# math-rendring-example-with-neovim-new-image-api
The same plugin but with extensive documantation

```lua
-- latex-math-preview.lua
--
-- This file is one lazy.nvim plugin spec. It is a "local" plugin: lazy.nvim
-- does not download anything from GitHub. Instead, this file returns a normal
-- plugin table at the bottom, and lazy.nvim calls M.setup() when a tex/typst
-- buffer opens.
--
-- What the plugin does:
--   * LaTeX: preview math, tikzcd, table, and figure snippets under the cursor.
--   * Typst: preview only $...$ math and #image(...) calls under the cursor.
--   * Draw the PNG with Neovim's experimental vim.ui.img API.
--   * Reserve virtual lines below the source so the image has a place to sit.
--
-- Important mental model:
--   vim.ui.img does not insert text into the buffer. It paints a terminal image
--   over the UI, more like a tiny floating picture. That is why this plugin must
--   keep recalculating the screen row/column during redraws and scrolling.

local M = {}

-- A namespace is Neovim's way to group decorations. Everything this plugin
-- draws in the buffer, such as virtual lines, is tagged with this namespace so
-- it can be cleared without touching decorations from other plugins.
local ns = vim.api.nvim_create_namespace("math_preview")

local file_patterns = { "*.tex", "*.md", "*.typ" }

local config = {
  -- Used by both pdftocairo and typst when producing PNG files.
  dpi = 220,

  -- Neovim does not expose terminal cell size in a portable way. This value
  -- converts image pixel height into terminal rows. You can override it from
  -- init.lua with:
  --   vim.g.latex_math_preview_cell_height_px = 22
  cell_height_px = tonumber(vim.g.latex_math_preview_cell_height_px) or 24,

  -- Delay before starting an expensive LaTeX compile. Typst uses a much shorter
  -- fixed delay because its compiler is fast and there is no cache.
  compile_delay_ms = tonumber(vim.g.latex_math_preview_compile_delay_ms) or 80,

  -- A very low zindex keeps terminal images below completion popups/statusline
  -- in terminals that support image layering.
  zindex = tonumber(vim.g.latex_math_preview_zindex) or -1073741825,
}

-- Runtime state for the one visible hover preview.
local state = {
  img_id = nil, -- id returned by vim.ui.img.set()
  range = nil, -- { start_line, start_col, end_line, end_col }
  hash = nil, -- stable id of the currently displayed source/render body
  last_pos = { row = nil, col = nil },
  pending_jobs = {}, -- LaTeX jobs keyed by hash
  generation = 0, -- cancellation token for delayed callbacks
  render_all = { running = false, cancelled = false },
}

local tex_template = [[
\documentclass[preview,border=1pt,varwidth]{standalone}
\usepackage{amsmath,mathtools,nicematrix,xcolor,libertinus-otf,graphicx}
%s
\begin{document}
{ \Large \selectfont
  \color[HTML]{FFFFFF}
%s
}
\end{document}
]]

local typst_template = [[
#set page(width: auto, height: auto, margin: 1pt, fill: none)
#set text(fill: white, size: 18pt)
%s
]]

-- ---------------------------------------------------------------------------
-- Tiny Generic Helpers
-- ---------------------------------------------------------------------------

-- trim(s)
--   Removes whitespace from the beginning and end of a string.
--
-- Lua note:
--   Strings are immutable, so gsub() does not modify s in place. It returns a
--   new string, and this function returns that new string.
local function trim(s)
  return (s:gsub("^%s+", ""):gsub("%s+$", ""))
end

-- notify_error(opts, msg)
--   Shows an error with vim.notify(), unless opts.silent is true.
--
-- Why opts.silent exists:
--   While typing, LaTeX/Typst snippets are often temporarily invalid. Showing a
--   notification for every half-written expression would be noisy, so insert
--   mode render calls pass { silent = true }.
local function notify_error(opts, msg)
  if not (opts and opts.silent) then
    vim.notify(msg, vim.log.levels.ERROR)
  end
end

-- cache_dir()
--   Returns the directory where LaTeX preview files live.
--
-- Neovim note:
--   stdpath("cache") is Neovim's standard per-user cache directory. It is the
--   right place for generated files that can be deleted safely.
local function cache_dir()
  return vim.fn.stdpath("cache") .. "/math-preview"
end

-- document_dir(buf)
--   Returns the folder containing the current document buffer.
--
-- Parameter:
--   buf is a Neovim buffer number. A buffer is Neovim's in-memory object for an
--   open file. The current buffer is vim.api.nvim_get_current_buf().
--
-- Why this matters:
--   LaTeX and Typst previews are compiled from temporary files in the cache.
--   Relative image paths should still resolve relative to the user's document,
--   not relative to the temporary cache file.
local function document_dir(buf)
  local name = vim.api.nvim_buf_get_name(buf)
  if name == "" then
    return ""
  end
  return vim.fn.fnamemodify(name, ":p:h")
end

-- A range is stored as:
--   { start_line, start_col, end_line, end_col }
-- Lines are 1-indexed because screenpos() and nvim_win_get_cursor() use
-- 1-indexed lines. Columns are byte columns, mostly from Treesitter.
--
-- cursor_in_range(cursor, range)
--   Returns true when the cursor is inside a renderable object's source range.
--
-- Parameter:
--   cursor is { line, col } from nvim_win_get_cursor().
--
-- Why single-line is special:
--   For $x$ on one line, line-only checking would trigger the preview even when
--   the cursor is somewhere else on the same line. Column checks avoid that.
local function cursor_in_range(cursor, range)
  if not range then
    return false
  end

  if range[1] == range[3] then
    return cursor[1] == range[1] and cursor[2] >= range[2] and cursor[2] <= range[4]
  end

  return cursor[1] >= range[1] and cursor[1] <= range[3]
end

-- same_start(a, b)
--   Compares only the start line/start column of two ranges.
--
-- Why not compare the whole range:
--   While editing, the end of the range can move as text changes. The beginning
--   is a stable enough identity for "this is the same preview location".
local function same_start(a, b)
  return a and b and a[1] == b[1] and a[2] == b[2]
end

-- line_count(text)
--   Counts how many text lines are in a string.
--
-- Lua note:
--   string.gsub() returns two values: the changed string and the number of
--   replacements. We ignore the changed string and use the count of "\n".
local function line_count(text)
  local _, n = text:gsub("\n", "\n")
  return n + 1
end

-- read_file_bytes(path)
--   Reads a file as raw bytes and returns them as a Lua string/blob.
--
-- Why raw bytes:
--   vim.ui.img.set() wants image bytes, not a file path. PNG bytes may contain
--   zero bytes and non-text data, so this is binary file reading.
--
-- Compatibility note:
--   readblob() is the Neovim-native path. The io.open fallback keeps the code
--   understandable and useful if readblob behaves differently.
local function read_file_bytes(path)
  local ok, bytes = pcall(vim.fn.readblob, path)
  if ok and bytes then
    return bytes
  end

  local file = io.open(path, "rb")
  if not file then
    return nil
  end

  bytes = file:read("*a")
  file:close()
  return bytes
end

-- PNG files store width/height in the IHDR chunk at fixed byte positions. This
-- avoids shelling out to identify/magick just to know how tall the image is.
--
-- png_size(bytes)
--   Returns width, height for a PNG image byte string.
--
-- Why this is safe:
--   Every valid PNG starts with the same 8-byte signature and stores width and
--   height in the IHDR chunk. The math below reads those big-endian integers.
local function png_size(bytes)
  if type(bytes) ~= "string" or #bytes < 24 or bytes:sub(1, 8) ~= "\137PNG\r\n\26\n" then
    return 0, 0
  end

  local w = bytes:byte(17) * 16777216 + bytes:byte(18) * 65536 + bytes:byte(19) * 256 + bytes:byte(20)
  local h = bytes:byte(21) * 16777216 + bytes:byte(22) * 65536 + bytes:byte(23) * 256 + bytes:byte(24)
  return w, h
end

-- image_height_cells(png_h, minimum_lines)
--   Converts a PNG pixel height into terminal rows.
--
-- Important idea:
--   vim.ui.img uses terminal-cell units for height. The PNG itself is measured
--   in pixels. config.cell_height_px is our calibration bridge between those
--   two coordinate systems.
--
-- minimum_lines:
--   Multi-line math must not be squeezed smaller than its visual line count, so
--   this function also applies a floor based on source/rendered line count.
local function image_height_cells(png_h, minimum_lines)
  local by_pixels = math.max(1, math.ceil(png_h / config.cell_height_px))
  return math.max(by_pixels, (minimum_lines or 1) * 2)
end

-- ---------------------------------------------------------------------------
-- Terminal Image Placement
-- ---------------------------------------------------------------------------

-- clear_preview()
--   Removes the currently visible terminal image and this plugin's virtual
--   lines from the current buffer.
--
-- Neovim note:
--   pcall() means "protected call". If vim.ui.img.del fails because the image
--   was already gone, the error is caught instead of breaking editing.
--
-- generation:
--   Incrementing state.generation invalidates delayed callbacks that were
--   scheduled before the clear.
local function clear_preview()
  if state.img_id and vim.ui and vim.ui.img then
    pcall(vim.ui.img.del, state.img_id)
  end

  state.img_id = nil
  state.range = nil
  state.hash = nil
  state.last_pos.row = nil
  state.last_pos.col = nil
  state.generation = state.generation + 1

  pcall(vim.api.nvim_buf_clear_namespace, 0, ns, 0, -1)
end

-- Virtual lines are attached after the source range. The image itself is drawn
-- by the terminal, so these fake lines are what make room in the buffer layout.
--
-- set_virtual_space(winid, range, height_cells)
--   Creates the bordered empty space where the image will appear.
--
-- Neovim note:
--   An extmark is an invisible marker attached to a buffer position. Here the
--   extmark owns virt_lines, which are screen-only lines shown below the source
--   without changing the file on disk.
local function set_virtual_space(winid, range, height_cells)
  local virt_lines = {}
  local border_line = string.rep("─", vim.api.nvim_win_get_width(winid) - 4)

  table.insert(virt_lines, { { border_line, "Comment" } })
  for _ = 1, height_cells do
    table.insert(virt_lines, { { "", "Normal" } })
  end
  table.insert(virt_lines, { { border_line, "Comment" } })

  vim.api.nvim_buf_clear_namespace(0, ns, 0, -1)
  vim.api.nvim_buf_set_extmark(0, ns, range[3] - 1, 0, {
    virt_lines = virt_lines,
  })
end

-- preview_anchor_screenpos(winid, range)
--   Finds the screen position where the preview should anchor.
--
-- Why screenpos():
--   Buffer coordinates are file positions. Terminal images need screen row/col.
--   vim.fn.screenpos() converts a buffer position to the visible screen grid.
--
-- Why use the logical line end:
--   Virtual lines appear after the full logical line, including wrapped screen
--   rows. Anchoring to the line end keeps the image aligned with that space.
local function preview_anchor_screenpos(winid, range)
  -- If a source line wraps on screen, virtual lines appear after the final
  -- wrapped screen row. Anchoring to the logical line end matches that behavior.
  local line = vim.api.nvim_buf_get_lines(0, range[3] - 1, range[3], false)[1] or ""
  local pos = vim.fn.screenpos(winid, range[3], math.max(#line, 1))
  if pos.row ~= 0 then
    return pos
  end

  return vim.fn.screenpos(winid, range[3], range[2] + 1)
end

-- update_img_pos(winid)
--   Moves the already-created terminal image to its current screen position.
--
-- Why this is separate from rendering:
--   Scrolling does not require recompiling math. It only changes where the
--   terminal image should be painted. The decoration provider calls this often.
local function update_img_pos(winid)
  if not state.img_id or not state.range then
    return
  end

  winid = winid or vim.api.nvim_get_current_win()
  local pos = preview_anchor_screenpos(winid, state.range)
  local target_row = pos.row == 0 and 9999 or pos.row + 2
  local target_col = pos.row == 0 and 1 or 2

  if state.last_pos.row == target_row and state.last_pos.col == target_col then
    return
  end

  state.last_pos.row = target_row
  state.last_pos.col = target_col
  pcall(vim.ui.img.set, state.img_id, { row = target_row, col = target_col })
end

-- preview_is_current(hash, range)
--   Checks whether an async render result still belongs to the active preview.
--
-- Why this exists:
--   Compilers finish later. By then you may have moved the cursor or edited the
--   source. This guard prevents old jobs from replacing the current preview.
local function preview_is_current(hash, range)
  return state.range
    and state.hash == hash
    and same_start(state.range, range)
    and cursor_in_range(vim.api.nvim_win_get_cursor(0), state.range)
end

-- Prepare state for a render attempt. Return false when the requested preview
-- is already shown and only needed a position refresh.
--
-- begin_preview(obj, hash, opts)
--   Sets state.range/state.hash for a new preview request.
--
-- opts.keep_old:
--   When true, the previous good image stays visible while a replacement is
--   compiling. This is what makes insert-mode live preview feel stable.
local function begin_preview(obj, hash, opts)
  local same_range = same_start(state.range, obj.range) and cursor_in_range(vim.api.nvim_win_get_cursor(0), obj.range)

  if same_range and state.hash == hash then
    update_img_pos()
    return false
  end

  if not same_range or not (opts and opts.keep_old) then
    clear_preview()
  else
    state.generation = state.generation + 1
  end

  state.range = obj.range
  state.hash = hash
  return true
end

-- show_png_bytes(bytes, hash, range, minimum_lines, opts)
--   Places raw PNG bytes with vim.ui.img and reserves virtual space for them.
--
-- Key detail:
--   The function creates the new image first and only then deletes the old one.
--   That avoids flicker while editing.
local function show_png_bytes(bytes, hash, range, minimum_lines, opts)
  if not preview_is_current(hash, range) or not bytes or not (vim.ui and vim.ui.img) then
    return
  end

  local winid = vim.api.nvim_get_current_win()
  local pos = preview_anchor_screenpos(winid, range)
  if pos.row == 0 then
    return
  end

  local _, png_h = png_size(bytes)
  local height_cells = image_height_cells(png_h > 0 and png_h or 92, minimum_lines)
  local old_img_id = state.img_id

  -- Create the new image before deleting the old one. This keeps live editing
  -- from flashing blank between successful renders.
  local ok, img_id = pcall(vim.ui.img.set, bytes, {
    row = pos.row + 2,
    col = 2,
    height = height_cells,
    zindex = config.zindex,
  })

  if not ok then
    return notify_error(opts, "vim.ui.img.set failed: " .. tostring(img_id))
  end

  state.img_id = img_id
  if old_img_id and old_img_id ~= img_id then
    pcall(vim.ui.img.del, old_img_id)
  end

  pcall(set_virtual_space, winid, range, height_cells)
  vim.schedule(update_img_pos)
end

-- show_png_file(path, hash, range, minimum_lines, opts, cleanup)
--   Reads a PNG file, optionally runs cleanup, then delegates to show_png_bytes.
--
-- cleanup:
--   Typst previews use temporary files and pass a cleanup function to delete
--   them after reading. Cached LaTeX previews pass no cleanup function.
local function show_png_file(path, hash, range, minimum_lines, opts, cleanup)
  if not preview_is_current(hash, range) then
    if cleanup then
      cleanup()
    end
    return
  end

  local bytes = read_file_bytes(path)
  if cleanup then
    cleanup()
  end

  show_png_bytes(bytes, hash, range, minimum_lines, opts)
end

-- defer_current(delay_ms, hash, range, callback, cleanup)
--   Runs callback later, but only if the same preview is still current.
--
-- Why delayed callbacks need protection:
--   Timers and external compiler jobs are asynchronous. Without this check,
--   old work could affect the UI after the cursor moved elsewhere.
local function defer_current(delay_ms, hash, range, callback, cleanup)
  state.generation = state.generation + 1
  local generation = state.generation

  -- Timer callback:
  --   Runs after delay_ms. It rechecks generation/hash/range before calling the
  --   real callback, because the cursor may have moved while the timer waited.
  vim.defer_fn(function()
    if generation ~= state.generation or not preview_is_current(hash, range) then
      if cleanup then
        cleanup()
      end
      return
    end

    callback()
  end, delay_ms)
end

-- ---------------------------------------------------------------------------
-- LaTeX Object Detection
-- ---------------------------------------------------------------------------

-- latex_environment_name(text)
--   Extracts the environment name from text that starts with \begin{...}.
--
-- Example:
--   "\\begin{figure}" -> "figure"
--   "\\begin{align*}" -> "align*"
--
-- Lua pattern note:
--   [%a%-%*]+ means "one or more letters, hyphens, or stars".
local function latex_environment_name(text)
  return text:match("^%s*\\begin%s*{%s*([%a%-%*]+)%s*}")
end

-- base_environment_name(name)
--   Removes a trailing star from a LaTeX environment name.
--
-- Example:
--   "align*" -> "align"
--
-- Why:
--   Starred and unstarred environments usually need the same preview handling.
local function base_environment_name(name)
  return name and name:gsub("%*$", "") or nil
end

-- is_latex_math_node(node_type)
--   Returns true for Treesitter node types that represent LaTeX math.
--
-- Treesitter note:
--   A parser labels pieces of source code with node types. For LaTeX, these are
--   names such as inline_formula or displayed_equation.
local function is_latex_math_node(node_type)
  return node_type == "inline_formula" or node_type == "displayed_equation" or node_type == "math_environment"
end

-- is_latex_preview_environment(name)
--   Returns true for non-math LaTeX environments this plugin can preview.
--
-- Current policy:
--   table and figure are treated as whole containers. tikzcd is previewed
--   directly. Bare tikzpicture is intentionally not a target now.
local function is_latex_preview_environment(name)
  name = base_environment_name(name)
  return name == "tikzcd" or name == "tikz-cd" or name == "figure" or name == "table"
end

-- node_range(node)
--   Converts a Treesitter node range into this plugin's range format.
--
-- Treesitter note:
--   node:range() returns zero-indexed lines. Neovim cursor/screen functions use
--   one-indexed lines, so we add 1 to the row numbers.
local function node_range(node)
  local start_row, start_col, end_row, end_col = node:range()
  return { start_row + 1, start_col, end_row + 1, end_col }
end

-- object_from_latex_node(buf, node)
--   Tries to convert one Treesitter node into a renderable preview object.
--
-- Return shape:
--   {
--     kind = "latex",
--     text = "... source text ...",
--     node_type = "... treesitter node type ...",
--     env = "figure" or nil,
--     range = { start_line, start_col, end_line, end_col },
--   }
--
-- Why return nil:
--   Most syntax nodes are not math/figure/table/tikzcd. Returning nil lets the
--   caller keep walking up or down the tree.
local function object_from_latex_node(buf, node)
  local node_type = node:type()
  local text = vim.treesitter.get_node_text(node, buf)

  if is_latex_math_node(node_type) then
    return { kind = "latex", text = text, node_type = node_type, range = node_range(node) }
  end

  if node_type:find("environment", 1, true) then
    local env = base_environment_name(latex_environment_name(text))
    if is_latex_preview_environment(env) then
      return { kind = "latex", text = text, node_type = node_type, env = env, range = node_range(node) }
    end
  end

  return nil
end

-- find_latex_container(buf, node)
--   Walks from a node up through its parents looking for table or figure.
--
-- Why containers win:
--   If the cursor is inside a tabular, includegraphics, caption, or math inside
--   a figure/table, the user wants the whole table/figure preview, not the small
--   inner piece.
local function find_latex_container(buf, node)
  local current = node
  while current do
    if current:type():find("environment", 1, true) then
      local text = vim.treesitter.get_node_text(current, buf)
      local env = base_environment_name(latex_environment_name(text))
      if env == "table" or env == "figure" then
        return current
      end
    end
    current = current:parent()
  end
  return nil
end

-- get_latex_at_cursor()
--   Finds the LaTeX thing under the cursor that should be previewed.
--
-- Algorithm:
--   1. Ask Treesitter for the smallest node at the cursor.
--   2. If it is inside a table/figure, return that outer container.
--   3. Otherwise walk upward until math or tikzcd is found.
--
-- pcall note:
--   Treesitter may not be ready or may not support the current buffer. pcall
--   catches that error and turns it into nil instead of crashing the plugin.
local function get_latex_at_cursor()
  local buf = vim.api.nvim_get_current_buf()
  local ok, node = pcall(vim.treesitter.get_node, { bufnr = buf })
  if not ok or not node then
    return nil
  end

  -- Tables and figures are treated as simple containers. If the cursor is
  -- anywhere inside one, render the whole outer environment and do not inspect
  -- inner tabular/tikzpicture/includegraphics nodes.
  local container = find_latex_container(buf, node)
  if container then
    return object_from_latex_node(buf, container)
  end

  while node do
    local obj = object_from_latex_node(buf, node)
    if obj then
      return obj
    end
    node = node:parent()
  end

  return nil
end

-- ---------------------------------------------------------------------------
-- Typst Object Detection
-- ---------------------------------------------------------------------------

-- line_offsets(lines)
--   Builds a lookup table from line number to absolute string position.
--
-- Why Typst needs this:
--   The Typst path uses plain string scanning instead of Treesitter. String
--   scanning works with one big string, but Neovim screen placement needs line
--   and column numbers. Offsets let us convert between those coordinate systems.
local function line_offsets(lines)
  local offsets = {}
  local pos = 1
  for i, line in ipairs(lines) do
    offsets[i] = pos
    pos = pos + #line + 1
  end
  return offsets
end

-- abs_to_line_col(offsets, lines, abs_pos)
--   Converts an absolute string position back to { line, column }.
--
-- Lua note:
--   This walks backward because most cursor positions are near the current line,
--   and the first offset <= abs_pos identifies the line.
local function abs_to_line_col(offsets, lines, abs_pos)
  for i = #offsets, 1, -1 do
    if abs_pos >= offsets[i] then
      return i, math.min(abs_pos - offsets[i], #lines[i])
    end
  end
  return 1, 0
end

-- cursor_abs_pos(lines, cursor)
--   Converts Neovim's cursor { line, column } to an absolute string position.
--
-- Neovim note:
--   nvim_win_get_cursor() returns a one-indexed line and a zero-indexed byte
--   column. This function preserves that convention while building the absolute
--   position used by Typst string scanning.
local function cursor_abs_pos(lines, cursor)
  local offsets = line_offsets(lines)
  local line = lines[cursor[1]] or ""
  local base = offsets[cursor[1]] or 1
  return base + math.min(cursor[2], #line), offsets
end

-- is_escaped_at(s, pos)
--   Returns true when the character at pos is escaped by an odd number of
--   backslashes before it.
--
-- Example:
--   \$ should not start/end Typst math. The dollar is escaped, so this function
--   returns true for the dollar position.
local function is_escaped_at(s, pos)
  local count = 0
  local i = pos - 1
  while i >= 1 and s:sub(i, i) == "\\" do
    count = count + 1
    i = i - 1
  end
  return count % 2 == 1
end

-- find_matching_paren(s, open_pos)
--   Finds the closing parenthesis matching s[open_pos].
--
-- Why this is more than string.find(")"):
--   #image(...) arguments can contain nested parentheses or quoted strings. A
--   small parser avoids stopping at a ")" inside a string or nested call.
local function find_matching_paren(s, open_pos)
  local depth = 0
  local quote = nil
  local escaped = false

  for i = open_pos, #s do
    local ch = s:sub(i, i)

    if quote then
      if escaped then
        escaped = false
      elseif ch == "\\" then
        escaped = true
      elseif ch == quote then
        quote = nil
      end
    elseif ch == '"' or ch == "'" then
      quote = ch
    elseif ch == "(" then
      depth = depth + 1
    elseif ch == ")" then
      depth = depth - 1
      if depth == 0 then
        return i
      end
    end
  end

  return nil
end

-- typst_object_range(offsets, lines, start_pos, end_pos)
--   Converts absolute Typst scan positions into the plugin's range format.
--
-- end_pos is inclusive:
--   The scanner passes the position of the last character in the object, so the
--   function uses end_pos + 1 to produce a range end column.
local function typst_object_range(offsets, lines, start_pos, end_pos)
  local start_line, start_col = abs_to_line_col(offsets, lines, start_pos)
  local end_line, end_col = abs_to_line_col(offsets, lines, end_pos + 1)
  return { start_line, start_col, end_line, end_col }
end

-- find_typst_image(full_text, offsets, lines, cursor_abs)
--   Finds a #image(...) call containing the cursor.
--
-- Why search all image calls:
--   Typst image syntax is regular enough that a direct text scan is simpler
--   than depending on a Treesitter Typst parser.
local function find_typst_image(full_text, offsets, lines, cursor_abs)
  local search = 1
  while true do
    local start_pos, call_end = full_text:find("#image%s*%(", search)
    if not start_pos then
      return nil
    end

    local open_pos = full_text:find("%(", call_end)
    local end_pos = open_pos and find_matching_paren(full_text, open_pos)
    if end_pos and cursor_abs >= start_pos and cursor_abs <= end_pos then
      return {
        kind = "typst",
        node_type = "typst_image",
        text = full_text:sub(start_pos, end_pos),
        range = typst_object_range(offsets, lines, start_pos, end_pos),
      }
    end

    search = (end_pos or call_end) + 1
  end
end

-- find_typst_math(full_text, offsets, lines, cursor_abs)
--   Finds a $...$ math span containing the cursor.
--
-- Simple strategy:
--   Look backward for the nearest unescaped "$", then forward for the matching
--   unescaped "$". If the cursor is between them, that is the math preview.
local function find_typst_math(full_text, offsets, lines, cursor_abs)
  local start_pos = nil
  for i = cursor_abs, 1, -1 do
    if full_text:sub(i, i) == "$" and not is_escaped_at(full_text, i) then
      start_pos = i
      break
    end
  end
  if not start_pos then
    return nil
  end

  local end_pos = nil
  for i = start_pos + 1, #full_text do
    if full_text:sub(i, i) == "$" and not is_escaped_at(full_text, i) then
      end_pos = i
      break
    end
  end
  if not end_pos or cursor_abs > end_pos then
    return nil
  end

  return {
    kind = "typst",
    node_type = "typst_math",
    text = full_text:sub(start_pos, end_pos),
    range = typst_object_range(offsets, lines, start_pos, end_pos),
  }
end

-- get_typst_at_cursor()
--   Finds Typst math or image syntax under the cursor.
--
-- Why it checks nearby positions:
--   Insert mode often reports the cursor just after the character you are
--   visually editing. Trying cursor_abs - 1 and cursor_abs + 1 makes live
--   preview less brittle while typing.
local function get_typst_at_cursor()
  local buf = vim.api.nvim_get_current_buf()
  local lines = vim.api.nvim_buf_get_lines(buf, 0, -1, false)
  local full_text = table.concat(lines, "\n")
  local cursor_abs, offsets = cursor_abs_pos(lines, vim.api.nvim_win_get_cursor(0))

  -- Insert mode can put the cursor byte just after the visual cursor position.
  -- Trying nearby bytes keeps live preview working at the edges of $...$.
  local candidates = {
    cursor_abs,
    math.max(1, cursor_abs - 1),
    math.min(#full_text, cursor_abs + 1),
  }

  local tried = {}
  for _, pos in ipairs(candidates) do
    if not tried[pos] then
      tried[pos] = true
      local obj = find_typst_image(full_text, offsets, lines, pos) or find_typst_math(full_text, offsets, lines, pos)
      if obj then
        return obj
      end
    end
  end

  return nil
end

-- ---------------------------------------------------------------------------
-- LaTeX Document Building
-- ---------------------------------------------------------------------------

-- strip_latex_delimiters(s)
--   Removes outer math delimiters such as $...$, $$...$$, \( ... \), or
--   \[ ... \].
--
-- Why:
--   The preview template adds its own math wrapper for normal formulas. Keeping
--   the original delimiters would create nested math mode and can break LaTeX.
local function strip_latex_delimiters(s)
  s = trim(s)

  for _, pattern in ipairs({
    "^%$%$(.*)%$%$$", -- $$ ... $$
    "^\\%[(.*)\\%]$", -- \[ ... \]
    "^%$(.*)%$$", -- $ ... $
    "^\\%((.*)\\%)$", -- \( ... \)
  }) do
    local inner = s:match(pattern)
    if inner then
      return trim(inner)
    end
  end

  return s
end

-- normalize_latex_math(s)
--   Converts common display math environments into tighter inline fragments.
--
-- Example:
--   \begin{align*} a &= b \\ c &= d \end{align*}
--   becomes:
--   \begin{aligned} a &= b \\ c &= d \end{aligned}
--
-- Why:
--   Full display environments add display skips and extra page spacing. The
--   aligned/gathered forms crop better inside the standalone preview document.
local function normalize_latex_math(s)
  s = strip_latex_delimiters(s)

  local env, body = s:match("^\\begin%s*{%s*([%a*]+)%s*}(.*)\\end%s*{%s*%1%s*}%s*$")
  if not env then
    return s
  end

  env = env:gsub("%*$", "")
  body = trim(body)

  if env == "equation" then
    return body
  elseif env == "align" then
    return "\\begin{aligned}" .. body .. "\\end{aligned}"
  elseif env == "gather" or env == "multline" then
    return "\\begin{gathered}" .. body .. "\\end{gathered}"
  end

  return s
end

-- latex_line_count(s)
--   Estimates how many visual rows a LaTeX snippet contains.
--
-- Why it counts "\\\\":
--   In LaTeX, \\ starts a new row in aligned arrays and similar constructs. The
--   result becomes a minimum height so multi-line math is not squeezed.
local function latex_line_count(s)
  local count = 1
  for _ in s:gmatch("\\\\") do
    count = count + 1
  end
  return count
end

-- document_tikz_libraries(buf)
--   Scans the user's document for \usetikzlibrary{...} and reuses those library
--   names in the preview document.
--
-- Why:
--   Hardcoding TikZ libraries is brittle. If the real document already says
--   which libraries it needs, the preview should copy that intent.
local function document_tikz_libraries(buf)
  local libs, seen = {}, {}
  for _, line in ipairs(vim.api.nvim_buf_get_lines(buf, 0, -1, false)) do
    for raw in line:gmatch("\\usetikzlibrary%s*{([^}]+)}") do
      for lib in raw:gmatch("[^,%s]+") do
        if not seen[lib] then
          seen[lib] = true
          table.insert(libs, lib)
        end
      end
    end
  end

  return #libs > 0 and "\\usetikzlibrary{" .. table.concat(libs, ",") .. "}" or ""
end

-- document_graphics_path(buf)
--   Produces a LaTeX \graphicspath pointing to the current document directory.
--
-- Why:
--   The temporary preview .tex file lives in the cache directory. Without
--   \graphicspath, \includegraphics{foo.png} would look in the cache rather
--   than next to your real .tex file.
local function document_graphics_path(buf)
  local dir = document_dir(buf)
  if dir == "" then
    return ""
  end

  dir = dir:gsub("\\", "/")
  if not dir:match("/$") then
    dir = dir .. "/"
  end
  return "\\graphicspath{{" .. dir .. "}}"
end

-- unwrap_latex_environment(s, env)
--   Removes an outer \begin{env}...\end{env} wrapper and returns the body.
--
-- Why used for table/figure:
--   table and figure are floats. Floats are not reliable inside tiny standalone
--   preview documents, so the preview renders their contents without the float
--   wrapper.
local function unwrap_latex_environment(s, env)
  for _, suffix in ipairs({ "", "%*" }) do
    local begin_pat = "\\begin%s*{%s*" .. env .. suffix .. "%s*}"
    local end_pat = "\\end%s*{%s*" .. env .. suffix .. "%s*}"
    local body = s:match("^%s*" .. begin_pat .. "%s*%b[]%s*(.-)%s*" .. end_pat .. "%s*$")
      or s:match("^%s*" .. begin_pat .. "%s*(.-)%s*" .. end_pat .. "%s*$")
    if body then
      return trim(body)
    end
  end
  return s
end

-- caption_to_plain_text(s)
--   Converts \caption{...} into italic normal text.
--
-- Why not \captionof:
--   The preview only needs to show the caption content visually. Plain italic
--   text avoids adding LaTeX caption semantics to the standalone document.
local function caption_to_plain_text(s)
  s = s:gsub("\\caption%s*%b[]%s*(%b{})", "\\par\\smallskip\\noindent\\textit%1")
  return s:gsub("\\caption%s*(%b{})", "\\par\\smallskip\\noindent\\textit%1")
end

-- latex_render_setup(s, buf)
--   Decides which extra LaTeX preamble lines are needed and what body should be
--   inserted into the standalone template.
--
-- Return values:
--   1. extra_preamble: package/library lines.
--   2. body: source to render inside \begin{document}.
--
-- Normal math is wrapped in \( \displaystyle ... \). tikzcd, figure, and table
-- are already full LaTeX structures, so they are inserted as prepared bodies.
local function latex_render_setup(s, buf)
  local env = base_environment_name(latex_environment_name(s))

  if env == "tikzcd" or env == "tikz-cd" then
    return table.concat({
      "\\usepackage{tikz-cd}",
      document_tikz_libraries(buf),
    }, "\n"), s
  end

  if env == "figure" then
    return table.concat({
      "\\usepackage{tikz}",
      "\\usepackage{mwe}",
      document_graphics_path(buf),
      document_tikz_libraries(buf),
    }, "\n"), caption_to_plain_text(unwrap_latex_environment(s, "figure"))
  end

  if env == "table" then
    return table.concat({
      "\\usepackage{array}",
      "\\usepackage{booktabs}",
      "\\usepackage{tabularx}",
      "\\usepackage{longtable}",
      "\\usepackage{multirow}",
      "\\usepackage{makecell}",
    }, "\n"), caption_to_plain_text(unwrap_latex_environment(s, "table"))
  end

  return "", "\\(\\displaystyle " .. s .. "\\)"
end

-- make_latex_task(buf, obj)
--   Builds a task table describing every file and string needed to render one
--   LaTeX object.
--
-- Task table idea:
--   Instead of passing many loose variables around, one table carries the hash,
--   paths, preamble, body, range, and height information together.
--
-- Cache key:
--   The hash is based on the exact preamble/body. If the same render input
--   appears again, it maps to the same PNG path.
local function make_latex_task(buf, obj)
  -- A task is a complete recipe for producing one LaTeX PNG. Separating this
  -- from rendering keeps the rest of the code simple: hover preview and
  -- :LatexMathPreviewRenderAll can both use the same task structure.
  local normalized = normalize_latex_math(obj.text)
  local preamble, body = latex_render_setup(normalized, buf)
  local hash = vim.fn.sha256(preamble .. "\n" .. body)
  local dir = cache_dir()

  return {
    obj = obj,
    hash = hash,
    range = obj.range,
    line_count = latex_line_count(normalized),
    tex_file = dir .. "/" .. hash .. ".tex",
    pdf_file = dir .. "/" .. hash .. ".pdf",
    png_file = dir .. "/" .. hash .. ".png",
    png_prefix = dir .. "/" .. hash,
    tmpdir = dir,
    preamble = preamble,
    body = body,
  }
end

-- write_latex_task(task)
--   Writes the generated standalone .tex file for a LaTeX task.
--
-- vim.fn.writefile note:
--   writefile() expects a list of lines. vim.split(..., "\n") turns the single
--   template string into that list.
local function write_latex_task(task)
  vim.fn.mkdir(task.tmpdir, "p")
  vim.fn.writefile(vim.split(string.format(tex_template, task.preamble, task.body), "\n"), task.tex_file)
end

-- ---------------------------------------------------------------------------
-- Typst Document Building
-- ---------------------------------------------------------------------------

-- typst_escape_string_path(path)
--   Escapes backslashes and double quotes so a filesystem path can safely live
--   inside a Typst string literal.
--
-- Example:
--   /tmp/a"b.png becomes /tmp/a\"b.png inside the generated Typst document.
local function typst_escape_string_path(path)
  return path:gsub("\\", "\\\\"):gsub('"', '\\"')
end

-- typst_rewrite_image_paths(s, buf)
--   Rewrites relative #image("...") paths to absolute paths.
--
-- Why:
--   Typst previews compile from temporary files. A relative image path should
--   still mean "relative to the user's Typst document", not "relative to /tmp".
local function typst_rewrite_image_paths(s, buf)
  local dir = document_dir(buf)
  if dir == "" then
    return s
  end

  -- gsub callback:
  --   Lua calls this function for each matching #image("...") prefix/path pair.
  --   The return value replaces that one match in the final string.
  return s:gsub('(#image%s*%(%s*)"([^"]+)"', function(prefix, path)
    if path:match("^/") or path:match("^%a+://") then
      return prefix .. '"' .. path .. '"'
    end

    local absolute = vim.fn.fnamemodify(dir .. "/" .. path, ":p")
    return prefix .. '"' .. typst_escape_string_path(absolute) .. '"'
  end)
end

-- typst_temp_files()
--   Allocates temporary .typ and .png paths for one Typst preview.
--
-- Why not cache:
--   Typst is fast and can compile directly to PNG, so temporary files keep this
--   path simple. cleanup_typst_task() deletes them after use.
local function typst_temp_files()
  local base = vim.fn.tempname()
  return base .. ".typ", base .. ".png"
end

-- make_typst_task(buf, obj)
--   Builds the task table for a Typst preview.
--
-- Difference from LaTeX:
--   Typst has no persistent cache here, no PDF step, and no package preamble.
--   The source is wrapped in typst_template and compiled straight to PNG.
local function make_typst_task(buf, obj)
  -- Typst tasks are deliberately smaller than LaTeX tasks. There is no cache
  -- and no PDF conversion step because typst compile can write PNG directly.
  local body = obj.text
  if obj.node_type == "typst_image" then
    body = typst_rewrite_image_paths(body, buf)
  end

  local typ_file, png_file = typst_temp_files()
  return {
    obj = obj,
    hash = "typst:" .. vim.fn.sha256(body),
    range = obj.range,
    line_count = line_count(body),
    typ_file = typ_file,
    png_file = png_file,
    body = body,
  }
end

-- cleanup_typst_task(task)
--   Deletes the temporary files used by one Typst preview task.
--
-- Why both files:
--   The .typ file is generated input, and the .png file is generated output.
--   Once vim.ui.img has read the PNG bytes, neither file is needed.
local function cleanup_typst_task(task)
  vim.fn.delete(task.typ_file)
  vim.fn.delete(task.png_file)
end

-- ---------------------------------------------------------------------------
-- External Commands
-- ---------------------------------------------------------------------------

-- compile_tex(tex_file, tmpdir, callback)
--   Runs xelatex asynchronously to turn a .tex file into a PDF.
--
-- vim.system note:
--   vim.system({ ... }, {}, callback) starts a process without blocking Neovim.
--   callback receives a result table with fields like code, stdout, stderr.
local function compile_tex(tex_file, tmpdir, callback)
  vim.system({
    "xelatex",
    "-interaction=nonstopmode",
    "-halt-on-error",
    "-output-directory=" .. tmpdir,
    tex_file,
  }, {}, callback)
end

-- convert_pdf_to_png(pdf_file, png_prefix, callback)
--   Runs pdftocairo asynchronously to turn the generated PDF into a PNG.
--
-- Why -singlefile:
--   pdftocairo normally can create numbered output files. -singlefile makes the
--   output path predictable: png_prefix .. ".png".
local function convert_pdf_to_png(pdf_file, png_prefix, callback)
  vim.system({
    "pdftocairo",
    "-png",
    "-singlefile",
    "-r",
    tostring(config.dpi),
    "-transp",
    pdf_file,
    png_prefix,
  }, {}, callback)
end

-- compile_typst_to_png(typ_file, png_file, callback)
--   Runs typst compile asynchronously and writes PNG directly.
--
-- Why --root /:
--   Typst restricts file access to a project root. Since image paths are
--   rewritten to absolute paths, root "/" lets Typst read those files.
local function compile_typst_to_png(typ_file, png_file, callback)
  vim.system({
    "typst",
    "compile",
    "--format",
    "png",
    "--ppi",
    tostring(config.dpi),
    "--root",
    "/",
    typ_file,
    png_file,
  }, {}, callback)
end

-- compile_latex_task(task, callback)
--   Ensures a LaTeX task's PNG exists, compiling/converting only when needed.
--
-- Callback contract:
--   callback(true, "cached" or "rendered") when a PNG is available.
--   callback(false, message) when xelatex or pdftocairo fails.
--   callback(nil, "already running") when another job owns this hash.
--
-- pending_jobs:
--   Autocommands can fire repeatedly. This table prevents duplicate compiler
--   processes for the exact same math snippet.
local function compile_latex_task(task, callback)
  -- callback(success, message)
  --   success == true   -> PNG is ready.
  --   success == false  -> command failed; message is user-facing.
  --   success == nil    -> another job for the same hash is already running.
  if vim.fn.filereadable(task.png_file) == 1 then
    return callback(true, "cached")
  end

  if state.pending_jobs[task.hash] then
    return callback(nil, "already running")
  end

  state.pending_jobs[task.hash] = true
  write_latex_task(task)

  -- First async step: xelatex creates a PDF.
  -- xelatex callback:
  --   Receives tex_res after the process exits. code == 0 means success.
  compile_tex(task.tex_file, task.tmpdir, function(tex_res)
    if tex_res.code ~= 0 then
      state.pending_jobs[task.hash] = nil
      return callback(false, "xelatex failed: " .. (tex_res.stderr or ""))
    end

    -- Second async step: pdftocairo rasterizes that PDF into a PNG.
    -- pdftocairo callback:
    --   Clears pending_jobs and reports whether the PNG is ready.
    convert_pdf_to_png(task.pdf_file, task.png_prefix, function(png_res)
      state.pending_jobs[task.hash] = nil
      if png_res.code == 0 then
        callback(true, "rendered")
      else
        callback(false, "pdftocairo failed: " .. (png_res.stderr or ""))
      end
    end)
  end)
end

-- ---------------------------------------------------------------------------
-- Render Current Object
-- ---------------------------------------------------------------------------

-- render_latex(obj, opts)
--   Renders the LaTeX object under the cursor, using the cache when possible.
--
-- Flow:
--   1. Build a LaTeX task.
--   2. Prepare preview state with begin_preview().
--   3. If PNG is cached, show it immediately.
--   4. Otherwise debounce, compile, convert, then show.
local function render_latex(obj, opts)
  local task = make_latex_task(vim.api.nvim_get_current_buf(), obj)
  if not begin_preview(obj, task.hash, opts) then
    return
  end

  -- show()
  --   Small local helper so both the cached path and compiled path can display
  --   the same task without repeating arguments.
  local function show()
    show_png_file(task.png_file, task.hash, task.range, task.line_count, opts)
  end

  if vim.fn.filereadable(task.png_file) == 1 then
    return show()
  end

  -- LaTeX is slow enough that we debounce it. If you move the cursor away or
  -- keep typing before the timer fires, defer_current drops the old callback.
  -- defer_current callback:
  --   Starts the compiler only if this preview is still active after the delay.
  defer_current(config.compile_delay_ms, task.hash, task.range, function()
    -- compile_latex_task callback:
    --   Runs when cached/compiled/failed status is known.
    compile_latex_task(task, function(success, msg)
      if success then
        vim.schedule(show)
      elseif success == false then
        -- External process callbacks do not run directly in the main Neovim
        -- event context, so UI work is scheduled back onto the main loop.
        -- vim.schedule callback:
        --   Displays the error later on Neovim's main event loop.
        vim.schedule(function()
          notify_error(opts, msg)
        end)
      end
    end)
  end)
end

-- render_typst(obj, opts)
--   Renders a Typst math/image object under the cursor.
--
-- Flow:
--   1. Build a temporary Typst task.
--   2. Prepare preview state.
--   3. Write a tiny .typ file.
--   4. Ask typst to compile directly to PNG.
--   5. Read PNG bytes and delete the temporary files.
local function render_typst(obj, opts)
  local task = make_typst_task(vim.api.nvim_get_current_buf(), obj)
  if not begin_preview(obj, task.hash, opts) then
    cleanup_typst_task(task)
    return
  end

  -- cleanup()
  --   Local wrapper so defer_current(), success, and failure paths can all
  --   delete the same temporary files without duplicating code.
  local function cleanup()
    cleanup_typst_task(task)
  end

  -- Typst is fast and uncached, but still async. The tiny delay keeps multiple
  -- TextChangedI events from spawning work for every intermediate keystroke.
  -- defer_current callback:
  --   Writes the temporary Typst file and starts typst only if still current.
  defer_current(10, task.hash, task.range, function()
    vim.fn.writefile(vim.split(string.format(typst_template, task.body), "\n"), task.typ_file)
    -- typst callback:
    --   Runs when typst compile exits.
    compile_typst_to_png(task.typ_file, task.png_file, function(res)
      if res.code == 0 then
        -- show_png_file reads the PNG and then calls cleanup().
        -- vim.schedule callback:
        --   Reads/displays the PNG on Neovim's main event loop.
        vim.schedule(function()
          show_png_file(task.png_file, task.hash, task.range, task.line_count, opts, cleanup)
        end)
      else
        -- On failure, delete temp files and optionally notify the user.
        -- vim.schedule callback:
        --   Cleans up and reports the Typst error on the main event loop.
        vim.schedule(function()
          cleanup()
          notify_error(opts, "typst failed: " .. (res.stderr or ""))
        end)
      end
    end)
  end, cleanup)
end

-- render_current(opts)
--   Entry point used by all cursor/text autocommands.
--
-- It decides which language path to use from vim.bo.filetype:
--   typst -> Typst scanner/compiler.
--   everything else loaded by this plugin -> LaTeX Treesitter/compiler.
local function render_current(opts)
  opts = opts or {}

  -- Do not render while in visual/operator-pending/etc. Those modes can have
  -- unusual cursor/range semantics and the preview would feel jumpy.
  if vim.fn.mode() ~= "n" and vim.fn.mode() ~= "i" then
    return clear_preview()
  end

  local is_typst = vim.bo.filetype == "typst"
  local obj = is_typst and get_typst_at_cursor() or get_latex_at_cursor()
  if not obj then
    return clear_preview()
  end

  if is_typst then
    render_typst(obj, opts)
  else
    render_latex(obj, opts)
  end
end

-- ---------------------------------------------------------------------------
-- Render All LaTeX Objects Into The Cache
-- ---------------------------------------------------------------------------

-- collect_latex_objects(buf)
--   Walks the whole Treesitter tree and returns every LaTeX previewable object.
--
-- Used by:
--   :LatexMathPreviewRenderAll
--
-- Why this is LaTeX-only:
--   RenderAll is a cache warmer. Typst previews intentionally are not cached.
local function collect_latex_objects(buf)
  local ok, parser = pcall(vim.treesitter.get_parser, buf)
  if not ok or not parser then
    return {}
  end

  local tree = (parser:parse() or {})[1]
  if not tree then
    return {}
  end

  local objects, seen = {}, {}

  -- add(obj)
  --   Adds an object once. The key includes range and text so duplicate tree
  --   visits do not schedule duplicate render tasks.
  local function add(obj)
    local range = obj.range
    local key = table.concat({ range[1], range[2], range[3], range[4], obj.text }, ":")
    if not seen[key] then
      seen[key] = true
      table.insert(objects, obj)
    end
  end

  -- walk(node)
  --   Recursive tree traversal. It checks the current node, then its children.
  --
  -- Why return after add:
  --   If a whole environment is renderable, we do not need to collect smaller
  --   renderable nodes inside it.
  local function walk(node)
    local obj = object_from_latex_node(buf, node)
    if obj then
      add(obj)
      return
    end

    for child in node:iter_children() do
      walk(child)
    end
  end

  walk(tree:root())
  -- table.sort callback:
  --   Orders objects by where they appear in the file. Returning true means
  --   object a should come before object b.
  table.sort(objects, function(a, b)
    if a.range[1] == b.range[1] then
      return a.range[2] < b.range[2]
    end
    return a.range[1] < b.range[1]
  end)

  return objects
end

-- render_all()
--   Implements :LatexMathPreviewRenderAll.
--
-- What it does:
--   Finds all LaTeX math/table/figure/tikzcd objects in the current buffer and
--   compiles their PNGs into the cache.
--
-- Safety:
--   It runs at most 6 compiler jobs at a time, even if the user sets a larger
--   vim.g.latex_math_preview_render_all_jobs value.
local function render_all()
  local buf = vim.api.nvim_get_current_buf()
  local objects = collect_latex_objects(buf)
  if #objects == 0 then
    vim.notify("No LaTeX math/table/figure previews found", vim.log.levels.INFO)
    return
  end

  state.render_all.cancelled = true
  state.render_all = { running = true, cancelled = false }

  local tasks, seen_hash = {}, {}
  for _, obj in ipairs(objects) do
    local task = make_latex_task(buf, obj)
    if not seen_hash[task.hash] then
      seen_hash[task.hash] = true
      table.insert(tasks, task)
    end
  end

  local max_jobs = math.max(1, math.min(6, tonumber(vim.g.latex_math_preview_render_all_jobs) or 6))
  local next_index, active, done = 1, 0, 0
  local ok_count, fail_count = 0, 0
  local token = state.render_all

  vim.notify("Rendering " .. #tasks .. " LaTeX previews with " .. max_jobs .. " async jobs", vim.log.levels.INFO)

  -- finish_if_done()
  --   Reports completion once all queued tasks have finished and no active jobs
  --   remain.
  local function finish_if_done()
    if done < #tasks or active > 0 then
      return
    end

    token.running = false
    -- vim.schedule callback:
    --   Shows the final RenderAll summary on Neovim's main event loop.
    vim.schedule(function()
      vim.notify(
        "LaTeX preview render-all finished: " .. ok_count .. " ready, " .. fail_count .. " failed",
        fail_count > 0 and vim.log.levels.WARN or vim.log.levels.INFO
      )
    end)
  end

  -- start_more()
  --   Starts new compile jobs until the concurrency limit is full.
  --
  -- Queue variables:
  --   next_index points to the next task not yet started.
  --   active counts currently running jobs.
  --   done counts finished jobs.
  local function start_more()
    if token.cancelled then
      token.running = false
      return
    end

    while active < max_jobs and next_index <= #tasks do
      local task = tasks[next_index]
      next_index = next_index + 1
      active = active + 1

      -- compile_latex_task callback:
      --   Updates queue counters when one render-all task finishes.
      compile_latex_task(task, function(success)
        active = active - 1
        done = done + 1
        if success == false then
          fail_count = fail_count + 1
        else
          ok_count = ok_count + 1
        end

        -- vim.schedule callback:
        --   Starts more queued work from Neovim's main event loop.
        vim.schedule(function()
          start_more()
          finish_if_done()
        end)
      end)
    end

    finish_if_done()
  end

  start_more()
end

-- ---------------------------------------------------------------------------
-- Commands And Autocommands
-- ---------------------------------------------------------------------------

-- clear_cache()
--   Implements :LatexMathPreviewClearCache.
--
-- What it clears:
--   The visible preview, pending job bookkeeping, the RenderAll token, and all
--   files under stdpath("cache")/math-preview.
local function clear_cache()
  clear_preview()
  state.pending_jobs = {}
  state.render_all.cancelled = true

  local dir = cache_dir()
  if vim.fn.isdirectory(dir) ~= 1 then
    vim.notify("Math preview cache is already empty", vim.log.levels.INFO)
    return
  end

  for _, file in ipairs(vim.fn.globpath(dir, "*", false, true)) do
    vim.fn.delete(file, "rf")
  end

  vim.notify("Math preview cache cleared: " .. dir, vim.log.levels.INFO)
end

-- M.setup()
--   lazy.nvim calls this function when the plugin loads.
--
-- Responsibilities:
--   * Register the redraw hook that keeps terminal images positioned.
--   * Create user commands.
--   * Create autocommands that render/refresh previews while editing.
function M.setup()
  -- Decoration providers run during redraw. That makes on_win the right hook
  -- for keeping a terminal image glued to its source line while scrolling.
  vim.api.nvim_set_decoration_provider(ns, {
    -- on_win callback
    --   Runs during redraw for each window. It should stay cheap: only update
    --   image position, never compile.
    on_win = function(_, winid, bufnr)
      if state.img_id and state.range and bufnr == vim.api.nvim_get_current_buf() then
        update_img_pos(winid)
      end
    end,
  })

  local group = vim.api.nvim_create_augroup("MathPreview", { clear = true })

  vim.api.nvim_create_user_command("LatexMathPreviewClearCache", clear_cache, {
    desc = "Clear cached LaTeX math preview images",
  })

  vim.api.nvim_create_user_command("LatexMathPreviewRenderAll", render_all, {
    desc = "Asynchronously render all LaTeX math/table/figure previews into the cache",
  })

  vim.api.nvim_create_autocmd("CursorMoved", {
    group = group,
    pattern = file_patterns,
    -- Normal-mode movement can show errors because the user is not in the
    -- middle of typing an incomplete snippet.
    callback = function()
      render_current()
    end,
  })

  vim.api.nvim_create_autocmd("CursorMovedI", {
    group = group,
    pattern = file_patterns,
    -- Insert-mode movement keeps the old image and stays silent so typing does
    -- not produce noisy errors for half-written math.
    callback = function()
      render_current({ keep_old = true, silent = true })
    end,
  })

  vim.api.nvim_create_autocmd({ "TextChangedI", "TextChangedP" }, {
    group = group,
    pattern = file_patterns,
    -- TextChangedI/TextChangedP are what make live preview update as you edit.
    callback = function()
      render_current({ keep_old = true, silent = true })
    end,
  })

  vim.api.nvim_create_autocmd({ "WinScrolled", "WinResized" }, {
    group = group,
    pattern = file_patterns,
    -- Scrolling/resizing only changes screen coordinates. Repositioning is
    -- enough; recompiling would be wasteful.
    callback = function()
      update_img_pos()
    end,
  })

  vim.api.nvim_create_autocmd("InsertLeave", {
    group = group,
    pattern = file_patterns,
    -- Once insert mode ends, do a non-silent refresh so a real final error can
    -- be reported to the user.
    callback = function()
      render_current({ keep_old = true })
    end,
  })
end

return {
  dir = vim.fn.stdpath("config"),
  name = "latex-math-preview",
  ft = { "tex", "markdown", "typst" },
  dependencies = { "nvim-treesitter/nvim-treesitter" },
  config = M.setup,
}
```

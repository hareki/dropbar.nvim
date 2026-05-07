# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
make all              # format-check, lint, test, docs (full CI suite)
make format-check     # stylua . --check
make format           # stylua . (auto-format Lua files)
make lint             # luacheck .
make test             # run all tests with plenary.busted
make test-file-NAME   # run a specific test (e.g., make test-file-bar runs tests/bar_spec.lua)
make docs             # regenerate doc/dropbar.txt from README.md and source
make deps             # clone test dependencies into deps/
make clean            # remove deps/ and doc/
```

Tests require the deps to be cloned first (`make deps`). The test runner uses `plenary.busted` via a headless Neovim invocation.

## Architecture

dropbar.nvim is an IDE-like breadcrumb winbar plugin. The core data model is a three-layer hierarchy:

```
dropbar_t (winbar)
  └── dropbar_symbol_t[] (components from sources)
        └── dropbar_menu_t (dropdown on click)
              └── dropbar_menu_entry_t[] (rows)
                    └── dropbar_symbol_t[] (components)
```

**Global state** lives in `_G.dropbar`:
- `_G.dropbar.bars[buf][win]` — all active dropbars
- `_G.dropbar.menus[win]` — all open menus
- `_G.dropbar.callbacks[buf][win]` — click callbacks, indexed and referenced from the winbar string via `%@v:lua.dropbar.callbacks.buf{N}.win{N}.fn{N}@`

**Entry point**: `plugin/dropbar.lua` hooks into the first `FileType` event to lazily load the plugin. `lua/dropbar.lua` runs setup, registers autocommands for bar attachment/updates, and populates `_G.dropbar`.

**Key files**:
- `lua/dropbar/bar.lua` — `dropbar_symbol_t` and `dropbar_t` implementations; rendering, truncation, pick mode
- `lua/dropbar/menu.lua` — `dropbar_menu_entry_t` and `dropbar_menu_t`; mouse hover/preview, fuzzy find
- `lua/dropbar/configs.lua` — all default config values; `configs.eval()` wraps function-valued options for lazy evaluation
- `lua/dropbar/api.lua` — public API (`pick`, `goto_context`, `fuzzy_find`)
- `lua/dropbar/hlgroups.lua` — defines and initializes all highlight groups
- `lua/dropbar/sources/` — symbol providers: `lsp`, `treesitter`, `path`, `markdown`, `terminal`
- `lua/dropbar/utils/` — helpers for bar/menu operations, fuzzy find, highlights

**Sources** implement a single function `get_symbols(buf, win, cursor)` returning `dropbar_symbol_t[]`. The `path` source handles file hierarchy; `lsp`, `treesitter`, and `markdown` provide code symbols. Source selection is configured per-filetype via `opts.bar.sources`.

**Rendering pipeline**: `dropbar_t:update()` calls each source's `get_symbols()`, collects results into `components`, then `dropbar_t:cat()` builds the winbar string by concatenating `symbol:cat()` calls (which emit `%@callback@icon name%X` click regions). `dropbar_t:truncate()` shortens the string when it exceeds window width.

## Documentation

`doc/dropbar.txt` is auto-generated — never edit it directly. Edit `README.md` and run `make docs` to regenerate.

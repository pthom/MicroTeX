# MicroTeX — ImGui Bundle fork

This is a fork of [NanoMichael/MicroTeX](https://github.com/NanoMichael/MicroTeX),
maintained as part of [Dear ImGui Bundle](https://github.com/pthom/imgui_bundle)
to provide native LaTeX math rendering inside `imgui_md` (markdown) documents.

The base branch is `openmath` (the openmath rewrite of MicroTeX, which removes
the tinyxml2 dependency and adds Unicode math symbol support). All
ImGui-Bundle-specific changes live on the `imgui_bundle` branch on top of the
upstream `openmath` tip.

## What's different from upstream

Each adaptation is bracketed in source by `// [ADAPT_IMGUI_BUNDLE]` comments
(or `#[[ [ADAPT_IMGUI_BUNDLE] ]]` in CMake), and each commit on the
`imgui_bundle` branch is prefixed with `[Bundle]`. To see all bundle-specific
diffs against upstream:

```bash
git log official/openmath..imgui_bundle --oneline
git diff  official/openmath..imgui_bundle
```

The current set of patches:

### 1. `lib/microtexexport.h` — add a `MICROTEX_STATIC` branch

The original MSVC branch in `microtexexport.h` only knows two states:

- `MICROTEX_LIBRARY` defined → `__declspec(dllexport)` (building a DLL)
- `MICROTEX_LIBRARY` undefined → `__declspec(dllimport)` (consuming a DLL)

There is no static-library branch. Static-lib consumers see `dllimport`,
emit `__imp_*` references, and the linker fails to resolve them against
the static `microtex.lib` (which only contains plain symbols).

The fix adds a third branch: when `MICROTEX_STATIC` is defined, expand
`MICROTEX_EXPORT` to nothing. Consumers reference plain symbols directly.

### 2. `lib/CMakeLists.txt` — proper `MICROTEX_*` options + static-lib wiring

Lifts the previously undocumented internal cache variables to proper
`option()` / cache `STRING` declarations with documented defaults that
match the original upstream behavior. No behavior change for existing
consumers; new option names are clearer and discoverable via `cmake -L`.

| Old name                  | New name                          | Default |
|---------------------------|-----------------------------------|---------|
| `_BUILD_STATIC`           | `MICROTEX_BUILD_STATIC`           | `OFF`   |
| `_HAVE_AUTO_FONT_FIND`    | `MICROTEX_HAVE_AUTO_FONT_FIND`    | `ON`    |
| `_GLYPH_RENDER_TYPE` / `GLYPH_RENDER_TYPE` | `MICROTEX_GLYPH_RENDER_TYPE` | `0` |

The upstream `_DISABLE_ALIAS` variable is dropped entirely (not renamed):
`VERSION` / `SOVERSION` are no-ops on static libraries anyway, so gating
on `NOT MICROTEX_BUILD_STATIC` alone is the complete and correct
condition. Also fixes a typo in the original `_HAVE_AUTO_FONT_FIND`
block (`${_HAVE_AUTO_FONT_FIND_}` with a trailing underscore — expanded
to empty).

When `MICROTEX_BUILD_STATIC` is `ON`, the microtex target exposes
`MICROTEX_STATIC` as a `PUBLIC` compile definition, so consumers linking
against `microtex` automatically inherit it via `target_link_libraries()`.
This pairs with patch #1 to make static linking work cleanly on MSVC.

For `SHARED` builds, the existing `MICROTEX_LIBRARY` `PRIVATE` define is
unchanged: microtex sees `dllexport`, consumers see `dllimport`.

## Upstreaming

Upstream MicroTeX has been dormant since mid-2023. These patches are
small, self-contained, and platform-improving rather than ImGui-Bundle
specific — they would benefit any static-lib consumer on Windows.
A future PR to NanoMichael/MicroTeX with the same changes would let us
drop this fork.

# Provenance & License

These files are **verbatim copies** of source pulled from the upstream Chromium
project, included here for study. They are **not** authored by this repository.

| Local path | Upstream path (chromium/chromium @ `main`) |
|---|---|
| `docs/overview.md` | `docs/accessibility/overview.md` |
| `docs/how_a11y_works.md` | `docs/accessibility/browser/how_a11y_works.md` |
| `blink-accessibility/ax_object_cache_lifecycle.h` | `third_party/blink/renderer/modules/accessibility/ax_object_cache_lifecycle.h` |
| `blink-accessibility/blink_ax_tree_source.h` | `third_party/blink/renderer/modules/accessibility/blink_ax_tree_source.h` |
| `blink-accessibility/ax_object_cache_impl.h` | `third_party/blink/renderer/modules/accessibility/ax_object_cache_impl.h` |
| `blink-accessibility/ax_object.h` | `third_party/blink/renderer/modules/accessibility/ax_object.h` |
| `blink-accessibility/ax_node_object.h` | `third_party/blink/renderer/modules/accessibility/ax_node_object.h` |

- **Source mirror:** `https://raw.githubusercontent.com/chromium/chromium/main/...`
- **Retrieved:** 2026-06-28, from the `main` branch (unpinned; treat as a snapshot).
- **License:** Chromium is distributed under a BSD-style license. The C++
  headers carry their original copyright headers (Google / Apple). See the
  Chromium `LICENSE` file: https://chromium.googlesource.com/chromium/src/+/main/LICENSE

> Note: `ax_object_cache_impl.cc` (~270 KB) is the implementation behind
> `ax_object_cache_impl.h`. It was read during research and is quoted in the
> curated docs, but not copied wholesale here to keep the artifact lean.
> Browse it at:
> https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/modules/accessibility/ax_object_cache_impl.cc

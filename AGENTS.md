# Agent notes for this Helium fork

This is a personal fork of [Helium](https://github.com/imputnet/helium). Work here is for this fork only.

## Workspace layout

This repo lives in a build workspace at `~/workspace/ws-helium/`:

- `ws-helium/helium/` — this repo (the fork of imputnet/helium, aka the
  `helium-chromium` submodule in platform repos).
- `ws-helium/helium-macos/` — clone of imputnet/helium-macos, the macOS build
  harness. Its `helium-chromium` submodule is wired to `../helium` (this fork),
  so builds pick up local patch work — **commit fork changes before building**.
- `ws-helium/build-and-run.sh` — one-command dev build: checks deps, syncs the
  submodule to this fork's `main`, runs `he setup` (first time only, downloads
  Chromium — hours), `he build`, `he run`. Subcommands: `sync`, `build`, `run`.
- The Chromium tree materializes at `ws-helium/helium-macos/build/src/` after
  setup. For fast iteration, edit files there directly and re-run
  `./build-and-run.sh build` (incremental ninja), then fold changes back into
  a patch with quilt (see helium-macos docs) — do not re-run setup.
- **Never start a build yourself.** Builds (`./build-and-run.sh`, `he build`,
  ninja, etc.) are run by the user only — they are long and hog the machine.
  Verify patch work with the scratch-tree method and `check_patch_files.py`,
  then report that the tree is ready and let the user build.

## What this repo is

There is **no Chromium source code here**. This repo is a patch set (plus
config/tooling) applied on top of ungoogled-chromium, which itself sits on top
of Chromium. Key files:

- `chromium_version.txt` — the pinned Chromium version (e.g. `151.0.7922.137`).
  All patch contexts must match this exact tag.
- `patches/series` — ordered list of patches. Order matters: a patch applies to
  the tree as left by every patch above it. Blank lines group related patches.
- `patches/helium/ui/layout/` — Helium's tab strip / browser layout patches.
  `vertical.patch` is the vertical (sidebar) tab strip; `zen-mode.patch` and
  `helium/ui/native-frame-materials.patch` also modify vertical tab strip files.
- Actual builds happen in the platform repos (`helium-macos` etc.), which check
  out Chromium, apply this series, and compile. You cannot compile from here.

## How to author a new feature patch without a Chromium checkout

1. **Fetch pristine upstream files from gitiles** at the pinned tag:
   `https://chromium.googlesource.com/chromium/src/+/refs/tags/$(cat chromium_version.txt)/<path>?format=TEXT`
   returns base64 (pipe through `base64 -d`). Append `/?format=JSON` to a
   directory path to list it (strip the first line — it's a `)]}'` guard).
2. **Find every patch that already touches your target files** — context lines
   must match the _patched_ state, not upstream:
   `grep -rn "^--- a/<path>" patches/`
3. **Reconstruct the patched state** in a scratch git repo: copy the upstream
   files in, commit, then apply the touching patches in `series` order with
   `git apply --include='<path-glob>' patches/...`. Caveat: `git apply` is
   atomic — the mini-tree must contain _every_ file the patch touches under
   your include globs, or the whole patch fails.
4. **Make your edits** in the scratch tree, then generate the patch with
   `git diff` (after `git add -N` for new files) and strip the `diff --git`,
   `index`, and `new file mode` lines to match the house style
   (`--- a/...` / `+++ b/...`; new files use `--- /dev/null`).
5. **Add it to `patches/series`** in a position that respects file overlap:
   after every patch whose changes yours builds on. Patches later in the series
   that touch the same files tolerate line offsets, but not context conflicts.
6. **Verify**: re-run step 3 from pristine upstream files but in _actual series
   order including your new patch_, and diff the result against your scratch
   tree. Also run `python3 devutils/check_patch_files.py` (checks series ↔
   files consistency).

New Helium source files use this header (GPL-3.0):

```
// Copyright 2026 The Helium Authors
// You can use, redistribute, and/or modify this source code under
// the terms of the GPL-3.0 license that can be found in the LICENSE file.
```

## Tooling gotchas

- `devutils/validate_patches.py -r` (remote full-series validation) currently
  crashes with `TypeError: 'NoneType' object is not subscriptable` while
  parsing Chromium's DEPS — pre-existing breakage with newer DEPS syntax, not
  a signal about your patch. Use the scratch-tree method above instead.
- The Claude Code shell here is zsh: `v="--a --b"; cmd $v` passes ONE argument
  (no word splitting). Use `bash <<'SCRIPT'` heredocs or arrays for scripts.
- `devutils/check_patch_files.py` exits 0 silently on success.

## Vertical tab strip architecture (Chromium 151, post-Helium patches)

Helium's sidebar is Chromium's new collection-based vertical tab strip,
heavily patched:

- `chrome/browser/ui/views/frame/vertical_tab_strip_region_view.{h,cc}` —
  the sidebar container (`final`). Its base
  `BaseTabStripRegionView` (`frame/base_tab_strip_region_view.{h,cc}`) exposes
  `GetTabAnchorViewAt(int model_index)` → the tab's `TabView`, and holds a
  protected `tab_strip_model_`. Manually-positioned children (resize area,
  shadow frame) are excluded from its `FlexLayout` via
  `SetProperty(views::kViewIgnoredByLayoutKey, true)` and positioned in
  `Layout(PassKey)`.
- `chrome/browser/ui/views/tabs/common/` — shared horizontal/vertical
  machinery: `TabStripView` (pinned + unpinned `views::ScrollView`s),
  `TabView` (per-tab view; custom `LayoutDelegate`, do not add children
  casually), `TabCollectionNode` (view-model tree mirroring `TabStripModel`).
- `chrome/browser/ui/views/tabs/vertical/` — vertical-only views; its
  `BUILD.gn` splits headers into `source_set("vertical")` and `.cc` files into
  `source_set("impl")` — add new files to both lists.
- Helium removed upstream's `VerticalTabStripTopContainer` (vertical.patch);
  don't reference it.
- To observe raw key events per-window (incl. modifier-only presses), use
  `views::EventMonitor::CreateWindowMonitor` — on macOS `flagsChanged`
  NSEvents surface as `ui::EventType::kKeyPressed/kKeyReleased`, and modifier
  state is in `event.flags()` (`ui::EF_COMMAND_DOWN` etc.). Window
  deactivation (Cmd+Tab) eats the release event — pair with a
  `views::WidgetObserver` activation check.
- The scroll views use layered scrolling: overlay views painted above them
  don't get damaged when contents scroll — subscribe via
  `views::ScrollView::AddContentsScrolledCallback` to repaint.
- `gfx::SlideAnimation`/`LinearAnimation` with a raw `gfx::AnimationDelegate`
  tick on a `base::Timer` capped at 60 fps. For UI animations, inherit
  `views::AnimationDelegateViews` instead — it attaches a
  `CompositorAnimationRunner` so animations tick with vsync at the display's
  refresh rate. It tracks only one `gfx::AnimationContainer`, so multiple
  animations on one delegate must share a container via `SetContainer()`
  (see `BrowserView` zen reveal animations in zen-mode.patch).

Worked example of all of the above:
`patches/helium/ui/layout/vertical-tab-shortcut-hints.patch` (Cmd-held ⌘n
hint chips next to vertical tabs).

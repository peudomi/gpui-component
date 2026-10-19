# Upstream merge policy

This repository is a fork of `longbridge/gpui-component` that builds against the
GPUI fork [`peudomi/gpui`](https://github.com/peudomi/gpui) instead of the
`gpui-pre-*` snapshots upstream takes from crates.io. Almost everything this
fork changes exists only to make upstream's code compile and behave on top of
`peudomi/gpui`; fork change 5 is the one hook the app needs. Keep that diff as
small as possible.

## Branch structure

`main` is the trunk of this fork. There is no separate mirror branch. To refer
to the upstream state, use the `upstream/main` remote-tracking ref after
`git fetch upstream`.

## Fork changes

Show them with `git diff $(git merge-base HEAD upstream/main) HEAD`. They fall
into five kinds, and a merge must keep all of them:

1. **GPUI source.** The workspace dependency keys for crates that
   `peudomi/gpui` provides point at it as a git dependency with no `rev`
   (`Cargo.lock` pins the commit):

   ```toml
   gpui = { git = "https://github.com/peudomi/gpui.git" }
   gpui_platform = { git = "https://github.com/peudomi/gpui.git", features = [...] }
   gpui_web = { git = "https://github.com/peudomi/gpui.git" }
   gpui_macros = { git = "https://github.com/peudomi/gpui.git" }
   reqwest_client = { git = "https://github.com/peudomi/gpui.git" }
   ```

   Keep upstream's `features` on each key. `[profile.dev.package]` entries that
   name a `gpui-pre-*` package are renamed to the `peudomi/gpui` package
   (`gpui-pre` to `gpui`, `gpui-pre-platform` to `gpui_platform`, and so on).
   Every other dependency (`sum-tree`, `reqwest`, `gpui-fast*`, ...) follows
   upstream, unless a type crosses into a `peudomi/gpui` crate and compilation
   requires the same source. An exact pin that cannot resolve against the
   dependencies of `peudomi/gpui` is raised to the version they need (for
   example `wasm-bindgen` in `crates/base/examples/wasm/Cargo.toml`).
2. **Rounded content masks.** `peudomi/gpui` adds `corner_radii` to
   `gpui::ContentMask`, so every struct literal needs `..Default::default()`:
   `ContentMask { bounds, ..Default::default() }`.
3. **GPUI API drift.** `peudomi/gpui` follows Zed `main` daily, so it can be
   ahead of the snapshot upstream compiles against. Minimal source adaptations
   for those differences live here (for example `9bd1745e`). Prefer the shape
   Zed's own callers use for the new API.
4. **No `gpui-kit` facade in GPUI macros.** Upstream's `gpui-pre-macros`
   rewrites `gpui::` paths in macro output to `gpui_kit::`; `peudomi/gpui`'s
   `gpui_macros` does not. This fork does not support consumers that depend
   only on `gpui-kit`. Workspace crates that do so and use GPUI macros get
   `extern crate gpui_kit as gpui;` at their crate root (`crates/story`,
   `examples/tig`), and `IntoPlot`'s package lookup in
   `crates/component-macros/src/crate_path.rs` includes `gpui`.

5. **Closed dock extent hook.** `DockAreaRenderer::closed_dock_extent` in
   `crates/base/src/dock/dock_area.rs` lets a renderer decide how much room a
   closed dock keeps. Its default returns `CLOSED_BOTTOM_STRIP` for the bottom
   dock and zero otherwise, so upstream's behavior is unchanged. Base reads it
   in `render_dock` for a closed dock, and in `resize_dock` and
   `end_dock_resize` where upstream uses `CLOSED_BOTTOM_STRIP`. A dock may be
   dragged below its minimum and shut only when its closed extent is above
   zero (upstream: only the bottom dock), since a dock that keeps nothing also
   loses the handle that would drag it open again. If upstream
   adds another use of `CLOSED_BOTTOM_STRIP` in the resize or render path,
   route it through the hook too.

## Merge procedure

```shell
git fetch upstream
git merge upstream/main
```

- **Root `Cargo.toml`**: accept everything upstream changed (members, versions,
  new dependencies), then reapply fork change 1. If upstream adds a key for
  another `gpui-pre-*` crate, redirect it too when `peudomi/gpui` has a crate
  of that name under `crates/`; otherwise keep upstream's entry.
- **`Cargo.lock`**: take upstream's (`git checkout --theirs Cargo.lock`) and let
  the next build re-resolve it. That drops the `gpui-pre-*` entries and pins
  `peudomi/gpui` at its current `main`.
- **Files the fork changed for `ContentMask`**: take upstream's side and add
  `..Default::default()` back to each literal.
- **Compile errors after the merge**: a missing `corner_radii` field is fork
  change 2. `cannot find module or crate gpui` from a GPUI macro in a crate
  that depends only on `gpui-kit` is fork change 4. Anything else that comes from GPUI is fork change 3; fix it in the
  smallest way and note it in the commit.

Do not edit upstream's CI scripts or workflows. Checks that assume `gpui-pre`
pins (for example `script/check-gpui-pin.ts`) cannot pass on this fork.

## Following `peudomi/gpui`

When `peudomi/gpui` `main` moves without an upstream merge here, move the pin
with `cargo update -p gpui` (it updates every crate from that git source),
verify, and commit as `chore: update gpui lock`.

## Verification (mandatory)

On Linux, after `script/install-linux.sh` for the system packages:

```shell
cargo check --workspace --exclude gpui-shell --exclude webview
cargo test --workspace --exclude gpui-shell --exclude gpui-kit-recipes \
  --exclude gpui-kit --features gpui-component-story/test-support $TEST_EXCLUDES
cargo test -p gpui-kit --features test-support --lib --test ui
```

These mirror the Linux `lint` and `test` jobs in `.github/workflows/ci.yml`
without `--locked`; `$TEST_EXCLUDES` is the example-crate list the `test` job
computes. When upstream changes those jobs, follow them, leaving out the
example crates as the `test` job does and skipping steps that need a real
renderer.

Because of fork change 4, the tests that check `gpui-kit` works without a
direct GPUI dependency are known failures and are left out: every test target
of `gpui-kit-recipes`, and the `integration` target of `gpui-kit`. Do not add
the alias to them; that would defeat what they test.

## Automated daily sync

A scheduled Claude Code cloud agent runs this once a day in a Linux sandbox,
right after the daily sync of `peudomi/gpui`. The GPUI baseline is
`peudomi/gpui` `origin/main` as it stands after that sync, never one of its
review branches.

1. **Skip when nothing changed.** If `upstream/main` is already an ancestor of
   `HEAD` and the `peudomi/gpui` commit pinned in `Cargo.lock` equals
   `git ls-remote https://github.com/peudomi/gpui.git main`, stop.
2. **Merge** with the procedure above on a work branch, keeping the default
   merge message (`Merge remote-tracking branch 'upstream/main'`). Fixes made
   during the merge go into the merge commit. If upstream has nothing new but
   `peudomi/gpui` moved, do only the pin update from
   [Following `peudomi/gpui`](#following-peudomigpui).
3. **Verify** with the checks above, then type-check the macOS build:
   `cargo check -p gpui-base -p gpui-component --target aarch64-apple-darwin`,
   with `gpui`, `gpui_platform`, `gpui_web`, `gpui_macros` and `reqwest_client`
   patched through `--config 'patch."https://github.com/peudomi/gpui.git".<crate>.path="..."'`
   to a local `peudomi/gpui` checkout of the baseline commit that carries the
   temporary Apple build edit described in that repository's `UPSTREAM.md`
   (automated sync, step 3). Discard the edit and the resulting `Cargo.lock`
   change afterwards. Errors in this repository's code are real failures;
   record a failure caused by missing tooling as *unverified*.
4. **Land on `main` directly** only if all of these hold:
   - no content conflict (`UU`/`AA`) other than root `Cargo.toml`/`Cargo.lock`
     and `ContentMask` literals resolved as described above,
   - no hand edit beyond fork changes 1, 2, 4 and 5 (any fork change 3 needs review),
   - every Linux check passes, and the macOS check found no real error.
5. **Otherwise ask for review**: commit as far as it got, push it to
   `claude/upstream-sync-<YYYY-MM-DD>`, leave `main` untouched, and notify the
   owner. The run's final report lists the upstream range, each conflict and
   how it was resolved, every hand edit, and failing or unverified checks.

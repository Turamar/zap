# WARP.md — Engineer Handbook

How to build, check, test, and style code in this repo. Process (issues, specs,
reviews) lives in [CONTRIBUTING.md](CONTRIBUTING.md); this file is commands and
conventions. Feature specs live under [`specs/`](specs/), agent workflow skills
under [`.agents/skills/`](.agents/skills/).

## Prerequisites

- Stable Rust toolchain (`rustup`; see `script/install_rust`). No local toolchain?
  validate through CI instead (see `.github/workflows/pr-check.yml`).
- `./script/bootstrap` — platform-specific setup.

## Build & run

```bash
cargo run              # build and run Zap
cargo check            # fast compile check (default-members subset, see below)
cargo check -p <crate> # check one crate, e.g. -p warp_terminal
```

The workspace is `members = ["crates/*", "app"]`, but `default-members` is narrowed
to the hot subset (`app`, `channel_versions`, `command`, `editor`, `markdown_parser`,
`sum_tree`, `warpui`, `warp_completer`, `warp_terminal`, `warp_util`). Plain
`cargo check` / `cargo test` only cover that subset; other crates need `-p`.

## Before a PR: `./script/presubmit`

This is the gate. Run it locally (or rely on CI) before pushing. It runs, in order:

1. `cargo fmt -- --check`
2. `cargo clippy --workspace --exclude warp_completer --all-targets --tests -- -D warnings`,
   then `cargo clippy -p warp_completer --all-targets --tests -- -D warnings`
   (completer uses default features, not all-features)
3. `clang-format` check over `crates/warpui/src/` and `app/src/` (C/C++/ObjC files)
4. `wgslfmt --check` over `*.wgsl` shaders
5. `PSScriptAnalyzer` over PowerShell scripts (skipped without `pwsh`, except in CI
   where missing `pwsh` fails the run)
6. `cargo nextest run --no-fail-fast --workspace --exclude command-signatures-v2`,
   plus `cargo nextest run -p warp_completer --features v2`
7. `cargo test --doc`

## Running tests

```bash
cargo nextest run                                                  # full unit suite
cargo nextest run -p <crate>                                       # one crate
cargo test -p warp_terminal --lib escape_sequences                 # one module
cargo nextest run -p warp_completer --features v2                  # completer v2 mode
cargo test --doc                                                   # doc tests
```

Conventions: unit tests live in `${filename}_tests.rs` (or `mod_test.rs`), wired at
the end of the source file via `#[cfg(test)] #[path = "..."] mod tests;`.
Integration tests use the `crates/integration` framework; examples in
`app/src/integration_testing/`. Timeouts/retries are configured in
`.config/nextest.toml` (slow test = 30s; integration tests get retries on CI).

## Style

- `cargo fmt` must be clean; do not hand-format.
- `cargo clippy` with `-D warnings` must be clean (see the exact invocations above).
- No redundant type annotations on closure params.
- One consolidated `use` block at the top; no long fully-qualified paths inline
  (`#[cfg]` branches excepted).
- Context params are named `ctx` and go last; a closure param, if any, goes last.
- Delete unused params outright (no `_` prefix); update call sites.
- Inline format args: `"{x}"`, not `"{}", x`.
- `match` stays exhaustive — no `_` wildcard unless genuinely needed.
- Do not touch unrelated comments/code/formatting as a side effect of a change.

## Architecture map (short)

```
app/  — main binary: wiring, entry points, platform glue, UI view roots
  ↑ product-domain crates: ai / computer_use / vim / onboarding /
  ↑                        warp_completer / languages … (+ code_review, in app/)
  ↑ framework crates: warpui / warpui_core / editor / ui_components /
  ↑                   sum_tree / syntax_tree
  ↑ infra crates: warp_core / warp_util / http_client / websocket /
                  ipc / jsonrpc / persistence / managed_secrets …
```

- Terminal core: `crates/warp_terminal/` + `app/src/terminal/` (427 files).
  `TerminalModel::lock()` deadlocks easily — never re-lock up a stack that already
  holds it; pass the locked reference down; keep lock scope minimal.
- Agent UI/conversations: `app/src/ai/` (largest subtree, ~390 files); grep by
  subtopic (`agent_*`, `conversation_*`, `mcp`, `tool_*`). Model clients and the
  tool-call protocol: `crates/ai/`.
- Keymap/hotkeys: `crates/warpui/.../key_events.rs` (winit event loop) +
  `app/src/util/bindings.rs`.
- Settings: `crates/settings*`; UI in `app/src/settings_view/`.
- Feature flags: `FeatureFlag` enum + `DOGFOOD/PREVIEW/RELEASE_FLAGS` in
  `crates/warp_features/src/lib.rs`; prefer runtime `is_enabled()` over `#[cfg]`;
  remove flag + dead branches after rollout.
- Persistence: Diesel + SQLite; migrations in `crates/persistence/migrations/`
  (`up.sql`/`down.sql` per migration); never hand-edit the generated
  `crates/persistence/src/schema.rs`.
- Child processes: never `std::process::Command` directly (flashes a console window
  on Windows) — always go through `crates/command`.
- Platform code: isolate behind `#[cfg(target_os = "...")]`.

## Branches & commits

- Default branch is `main`. Branch from it; prefix work branches `fix/<topic>`
  (CI's light check runs on `fix/**` and PRs to `main`).
- Commit messages explain *what and why*. Keep each PR to one logical change; every
  changed line must trace back to the request — no drive-by refactors.
- Add a changelog trailer (`CHANGELOG-NEW-FEATURE`, `CHANGELOG-IMPROVEMENT`, or
  `CHANGELOG-BUG-FIX`); omit only for docs/refactor-only changes.

## Releases (maintainers)

- `pr-check.yml`: light validation per push/PR.
- `zap_release.yml`: full 3-OS build on `v*` tag pushes (plus manual dispatch) with
  `CHANNEL=oss`; creates the GitHub Release with auto-generated notes over the
  previous `v*` tag. Never delete or rewrite pushed `v*` tags.
- Oss-channel autoupdate pulls from this fork's GitHub Releases
  (`app/src/autoupdate/github.rs`).

## License

AGPL-3.0-only for the whole tree, except `crates/warpui` and `crates/warpui_core`
(MIT). Keep all copyright/license notices intact, including third-party files under
`app/assets/…`, `resources/bundled/skills/*/LICENSE.txt`, and `lib/*/LICENSE-*`;
new bundled third-party code must be registered in the `ADDITIONAL_LICENSES` lists
(`script/prepare_bundled_resources`, `script/windows/prepare_bundled_resources.ps1`).

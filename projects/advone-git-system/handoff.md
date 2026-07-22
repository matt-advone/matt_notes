---
type: ProjectStatus
related_to: "[[tolaria]]"
status: Active
---

# advone-git-system — Handoff

## Last updated
2026-07-21 (session 3)

## What this project is
A cross-platform (macOS + Windows) desktop app that lets non-technical users
browse and sync their GitHub repos **without installing git**. Sign in with
GitHub inside the app, see a branded home page of repos you can access (with
descriptions), then either Sync (already cloned) or Clone (pick a folder). The
app remembers clone locations so it never re-clones and auto-pulls every 10
minutes. All syncs are read-only (pull) — never commits/pushes.

**Where it lives:** `/Users/mattperzel/src/advone-git-system`

Originally requested as Electron; Matt asked to switch to **Tauri v2** (latest).
Stack: Tauri v2 + React 18 + TS + Vite 5 + Tailwind v4. Git via `isomorphic-git`
in the webview. Theme copied from advantageone.tech (dark navy `#061423`,
primary `#99cbff`/`#4e96d5`, secondary `#bf3701`, fonts Plus Jakarta Sans +
Inter).

## What was done this session (key decisions)
- Installed Rust 1.97.1 + `tauri-cli` 2.11.4 (Rust wasn't present; installed
  rustup non-interactively). Icons generated via a custom Node PNG encoder +
  `cargo tauri icon` (no image tooling needed).
- Scaffolded the project manually (the official `create-tauri-app` needs a
  TTY and failed). Wrote tauri.conf.json v2, Cargo.toml, capabilities,
  lib.rs/main.rs.
- **Key architecture decision:** isomorphic-git can't use the browser `fetch`
  for github.com (no CORS headers on git endpoints) and has no Buffer global in
  a webview. Solved by: (a) polyfilling `globalThis.Buffer` from the `buffer`
  pkg + a minimal `process` global, (b) writing an isomorphic-git HTTP client
  on top of `@tauri-apps/plugin-http` (Rust reqwest → bypasses CORS), and
  (c) implementing isomorphic-git's `fs.promises` contract as a thin adapter
  over custom `fs_*` Rust commands (read/write/mkdir/readdir/stat/lstat/
  unlink/readlink/symlink/rm/rmdir) — avoiding the scoped fs plugin entirely so
  any user-chosen absolute path works.
- Sync engine (`lib/sync.ts`): fetch origin branch → resolve
  `refs/remotes/origin/<branch>` → `writeRef` local branch (force) → `checkout`
  (force). Equivalent to a hard reset to remote tip — correct for read-only,
  no conflicts ever.
- GitHub auth: OAuth **Device Flow** (no client secret). Client ID from
  `VITE_GITHUB_CLIENT_ID` env or a runtime `github-client-id.txt` in appData.
  Token stored in `auth.json` written 0600 (Unix) via a `fs_write_file_secret`
  Rust command. Repo registry in `registry.json`.
- UI: Login (device-flow), Repos home (hero + grid + search/filter + sync/clone
  buttons + status badges), RepoDetail (header, local-copy panel, reveal in
  Finder, forget). Auto-sync loop every 10 min in `AppStateProvider`.

## Current state — what works / verified
- `npm run build` (tsc -b + vite build) passes clean (only harmless
  dynamic-import warnings).
- `cargo build` of the Rust backend compiles clean.
- `npm run tauri:dev` launches the app window with **no runtime errors / no
  panics** — verified the binary ran. (Did NOT exercise a real clone/sync end
  to end because that needs a configured GitHub OAuth Client ID.)

## NOT yet verified / known gaps
- **No GitHub OAuth Client ID configured yet.** Login flow + clone + sync have
  not been exercised against a real repo. Create the OAuth App (README) and set
  `VITE_GITHUB_CLIENT_ID` to test.
- The isomorphic-git fs adapter (`tauri-fs.ts`) and http client (`git-http.ts`)
  are the integration risk points — both written carefully against the actual
  isomorphic-git v1.38 contract (read its source to confirm the `fs.promises`
  + http `request` shapes) but untested against a real clone. First thing to
  debug if a clone fails: check devtools console (auto-opens in dev) and these
  two files.
- `git-http.ts` buffers the entire response into one Uint8Array chunk (no
  streaming) — fine for moderate repos, may be slow for very large ones.
- Signing is **wired but disabled**. Produces unsigned builds until certs are
  provided. README documents that **ACM certs can't be used** for code signing
  (not exportable with private key) and what to use instead (Apple Developer ID
  + notarization for macOS; OV/EV or Azure Trusted Signing for Windows).
- Token stored as plaintext file (0600). Stronghold plugin is the documented
  upgrade path.

## Next steps
1. Create the GitHub OAuth App, set `VITE_GITHUB_CLIENT_ID`, and do a real
   clone + sync end-to-end. Debug `tauri-fs.ts`/`git-http.ts` as needed.
2. Acquire signing certs (Apple Developer ID; Windows OV/EV or Azure Trusted
   Signing). Wire env vars per README and produce signed `tauri:build`.
3. Add a GitHub Actions workflow using `tauri-apps/tauri-action` with the
   signing secrets.
4. (Optional) Migrate token storage to `tauri-plugin-stronghold`.
5. (Optional) Streaming http body in `git-http.ts` for large repos.
6. (Optional) UI polish: empty-state illustrations, per-repo sync log/history.

## Reminders / gotchas
- Rust must be on PATH for tauri commands: `. "$HOME/.cargo/env"` (already
  added to shell profile by rustup; new shells are fine).
- `tauri dev` opens devtools automatically in debug builds.
- tauri.conf.json `bundle.targets` is `"all"` — on a Mac build it will try to
  make Windows targets too and fail; set target-specific via
  `tauri build --target <triple>` or narrow `targets` when building on one OS.
- Capabilities HTTP scope covers `*.github.com`, `api.github.com`,
  `raw.githubusercontent.com`, `objects.githubusercontent.com`,
  `codeload.github.com` — add more if you ever proxy elsewhere.
- CSP in tauri.conf.json is intentionally minimal (everything goes through the
  Rust http plugin). Don't relax without reason.


## Session 3 — 2026-07-21: Branded app icon

- Generated a proper branded app icon from the Advantage One logo
  (`~/Downloads/AO-Horizontal-Color.png`). The swirl mark was extracted from
  the 102px-tall logo, unmixed into orange (#F15A22) / gray (#B1B3B6) layers,
  upscaled with edge re-sharpening (blur + smoothstep on the coverage masks),
  and composed onto a macOS-style dark charcoal-navy rounded tile at 1024px.
- Ran `npx tauri icon src-tauri/icons/source.png` — regenerated ALL icon
  sizes (desktop .icns/.ico/pngs, iOS AppIcon set, Android mipmaps).
  `src-tauri/icons/source.png` is now the 1024px master.
- Alternate light-tile variant saved at `~/Downloads/ao-app-icon-light-alt.png`;
  regeneration script at `~/Downloads/ao-app-icon-generator.py` (needs Pillow,
  plus `mark.png` extracted from the horizontal logo).
- Not yet verified: rebuilt app bundle with the new .icns (run
  `npm run tauri build` or dev to see it in the Dock).

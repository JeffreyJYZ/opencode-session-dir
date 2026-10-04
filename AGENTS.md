# AGENTS.md

Repository: `@jeffreyjyz/opencode-session-dir` — an opencode v2 plugin that binds
extra working directories to a single session, durably across restarts. Its
behaviour was **inspired by** `opencode-add-dir` (0xkuze), which is a *reference
only* — no code is shared or copied, and that repository is never edited here.
Sits beside `cmduse`, `oc-cmd-compare` (`mpc`), `reqshape`,
`opencode-context` and `opencode-shell-rc` in `~/dev/cmdcode-tools/`.

## Layout

```
src/index.ts   server half (v2 `{ id, setup }`): permission.evaluate + session.context hooks
src/tui.tsx    TUI half (./tui): /session-dir, /session-dir-list, /session-dir-remove
src/state.ts   the one shared store: $XDG_DATA_HOME/opencode/session-dir/bindings.json
src/context.ts system-prompt injection for the bound dirs
src/ids.ts     plugin id, slash names, keymap command ids
scripts/build-tui.ts  solid-transformed TUI bundle (see Traps)
index.js, tui.js      root shims for local-directory plugin loading (see Traps)
```

## Core rules

- **Session-scoped only.** There is exactly one binding mode: durable, keyed by
  session id. Do not add a global or process-wide mode — that is what
  `opencode-add-dir` already does, and it is the reason this plugin exists.
- **The server never decides scope from anything but `event.sessionID`.** Both
  hooks filter through `bindingsFor(String(event.sessionID))`; a binding for
  another session must never authorize this one.
- **Allow, never deny.** `permission.evaluate` only turns `ask` into `allow`,
  and only for `action === "external_directory"` with `effect === "ask"`. A
  configured `deny` always wins.
- **Match directory boundaries, not string prefixes.** Resources are canonical
  boundaries (`<dir>/*`); strip the trailing `/*` and compare with
  `isChildOf`, so `/boundary` is never inside `/bound`.
- **The two halves share nothing but the file.** TUI writes, server reads via a
  short mtime cache (`freshBindings`). Writes go through `addBinding` /
  `removeBinding`, which invalidate the cache.

## Traps

- **`permission.evaluate` and `session.context` both carry `sessionID`** — use
  it. `PermissionEvaluation.sessionID` is on the event; `session.context`'s event
  exposes `sessionID` and a mutable `system` array.
- **Delete the session key when its last binding is removed** (in
  `removeBinding`), otherwise `bindings.json` accumulates empty arrays.
- **`keymap.layer()` must run inside a render.** Registering it directly in
  `setup` throws `Keymap.Provider is missing`, so the app slot mounts a no-op
  whose `render` calls it (cast to `() => JSX.Element`). This is the same shape
  `opencode-context` uses.
- **Directory-browser dialogs are imperative** (`ui.dialog.select` / `prompt` /
  `confirm` / `alert`); nothing in the TUI half needs JSX, but the file is
  `.tsx` and built through the solid transform so future JSX stays reactive.
- **Build the TUI with `@opentui/solid`'s transform**, not a plain `bun build`.
  `@opentui/*` and `solid-js` stay external because the host rewrites the
  entry's imports to its own module instances.
- **`biome check .` aborts on a nested root configuration when a `.delta/`
  directory is present**; this repo's `biome.json` excludes `**/.delta`.
- **Local-directory plugins resolve `index` / `server` / `tui` beside the
  package root, not the `exports` map.** The root `index.js` / `tui.js` shims
  re-export `dist/`; npm installs use `exports`. Keep them in sync.

## Tests

Hermetic: every test points `HOME` and `XDG_DATA_HOME` at a temp dir. The plugin
test stubs `ctx.permission.hook` / `ctx.session.hook`, captures the callbacks,
and drives them directly — no opencode process needed.

```sh
bun install
bun test
bun run typecheck
bun run build
```

## Publishing (NEVER without explicit go)

Same policy as the sibling repos: local commits only until the user says push.
Publishing from this account must go through `npm stage publish` + the user's
`npm stage approve` — a bare `npm publish` leaves ghost versions (see
`cmduse/AGENTS.md`). Note that `npm stage publish` leaves a `0.0.0-stage` version
in the packument; it is the staging placeholder, not a ghost. Verify with the
raw packument, not `npm view` (its cache can lag). After a publish, opencode's
per-package install cache can lag npm: `npm cache clean`, remove
`~/.cache/opencode/npm/@jeffreyjyz/opencode-session-dir@latest`, then the user
restarts. **Never restart or reload opencode yourself.**

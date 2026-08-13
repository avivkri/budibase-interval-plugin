# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Budibase custom **component** plugin ("Interval"). It renders a widget that fires a `trigger` event every N seconds, so app builders can attach a list of actions to a timer (originally: auto-refreshing a data provider).

There is no test suite and no linter configured — `build` / `watch` are the only scripts.

## Commands

```bash
yarn                 # install (CI uses yarn + Node 16)
yarn build           # rollup -c  → dist/
yarn watch           # rollup -cw, inline sourcemaps

budi plugins --build # equivalent build via the Budibase CLI
budi plugins --watch
```

## Build pipeline (rollup.config.js)

The rollup config does more than bundle; several custom inline plugins matter:

- `validateSchema()` runs `@budibase/backend-core/plugins`'s `validate()` against `schema.json` at build start — an invalid schema fails the build before anything is emitted.
- `svelte` and `svelte/internal` are **external** and mapped to globals; the Budibase client app supplies them at runtime. Never bundle Svelte.
- Output is a single IIFE at `dist/plugin.min.js`.
- `hash()` sha1s the built JS and writes `hash` + `version` into `dist/schema.json` after the copy step. Budibase uses that hash for cache-busting, so it must be regenerated on every change — never hand-edit `dist/`.
- `bundle()` tars `plugin.min.js`, `schema.json`, `package.json` into `dist/${pkg.name}-${pkg.version}.tar.gz` (e.g. `Interval-1.1.1.tar.gz`). That tarball is the release artifact.

## Release

`.github/workflows/release.yml` runs on every push to `main`/`master`: it reads `version` from `package.json`, builds, and publishes a GitHub release tagged `v<version>` with `dist/*.tar.gz` attached. **Bumping `package.json` version is what cuts a release** — a push without a bump will collide with the existing tag. `package.json` version is also the plugin version surfaced to Budibase (via `index.js` and the `hash()` step), so it is the single source of truth.

## Architecture

Registration and error-boundary layers are Budibase plugin boilerplate; only `src/` is plugin-specific.

```
index.js              pushes { Component, schema, version } onto
                      window["##BUDIBASE_CUSTOM_COMPONENTS##"] and calls
                      window.registerCustomComponent if present
  └─ lib/Wrapper.svelte    spreads $$props through
     └─ lib/Boundary.js    svelte-error-boundary; renders errors in-place
        └─ src/Component.svelte   applies Budibase `styleable` from sdk context
           └─ src/lib/Interval.svelte   the actual setInterval logic
```

`lib/` is generic scaffolding copied from the Budibase plugin template — changes there are rarely what you want. Behaviour changes belong in `src/lib/Interval.svelte`; layout/context wiring in `src/Component.svelte`.

### schema.json ↔ props contract

Each entry in `schema.settings[].key` arrives as an exported prop on `src/Component.svelte`. Adding or renaming a setting requires editing **both** files or the prop silently stays `undefined`. Current keys: `interval`, `display`, `text`, `trigger`.

`trigger` is a `type: "event"` setting — Budibase passes it down as a function, which is why `Component.svelte` wires it as `on:trigger={trigger}` rather than calling it directly.

Settings can be conditionally shown in the builder via `dependsOn` (see how `text` hangs off `display`).

### Interval lifecycle

`src/lib/Interval.svelte` starts `window.setInterval` at component init when `interval > 0` and clears it in `onDestroy`. The interval is **not** reactive — changing the `interval` prop at runtime does not restart the timer.

The stopwatch icon is imported as a raw string via `rollup-plugin-svg` and injected with `{@html}`.

## Upstream

Fork of `MartinPicc/budibase-interval-plugin`; `schema.json`'s `info` field and the README still point issues at that repo.

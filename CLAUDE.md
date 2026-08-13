# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Budibase custom **component** plugin ("Interval"). It renders a widget that fires a `trigger` event every N seconds, so app builders can attach a list of actions to a timer (originally: auto-refreshing a data provider).

Built for **Svelte 5**, required by Budibase 3.24.0+. Migrated from the Svelte 3 layout via `budi plugins --migrate-svelte5`, so the build config and `lib/` wrapper deliberately match Budibase's official plugin skeleton — prefer keeping them in sync with it rather than hand-tuning.

There is no test suite and no linter configured — `build` / `watch` are the only scripts.

## Commands

```bash
npm install          # both package-lock.json and yarn.lock are committed and in sync;
                     # `budi` prefers yarn.lock and falls back to npm when yarn is absent
npm run build        # rollup -c rollup.config.mjs  → dist/
npm run watch        # rollup -cw, inline sourcemaps

budi plugins --build # equivalent build via the Budibase CLI (also runs verification)
budi plugins --watch
```

## Build pipeline (rollup.config.mjs)

The rollup config does more than bundle; several custom inline plugins matter:

- `validateSchema()` runs `@budibase/backend-core/plugins`'s `validate()` against `schema.json` at build start — an invalid schema fails the build before anything is emitted.
- All of `svelte` and `svelte/*` are **external**, mapped by the `output.globals` function to the globals the Budibase client publishes (`svelte`, `svelteInternal`, `svelteStore`, …; see `packages/client/src/svelteGlobals.ts` upstream). Never bundle Svelte.
- The Svelte plugin compiles with `compilerOptions.compatibility.componentApi: 4`.
- Output is a single IIFE at `dist/plugin.min.js` (~5KB, since Svelte is external).
- `hash()` sha1s the built JS and writes `hash` + `version` into `dist/schema.json` after the copy step. Budibase uses that hash for cache-busting, so it must be regenerated on every change — never hand-edit `dist/`.
- `bundle()` tars `plugin.min.js`, `schema.json`, `package.json` into `dist/${pkg.name}-${pkg.version}.tar.gz` (e.g. `Interval-1.2.0.tar.gz`). That tarball is the release artifact.

`schema.json` must keep `metadata.svelteMajor: 5` — that is how Budibase tells Svelte 5 plugins from legacy ones.

## Release

`.github/workflows/release.yml` runs on every push to `main`/`master`: it reads `version` from `package.json`, builds, and publishes a GitHub release tagged `v<version>` with `dist/*.tar.gz` attached. **Bumping `package.json` version is what cuts a release** — a push without a bump will collide with the existing tag. `package.json` version is also the plugin version surfaced to Budibase (via `index.js` and the `hash()` step), so it is the single source of truth.

Actions were disabled on this fork (GitHub's default for forks that contained workflows) until 2026-08-13; they are now enabled, and the workflow also accepts `workflow_dispatch` for manual runs. The job declares `permissions: contents: write` because the repository's default `GITHUB_TOKEN` scope is read-only, which would otherwise 403 on the release step.

## Architecture

Registration and the error boundary are Budibase plugin boilerplate; only `src/` is plugin-specific.

```
index.js              pushes { Component, schema, version } onto
                      window["##BUDIBASE_CUSTOM_COMPONENTS##"] and calls
                      window.registerCustomComponent if present
  └─ lib/Wrapper.svelte    <svelte:boundary> + {#snippet failed} error UI,
                           spreads $$restProps through
     └─ src/Component.svelte   applies Budibase `styleable` from sdk context
        └─ src/lib/Interval.svelte   the actual setInterval logic
```

`lib/Wrapper.svelte` is generated verbatim by `budi plugins --migrate-svelte5`; changes there are rarely what you want and will be overwritten if the migration is re-run. Behaviour changes belong in `src/lib/Interval.svelte`; layout/context wiring in `src/Component.svelte`.

### Legacy-mode dependency (important)

`src/` is written in Svelte 4 style (`export let`, `createEventDispatcher`, `on:trigger`), which compiles to **legacy mode**. Legacy-mode components call Svelte's internal `init()`, which throws `Cannot read properties of null (reading 'u')` unless the legacy-mode flag is enabled at runtime.

The compiled bundle does `import "svelte/internal/flags/legacy"` to set that flag, but the config's `globals` function maps every `svelte/*/internal/*` id onto the single `svelteInternal` global, which **erases that side effect**. The plugin therefore relies on the *host page* having already enabled legacy mode — true for Budibase, whose client bundle still contains non-runes components. Verified working against real Svelte 5 with the flag set; fails without it.

Consequences:
- Converting `src/` to runes (`$props()`, `$effect`, callback props) would remove this dependency, at the cost of diverging from the official skeleton.
- Testing this plugin outside Budibase requires importing `svelte/internal/flags/legacy` first.
- `new Component()` (the Svelte 4 class API) throws, because the config maps `svelte/legacy` onto the plain `svelte` global, which has no `createClassComponent`. Budibase renders plugins via `<svelte:component>`, which does not use `new`, so this does not bite in practice.

### schema.json ↔ props contract

Each entry in `schema.settings[].key` arrives as an exported prop on `src/Component.svelte`. Adding or renaming a setting requires editing **both** files or the prop silently stays `undefined`. Current keys: `interval`, `display`, `text`, `trigger`.

`trigger` is a `type: "event"` setting — Budibase passes it down as a function, which is why `Component.svelte` wires it as `on:trigger={trigger}` rather than calling it directly. Props reach `src/Component.svelte` through `$$restProps` on the wrapper, so they are not declared anywhere in `lib/`.

Settings can be conditionally shown in the builder via `dependsOn` (see how `text` hangs off `display`).

### Interval lifecycle

`src/lib/Interval.svelte` starts `window.setInterval` at component init when `interval > 0` and clears it in `onDestroy`. The interval is **not** reactive — changing the `interval` prop at runtime does not restart the timer.

The stopwatch icon is imported as a raw string via `rollup-plugin-svg` and injected with `{@html}`.

## Upstream

Fork of `MartinPicc/budibase-interval-plugin`; `schema.json`'s `info` field and the README still point issues at that repo.

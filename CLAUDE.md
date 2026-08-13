# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Canonical documentation

This is one of seven Budibase plugin forks that share an identical build, release, and upgrade setup. **The full documentation lives in `../minikube-ground/dev-lab-setup/docs/budibase-plugins.md`** (the `minikube-ground` repo, cloned as a sibling of this one) — read it before changing the build, the release workflow, or the `svelte` version. It covers the rollup pipeline, the release mechanics, the codemod's known gaps, and the mandatory rebuild procedure after a Budibase upgrade.

Only the facts specific to this repo are below.

## This plugin

A timer component: it fires a `trigger` event every N seconds so app builders can attach actions to it (originally, auto-refreshing a data provider). Fork of `MartinPicc/budibase-interval-plugin` (unmaintained since 2023).

- Plugin name / component key: `Interval` → `plugin/Interval`, doc id `plg_Interval`
- Current version: 1.2.1 · branch `main`
- Settings: `interval` (number, seconds, min 1), `display` (boolean), `text` (text, shown only when `display`), `trigger` (event)
- Components: `src/Component.svelte`, `src/lib/Interval.svelte`; no children

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

## Repo-specific notes

- **The `schema.json` ↔ props contract.** Each `schema.settings[].key` arrives as an exported prop on `src/Component.svelte`; adding or renaming a setting requires editing **both** files or the prop silently stays `undefined`. `trigger` is a `type: "event"` setting, which Budibase passes down as a function — hence `on:trigger={trigger}` rather than calling it. Props reach `src/Component.svelte` via `$$restProps` on the wrapper, so they are declared nowhere in `lib/`. Settings can be conditionally shown with `dependsOn` (see `text` hanging off `display`).
- **The interval is not reactive.** `src/lib/Interval.svelte` starts `window.setInterval` at component init when `interval > 0` and clears it in `onDestroy`; changing the `interval` prop at runtime does not restart the timer.
- The stopwatch icon is imported as a raw string via `rollup-plugin-svg` and injected with `{@html}`.
- `schema.json`'s `info` field and the README still point issues at the upstream repo.

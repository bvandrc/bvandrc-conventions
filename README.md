# bvandrc-conventions

Coding conventions shared across my projects, synced into each repo for both
AI agents and humans.

## What's here

| File | Scope |
| --- | --- |
| [`conventions/typescript.md`](conventions/typescript.md) | Language-level TypeScript/JavaScript rules |
| [`conventions/react.md`](conventions/react.md) | Component, JSX, and accessibility rules |
| [`conventions/playwright.md`](conventions/playwright.md) | Test layout, test IDs, accessibility scans |
| [`conventions/git.md`](conventions/git.md) | Branch naming and pull request review practice |
| [`conventions/biome.base.json`](conventions/biome.base.json) | Shared Biome lint and format settings |

`react.md` and `playwright.md` both build on `typescript.md`. `git.md` stands
alone and applies to every repo, whatever the stack. `biome.base.json` is the
executable half of `typescript.md` — sync the two together.

## How consuming repos use these

Each consuming repo commits a **copy** of these files under its own
`conventions/`, kept current by a scheduled GitHub Action that opens a pull
request whenever this repo changes.

A repo syncs only the files it needs — the workflow names them explicitly, so
a TypeScript project with no React syncs `typescript.md` and `git.md` and
skips the rest. Because of that, a file here may reference the ones it builds
on, but never the ones that build on it: `react.md` may point at
`typescript.md`, while `typescript.md` names no framework file, since it
cannot know which of them a given repo has.

The files are copied rather than referenced because agent instruction files
are read at session start, before any dependency is installed — a remote or
web session clones the repo and begins immediately, so anything not committed
is simply absent. Committing them also means convention changes show up in
pull request diffs instead of appearing silently.

**Edit here, never downstream.** A downstream edit is overwritten by the next
sync. Change a rule in this repo and let the sync PR carry it out.

## Setting up a new consumer

The sync logic lives here, in
[`.github/workflows/sync.yml`](.github/workflows/sync.yml), as a reusable
workflow. A consuming repo only declares when to run and which files it wants,
so a fix to the sync itself reaches every project without editing them.

Add `.github/workflows/sync-conventions.yml` to the repo:

```yaml
name: Sync Conventions

on:
  schedule:
    - cron: '0 9 * * 1' # Mondays, 09:00 UTC
  workflow_dispatch: # Allows manual triggering

permissions:
  contents: write
  pull-requests: write

jobs:
  sync:
    uses: bvandrc/bvandrc-conventions/.github/workflows/sync.yml@main
    with:
      files: typescript.md react.md playwright.md git.md biome.base.json
```

Then, in the consuming repo:

1. Set `files` to the subset that repo needs. A backend TypeScript project
   might use `typescript.md git.md`.
2. Import the same files from its `CLAUDE.md`, one `@conventions/<file>` per
   line. The `files` input decides what exists on disk; the imports decide
   what Claude loads, and the two should match.
3. Enable **Settings → Actions → General → Allow GitHub Actions to create and
   approve pull requests**. It is off by default, and without it the run fails
   with `GitHub Actions is not permitted to create or approve pull requests`
   *after* pushing the branch — so the sync looks half-done.
4. Run it once via **workflow_dispatch** to seed `conventions/`.

## Biome config

`biome.base.json` rides the same sync as the markdown: it lands in
`conventions/` like everything else, and the repo's `biome.json` extends it.

```json
{ "extends": ["./conventions/biome.base.json"] }
```

It is distributed by copy rather than as an npm package so there is one
mechanism to understand, and so config changes land in the same sync PR as the
prose they enforce. The tradeoff is that upgrades arrive on the sync schedule
instead of being pinned per repo — if a repo ever needs to hold a version
back, that is the signal to move the config to a package.

**What belongs where.** The base holds settings every repo agrees on:
formatter style, `recommended`, and the rules that make `typescript.md`
enforceable rather than aspirational.

| Convention | Rule |
| --- | --- |
| Named exports, no defaults | `style/noDefaultExport` |
| `import type` for type-only imports | `style/useImportType` |

The repo's own `biome.json` holds everything the shared file cannot know:
`files.includes`, test-file `overrides`, and framework rules. File naming is
deliberately *not* enforced — `typescript.md` wants kebab-case for modules
while `react.md` wants PascalCase components and camelCase hooks, so
`useFilenamingConvention` belongs downstream where the mix is known. Same for
the `noDefaultExport` escape hatch: a repo with lazy-loaded route components
overrides the rule for those paths locally.

**Validating it here.** This repo installs Biome itself and keeps a `biome.json`
that extends the base, so the shim consuming repos write is dogfooded and the
base config is checked before it syncs anywhere:

```bash
pnpm install
pnpm check      # pnpm check:fix to apply
```

An unknown rule, a misspelled key, or a schema the installed Biome no longer
accepts fails the command — the config is validated, not just parsed.

Two things to know:

- **Sync it alongside `typescript.md`.** The sync deletes `conventions/`
  before copying, so a repo that lists `typescript.md` but forgets
  `biome.base.json` gets a `biome.json` pointing at a file that no longer
  exists. The failure is loud, but avoidable.
- **The `$schema` version is pinned** in the base file and tracks the Biome
  version this repo installs. Bump both together — `pnpm check` fails if the
  config drifts past what that version accepts.

### Notes

- **Permissions stay in the caller.** A called workflow cannot widen its own
  permissions, so `contents: write` and `pull-requests: write` have to be
  declared by each consumer.
- **Triggers stay in the caller** too, so cron frequency is a per-repo choice.
- `peter-evans/create-pull-request` force-pushes its branch on every run, so
  don't stack manual commits on an open sync PR — the next run discards them.
- Scheduled workflows are disabled after 60 days of repository inactivity;
  `workflow_dispatch` is the manual recovery.

## Consuming repos

- [bvandrc-project-template-react-frontend](https://github.com/bvandrc/bvandrc-project-template-react-frontend)

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

`react.md` and `playwright.md` both build on `typescript.md`. `git.md` stands
alone and applies to every repo, whatever the stack.

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

Add `.github/workflows/sync-conventions.yml` to the repo. This is the working
copy from
[bvandrc-project-template-react-frontend](https://github.com/bvandrc/bvandrc-project-template-react-frontend/blob/main/.github/workflows/sync-conventions.yml) —
the only line that normally changes per project is `FILES`.

```yaml
name: Sync Conventions

on:
  schedule:
    - cron: '0 9 * * 1' # Mondays, 09:00 UTC
  workflow_dispatch: # Allows manual triggering
  repository_dispatch: # Sent by bvandrc-conventions when it changes
    types: [conventions-updated]

permissions:
  contents: write
  pull-requests: write

concurrency:
  group: ${{ github.workflow }}
  cancel-in-progress: true

env:
  CONVENTIONS_REPO: bvandrc/bvandrc-conventions
  SYNC_MESSAGE: 'chore: sync convention docs from bvandrc-conventions'

jobs:
  sync:
    name: Sync conventions
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repo
        uses: actions/checkout@v7

      - name: Checkout conventions
        uses: actions/checkout@v7
        with:
          repository: ${{ env.CONVENTIONS_REPO }}
          path: .conventions-src

      # Only the files listed here are synced — a project on a different stack
      # syncs a different subset. Add or remove entries to change what this
      # repo tracks.
      - name: Copy conventions into place
        env:
          FILES: typescript.md react.md playwright.md git.md
        run: |
          rm -rf conventions
          mkdir -p conventions
          for file in $FILES; do
            cp ".conventions-src/conventions/$file" "conventions/$file"
          done
          rm -rf .conventions-src

      - name: Open pull request on drift
        uses: peter-evans/create-pull-request@v7
        with:
          branch: chore/sync-convention-docs
          delete-branch: true
          commit-message: ${{ env.SYNC_MESSAGE }}
          title: ${{ env.SYNC_MESSAGE }}
          body: |
            Automated sync from https://github.com/${{ env.CONVENTIONS_REPO }}

            Edit conventions upstream — this directory is overwritten on every
            sync, so changes made here are lost.
```

Then, in the consuming repo:

1. Set `FILES` to the subset that repo needs. A backend TypeScript project
   might use `typescript.md git.md`.
2. Import the same files from its `CLAUDE.md`, one `@conventions/<file>` per
   line. The `FILES` list decides what exists on disk; the imports decide what
   Claude loads, and the two should match.
3. Enable **Settings → Actions → General → Allow GitHub Actions to create and
   approve pull requests**. It is off by default, and without it the last step
   fails with `GitHub Actions is not permitted to create or approve pull
   requests` *after* pushing the branch — so the sync looks half-done.
4. Run it once via **workflow_dispatch** to seed `conventions/`.

### Notes on the workflow

- `rm -rf conventions` before copying is deliberate: it means a file dropped
  from `FILES`, or retired from this repo, disappears downstream instead of
  lingering forever.
- `.conventions-src` is deleted before the last step, or it would be committed
  along with the sync.
- `peter-evans/create-pull-request` force-pushes its branch on every run, so
  don't stack manual commits on an open sync PR — the next run discards them.
- Scheduled workflows are disabled after 60 days of repository inactivity;
  `workflow_dispatch` is the manual recovery.
- The `repository_dispatch` trigger is inert until something sends the event.
  Doing so needs a workflow here holding a PAT with write access to each
  consumer, which does not exist yet.

## Consuming repos

- [bvandrc-project-template-react-frontend](https://github.com/bvandrc/bvandrc-project-template-react-frontend)

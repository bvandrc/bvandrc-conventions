# bvandrc-conventions

Coding conventions shared across my projects, synced into each repo for both AI agents and humans.

## What's here

| File | Scope |
| --- | --- |
| [`conventions/typescript.md`](conventions/typescript.md) | Language-level TypeScript/JavaScript rules |
| [`conventions/react.md`](conventions/react.md) | Component, JSX, and accessibility rules |
| [`conventions/playwright.md`](conventions/playwright.md) | Test layout, test IDs, accessibility scans |
| [`conventions/ts-unit-testing.md`](conventions/ts-unit-testing.md) | TypeScript unit test layout, naming, fixtures, assertions |
| [`conventions/all.md`](conventions/all.md) | Practice for every repo: branches, formatting, comments, testing, pull request reviews |
| [`conventions/biome.base.json`](conventions/biome.base.json) | Shared Biome lint and format settings |

`react.md`, `playwright.md`, and `ts-unit-testing.md` all build on `typescript.md`. `all.md` stands alone and applies to every repo, whatever the stack. `biome.base.json` is the executable half of `typescript.md` — sync the two together.

## How consuming repos use these

Each consuming repo commits a **copy** of these files under its own `conventions/`, kept current by a scheduled GitHub Action that opens a pull request whenever this repo changes.

Where a sync lands depends on the branch it runs from. On the default branch — every scheduled run, and a manual run left on `main` — it opens a pull request. Dispatch it manually from any other branch and it commits straight to that branch instead, so work already in flight can pull in the current conventions without a second pull request to merge.

A repo syncs only the files it needs — the workflow names them explicitly, so a TypeScript project with no React syncs `typescript.md` and `all.md` and skips the rest. Because of that, a file here may reference the ones it builds on, but never the ones that build on it: `react.md` may point at `typescript.md`, while `typescript.md` names no framework file, since it cannot know which of them a given repo has.

The files are copied rather than referenced because agent instruction files are read at session start, before any dependency is installed — a remote or web session clones the repo and begins immediately, so anything not committed is simply absent. Committing them also means convention changes show up in pull request diffs instead of appearing silently.

**Edit here, never downstream.** A downstream edit is overwritten by the next sync. Change a rule in this repo and let the sync PR carry it out.

## Setting up a new consumer

The sync logic lives here, in [`.github/workflows/sync.yml`](.github/workflows/sync.yml), as a reusable workflow. A consuming repo only declares when to run and which files it wants, so a fix to the sync itself reaches every project without editing them.

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
  id-token: write # For apply-with-claude

jobs:
  sync:
    uses: bvandrc/bvandrc-conventions/.github/workflows/sync.yml@main
    with:
      files: typescript.md react.md playwright.md ts-unit-testing.md all.md biome.base.json
      apply-with-claude: true
    secrets: inherit
```

Then, in the consuming repo:

1. Set `files` to the subset that repo needs. A backend TypeScript project might use `typescript.md all.md`.
2. Import the same files from its `CLAUDE.md`, one `@conventions/<file>` per line. The `files` input decides what exists on disk; the imports decide what Claude loads, and the two should match.
3. Enable **Settings → Actions → General → Allow GitHub Actions to create and approve pull requests**. It is off by default, and without it the run fails with `GitHub Actions is not permitted to create or approve pull requests` *after* pushing the branch — so the sync looks half-done.
4. Set up [Claude](#applying-conventions-with-claude), or drop `apply-with-claude`, `id-token`, and `secrets` from the file above to only sync.
5. Run it once via **workflow_dispatch** to seed `conventions/`.

For a repo syncing every markdown file, the `CLAUDE.md` import looks like this:

```markdown
## Code conventions

Conventions live outside this file, synced from https://github.com/bvandrc/bvandrc-conventions — follow all of them:

- @conventions/typescript.md — language-level TypeScript/JavaScript rules
- @conventions/react.md — component, JSX, and accessibility rules
- @conventions/playwright.md — test layout, test IDs, and accessibility scans
- @conventions/ts-unit-testing.md — TypeScript unit test layout, naming, fixtures, and assertions
- @conventions/all.md — practice for every repo: branches, formatting, comments, testing, markdown, PR reviews
```

## Biome config

`biome.base.json` rides the same sync as the markdown: it lands in `conventions/` like everything else, and the repo's `biome.json` extends it.

```json
{ "extends": ["./conventions/biome.base.json"] }
```

It is distributed by copy rather than as an npm package to match the way convention `.md` files are synced.

**What belongs where.**

- **The base** holds:
  - Formatter style, including `indentWidth` and `lineWidth`, stated explicitly so they survive a change to Biome's defaults.
  - The rules it raises from warning to error, including `style/useImportType` for the `import type` rule in `typescript.md`.
- **The repo's own `biome.json`** holds everything the shared file cannot know: `files.includes`, test-file `overrides`, and framework rules.

The base deliberately stops short of enforcing every rule the conventions state, as the specifics may vary per repo. `noDefaultExport` is absent entirely, and `useFilenamingConvention` runs at Biome's defaults — which accept camelCase, kebab-case, snake_case, or a name matching an export — so it permits both a kebab-case util and a PascalCase component without requiring either where `typescript.md` and `react.md` ask for them. A repo that wants the stricter rules configures them itself.

**Validating it here.**

```bash
pnpm install
pnpm check      # pnpm check:fix to apply
```

**Sync it alongside `typescript.md`.** The sync deletes `conventions/` before copying, so a repo that lists `typescript.md` but forgets `biome.base.json` gets a `biome.json` pointing at a file that no longer exists. The failure is loud, but avoidable.

### Notes

- **Permissions stay in the caller.** A called workflow cannot widen its own permissions, so `contents: write` and `pull-requests: write` have to be declared by each consumer.
- **Triggers stay in the caller** too, so cron frequency is a per-repo choice.
- `peter-evans/create-pull-request` force-pushes its branch on every run, so don't stack manual commits on an open sync PR — the next run discards them. This only applies to the pull request path; a dispatch from a non-default branch commits to that branch and force-pushes nothing.
- **The default branch is read from the event payload**, not hardcoded, so a repo whose default is not `main` behaves the same. The run fails rather than guessing if the payload has no `default_branch`.
- Scheduled workflows are disabled after 60 days of repository inactivity; `workflow_dispatch` is the manual recovery.

## Applying conventions with Claude

A sync PR only copies the rules. With `apply-with-claude: true`, a second job then has Claude bring the code into line with them, on the same pull request. It runs whenever the sync opens or updates the PR:

1. Checks out the sync commit and installs dependencies — through the repo's `.github/actions/setup` if it has one, or pnpm when there is a `pnpm-lock.yaml`.
2. Runs Claude, which commits `refactor: apply updated conventions` for the new or changed rules, then `refactor: fix existing convention drift` for anything else out of line, running the repo's format and check scripts before each, and pushes to the PR branch.
3. Comments on the PR with Claude's summary of what changed under which rule, and what it left alone.

### Setup, per consuming repo

1. Install the [Claude GitHub App](https://github.com/apps/claude) on the repo. Installing it for all repositories on the account covers new repos too. Claude pushes with the app's token, which is scoped to the one repo the run is in.
2. Add a repository secret under **Settings → Secrets and variables → Actions**: `ANTHROPIC_API_KEY` from console.anthropic.com, or `CLAUDE_CODE_OAUTH_TOKEN` from `claude setup-token` to bill a Pro/Max subscription.
3. In `sync-conventions.yml`, grant `id-token: write`, set `apply-with-claude: true`, and add `secrets: inherit`, as in the example above.

The key is stored in each repo rather than once here on purpose. Running Claude centrally would need a token here that can push to every consuming repo; per-repo, each run can only touch its own repo.

### Notes

- **The job inherits the caller's permissions.** It declares none itself, because asking for `id-token: write` would fail validation for callers that haven't granted it, whether they opted in or not.
- **CI runs on the PR once Claude pushes.** The sync's own push uses `GITHUB_TOKEN`, which starts no workflows; a push with the app's token does. If Claude finds nothing to change, CI doesn't run on the sync PR.
- **A newer sync discards Claude's commits** along with the rest of the branch, then Claude runs again against the new sync commit.
- **Only the pull request path runs Claude.** A dispatch from a non-default branch commits the sync to that branch and stops there.

## Consuming repos

- [bvandrc-project-template-react-frontend](https://github.com/bvandrc/bvandrc-project-template-react-frontend)

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

The files are copied rather than referenced because agent instruction files
are read at session start, before any dependency is installed — a remote or
web session clones the repo and begins immediately, so anything not committed
is simply absent. Committing them also means convention changes show up in
pull request diffs instead of appearing silently.

**Edit here, never downstream.** A downstream edit is overwritten by the next
sync. Change a rule in this repo and let the sync PR carry it out.

## Consuming repos

- [bvandrc-project-template-react-frontend](https://github.com/bvandrc/bvandrc-project-template-react-frontend)

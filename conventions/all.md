# General conventions

Applies to every repo, independent of language or framework.

## Git and pull requests

- **Branches**
  - **Naming**: Name work branches `<type>/<slug>`, where `<type>` is a Conventional Commits type — `feat`, `fix`, `chore`, `refactor`, `ci`, `docs`, `style`, `test`, `perf`, or `build` — and `<slug>` is a short kebab-case description of the change. Never use a `claude/` prefix or a random session suffix.
  - **Renaming an assigned branch**: The naming rule above overrides the branch name a session is assigned by default — if you were given one, rename it before the first push.
- **Commit messages and PR titles**: [Conventional Commits](https://www.conventionalcommits.org) — `<type>(<scope>): <summary>`, using the branch types above.
  - **Summary**: lowercase, imperative, and about what changes for whoever uses the code, not how (`fix: keep saved filters when another device syncs`).
  - **Scope**: optional, lowercase, and names the area the change is in (`test(e2e)`, `fix(sync-outbox)`). Leave it off rather than invent one.
  - **Breaking changes**: mark them with `!` before the colon (`feat!: …`), in a repo that versions a public API.
  - **Squash merges**: a PR lands as one commit whose message is the PR title plus ` (#<number>)`, so the title is the commit message and follows the same form.
  - **Squashing several PRs together**: only on command, the commits from several PRs are force-squashed into one. Its message takes the same form, with one summary covering them all and every PR number listed at the end, oldest first (`chore: update dependencies (#534, #538, #539)`).
- **PR review threads**: Always reply on the thread with what changed (or why it wasn't changed), then mark the thread resolved. Do this for every thread you act on, not just the ones that needed discussion.
- **Labels**: Every PR and issue carries a _type_ label, and may carry action and other labels beside it. More than one type label is fine where a change really is more than one kind, like a fix that also adds tests.
  - **Type labels**: one per Conventional Commits type, and a PR's labels include the one for its title's type. A label takes its type's name (`fix`, `docs`, `chore`, ...), spelled out where the prefix abbreviates it (`feat` is `feature`, `perf` is `performance`).
    - **`bug` and `fix`**: `bug` is for issues, which report the problem, and `fix` is for PRs, which solve it. A PR that fixes a bug is `fix`, never `bug`.
  - **Issues**: titled the way the PR expected to close them will be (`perf: …`, `feat: …`), except a bug, which is `bug: …`, and labeled to match: `bug` for a bug, and otherwise that PR's type label. `question` goes on beside it while the issue is still waiting on an answer before any work can start, and comes off once it has one.
  - **Action labels**: a label a workflow reads, like `run ci`, `preview`, or `refresh-data`, is named for what adding it does, and its GitHub description says so. A repo that has any documents them in its own `CLAUDE.md`, since what each does is that repo's, in a table of the label, what adding it does, and the workflow that reads it.
  - **Other labels**: labels like `a11y`, `question`, `help wanted`, `duplicate`, and `wontfix` go on beside the type label, not in place of it, and have no title prefix of their own, except `a11y`, which matches the `a11y` scope (`fix(a11y): …`). Close an issue labeled `duplicate` or `wontfix` with GitHub's matching reason too, _duplicate_ or _not planned_.
  - **GitHub's defaults**: every new repo starts with GitHub's default labels. Bring them in line with the table below, where renaming one carries it across every issue and PR that has it.

| GitHub default | Becomes |
| --- | --- |
| `enhancement` | `feature` |
| `documentation` | `docs` |
| `accessibility` | `a11y` |
| `bug` | kept, for issues only; a PR labeled `bug` is relabeled `fix` |
| `invalid`, `good first issue` | deleted |
| `question`, `help wanted`, `duplicate`, `wontfix` | kept |

A repo that already has a `perf` label renames it to `performance`.

## Tooling

- **Formatting**: Run the repo's format script after making edits and before every commit. Each language's conventions say what it is.

## Testing

- **Testing internals**: Never widen a module's API for a test. Where something is only public so a test file can reach it, keep it internal and drive it through the public thing that uses it — the test then pins behaviour someone can actually depend on, and survives the internal being renamed or absorbed. A repo's test-support module is the exception: being imported by tests is its whole job.
- **Checking a new test can fail**: A test that passes against broken source pins nothing. Before keeping a new case, break the behaviour it names and confirm that case — and ideally only that case — goes red.
- **Trivial cases**: Don't write a case whose whole subject is one plain line — a one-line function or wrapper, a guard, a ternary, a fallback, a prop spread, or a single attribute or class. It restates the line rather than pinning behaviour anyone could get wrong, and is one more case to read and keep in step. Where that line feeds something larger, the case for the larger thing covers it. A single assertion still earns its place where it checks that two sources stay in step, like a theme's tokens against the CSS they mirror, since that pins no one line at all.

## Comments

- **Where a comment goes**: A doc comment (i.e., above a function) is for _callers_ — what the thing is/does, what it gives back, what it costs them, etc. Any note about _how_ it does that, or explaining a cast, a workaround, or a library's behaviour, etc., goes in a line comment _within_ the function _at_ the code it describes, since it doesn't need to be known to the caller. This goes for any function, internal to a file or exported.
- **Doc comment shape**: A doc comment that runs to more than one line opens with its summary alone — what the thing is, usually one sentence — then a blank line before any specifics: conditions, defaults, the reason it's built the way it is. A comment that is only a summary, however many lines it wraps to, needs no gap.

## Writing

- **Oxford comma**: A list of three or more takes a comma before its final `and` or `or` — in prose, comments, documentation, and commit messages alike.

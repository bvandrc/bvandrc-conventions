# Playwright conventions

Builds on the language-level rules in `./typescript.md` — follow those too.

- **Layout**: All Playwright tests live in `playwright/`, split by project: `playwright/e2e/`, `playwright/a11y/`, `playwright/lighthouse/`, with shared helpers under `playwright/support/`. Type checking uses `playwright/tsconfig.json`, separate from the app's.
- **Selectors**
  - **Test ID registry**: Define every `data-testid` value in one registry before using it in a test. Where the repo has unit tests as well, that registry is the shared test-support module both kinds of test import (`shared/test-support/selectors.ts`), not one under `playwright/` — a component test and an end-to-end test reaching for the same element must not be naming it twice, and a testid that moves has to break both at once. Playwright on its own keeps it at `playwright/support/constants/selectors.ts`.
    - Nest by component.
    - Build strings with the `testId()` helper — never a hand-written `[data-testid="..."]`.
    - Name a container's own testid `SELF`.
  - **Locators**: Reach for `data-testid` first. Text, role, and class-based locators break on copy edits, markup changes, and styling churn, so use them only where a testid isn't an option — third-party markup you don't control, or an assertion whose whole point is the visible text or the accessible role.
- **Accessibility tests**: axe runs at WCAG 2.1 AA plus best-practice on desktop and mobile, and violations fail CI. Cover each new meaningful UI state with a `checkA11y(page)` scan in `playwright/a11y/`.

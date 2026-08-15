# TypeScript / JavaScript conventions

Language-level rules, independent of framework or project. Framework-specific
conventions live in their own files alongside this one and build on it.

- **File naming**: kebab-case for utils and plain modules (`auth-utils.ts`).
- **Modules**
  - **Exports**: Named exports. No default exports, unless something requires
    one.
  - **Type-only imports**: Use `import type` for imports only used as types.
- **Constant objects and maps**
  - **Casing**:
    - UPPER_CASE for the constant's name.
    - UPPER_CASE for keys that name entries (namespace/enum-style, e.g.
      `ROUTES.HOME`, `SELECTORS.TASK_FORM.SUBMIT_BTN`).
    - camelCase for keys that are typed properties of an entry (e.g. `color`,
      `icon` in `FEATURES`).
    - camelCase for function-valued keys (e.g.
      `SELECTORS.TASK_CARD.rankFieldBadge(field)`).
  - **Exhaustive maps**: When a value is keyed by a union, declare the map at
    module level with `satisfies Record<Variant, string>` so a missing key is
    a type error.
- **Comments/JSDoc**:
  - Describe *what* and *why* from the caller's perspective.
  - Don't restate implementation.
  - Don't repeat what the type signature conveys.
  - Keep to 1–2 lines.
  - No hedge prefixes.
- **Tooling**
  - **Linting and formatting**: Biome is the linter *and* formatter — no
    eslint/prettier here. Shared rules live in `./biome.base.json`, which the
    repo's own `biome.json` extends; keep repo-specific settings in that
    `biome.json` rather than editing the synced base.
  - **es-toolkit**: Use `es-toolkit` functions when simpler than the builtin
    equivalents — especially `omit`/`pick`.

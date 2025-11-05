# Repository Guidelines

## Project Structure & Module Organization
This repo is a Yarn workspaces monorepo. All runtime code lives under `packages/*`; each package keeps its implementation in `src/` and colocated tests in `__tests__/`. Shared build tooling and scripts sit under `scripts/`, compiler experiments live in `compiler/`, manual verification fixtures are in `fixtures/`, and prose docs belong in `docs/`. Use `flow-typed/` for shared library stubs and keep asset snapshots with the package that owns them.

## Build, Test, and Development Commands
- `yarn install` boots the workspace and links cross-package dependencies; rerun when adding packages.
- `yarn build` runs the Rollup pipeline across release channels; prefer this before publishing artifacts.
- `yarn test` invokes the custom Jest runner; append `--runTestsByPath packages/react/src/__tests__/ReactHooks-test.js` to scope runs.
- `yarn test-stable` and `yarn test-www` validate stable and www channel builds.
- `yarn lint`, `yarn flow`, and `yarn prettier` enforce lint rules, Flow typing, and formatting; use `yarn prettier-all` for large refactors.

## Coding Style & Naming Conventions
Formatting is fixed by Prettier (2-space indentation, semicolons, double quotes); do not hand-tune whitespace. Follow ESLint warnings emitted by `scripts/eslint-rules`. Name React components with PascalCase, hooks with `use*`, and internal modules with dash-separated package names (e.g. `react-server-dom-webpack`). Flow files carry `.js` with `@flow`; TypeScript utilities live in `.ts` or `.tsx`.

## Testing Guidelines
Write Jest tests alongside the code in `__tests__/` using the `*-test.(js|ts)` naming convention. Snapshot tests should live with their owning package to keep release builds reproducible. Validate type coverage with `yarn flow` (or `yarn flow-ci` on CI) and run `yarn lint-build` when altering Rollup config or bundle outputs. Add DOM fixture cases under `fixtures/` only when manual QA is unavoidable.

## Commit & Pull Request Guidelines
Commits follow the existing history: `[Scope] Imperative summary (#issue)` (e.g. `[eslint] Fix useEffectEvent checks`). Keep messages concise and describe both problem and fix. Every pull request should include context, a test plan (commands run), affected release channels, and links to issues or RFCs. Attach screenshots for DevTools or timeline changes, and ensure CI passes before requesting review.

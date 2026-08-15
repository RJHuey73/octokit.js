# CLAUDE.md

Guidance for Claude Code (claude.ai/code) when working in this repository.

## What This Is

**octokit.js** (npm package `octokit`) is the all-batteries-included GitHub
SDK for browsers, Node.js, and Deno. It is a thin composition layer over the
Octokit ecosystem, not a monorepo of its own packages: `package.json` pulls
in `@octokit/core`, `@octokit/app`, `@octokit/oauth-app`,
`@octokit/plugin-paginate-rest`, `@octokit/plugin-paginate-graphql`,
`@octokit/plugin-rest-endpoint-methods`, `@octokit/plugin-retry`,
`@octokit/plugin-throttling`, `@octokit/request-error`, `@octokit/types`,
and `@octokit/webhooks` as regular npm dependencies, and this repo's own
code (`src/`) just wires them together with sane defaults. This repo is
`RJHuey73/octokit.js`, a fork of `octokit/octokit.js`.

## Layout

The entire implementation is four small files:

| Path | Purpose |
|------|---------|
| `src/index.ts` | Public entry point — re-exports `Octokit`, `RequestError`, `App`, `OAuthApp`, `createNodeMiddleware`, and the `PageInfoForward`/`PageInfoBackward` types. |
| `src/octokit.ts` | Builds `Octokit` by composing `@octokit/core` with the `restEndpointMethods`, `paginateRest`, `paginateGraphQL`, `retry`, and `throttling` plugins via `.plugin(...).defaults(...)`. Sets the default `userAgent` (`octokit.js/${VERSION}`) and default `onRateLimit`/`onSecondaryRateLimit` throttle handlers (retry once, then give up). |
| `src/app.ts` | Builds `App` and `OAuthApp` by taking `@octokit/app`'s and `@octokit/oauth-app`'s default exports and calling `.defaults({ Octokit })` so they use this repo's configured `Octokit`. Re-exports `createNodeMiddleware`. |
| `src/version.ts` | Single `export const VERSION` string, rewritten by `semantic-release-plugin-update-version-in-files` at release time — don't hand-edit for a feature change. |
| `scripts/build.mjs` | Release build: cleans `pkg/`, esbuilds `src/**/*.ts` unbundled to `pkg/dist-src`, esbuilds a bundled ESM entry to `pkg/dist-bundle`, copies `LICENSE`/`README.md`, and writes a trimmed `pkg/package.json` (strips `scripts`/`prettier`/`release`/`jest`, adds `exports`/`types`/`sideEffects: false`). `tsc -p tsconfig.json` (run after this script by `npm run build`) separately emits declarations to `pkg/dist-types`. |
| `test/smoke.test.ts` | Minimal "does it construct and export the right shape" checks for `Octokit`, `App`, `OAuthApp`, `RequestError`. |
| `test/app.test.ts` | Behavioral tests for `App` against the README's own examples (e.g. `app.eachRepository.iterator`), using `nock` to mock the GitHub API and `mockdate` to freeze time for JWT generation. |
| `test/typescript-validate.ts` | Exercised only via `npm run test:typescript` — a hand-written file of TypeScript usage patterns that must typecheck under strict compiler options; not run by vitest. |
| `README.md` | The primary user-facing doc — usage examples for the `Octokit`, `App`, and `OAuthApp` clients. `test/app.test.ts` tests against examples taken directly from it, so keep them in sync if you change either. |
| `MAINTAINING.md` | Release process (semantic-release, conventional commits, the `beta` branch flow for breaking changes) and the cross-repo merge order for coordinated changes across the Octokit ecosystem. |

There is no `src/action.ts` yet — the README's table of contents lists an
"Action client" as one of octokit.js's three integrated libraries, but it is
not implemented in this repo's `src/`. Don't assume it exists; check `src/`
directly rather than trusting the README's feature list.

## Commands

```bash
npm install                # install dependencies
npm test                   # vitest run --coverage (100% coverage threshold, see vite.config.js)
npm run test:typescript    # strict standalone tsc check of test/typescript-validate.ts
npm run lint                # prettier --check over src, test, scripts, and top-level md/json
npm run lint:fix            # prettier --write, same scope
npm run build                # scripts/build.mjs (produces pkg/) then tsc -p tsconfig.json (emits pkg/dist-types)
```

Run a single test file with `npx vitest run test/smoke.test.ts`, or a single
test by name with `npx vitest run -t "test name"`.

`npm test` runs `pretest` first, which is `npm run -s lint` — so a failing
Prettier check blocks `npm test` even if the tests themselves would pass.

CI (`.github/workflows/test.yml`) runs on Node 20/22/24 via `npm test`, then
separately runs `npm run test:typescript`, `npm run lint`, and `npm run
build` once on Node LTS. Match that locally before pushing: lint + test +
test:typescript + build all need to pass.

## Conventions

- **This package is a composition, not an implementation.** Almost all real
  logic (REST/GraphQL request handling, auth strategies, retry/throttle
  behavior, App/OAuthApp/webhook logic) lives in the upstream `@octokit/*`
  packages listed in `dependencies`. Changes here are typically about how
  those pieces are wired together and defaulted (see `src/octokit.ts`,
  `src/app.ts`), not about reimplementing API client behavior — if a bug
  looks like it belongs in `@octokit/core`/`@octokit/app`/etc., it likely
  needs to be fixed upstream in that package's own repo instead.
- **`Octokit`/`App`/`OAuthApp` are exported as both values and types.** Each
  is `X.plugin(...)`/`X.defaults(...)` at the value level, with
  `export type X = InstanceType<typeof X>;` alongside it — preserve this
  pattern (see `src/octokit.ts`, `src/app.ts`) rather than introducing a
  separate interface if you extend these.
- **`test/app.test.ts` is README-driven.** Its test names literally reference
  README section examples (e.g. `` Readme example: `app.eachRepository.iterator` ``).
  If you change one, check whether the other needs a matching update.
- **Conventional commits drive releases.** `semantic-release` (configured in
  `package.json`'s `release` block, run via `.github/workflows/release.yml`)
  parses commit subjects: `fix: ...` → patch, `feat: ...` → minor,
  `BREAKING CHANGE:` in the body → major. `MAINTAINING.md` requires breaking
  changes go through a short-lived `beta` branch off `main`, merged back only
  after further testing — don't land a `BREAKING CHANGE:` commit straight to
  `main`.
- **ESM-only, Node >= 20.** `package.json` sets `"type": "module"`; there is
  no CommonJS build. Consumers on older TypeScript setups need
  `"moduleResolution": "node16", "module": "node16"` because this package
  uses conditional `exports` (documented in the README's usage section) —
  keep that guidance in sync with `scripts/build.mjs`'s generated `exports`
  map if it changes.
- **Formatting is Prettier, not ESLint.** There is no ESLint config in this
  repo; `npm run lint` / `lint:fix` are pure `prettier --check`/`--write`
  over `src`, `test`, `scripts`, and top-level `.md`/`.json` files.

## Testing

- Vitest (`vite.config.js`) with `@vitest/coverage-v8`, coverage scoped to
  `src/**/*.ts`, and a **100% coverage threshold** — new code in `src/`
  needs tests or an explicit `/* v8 ignore next ... -- @preserve */` comment
  (see the two throttle-handler functions in `src/octokit.ts` for the
  established pattern: internals of a third-party plugin that aren't worth
  asserting on directly).
- Network calls in tests are mocked with `nock`, not real HTTP — see
  `test/app.test.ts`'s `nock("https://api.github.com")` setup with
  `nock.cleanAll()` in `beforeEach`.
- `MockDate` freezes `Date` in `test/app.test.ts` because JWT generation for
  GitHub App auth is time-sensitive (`iat`/`exp` claims) and the test asserts
  against a hardcoded expected JWT string.
- `test/typescript-validate.ts` is **not** part of the vitest suite — it's a
  separate strict-mode type-only check (`npm run test:typescript`) asserting
  that certain usage patterns compile under
  `--noUnusedLocals --esModuleInterop --module node16 --strict
  --allowImportingTsExtensions --exactOptionalPropertyTypes`.

## Gotchas

- **`src/version.ts`'s `VERSION` is always `"0.0.0-development"` in the repo**
  — it's rewritten to the real release version only during the
  `semantic-release` publish step (`semantic-release-plugin-update-version-in-files`,
  configured in `package.json`). Don't hand-edit it to "fix" the version
  string; it's not meant to reflect the current released version in-tree.
- **`pkg/` is a generated release artifact, not source.** `scripts/build.mjs`
  deletes and regenerates it wholesale (`rm("pkg", { recursive: true,
  force: true })`) and it isn't committed — don't edit anything under `pkg/`
  by hand.
- **The published package's `exports` map, `types` path, and `sideEffects`
  flag are synthesized by `scripts/build.mjs`**, not hand-written in the
  root `package.json` — if you need to change the public export shape,
  change it there (and in `src/index.ts`'s actual exports), not by editing
  `pkg/package.json` after the fact.
- **This is a fork.** `package.json`'s `repository` field still points at
  `octokit/octokit.js` (upstream); be mindful that README/CONTRIBUTING.md
  wording (e.g. "fork the repository", issue links) describes the upstream
  project's contribution flow, not anything specific to this fork.

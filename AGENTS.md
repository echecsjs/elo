# AGENTS.md

Agent guidance for the `@echecs/elo` repository — a TypeScript library
implementing the ELO Rating System following FIDE rules.

**See also:** [`REFERENCES.md`](REFERENCES.md) |
[`COMPARISON.md`](COMPARISON.md) | [`SPEC.md`](SPEC.md)

**Backlog:** tracked in [GitHub Issues](https://github.com/mormubis/elo/issues).

---

## Project Overview

Pure calculation library, no runtime dependencies. Exports six functions
(`delta`, `expected`, `initial`, `kFactor`, `performance`, `update`) and six
types (`GameOptions`, `GameType`, `KFactorOptions`, `PlayerOptions`, `Result`,
`ResultAndOpponent`). All source lives in `src/index.ts`; tests in
`src/__tests__/index.spec.ts`.

---

## Commands

### Build

```bash
pnpm run build          # bundle TypeScript → dist/ via tsdown
```

### Test

```bash
pnpm run test                          # run all tests once
pnpm run test:watch                    # watch mode
pnpm run test:coverage                 # with coverage report

# Run a single test file
pnpm run test src/__tests__/index.spec.ts

# Run a single test by name (substring match)
pnpm run test -- --reporter=verbose -t "kFactor"
```

### Lint & Format

```bash
pnpm run lint           # ESLint + tsc type-check (auto-fixes style issues)
pnpm run lint:ci        # strict — zero warnings allowed, no auto-fix
pnpm run lint:style     # ESLint only (auto-fixes)
pnpm run lint:types     # tsc --noEmit type-check only
pnpm run format         # Prettier (writes changes)
pnpm run format:ci      # Prettier check only (no writes)
```

### Full pre-PR check

```bash
pnpm lint && pnpm test && pnpm build
```

---

## Validation

Input validation is mostly provided by TypeScript's strict type system at
compile time. There is no runtime validation library — the type signatures
enforce correct usage. Do not add runtime type-checking guards (e.g. `typeof`
checks, assertion functions) unless there is an explicit trust boundary.

---

## Architecture Notes

- `expected(a, b)` — win probability for player A; clamps rating diff to ±400
  (FIDE §8.3.1).
- `kFactor(options)` — returns `10 | 20 | 40`. Decision order:
  1. `gameType === 'blitz' || 'rapid'` → 20
  2. `gamesPlayed <= 30` or `(age < 18 && rating < 2300)` → 40
  3. `rating >= 2400 || everHigher2400` → 10
  4. Otherwise → 20
- `delta(actual, expected, k)` — `k * (actual − expected)`.
- `update(a, b, resultOrOptions)` — accepts bare numbers or `PlayerOptions`
  objects for `a`/`b`, and a bare `Result` (0 | 0.5 | 1) or `GameOptions` object
  as the third argument. Results are rounded to the nearest integer.
- `performance(games)` — FIDE §8.2.3 performance rating via `DP_TABLE` lookup;
  throws `RangeError` for an empty array or out-of-range result values.
- The library has **no runtime dependencies**; keep it that way.
- **ESM-only** — the package ships only ESM. Do not add a CJS build.

---

## Release Protocol

Step-by-step process for releasing a new version. CI auto-publishes to npm when
`version` in `package.json` changes on `main`.

1. **Verify the package is clean:**

   ```bash
   pnpm lint && pnpm test && pnpm build
   ```

   Do not proceed if any step fails.

2. **Decide the semver level:**
   - `patch` — bug fixes, internal refactors with no API change
   - `minor` — new features, new exports, non-breaking additions
   - `major` — breaking changes to the public API

3. **Update `CHANGELOG.md`** following
   [Keep a Changelog](https://keepachangelog.com) format:

   ```markdown
   ## [x.y.z] - YYYY-MM-DD

   ### Added

   - …

   ### Changed

   - …

   ### Fixed

   - …

   ### Removed

   - …
   ```

   Include only sections that apply. Use past tense.

4. **Update `README.md`** if the release introduces new public API, changes
   usage examples, or deprecates/removes existing features.

5. **Bump the version:**

   ```bash
   npm version <major|minor|patch> --no-git-tag-version
   ```

6. **Commit and push:**

   ```bash
   git add package.json CHANGELOG.md README.md
   git commit -m "release: @echecs/elo@x.y.z"
   git push
   ```

   **The push is mandatory.** The release workflow only triggers on push to
   `main`. A commit without a push means the release never happens.

7. **CI takes over:** GitHub Actions detects the version bump, runs format →
   lint → test, and publishes to npm.

Do not manually publish with `npm publish`.

# AGENTS.md — ts-operator-overload

Typed operator overloading for TypeScript using `// @operator...` pragma comments placed above methods of classes, interfaces and types.
A compile-time AST transformer + tsserver plugin that rewrites operator expressions (`a + b`, `~x`, `x += y`, `++x`) into annotated method calls (`a.add(b)`), with type-aware resolution.

## Project Facts

- **Runtime**: Node.js, CommonJS (`.cjs` files). There is no build or bundling step for `src/` — the files ship as published.
- **Only dependency**: `typescript` (^5.6.3, a devDependency, used as a library). No runtime deps.
- **Package manager**: use `pnpm` for root installs. Note that `scripts/run-tests.cjs` internally shells out to `npm --prefix test/<case> install` for the integration cases (eslint/vite/nextjs), so `npm` must also be on PATH when running the full suite.
- **Style**: English, 4-space indent — mirror the existing files.

## Commands

```bash
pnpm install              # install devDeps (typescript)
pnpm test                 # full unified suite (node scripts/run-tests.cjs)
node scripts/build-tsc.cjs --project <tsconfig> --outDir <dir>   # compile with the transformer applied (custom tsc driver)
pnpm run build:examples   # build test/tsc-operators-all via the custom driver
```

- `pnpm test` runs, in order: `api-export-shape`, `eslint:install`/`eslint:lint`, `vite:install`/`vite:build`, `nextjs:install`/`nextjs:build` (npm-based integration builds), then `tsc-all`, `tsserver-smoke`, `tsserver-typed`.
- For a quick iteration loop on transformer logic, exercise a single case directly: `node scripts/build-tsc.cjs --project test/<case>/tsconfig.json --outDir /tmp/out`, then diff the emitted JS against `test/<case>/expected.txt`.
- There is no lint or format config for the repo itself; do not invent one.

## Repository Layout

```
index.cjs            # public entry: re-exports src/index.cjs
index.d.ts           # public TypeScript typings
src/
  index.cjs          # exports createOperatorOverloadTransformer, createTscTransformer, tsserverPlugin, shouldSuppressOperatorDiagnostic
  core/
    operator-map.cjs           # SyntaxKind -> operator text maps (binary, unary, compound-assign, ++/--)
    type-utils.cjs             # type-driven resolution of annotated operator methods
    create-transformer.cjs     # the actual ts.TransformerFactory (AST rewriting)
    annotation-diagnostics.cjs # custom diagnostics (code 93001) for invalid @operator annotations
    diagnostic-utils.cjs       # suppresses native TS errors (codes 2356, 2362, 2363, 2365, 2367) where an overload applies
  adapters/
    tsc/index.cjs              # createTscTransformer — thin alias over the core transformer
  plugins/
    tsserver/
      index.cjs                    # tsserver plugin proxy (modes: suppress | shadow | hybrid)
      shadow-language-service.cjs  # shadow program: rewrites files, position mappers, serves diagnostics/quickInfo/completions/inlayHints
      hybrid-coordinator.cjs       # debounced shadow warmup with cooldown/failure budget
scripts/
  build-tsc.cjs        # standalone tsc-like CLI driver applying the transformer + annotation diagnostics
  run-tests.cjs        # unified test runner (all suites)
test/                  # each case folder has tsconfig.json + main.ts + expected.txt (or expect-error.txt)
```

## How It Works (core flow)

1. **Annotation detection** — `parseOperatorComment` (`src/core/type-utils.cjs`, duplicated in `annotation-diagnostics.cjs`) reads the leading comment ranges of a method declaration and matches `/@operator\s*([+\-*/%<>=!&|^~?]{1,3})/`.
2. **Operator mapping** — `src/core/operator-map.cjs` converts `ts.SyntaxKind` operator tokens to text (`+=` maps to `+`, `++` maps to `+`, etc.).
3. **Resolution** (`src/core/type-utils.cjs`):
   - `resolveBinaryAnnotatedMethod`: instance method with 1 param (`a + b` becomes `a.add(b)`), static with 2 params on the left type, or static with 2 params on the right type (only in the `allowRightOperand` flow and when the right type is not number-like).
   - `resolveUnaryAnnotatedMethod`: 0-arg (`~a` becomes `a.neg()`) or 1-arg instance method.
   - `resolveIncrementAnnotatedMethod`: `++a` becomes an assignment of `a.add(1)` (instance 1-arg or static 2-arg).
   - `resolveSmartBinaryFallback`: derives missing operators — `!=` negates the `==` result; `-` reuses `+` with native or overloaded negation of the right operand.
   - Union and intersection types are traversed; `seen` Sets guard recursion. Argument assignability is checked with `checker.isTypeAssignableTo`.
4. **Rewriting** (`src/core/create-transformer.cjs`): a visitor over `BinaryExpression`, `PrefixUnaryExpression`, `PostfixUnaryExpression`; builds `CallExpression`/`Assignment` nodes via `context.factory`. It reads types from pre-transform nodes (`getOriginalNode`) and computes post-rewrite types via `getEffectiveExpressionType` (needed for chained overloads).
5. **Diagnostics**: `annotation-diagnostics.cjs` emits custom errors (93001) for invalid annotation shapes; `diagnostic-utils.cjs` filters native TS operator errors where the transformer will rewrite anyway.

## Tsserver Plugin Modes (`src/plugins/tsserver/`)

- Config keys come from the `plugins` entry in `tsconfig.json`: `mode` (`"suppress"` | `"shadow"` | `"hybrid"`), `allowRightOperand`, `shadowScope` (`"file"` | `"project"`), `autoFallbackToSuppress`, `maxShadowFiles`, `shadowFeatures`/`hybridFeatures` (per-feature toggles: `diagnostics`, `quickInfo`, `completions`, `completionDetails`, `inlayHints`), `hybridDebounceMs`, `hybridMaxBuildMs`, `hybridFailureDisableAfter`, `hybridCooldownMs`.
- **suppress**: passthrough language service + native diagnostics filtered by `shouldSuppressOperatorDiagnostic` + annotation diagnostics appended.
- **shadow**: builds a shadow program whose files are pre-rewritten by `collectRewriteEdits`/`applyEditsToText` (in `shadow-language-service.cjs`), then serves all LS features from it, remapping positions via `createPositionMapper` (edits change offsets; every LS response must be remapped back).
- **hybrid**: serves from the host LS, warms the shadow in the background (`hybrid-coordinator.cjs`: debounce, build-time budget, consecutive-failure cooldown), and switches to the shadow once it is ready.

## Test Conventions

- `test/tsc-*` cases: a folder with `tsconfig.json`, `main.ts`, and either
  - `expected.txt` — each non-empty line must appear (trimmed) in the emitted `dist/main.js` (e.g. `base.add(2)`), or
  - `expect-error.txt` — the build must fail and stderr must contain the snippet (used for invalid annotations, e.g. non-static 2-arg methods).
- The runner auto-discovers all such folders under `test/` (`collectCaseFolders`); it skips the `dist` and `node_modules` dirs.
- `test/tsserver-smoke` and `test/tsserver-typed` drive the plugin through a real `ts.LanguageService` in-process.
- `test/eslint-basic`, `test/vite-basic`, `test/nextjs-basic` (and siblings) are integration sandboxes with their own `package.json`; the runner installs and builds them with npm `--prefix`.
- **Adding a regression test** = create a new `test/tsc-<name>/` folder with the three files; no registration is needed.

## Key Modules for Bug Hunting / Optimization

| Area | File(s) | Notes |
|---|---|---|
| AST rewrite correctness | `src/core/create-transformer.cjs` | visitor, compound assign, ++/--, smart fallback node construction |
| Resolution logic | `src/core/type-utils.cjs` | candidate search order, arity/static rules, assignability, recursion guards |
| Perf hotspot | `src/core/type-utils.cjs` + `create-transformer.cjs` | `getEffectiveExpressionType` recomputes types recursively per node; `checker.isTypeAssignableTo` calls dominate cost — profile with `node --cpu-prof` |
| Shadow LS | `src/plugins/tsserver/shadow-language-service.cjs` (~1000 lines) | edit application, overlap removal, position remapping — off-by-one bugs live here |
| Warmup policy | `src/plugins/tsserver/hybrid-coordinator.cjs` | debounce/seq/cooldown state machine |
| Error suppression | `src/core/diagnostic-utils.cjs` | suppressed codes: 2356, 2362, 2363, 2365, 2367 |
| Custom diagnostics | `src/core/annotation-diagnostics.cjs` | code 93001, annotation shape rules |

## Conventions & Gotchas

- Everything is plain `.cjs` CommonJS with `require`/`module.exports` — do not introduce ESM or TypeScript in `src/`.
- The transformer must only rewrite when the types actually match an annotated method; native number/string operator semantics must never be touched (`isNumberLike` guards).
- The tsserver plugin must never crash the LS: all shadow/hybrid paths have fallbacks to the host service. Preserve that invariant.
- Keep dual consumers in sync: the transformer (`create-transformer.cjs`) and the shadow rewriter (`collectRewriteEdits` in `shadow-language-service.cjs`) implement the same rewrite semantics twice — a fix in one usually needs mirroring in the other.
- `parseOperatorComment` exists in two places (type-utils, annotation-diagnostics); keep both consistent if the regex changes.
- No `package-lock.json` is committed; run installs via `pnpm install` at the root. The integration sandboxes resolve their own deps via npm.

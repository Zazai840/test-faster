# Change impact: which public entrypoints can a diff reach?

Research ticket for `merge-gate`. Question: given a diff, how do we cheaply and language-agnostically find every public entrypoint (HTTP route, CLI command, exported API, UI component) the change can reach, so the gate checks only affected behaviour? This note compares the candidate techniques, gives precision/recall trade-offs and realistic timings on a ~100k-LOC repo, and covers dynamic dispatch and reflection.

Date: 2026-11-13. Measurements in this note were taken on a laptop (Apple Silicon, macOS) with the throwaway script described in [Experiment](#experiment).

## Answer

Use a **two-sided static graph built with tree-sitter, and let it widen, never narrow, when it meets something it cannot resolve**:

1. **Entrypoint inventory (what counts as public).** Found from declarations, not inferred: package manifests (`package.json` `exports`/`bin`, `pyproject.toml` `[project.scripts]`/entry points, `__all__`), framework conventions expressed as tree-sitter queries (route decorators, `app.get(...)`, Next.js `app/**/page.tsx` and `route.ts`, CLI decorators), and, because the gate may boot the app in memory, the framework's own route table (FastAPI `app.routes`, Flask `url_map`, a Click group's commands). Cached per base commit.
2. **Reachability (what the diff touches).** Changed lines map to enclosing definitions via tree-sitter. Propagate backwards over a file-level import graph, refined to definition level by *name*: a definition is affected if its body mentions an affected name exported by a file it imports. Identifier-shaped string literals count as mentions. Stop at entrypoints.
3. **Escalation instead of guessing.** If an affected definition is reachable only through a dynamic site (computed property call, `getattr` with a non-literal, `import(x)`, eval, DI/plugin registry), or a changed file has no static importer and is not itself an entrypoint, fall back to the file-level closure (or "every entrypoint of the package") and mark the verdict input as `reach: widened`. The LLM is asked only about these leftovers.
4. **Optional dynamic confirmation.** When the repo's declared tests run anyway, record per-test coverage (coverage.py dynamic contexts, `NODE_V8_COVERAGE`) and union the observed reach with the static reach. Union only raises recall.

Do **not** make SCIP indexes or language servers the primary mechanism. They are precise but need a per-language toolchain plus installed dependencies, cost 8-15 s and ~0.7-0.9 GB on a 95k-LOC TypeScript repo before any query, and still miss reflection. Keep them as an optional precision pass for a stack where the indexer already works.

Do **not** rely on test-impact tools (Jest `--findRelatedTests`, `vitest related`, pytest-testmon, Nx/Turborepo/Bazel affected) for entrypoints. They answer "which tests" not "which public behaviour", they are per-ecosystem, and the monorepo ones are package/target-granular. They are useful as a cross-check and for picking which declared tests to run.

### Key numbers

| | Value | Source |
|---|---|---|
| Build tree-sitter import graph + def-level name index, 95k-LOC TS (hono) | 1.5-7.8 s single-threaded Python bindings (1.5 s warm) | Experiment |
| Same, 160k-LOC Python (pydantic, vendored corpus excluded) | 4.3-6.5 s | Experiment |
| Reverse closure per diff once graph is built | 1-440 ms | Experiment |
| scip-typescript index of the same 95k-LOC TS repo | 8.2 s without deps (860 MB RSS); 14.8 s with deps (690 MB) plus 16 s `pnpm install` | Experiment |
| LLM tokens for reachability | 0. Only `reach: widened` cases go to the model, as a short list of dynamic sites (estimate: low single-digit k tokens) | Design |
| Python static call graph, best published (PyCG) | 99.2% precision, 69.9% recall, 0.38 s per kLOC | [PyCG, ICSE 2021](https://arxiv.org/abs/2103.00587) |
| JS static call graph recall, Jelly | 75.9% static, 88.1% with approximate interpretation (which executes code) | [Laursen et al., PLDI 2024](https://doi.org/10.1145/3656424) |

The graph build is the dominant cost and is per-file, so it caches by content hash; a PR touching a handful of files re-parses only those. A production implementation using tree-sitter query captures in native code (instead of a Python-level node walk) and a worker pool should be well under the 1.5 s warm figure, but that was not measured.

## Comparison

Precision here means "selected entrypoints that really are affected"; recall means "affected entrypoints that were selected". For a merge gate, **recall is the safety property** (a miss means unchecked behaviour ships) and precision is the speed property (a false hit means wasted checks).

| Technique | Language-agnostic? | Needs build/deps? | Precision | Recall | Dynamic dispatch / reflection | ~100k LOC time |
|---|---|---|---|---|---|---|
| Tree-sitter file-level import closure | Yes: one grammar + one import query + resolver per language | No | Low on hub files (a change in a shared util reaches most of the repo) | High for static imports; misses `import(x)`, lazy string imports, convention-wired files | Over-approximates dynamic *method* dispatch (whole file is affected); misses dynamic *loading* | 2-8 s cold, cacheable; closure ms |
| Tree-sitter def-level, name-based (recommended) | Yes, same inputs plus a "definition" node list per grammar | No | Medium: name collisions add false hits; big win for leaf symbols in hub files | Close to file-level for statically named calls; needs string-literal names to keep reflection tables | Name matching is a class-hierarchy-style over-approximation of virtual/duck dispatch (good); misses computed names unless escalated | Same parse; closure ≤0.5 s |
| Stack graphs | Partly: needs a hand-written graph DSL rule set per language; only Java, JavaScript, Python, TypeScript exist | No, file-incremental by design | High (scope-aware name resolution) | High for static bindings | Same blind spot for computed names | Not measured; repo is **archived (Sep 2025)** |
| SCIP indexes (scip-typescript, scip-python, ...) | No: one indexer per language, ~10 exist | Yes: tsconfig + installed deps; Python venv | High (compiler-grade references, implementations) | High for static references; no reflection | Resolves interface/implementation relations; nothing for computed names | 8-15 s + dependency install for 95k LOC TS |
| Language servers (LSP `references`, `callHierarchy/incomingCalls`) | No: one server per language | Yes, same as SCIP | High | High for static | Same as SCIP | Startup ≈ full type check, then one round trip per symbol; poor fit for batch |
| Whole-program call-graph tools (PyCG, Jelly) | No: per language, research tools | Source only (Jelly optionally runs code) | ~99% | 70-88% | Explicitly unsupported: PyCG ignores eval, getattr/setattr | PyCG ≈ 0.38 s/kLOC → ~40-60 s for 100-160k LOC; PyCG is **archived** |
| Test impact, static import graph (Jest `--findRelatedTests`/`-o`, `vitest related`) | No: JS/TS only | Yes (runner config) | File-level | Misses dynamic imports by their own docs | Same as file-level | Seconds (not measured: running repo code was not allowed in this session) |
| Test impact, dynamic coverage (pytest-testmon, coverage contexts, Ekstazi) | Per runtime | Needs a prior full run | Good (observed lines) | High *for tested paths*; blind to untested entrypoints | Sees whatever reflection actually ran | Baseline costs a full test run; selection is fast |
| Monorepo affected (Nx, Turborepo, Bazel `rdeps`) | Only for repos that adopted the tool | Declared project graph | Very low (package/task granularity) | Depends on declared inputs; Nx documents misses when inputs are incomplete | Not applicable | Near-instant |

### Evidence behind the table

**Tree-sitter.** Designed to be "general enough to parse any programming language", "fast enough to parse on every keystroke" and "robust enough to provide useful results even in the presence of syntax errors" ([README](https://github.com/tree-sitter/tree-sitter)). Its tags queries define the definition/reference vocabulary (`@definition.function`, `@reference.call`, ...) and each grammar ships `queries/tags.scm` ([code navigation docs](https://tree-sitter.github.io/tree-sitter/4-code-navigation.html)). The docs do not describe scope-aware resolution: tags are names, which is why this note treats the tree-sitter approach as name-based. Pants does the same thing commercially for Python: a Rust, multithreaded import parser, and an opt-in `string_imports` option to "infer a target's dependencies based on strings that look like dynamic dependencies" ([Pants python-infer](https://www.pantsbuild.org/stable/reference/subsystems/python-infer)).

**File-level is a sound granularity.** The FSE 2016 study of static regression test selection on 985 revisions of 22 projects found class-level static selection comparable to dynamic Ekstazi and only occasionally unsafe, while method-level static selection "performs rather poorly" ([Legunsen et al.](https://mir.cs.illinois.edu/~marinov/publications/LegunsenETAL16StaticRTS.pdf)). Ekstazi itself tracks dynamic *file* dependencies and cut end-to-end test time 32% on average over 615 revisions ([Ekstazi](https://github.com/gliga/ekstazi)). Takeaway: file-level is the safe floor; finer granularity must be justified per change and must fall back to file-level when unsure.

**Stack graphs.** Purely syntactic, file-incremental name resolution; "most commits change only a small fraction of a repository's files", and no package-specific build is run ([Creager and van Antwerpen, 2022](https://arxiv.org/abs/2211.01224)). The repository was archived on 9 Sep 2025 and "is no longer supported or updated by GitHub" ([github/stack-graphs](https://github.com/github/stack-graphs)); rule sets exist only for Java, JavaScript, Python and TypeScript. Good ideas to borrow (per-file partial graphs, caching), not a dependency to take.

**SCIP.** Language-agnostic *format*, but one indexer per language: Java/Scala/Kotlin, TS/JS, Rust, C/C++, Ruby, Python, C#/VB, Dart, PHP ([sourcegraph/scip](https://github.com/sourcegraph/scip)). scip-typescript needs `npm install`/`yarn install` first and recommends `@types/*` for JS ([scip-typescript](https://github.com/sourcegraph/scip-typescript)); scip-python is a pyright fork that needs Python 3.10+ and the project's virtualenv active, with OOM advice for big repos ([scip-python](https://github.com/sourcegraph/scip-python)). SCIP is ~4-5x smaller than LSIF ([announcing SCIP](https://sourcegraph.com/blog/announcing-scip)), so LSIF is strictly worse here.

**Language servers.** `callHierarchy` arrived in LSP 3.16 and `implementation` in 3.6 ([LSP 3.17 spec](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/)). They use the same type checkers as SCIP, so the startup cost and environment needs are the same, plus a round trip per symbol.

**Test impact tools.** Jest `--onlyChanged` "requires a static dependency graph (ie. no dynamic requires)" ([Jest CLI](https://jestjs.io/docs/cli)). `vitest related` "works with static imports ... but not the dynamic ones (e.g., `import(filepath)`)" ([Vitest CLI](https://vitest.dev/guide/cli)). pytest-testmon needs a full baseline run with `--testmon` and does not track static files or network services ([testmon](https://testmon.org/)). Nx selects a task "when your change touches one of its inputs", and documents that incomplete `inputs` cause misses and that lock-file changes affect every task ([Nx affected](https://nx.dev/docs/features/ci-features/affected)). Turborepo `--affected` is package-level by default: "if any file in a package changed, all of its tasks are selected", and a too-shallow checkout makes every package changed ([Turborepo run](https://turborepo.dev/docs/reference/run)). Bazel `rdeps` works over "the graph implicitly defined by all rule declarations in all BUILD files" ([Bazel query](https://bazel.build/query/language)). Meta's learned predictive test selection catches >95% of individual failures and >99.9% of faulty changes at half the test cost ([Machalica et al., ICSE-SEIP 2019](https://arxiv.org/abs/1810.05286)), which needs history the gate will not have on day one.

**Static call graphs in dynamic languages.** PyCG: 99.2% precision, 69.9% recall, 0.38 s per kLOC; it "ignore[s] code generation schemes, such as calls to the eval built-ins" and "the effects of functions such as getattr and setattr are ignored" ([PyCG paper](https://www.spinellis.gr/pubs/conf/2021-ICSE-pycg/html/journal.pdf); repo archived 2023). For JavaScript, "dynamic property accesses are the main reason for unsoundness"; Jelly's approximate interpretation lifts recall from 75.9% to 88.1% "with a negligible reduction in precision" but does so by executing code ([Laursen, Xu, Møller, PLDI 2024](https://doi.org/10.1145/3656424); [Jelly](https://github.com/cs-au-dk/jelly)). In a 2018 comparison of five static JS call-graph tools, most tools found edges that every other tool missed ([Antal et al., SCAM 2018](https://arxiv.org/abs/2405.07206)). Takeaway: even the best per-language analyzers miss 10-30% of real edges, so the gate's design must assume static reach is incomplete.

## Dynamic dispatch and reflection

Sorted by how the recommended design handles each case.

| Pattern | Example | Handling |
|---|---|---|
| Virtual / duck-typed method calls | `handler.process(x)` | Name-based matching treats every `process` as a candidate. Over-approximate, so recall is kept. No action. |
| String-keyed dispatch tables, lazy re-exports | pydantic `_dynamic_imports = {'AnyUrl': (pkg, '.networks')}` + module `__getattr__` | Identifier-shaped string literals count as name mentions; module-shaped strings count as import edges (Pants does the same with `string_imports`). Measured: recovers the missed public export below. |
| Computed names | `obj[key]()`, `getattr(x, name)()`, `import(path)`, `import_module(var)`, `eval` | Detected syntactically (tree-sitter query per language). If an affected definition's file is reachable only through such a site, escalate to file-level closure and mark `reach: widened`. Measured counts: pydantic 153 sites in 49 files; hono 25 sites in 10 files. |
| Framework conventions | Next.js `app/blog/page.tsx` → `/blog` ([Next.js project structure](https://nextjs.org/docs/app/getting-started/project-structure)); Django URLconf strings; pytest plugins; `pyproject` entry points | Per-framework entrypoint adapters (tree-sitter queries + path rules). A changed file with zero static importers that is not a known entrypoint is a red flag: escalate. |
| Runtime registration | decorators adding to a registry, DI containers, plugin loaders | Boot the app in memory (allowed by the gate's constraints) and read the framework's route table: FastAPI builds its OpenAPI schema from `app.routes`, which includes routes from included routers ([FastAPI](https://fastapi.tiangolo.com/how-to/extending-openapi/)). Map each route to its handler's source location, then intersect with the static reach. |
| Reach only visible at run time | anything above that still slips through | Union with per-test coverage when tests run anyway: coverage.py `dynamic_context = test_function` records which test ran which line ([coverage contexts](https://coverage.readthedocs.io/en/latest/contexts.html)); note the default `sysmon` core on Python 3.14+ does not support dynamic contexts, so force `ctrace` there ([coverage config](https://coverage.readthedocs.io/en/latest/config.html)). Node writes V8 coverage JSON per process with `NODE_V8_COVERAGE` and propagates it to subprocesses ([Node CLI](https://nodejs.org/api/cli.html)). |

## Experiment

Throwaway script (not committed): Python 3.14, `tree-sitter` 0.26.1 with the TypeScript and Python grammars, a Python-level walk over every node collecting import statements, definitions (with line spans) and identifiers. Repos: [honojs/hono](https://github.com/honojs/hono) (458 TS files, 95,495 lines, 76 public subpath exports in `package.json`) and [pydantic/pydantic](https://github.com/pydantic/pydantic) (412 Python files, 160,345 lines after excluding its vendored `tests/benchmarks/corpus`; 22 public modules). Shallow clones taken 2026-11-13.

| Change (file, symbol) | File-level closure | Def-level, name-based | Public entrypoints file → def |
|---|---|---|---|
| hono `src/utils/url.ts` `getPath` (hub) | 220 files (93 tests) | 194 (85 tests) | 59 → 55 of 76 |
| hono `src/helper/cookie/index.ts` `getCookie` | 25 (16 tests) | 22 (13 tests) | 5 → 5 of 76 |
| pydantic `_internal/_utils.py` `lenient_issubclass` (hub) | 308 (259 tests) | 238 (196 tests) | 19 → 18 of 22 |
| pydantic `networks.py` `AnyUrl` | 308 (259 tests) | 6 (4 tests) | 19 → 2 of 22 |
| pydantic `alias_generators.py` `to_camel` | 3 | 3 | 1 → 1 of 22 |

Findings:

- **Def-level pays off for leaf symbols in hub files** (`AnyUrl`: 308 → 6 files, 98% fewer) and barely helps for genuinely central symbols (`getPath`, `lenient_issubclass`: 10-25% fewer). That is the correct behaviour: a central change really does reach most of the API.
- **Reflection caused a real miss, and the cheap fix worked.** Without string-literal names, def-level for `AnyUrl` returned 4 files and only `pydantic.networks`; it dropped the top-level `pydantic.AnyUrl` export because `pydantic/__init__.py` names it only as a string key in a lazy-import table. Counting identifier-shaped strings recovered it (6 files, entrypoints `pydantic` and `pydantic.networks`), matching a grep ground truth.
- **Name collisions were handled by the import filter.** hono's `aws-lambda` adapter defines `getCookies` (a different symbol) and does not import the cookie helper; it was correctly excluded.
- **SCIP cost on the same TS repo:** 8.2 s / 860 MB without dependencies, 14.8 s / 690 MB after a 16 s `pnpm install` (run on Node 26, which scip-typescript does not officially support). That is before any query and only covers one language.
- Not measured: `vitest related` on hono (running the cloned repo's test code was not permitted in this session), stack graphs, LSP query latency, and per-test coverage overhead.

## Main risk

**Silent recall loss.** Any pruning that drops an affected entrypoint means the gate never checks that behaviour and still says "merge". Published static analyzers for Python and JS miss 10-30% of real call edges, mostly through reflection and framework wiring, and the measured `AnyUrl` miss shows how ordinary the trigger is. Mitigations, in order: never prune below the file-level closure without a resolved static path; escalate on unresolved dynamic sites and orphan files; union with observed coverage when available; and measure it. On the bug benchmark, record for each bug whether its failing entrypoint was in the selected set ("selection recall") separately from whether the gate caught the bug, so pruning errors are visible rather than blended into the headline score.

Secondary risk: hub changes select most entrypoints, so the 2-minute budget must be met by the checks themselves running fast, not by the selection always being small.

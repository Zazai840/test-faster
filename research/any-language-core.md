# Research: how a language-agnostic core works in practice

Ticket question: (a) how tree-sitter gives `merge-gate` symbols, imports and calls for any language, how fast, and how far it reaches; (b) how to auto-discover a repo's own declared check commands (typecheck, lint, test, build) and run only the ones relevant to the changed files, what prior art does this, and where it breaks.

Settled constraints this answer respects: language-agnostic from day one, under 2 minutes per verdict for a PR under 500 changed lines, at most 30k LLM tokens, no execution past "app in memory with fakes" (no network, no real services).

## Direct answer

1. **Parsing is a solved, cheap layer.** tree-sitter plus a prebuilt grammar bundle parses any of ~370 languages at 100k-300k lines/s on one laptop core. Indexing a whole 130k-225k-line repo for definitions and call sites took 0.55-1.4 s in our experiment. Parsing never threatens the 2-minute budget.
2. **But tree-sitter only gives names, not meaning.** Tags queries (`tags.scm`) give definitions and call sites by *name*. Measured on four real repos, only 14-32% of call sites resolve to exactly one definition by name; 28-38% are ambiguous, and the rest point outside the repo. So the tree-sitter graph is an *over-approximate* blast radius: good for deciding what to check and what to show the AI, never good enough to prove anything. Precise name resolution exists (stack-graphs) but GitHub archived it in September 2025 with only 4 languages.
3. **The real correctness signal comes from the repo's own tools** (`tsc`, `mypy`, `ruff`, `eslint`, the test runner). Discovery should be a deterministic, layered read of declared entry points: package.json scripts, Makefile/justfile/Taskfile targets, tox/nox, pre-commit hooks, then CI workflow `run:` steps. Every mature tool in this space (Trunk, MegaLinter, pre-commit, Nx, moon) works the same way: a per-tool *descriptor* says which files a tool cares about and which config file proves it is in use. None of them reads intent from the code.
4. **Unusual repos fail in predictable ways**: compound scripts (`"test": "xo && npm run build && tsc && ava"`), checks hidden in `scripts/*.sh`, Makefile prerequisites that silently install things from the network, monorepos where root scripts just delegate to turbo/nx, and repos where CI YAML is the only declaration. The LLM fallback for "figure out how to run this repo" is unreliable: published agents succeed 6.7%-86% depending on benchmark difficulty, so the gate must never depend on it.

## Recommended approach

### Part A: the parsing layer

- **Library**: `py-tree-sitter` 0.26 + `tree-sitter-language-pack` 1.25 (MIT, 371 grammars, Python/Node/Rust bindings). Grammars download on first use, so the gate must call `download()`/`prefetch()` in a setup step, with `PackConfig(cache_dir=...)` pinned, and parse offline afterwards. ([language pack README](https://github.com/Goldziher/tree-sitter-language-pack))
- **Symbols and calls**: run each language's `tags.scm` and read the standard captures `@definition.{class,function,interface,method,module}`, `@reference.{call,class,implementation}`, `@name`, `@doc` ([tree-sitter code navigation docs](https://tree-sitter.github.io/tree-sitter/4-code-navigation.html)). This is exactly what Aider's repo map does across 39 languages, and it fits the map into a 1k-token default budget ([Aider repo map](https://aider.chat/docs/repomap.html), [Aider languages](https://aider.chat/docs/languages.html)).
  - Gotcha found in the experiment: the TypeScript `tags.scm` only covers types and interfaces. It emits **no** `@reference.call`; you must concatenate the JavaScript tags query with it (TS grammars extend JS). Without this, TS call graphs are empty.
  - Coverage gap: only about 71 of the 371 bundled grammars ship a tags query (our count from the README table). For the rest, fall back to per-file "changed hunks only" with no graph.
- **Imports**: tags queries do not cover imports. The language pack's `process()` returns `ImportInfo`/`ExportInfo`/`SymbolInfo` generically. In our test it found static ES and Python imports, but it **missed** `await import("./lazy")`, dropped aliases (`bar as baz` came back as `bar`), returned the whole statement rather than the module specifier, and skipped TS class methods. Treat it as a starting point; resolving `"./util"` to a file is per-language (tsconfig `paths`, package `exports`, Python `sys.path`) and needs a small resolver per supported stack.
- **Structural rules**: use [ast-grep](https://ast-grep.github.io/guide/rule-config.html) (28 built-in languages, more via a compiled tree-sitter grammar in `sgconfig.yml` `customLanguages` ([custom languages](https://ast-grep.github.io/advanced/custom-language.html))) for any pattern rules the gate itself ships, such as "new `eval`" or "removed exported symbol". It does single-file structural matching only. It does no cross-file name resolution or type checking.
- **Precision upgrade, opt-in per stack**: when a stack's own indexer exists (scip-typescript, scip-python, scip-java, rust-analyzer, scip-clang and others ([SCIP](https://github.com/sourcegraph/scip))), it gives exact references. Do not make it the core: it needs a build environment per language and is far slower than tree-sitter.
- **How to use the ambiguous graph safely**: for check *selection* and for picking what context to send to the AI, an over-approximation is the safe direction. Treat an ambiguous name as "all candidates". For *findings* (for example "this caller now breaks"), require confirmation from the repo's typechecker or a test, never from a name match.

### Part B: discovering and running the repo's own checks

Discovery order (stop adding sources once the repo declares checks in a higher layer, but always record the others for the report):

| Layer | Source | How to read it without running anything |
|---|---|---|
| 0 | Gate config in repo (for example `.merge-gate.yml`) | Explicit wins. Ship `merge-gate init` that writes the discovered result here for the owner to edit. |
| 1 | `package.json` `scripts` (root and each workspace from `pnpm-workspace.yaml` / `workspaces`) | JSON. Classify by script name (`typecheck`, `lint`, `test`, `build`) and by the binary called (`tsc`, `eslint`, `vitest`, `jest`). |
| 1 | `pyproject.toml` | `[tool.tox]`, `[tool.hatch.envs.*.scripts]`, `[tool.poe.tasks]`, and tool sections (`[tool.mypy]`, `[tool.ruff]`, `[tool.pytest]`, `[tool.pyright]`) as evidence a tool is in use. |
| 1 | `tox.ini` / `noxfile.py` | `tox list`, `nox --list --json` (name, description, tags, Python version) ([nox usage](https://nox.thea.codes/en/stable/usage.html)). |
| 1 | `justfile` / `Taskfile.yml` / `Makefile` | `just --dump --dump-format json` ([just README](https://github.com/casey/just)); `task --list-all --json` ([Taskfile CLI](https://taskfile.dev/reference/cli/)); Makefile by reading `.PHONY` targets and their recipes (`make -pn` dry runs can still execute `$(shell ...)`). |
| 2 | `.pre-commit-config.yaml` | Hooks already carry `files`, `exclude`, `types`/`types_or` filters and `pass_filenames`, and `pre-commit run --from-ref BASE --to-ref HEAD` runs only hooks whose filters match the changed files ([pre-commit](https://pre-commit.com/)). This is the best ready-made "relevant to changed files" mapping. |
| 2 | `lefthook.yml` | `glob` + `root` per command; note globs always match from the git root even when `root` is set ([lefthook root](https://lefthook.dev/configuration/root)). |
| 3 | `nx.json` / `turbo.json` / `moon.yml` | Defer to the monorepo tool's own affected logic (below) instead of reimplementing it. |
| 4 | `.github/workflows/*.yml` | Extract `run:` steps from jobs triggered by `pull_request`, honour `paths` filters ([workflow syntax](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax)). Drop steps that use `services:`, deploy, publish, or are clearly install/network steps. |
| 5 | Config-file presence only (no declared command) | Trunk-style descriptors: `eslint.config.*` present means run `eslint` on changed TS/JS; `[tool.ruff]` means `ruff check`. Mark these "inferred" in the verdict. |
| 6 | LLM fallback | Only to *classify* ambiguous discovered commands (send names and command strings, about 1-3k tokens). Never to invent commands. |

How to pick only the relevant commands for a change:

- **Per-file tools** (linters, formatters): pass only changed files, using the tool's own include/exclude config. MegaLinter models this as `cli_lint_mode: file | list_of_files | project`; pylint, mypy and ruff all default to `list_of_files` ([MegaLinter Python descriptor](https://raw.githubusercontent.com/oxsecurity/megalinter/main/megalinter/descriptors/python.megalinter-descriptor.yml)). Note mypy and `tsc` give *incomplete* answers on a file list, because type errors appear in unchanged callers. Run project-mode typecheckers on the affected package instead, relying on their incremental caches.
- **Report only new problems**: Trunk's "hold the line" lints both the upstream and the working-tree version of each modified file, diffs the results, and reports only new issues at line level ([Trunk blog](https://trunk.io/blog/trunk-check-hold-the-line), [docs](https://docs.trunk.io/code-quality/overview/prevent-new-issues)). The gate needs this so legacy lint debt does not block every PR.
- **Monorepos**: Nx selects a task when a changed file matches one of that task's `inputs`, and by default a lock-file change affects every task ([Nx affected](https://nx.dev/docs/features/ci-features/affected)). moon `--affected` exits early when no changed file intersects a task's `inputs`, whose default is every file in the project (`**/*`) ([moon run-task](https://moonrepo.dev/docs/run-task), [moon project config](https://moonrepo.dev/docs/config/project)). Nx infers tasks from tool config files through plugins (`createNodes` = glob + function) ([Nx inferred tasks](https://nx.dev/docs/concepts/inferred-tasks)). When a repo has Nx/moon/turbo, call their affected command. Otherwise map each changed file to its nearest manifest (package.json / pyproject.toml / Cargo.toml) and run that package's checks plus the checks of packages whose manifests depend on it.
- **No-network enforcement**: run every discovered command in a sandbox with networking off, after a separate, cacheable "prepare" step that installs dependencies. Many declared commands quietly install things (see failure modes).

## Experiments (temp folder, not committed)

Environment: MacBook, Python 3.14, `tree-sitter` 0.26.1, `tree-sitter-language-pack` 1.25.0, single thread. The grammar cache was kept inside the worktree. Downloading 3 grammars took 4.6 s once.

### Parse and tags speed

| Corpus | Files | Lines | Parse | Speed | Files with parse errors |
|---|---|---|---|---|---|
| npm's JS sources | 1,135 | 177k | 0.88 s | 203k lines/s overall | 1 |
| TS in global node_modules (first 3,000 `.ts`) | 3,000 | 325k | 2.34 s | 139k lines/s overall; 309k median per file over all 5,515 | 91 (mostly large `.d.ts` files) |
| CPython 3.14 stdlib | 2,289 | 1.13M | 4.6 s | 281k lines/s median per file | 4 (PEP 695 type-param tests, intentional bad syntax) |

One run that kept all 2,289 Python trees in memory at once took 22.8 s, so the gate should parse, extract, and drop each tree rather than hold them.

Whole-repo index (parse + tags query, definitions and call sites):

| Repo | Files | Lines | Definitions | Call/type refs | Wall time |
|---|---|---|---|---|---|
| pydantic | 3,162 | 226k | 18,580 | 50,868 | 1.06 s |
| trpc (TS monorepo) | 1,343 | 193k | 6,452 | 56,961 | 1.39 s |
| fastapi | 1,225 | 133k | 7,160 | 23,481 | 0.55 s |

### How far name-only resolution gets

Each `@reference.call` name was matched against all `@definition.*` names in the same repo:

| Repo | Resolves to exactly one definition | Ambiguous (2+ definitions) | No definition in repo (stdlib, deps, dynamic) |
|---|---|---|---|
| pydantic | 23% | 38% | 39% |
| trpc | 20% | 28% | 52% |
| fastapi | 14% | 34% | 53% |
| flask | 32% | 28% | 39% |

### What real repos declare

Shallow clones of six popular repos:

| Repo | Where the checks actually live | Discovery problem it shows |
|---|---|---|
| flask | `[tool.tox]` in pyproject + pre-commit + CI | Clean case: tox environments are the API. |
| fastapi | `scripts/lint.sh` (`mypy`, `ty check`, `ruff check`, `ruff format --check`), `scripts/test.sh`; no Makefile and no package scripts | Checks hide in shell scripts. Only discoverable via CI `run:` steps (`uv run --no-sync bash scripts/test-cov.sh`) or by name. |
| pydantic | Makefile (`lint`, `typecheck`, `test`) + pre-commit `repo: local` hooks with `language: system` | `typecheck` depends on `.pre-commit`, which runs `uv pip install pre-commit` if missing, so it needs network. `uv run` also syncs dependencies by default. |
| ky | `"test": "xo && npm run build && tsc --project tsconfig.test.json && ava"` | One compound script mixes lint, build, typecheck and test. It has to be split by parsing the shell `&&` chain. |
| trpc | pnpm workspace + turbo; root scripts delegate (`turbo --filter=./packages/* build`) | Root scripts are routers. Affected selection belongs to turbo, not to the gate. |
| uv | Rust + Python; 50+ workflow files and no task runner | CI YAML is the only declaration. Most workflows are release/bench/bot jobs that must be filtered out. |

## Prior art: how each tool detects what to run

| Tool | Detection mechanism | Changed-files mode | Lesson for merge-gate |
|---|---|---|---|
| Trunk Check | Per-linter `plugin.yaml`: `files: [typescript, javascript]`, `direct_configs` (for example `eslint.config.js`), `suggest_if: config_present`, `affects_cache` ([eslint plugin](https://raw.githubusercontent.com/trunk-io/plugins/main/linters/eslint/plugin.yaml), [plugins repo](https://github.com/trunk-io/plugins), ~110 tools) | Hold-the-line by default: lints upstream and working tree, reports only new issues | Copy the descriptor shape and hold-the-line. Trunk installs its own pinned tool versions, while the gate should use the repo's versions. |
| MegaLinter | YAML descriptors per language (`file_extensions`, `config_file_name`, `cli_lint_mode`); activation via `ENABLE`/`DISABLE` lists; 67 languages, 18 Docker flavours ([MegaLinter](https://megalinter.io/latest/), [activation](https://megalinter.io/latest/config-activation/)) | `VALIDATE_ALL_CODEBASE=false` checks only new or edited files against `DEFAULT_BRANCH` ([variables](https://megalinter.io/latest/config-variables/)); filters do not apply to `project`-mode linters ([filtering](https://megalinter.io/latest/config-filtering/)) | Docker-heavy and runs its own linter versions, so it is too slow and too opinionated for the core. The descriptor format is useful reading. |
| pre-commit | Explicit per-hook config; file matching by regex plus `identify` tags (extension, filename, shebang) | `--from-ref/--to-ref`; per-language isolated environments ([pre-commit](https://pre-commit.com/)) | Best existing changed-file to check mapping. Reuse it directly when present. |
| lefthook | Explicit `glob`, `root`, `{staged_files}`/`{push_files}` templates; no auto-detection ([root docs](https://lefthook.dev/configuration/root)) | Yes, via templates | Read it as a declaration source. |
| Nx | Inference plugins read tool configs and create tasks (`createNodes`); config merges plugin, then `targetDefaults`, then project files ([inferred tasks](https://nx.dev/docs/concepts/inferred-tasks)) | Task-level affected by `inputs`; lockfile affects all tasks by default ([affected](https://nx.dev/docs/features/ci-features/affected)) | Delegate to it when present. |
| moon | Language detected from config files in project root, otherwise `unknown`; can infer tasks from package.json scripts or migrate them ([project config](https://moonrepo.dev/docs/config/project), [migrate](https://moonrepo.dev/docs/commands/migrate/from-package-json)) | `--affected` intersects touched files with task `inputs` ([run-task](https://moonrepo.dev/docs/run-task)) | Delegate to it when present. |

Agent-based "figure out how to run it" research, for calibrating the LLM fallback:

- ExecutionAgent (ISSTA 2025): ran test suites of 33 of 50 projects across 14 languages ([arXiv 2412.10133](https://arxiv.org/abs/2412.10133)).
- Repo2Run (NeurIPS 2025): 86.0% of 420 Python repos, against SWE-agent at 9.0%; it needs Docker and network, and uses many turns ([arXiv 2502.13681](https://arxiv.org/abs/2502.13681)).
- EnvBench (JetBrains, 2025): the best method set up only 6.69% of hard Python repos and 29.47% of JVM repos ([arXiv 2503.14443](https://arxiv.org/abs/2503.14443)).

Conclusion: agentic setup is too slow, too token-hungry and too unreliable to sit on the gate's critical path. Deterministic discovery plus a one-time, owner-confirmed `merge-gate init` is the robust route.

## Failure modes on unusual repos, and the planned response

| Failure | Response |
|---|---|
| No declared commands at all | Fall back to layer 5 (config-presence descriptors), then tree-sitter-only checks. Say so in the verdict as reduced confidence ("no typecheck found"); never silently pass. |
| Compound scripts (`a && b && c`) | Split on top-level `&&`/`;` with a shell parser, then classify each part by binary. If it cannot be split safely, run it whole and label it by its strongest part. |
| Checks hidden in `scripts/*.sh` | Find them through CI `run:` steps and through script-name heuristics (`lint.sh`, `test.sh`). The init step confirms them with the owner. |
| Commands that install or fetch (`uv run` sync, `npx` downloads, `make` prerequisites, `pip install` fallbacks) | Separate a cacheable prepare phase from the check phase; run checks with networking off and offline flags (`uv run --no-sync`/`--offline`, `npm --offline`); report "needed network" as a configuration finding. |
| Monorepo root scripts that route to turbo/nx/moon | Detect the router and call its affected mode instead of the root script. |
| CI-only declarations full of release/bench/bot jobs | Keep only `pull_request`-triggered jobs; drop `services:`, docker, publish and deploy steps; honour `paths` filters. |
| File-list typecheck misses errors in unchanged callers | Run project-mode typecheck on the affected package (incremental), not on the file list. |
| Pre-existing lint debt | Hold-the-line diff against the base revision, as Trunk does. |
| Grammar lags new syntax (PEP 695, newest TS) | `has_error` on a changed file downgrades the structural layer for that file; the repo's own compiler remains the source of truth. |
| Language with no `tags.scm` (about 300 of 371) | Changed-hunk view only, no graph; the repo's own checks still run. |

## Key numbers

- Parse speed: about 140k-310k lines/s per core (median per file); whole-repo symbol index 0.5-1.4 s for 130k-225k lines. A 500-line PR touching 20 files parses in well under 0.1 s.
- Grammar coverage: 371 grammars in the language pack, about 71 with tags queries, 28 built into ast-grep, 39 with Aider repo-map support.
- Name-only call resolution: 14-32% unique, 28-38% ambiguous.
- Token cost of this layer: 0 for parsing and discovery. Optional LLM classification of ambiguous commands is about 1-3k tokens. A symbol-map context in the Aider style costs about 1k tokens by default.
- Agent-based environment setup success, for comparison: 6.7%-86% depending on benchmark.

## Main risk

The gate's verdict is only as strong as the checks it finds. On a repo with no machine-readable declaration, or with checks that need the network, the deterministic layer finds little and tree-sitter alone cannot prove correctness, because name matching is ambiguous more than a quarter of the time. Mitigation: make "what was and was not checked" a first-class part of every verdict, ship an `init` step that turns discovery into an owner-confirmed config file, and benchmark the discovery itself across a corpus of real repos (precision and recall of found commands), not only the bug benchmark.

## Open questions for later tickets

- Which file in the repo is the confirmed config (`.merge-gate.yml` or a section in an existing file)? This is an owner decision.
- Test selection inside a single test command (which tests to run for a change) is not covered here. It needs coverage maps or import graphs and is its own ticket.
- Whether to ship per-stack import resolvers for TS and Python in v1, or to rely on the typechecker for cross-file effects only.

## Sources

- tree-sitter: https://tree-sitter.github.io/tree-sitter/ and https://tree-sitter.github.io/tree-sitter/4-code-navigation.html
- tree-sitter-language-pack: https://github.com/Goldziher/tree-sitter-language-pack
- stack-graphs (archived 2025-09-09; Java, JS, Python, TS only): https://github.com/github/stack-graphs and https://github.blog/open-source/introducing-stack-graphs/
- ast-grep: https://ast-grep.github.io/reference/languages.html, https://ast-grep.github.io/advanced/custom-language.html, https://ast-grep.github.io/guide/rule-config.html
- Aider repo map: https://aider.chat/docs/repomap.html, https://aider.chat/docs/languages.html
- SCIP: https://github.com/sourcegraph/scip
- Trunk: https://github.com/trunk-io/plugins, https://trunk.io/blog/trunk-check-hold-the-line, https://docs.trunk.io/code-quality/overview/prevent-new-issues
- MegaLinter: https://megalinter.io/latest/, https://megalinter.io/latest/config-activation/, https://megalinter.io/latest/config-variables/, https://megalinter.io/latest/config-filtering/
- pre-commit: https://pre-commit.com/
- lefthook: https://lefthook.dev/configuration/root
- Nx: https://nx.dev/docs/concepts/inferred-tasks, https://nx.dev/docs/features/ci-features/affected
- moon: https://moonrepo.dev/docs/config/project, https://moonrepo.dev/docs/run-task, https://moonrepo.dev/docs/commands/migrate/from-package-json
- just: https://github.com/casey/just (README, `--dump-format json`); Taskfile: https://taskfile.dev/reference/cli/; nox: https://nox.thea.codes/en/stable/usage.html
- GitHub Actions workflow syntax: https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax
- ExecutionAgent: https://arxiv.org/abs/2412.10133; Repo2Run: https://arxiv.org/abs/2502.13681; EnvBench: https://arxiv.org/abs/2503.14443

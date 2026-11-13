# Property-based tests and diff-scoped mutation testing for the merge gate

Research ticket for `merge-gate`. Question: (a) can property-based tests (PBT) for changed functions be generated automatically and run in about 60 s, what do they catch, how flaky are they, what do LLM-written ones cost in tokens; (b) can mutation testing restricted to changed lines prove the existing tests guard the change, within the 2-minute budget.

Date: 2026-11-13. Sources are primary (tool docs, papers) unless marked. Numbers I derived rather than read are marked **[estimate]**.

Experiment note: I planned to time Hypothesis ghostwriter, mutmut and Stryker on a real repo (more-itertools at a few recent commits). Installing and running third-party packages was blocked by the sandbox's permission policy, so this note has no local measurements. All runtime figures below are published numbers or labelled estimates.

---

## 1. Short answer

**(a) Property-based tests: yes, in a narrow form.**
- Generation without an LLM works for three cases:
  - Python functions with type hints or defaults, through the Hypothesis ghostwriter.
  - HTTP APIs that publish an OpenAPI or GraphQL schema, through Schemathesis, which runs in-process against ASGI and WSGI apps.
  - TypeScript code that already has Zod, Valibot or Effect schemas, through schema-to-arbitrary adapters for fast-check.
- Everything else needs an LLM. Raw LLM-written properties are often wrong: only 26% of generated tests were both valid and sound in the main study. So a failing LLM-written property is evidence to triage, not a verdict.
- Running them fits easily in 60 s when generation is made deterministic (fixed seed, no example database, no deadline). Doing that also removes the generator's own flakiness.

**(b) Mutation testing on changed lines: yes, it fits the budget, but only with all of these cuts:**
- mutate only the changed lines that tests cover
- skip "arid" (uninteresting) nodes
- generate at most 1 mutant per line
- run only the tests that cover each mutant
- cap the total number of mutants
- reuse cached results from main

Google's production version of this setup cut the median from 820 mutants per change to 7.

What it can prove: whether the changed lines are guarded by tests. What it cannot prove: whether everything the change affects is guarded. 81% of the mutants that matter to a commit sit outside the changed lines.

---

## 2. Property-based testing

### 2.1 Tools that generate tests without an LLM (0 tokens)

| Tool | Stack | What it generates | Limits |
|---|---|---|---|
| Hypothesis ghostwriter (`hypothesis write`) | Python | One test per function. Modes: `fuzz` ("valid inputs only raise expected exceptions"), `idempotent`, `roundtrip`, `equivalent` (several functions return equal values for the same arguments), `binary_operation`, `ufunc`. `magic` picks a mode itself. | "Type annotations are helpful but not required". Parameters with no annotation or default get a placeholder. The output "may be ready-to-run, or include placeholders and `# TODO:` comments". Without a detected property it only checks "no error on valid input". Requires `black`. [docs](https://hypothesis.readthedocs.io/en/latest/reference/integrations.html) |
| CrossHair `diffbehavior` | Python | Uses an SMT solver to find an input on which two versions of a function behave differently. | "absence of an example difference does not guarantee that the functions are equivalent". Needs proper type annotations and deterministic code. "likely won't be able to detect differences in complex code". CPython only. [docs](https://crosshair.readthedocs.io/en/latest/diff_behavior.html) |
| Schemathesis | Any HTTP app with an OpenAPI 2.0-3.2 or GraphQL schema (also gRPC) | Property-based API tests. Checks include `not_a_server_error`, `status_code_conformance`, `content_type_conformance`, `response_schema_conformance`, `negative_data_rejection`, `positive_data_acceptance`, `ignored_auth`, `use_after_free` and others. | In-process only for Python apps: `schemathesis.openapi.from_asgi("/openapi.json", app)` and `from_wsgi`. The transport "sends this schema's cases to `app`, or over the network when `app` is `None`". [Python API](https://schemathesis.readthedocs.io/en/stable/reference/python/), [CLI](https://schemathesis.readthedocs.io/en/stable/reference/cli/) |
| fast-check plus schema adapters (`zod-fast-check`, `valibot-fast-check`, Effect `Arbitrary.make`) | TS/JS | Input arbitraries derived from an existing runtime schema. | I found no built-in way to derive arbitraries from plain TypeScript types. A fast-check maintainer prefers to keep schema generation in side packages. There is no ghostwriter equivalent, so the property itself must come from somewhere else. [ecosystem](https://fast-check.dev/docs/ecosystem), [zod-fast-check](https://classic.yarnpkg.com/en/package/zod-fast-check) |

Most useful for a merge gate: the **differential old-vs-new property**. Use `hypothesis write --equivalent` or CrossHair `diffbehavior` to compare the base version of a function with the head version.
- If the change intent says behaviour must stay the same (a refactor, a performance change, a rename), any difference found is a real finding.
- If the change intent says behaviour should change, the counterexample shows the AI judge exactly what changed, which is useful input for the intent-match check.

### 2.2 What PBT catches (published evidence)

- **Schemathesis** (Hatfield-Dodds and Dygalo, ICSE 2022 poster, [arXiv 2112.10328](https://arxiv.org/abs/2112.10328)):
  - 30 runs of 8 fuzzers against 16 open-source web services
  - finds 1.4x to 4.5x more unique defects than the second-best fuzzer on each target
  - the only fuzzer that found defects in 4 of the targets
  - the defects are server errors, schema violations and validation gaps
- **PBT compared with unit tests** (Ravi and Coblenz, OOPSLA 2025, [DOI 10.1145/3764068](https://doi.org/10.1145/3764068), [talk page](https://2025.splashcon.org/details/OOPSLA/102)): across 40 Python projects, each property-based test kills about **50x as many mutants** as an average unit test.
- **PBT in industry** (Goldstein et al., ICSE 2024, [paper](https://cis.upenn.edu/~bcpierce/papers/icse24-pbt-in-practice)): 30 interviews at Jane Street.
  - PBT's value is for complex code, mostly through a few high-leverage idioms: round-trip properties, differential testing against a reference or old version, and invariants.
  - The main weakness is that writing properties and generators is hard.
- **Kinds of bug found by LLM-written PBT** (Maaz et al., NeurIPS 2025 workshop, [arXiv 2510.09907](https://arxiv.org/abs/2510.09907)):
  - numerical errors (negative samples from `numpy.random.wald`)
  - logic errors (`slice_dictionary` repeating a chunk; `item_hash` returning `None`)
  - contract violations (malformed HSL string)
  - serialization failures and broken caches

### 2.3 LLM-written properties: accuracy and token cost

**Vikram, Lemieux, Sunshine and Padhye, "Can Large Language Models Write Good Property-Based Tests?"** ([arXiv 2307.04346](https://arxiv.org/abs/2307.04346), [full text](https://ar5iv.labs.arxiv.org/html/2307.04346))
- Setup: 40 Python API methods; 3,110 generated test functions; models were GPT-4, Gemini-1.5-Pro and Claude-3-Opus.
- Validity and soundness over all generated tests: **39% valid, 26% valid and sound**. Two-stage prompting works best: first extract up to 5 properties, then write one test per property.
  - GPT-4 two-stage: 55% valid, 42% valid and sound, **2.4 samples on average to get one valid and sound test**.
- Property coverage (does the test detect a violation of a documented property): **21% at best**.
- Failure modes:
  - assertions that hold only for some inputs
  - properties that are not in the docs
  - missed edge cases
  - runtime errors on extreme inputs, such as an `OverflowError` from `timedelta`
- The API documentation fed in was 82 to 1,198 tokens per method. The paper reports no token cost.

**PBT-Bench** ([arXiv 2605.15229](https://arxiv.org/abs/2605.15229), NeurIPS 2026 Datasets track)
- 100 problems across 40 Python libraries, with 365 injected semantic bugs.
- An agent must infer an invariant from the docs and write a Hypothesis strategy that exposes the bug.
- Bug recall: **42.1% to 83.4%** across 8 models with a PBT-guided prompt, and 31.4% to 76.7% with an open-ended prompt.
- The Hypothesis scaffolding helps mid-tier models by more than 20 points.

**Agentic PBT** (Maaz, DeVoe, Hatfield-Dodds and Carlini, [arXiv 2510.09907](https://arxiv.org/abs/2510.09907)): Claude Code with Opus 4.1 across 100 packages and 933 modules.
- Precision:
  - **56% of bug reports were valid** (95% CI 42-70%), and 32% were worth reporting.
  - Among the top 21 reports by score, 86% were valid.
- Cost: **2.37M tokens and 8.8 minutes per module**, which is $5.56 per bug report.
- The paper names intent ambiguity as a source of false positives. It also counts inputs that break implicit preconditions as false alarms.
- **This open-ended agentic mode is about 80x over the 30k-token budget per verdict. The gate cannot use it.**

**Token estimate for a gate-sized LLM property [estimate]:**

| Part | Tokens |
|---|---|
| Changed function, signature and docstring | ~0.5-1k |
| Relevant change intent | ~0.3-0.5k |
| Output (one or two `@given` tests) | ~0.4-0.6k |
| **Total per attempt** | **~1.5-2k** |
| At 2.4 attempts per valid and sound test | **~4-5k per function** |

A cap of **3 functions per pull request** keeps this at about 15k tokens, leaving room for mutant triage (section 3.5). The two-stage format from Vikram et al. (list properties, then write tests) is the right prompt shape.

**Making unsound LLM properties safe [design suggestion, not from a paper]:** run every LLM-written property against **both the base and the head version**.
- **Passes on base, fails on head:** a regression signal. This deterministic filter removes most unsound properties without spending tokens on triage.
- **Fails on both:** the property is wrong, or the bug was already there. Report it as a note, never a block.
- **Fails on head only, and the change intent says behaviour should change:** pass it to the AI judge with the minimal counterexample.

### 2.4 Runtime

- Defaults:
  - Hypothesis: `max_examples=100`, a `deadline` of 200 ms per example ([settings](https://hypothesis.readthedocs.io/en/latest/reference/api.html))
  - fast-check: `numRuns` 100, `timeout`, `interruptAfterTimeLimit` and `skipAllAfterTimeLimit` all disabled by default ([Parameters](https://fast-check.dev/docs/api/interfaces/Parameters/))
  - Schemathesis: `--max-time SECONDS` splits a time budget across its phases, and `--max-examples` caps cases per operation ([CLI](https://schemathesis.readthedocs.io/en/stable/reference/cli/))
- Estimate for pure functions: 100 examples at sub-millisecond to millisecond cost each is well under 1 s per property. Shrinking a failure adds seconds, not minutes. A hard budget of **about 20-30 s for the whole PBT stage** is realistic. **[estimate]**
- The cost to control is collection and import time of the test runner, not the examples. Run all generated properties in one process.
- Enforce the budget through the tools: Hypothesis `phases`, `max_examples` and a per-stage wall-clock kill; fast-check `interruptAfterTimeLimit`; Schemathesis `--max-time`.

### 2.5 Flakiness

- I found no empirical study of flakiness specific to property-based tests.
- General Python data (Gruber et al., [arXiv 2101.09077](https://arxiv.org/abs/2101.09077)): across 7,571 flaky tests, order dependency caused 59% and infrastructure problems caused 28%. Network and randomness APIs together caused only about 13%.
- PBT adds one source of flakiness: random generation. All three tools can switch it off:
  - **Hypothesis:** `derandomize=True` (seeded from a hash of the test), `database=None`, `deadline=None`. The built-in CI profile already sets the first two plus `deadline=None` ([settings](https://hypothesis.readthedocs.io/en/latest/reference/api.html)). The deadline is the usual cause of timing flakes on shared CI.
  - **fast-check:** a fixed `seed` (the default is `Date.now()`). Replay a failure with `seed` and `path`.
  - **Schemathesis:** `--seed` or `--generation-deterministic`.
- Leftover non-determinism in the code under test shows up as a Hypothesis `FlakyFailure` ("failed with a given value at least once, but a later call did not fail"). That is itself a code-health finding, not gate noise.
- Trade-off of derandomizing: the gate tests the same inputs on every run, so it explores less over time. That is the correct trade for a gate: same input, same verdict.

---

## 3. Mutation testing restricted to the change

### 3.1 Tool support for diff scoping

| Tool | Stack | Can it scope to the diff? | Incremental cache | Test selection per mutant |
|---|---|---|---|---|
| StrykerJS | JS/TS | Yes, by line ranges: `mutate: ["src/app.js:1-11"]`, built from the diff. There is no git `--since` flag in StrykerJS. | `--incremental` (since 6.2) reuses results from `reports/stryker-incremental.json`. It reuses a killed mutant if its killing test is unchanged, and a surviving mutant if no tests changed. It does not see changes to dependencies, environment variables or `.snap` files. [docs](https://stryker-mutator.io/docs/stryker-js/incremental/) | `coverageAnalysis: perTest` (the default). The TypeScript checker removes mutants that do not compile before running them. [config](https://stryker-mutator.io/docs/stryker-js/configuration/) |
| Stryker.NET | C# | Yes: `--since[:target]` "test only code changes since the given target". | `with-baseline` | yes |
| mutmut 3 | Python (pytest only, POSIX fork) | Function level only. `mutmut run "module.func*"`. No line or diff option. | Per-function source hash, with invalidation of callers (3.7.0). Only re-tests mutants in functions whose source changed. [docs](https://mutmut.readthedocs.io/en/latest/), [history](https://raw.githubusercontent.com/boxed/mutmut/main/HISTORY.rst) | Records which tests reach each function. `mutate_only_covered_lines` is opt-in. |
| Cosmic Ray | Python | Could not confirm a diff filter from its docs. | n/a | n/a |
| Mull | C/C++ (LLVM) | Yes: `gitDiffRef: origin/main`, plus optional `gitDiffPaths`. "only finds mutations in the source lines that are covered by a particular Git Diff changeset." [docs](https://mull.readthedocs.io/en/latest/IncrementalMutationTesting.html) | n/a | yes |
| PIT | JVM | Open-source PIT has only `targetClasses` and `targetTests`. Diff scoping at line level needs the commercial Arcmutate `+GIT(from[master])` ("by default only mutations on lines that have been modified"). [PIT](https://pitest.org/quickstart/maven/), [Arcmutate](https://docs.arcmutate.com/docs/git-integration.html) | `withHistory` | Per-test line coverage; tests run fastest-first ([FAQ](https://pitest.org/faq/)) |

Conclusion: there is no universal diff-scoped mutation tool. Native tools exist for JS/TS (Stryker with line ranges), .NET, C/C++ and JVM (paid). Python gets function granularity only.

The language-agnostic fallback is a small mutation engine built on tree-sitter. It would apply a few classic operators to the changed, covered, non-arid nodes, then run the repo's declared test command. Test selection would be coarser: test files that import the changed module rather than coverage per test.

### 3.2 Realistic numbers (Google, production)

From Petrović, Ivanković, Fraser and Just, "Practical Mutation Testing at Scale" (TSE 2021, [arXiv 2102.11378](https://arxiv.org/abs/2102.11378), [full text](https://ar5iv.labs.arxiv.org/html/2102.11378)).

**Setup:**
- Only changed lines that tests cover.
- **At most 1 mutant per line.**
- At most 7 times the number of files reported per change.
- Tests chosen by line coverage.

**Results:**
- Median mutants per change: **820 traditional, 77 with 1 per line, 7 with arid nodes suppressed as well**.
- Surviving mutants: median **2 live mutants per change**, 99th percentile 43.
- Kill rate: 87.5% of generated mutants are killed.
- Developer feedback: 82% of surfaced mutants with feedback were rated productive. This rose from about 15% before the arid-node heuristics to 89% at the end.
- Picking operators by their past productivity raised mutant survival by over 40% and "please fix" requests by almost 50%.
- Scale: more than 24,000 developers.
- Latency is not reported.

**Earlier version** (ICSE-SEIP 2018, [research.google](https://research.google/pubs/pub46584/)): about 70,000 diffs, 1.1M mutants and 150,000 actionable findings during code review.

**Facebook** (Beller et al., ICSE-SEIP 2021, [arXiv 2010.13464](https://arxiv.org/abs/2010.13464)): more than half of 15,000 mutants survived a mature test suite. Nearly half of 26 developers said they would act on a surviving mutant by adding or changing a test.

### 3.3 Runtime model for one pull request [estimate]

Runtime ≈ T_cov + (N × t̄) / W, where:
- T_cov is one coverage run of the tests selected for the change
- N is the number of mutants after filtering
- t̄ is the average time of the tests that cover one mutant
- W is the number of workers

| Step | Typical value |
|---|---|
| Changed lines (pull request under 500 lines, minus comments, imports, tests and arid nodes) | ~50-200 covered, non-arid lines |
| Mutants (1 per line, capped at about 7x the number of files or 50 sampled) | N ≈ 7-50 |
| Average covering-test time | t̄ ≈ 0.2-2 s (fast unit tests that touch the changed code) |
| Workers (laptop) | W ≈ 6-8 |
| Mutation time | **≈ 1-15 s** |
| Plus T_cov | **≈ 5-30 s** |

Total: **about 10-45 s**, which fits beside the other gate stages.

Without the cuts, mutating every line of a 500-line diff gives hundreds of mutants, each run against the whole test suite. That is minutes to hours, the 820-mutant median above.

Stryker's own figures for its main speedups:
- mutation switching (all mutants compiled in once, with one active per run): 20-70% faster
- per-test coverage: "40% and 60%" faster

[blog](https://stryker-mutator.io/blog/announcing-stryker-4-mutation-switching/)

### 3.4 How to cut runtime, in order of payoff

1. **Changed and covered lines only.** Uncovered changed lines get an immediate "untested" finding with no mutant to run. Google, Mull and PIT all do this, and PIT notes there is "little or no execution time cost" for uncovered lines.
2. **Arid-node suppression:** skip logging, debug output, configuration constants, dependency-injection wiring and similar. This is the biggest single cut, about 10x on top of 1 mutant per line.
3. **1 mutant per line**, choosing operators by past productivity.
4. **Test selection per mutant** from per-test coverage: Stryker `perTest`, mutmut stats, PIT coverage ordering. Run the fastest tests first and stop at the first kill.
5. **Type-check filter:** Stryker's TypeScript checker removes mutants that do not compile before any test runs.
6. **Incremental cache from main:** reuse results for unchanged code and tests (Stryker `--incremental`, mutmut function hashes, PIT `withHistory`).
7. **Hard cap and random sampling** when N is still too large. Gopinath et al. ([ISSRE 2015](https://ics.uci.edu/~iftekha/publication/how-hard-does-mutation-analysis-have-to-be-anyway)) show that a small constant random sample estimates the mutation score with statistically similar results whatever the program size. I could not verify the exact sample size they recommend; the PDF's TLS certificate failed. For a gate the question is per line, not a score, so use a cap, not a score estimate.

### 3.5 What a surviving mutant means, and the remaining AI work

- **Mutants as a stand-in for real faults:** mutants are coupled to **73% of real faults** (Just et al., FSE 2014, [paper](https://homes.cs.washington.edu/~mernst/pubs/mutation-effectiveness-fse2014.pdf)): 357 real faults in 5 Java programs. Weakest for algorithm-level faults.
- **Equivalent mutants:** a surviving mutant may be equivalent to the original, meaning it cannot be killed and so is not a test gap. This is the part that needs AI judgment.
  - Meta's ACH ([arXiv 2501.12862](https://arxiv.org/abs/2501.12862)) used Llama 3.1 70B as an equivalence detector.
  - Results: precision 0.79 and recall 0.47 as headline; **0.95 and 0.96 after stripping comments and dropping identical code**.
  - Of 9,095 LLM mutants that built and passed, 25% were syntactically identical to the original.
- **Token cost per survivor [estimate]:** about 1-2k (mutated hunk, surrounding function, change intent).
  - Google's median of 2 surviving mutants per change makes this about 2-4k tokens per verdict.
  - Capping triage at the 5 worst survivors bounds it at about 10k.
- **What it does not prove:** Ojdanic et al. (TOSEM 2023, [arXiv 2112.14566](https://arxiv.org/abs/2112.14566)) studied 10M mutants across 288 commits. **81% of commit-relevant mutants are outside the changed lines.**
  - So killing every mutant on the diff shows the changed lines are exercised and checked.
  - It does not show that callers and interactions the change affects are covered.
  - One mitigation is to add a small sampled set of mutants in direct callers of the changed functions, within the same cap.

---

## 4. Recommended approach for merge-gate

Every stage gets a wall-clock cap and a token cap. Deterministic tools run first, and AI handles only what is left.

1. **Coverage run (deterministic, 0 tokens).**
   - Run the repo's declared test command on the tests selected for the change, with coverage.
   - Output: which changed lines are covered, and per-test coverage where the native tool gives it.
   - Changed lines with no coverage: report a finding immediately.
2. **Diff-scoped mutation (deterministic, 0 tokens, cap about 45 s).**
   - Use the native tool when present:
     - Stryker with `mutate` line ranges, `perTest` and `--incremental` cached from main
     - mutmut with function patterns
     - Mull `gitDiffRef`
     - Stryker.NET `--since`
     - PIT with `targetClasses`
   - Otherwise use the generic tree-sitter mutator.
   - Always apply 1 mutant per line, arid filtering and a cap of about 7x files or 50 mutants.
3. **Deterministic PBT (0 tokens, cap about 20 s).**
   - For Python, CrossHair `diffbehavior` or Hypothesis ghostwriter `--equivalent` comparing the base and head versions of changed pure functions. Use ghostwriter `fuzz` for new functions.
   - Schemathesis in-process for changed HTTP endpoints when a schema exists.
   - Schema-derived arbitraries in TypeScript.
   - All runs derandomized.
4. **LLM properties (at most about 15k tokens, at most 3 functions, cap about 20 s run time).**
   - Only for changed functions where the change intent states a behaviour and step 3 found no property.
   - Two-stage prompt.
   - Filter by running on both base and head (section 2.3).
5. **AI triage of what is left (at most about 10k tokens).**
   - Surviving mutants: equivalent, or a real test gap.
   - Property counterexamples: matches the change intent, or a bug.

Total: under 30k tokens, and about 1-1.5 minutes of wall clock for the test stages. **[estimate]** This leaves little room for the other gate stages (type-check, lint, intent review). The planning sizer should confirm this on the benchmark.

---

## 5. Main risks

1. **Unsound LLM properties produce false blocks.** Only 26% of raw LLM properties are valid and sound, and the agentic study had 56% valid reports. Mitigation: the base/head differential filter, and never block on an LLM-written property alone.
2. **Diff-only mutation gives false confidence.** 81% of the mutants that matter to a commit are outside the diff. The gate should word this finding as "changed lines are guarded", not "change is safe".
3. **Uneven tool support breaks the language-agnostic promise.** Python has no diff or line scoping (mutmut works per function). JVM line scoping is commercial (Arcmutate). TypeScript has no ghostwriter. The generic tree-sitter fallback has coarse test selection, so it is slower and gives weaker signal.
4. **Runtime depends on the repo's tests.** Every number above assumes fast unit tests cover the changed code. If the only covering tests are slow integration tests, mutation will overrun its budget. The gate then needs a "skipped: tests too slow" outcome rather than a silent pass.
5. **Equivalent mutants cost tokens.** The cap on triage keeps this bounded, but the uncapped tail (99th percentile of 43 surviving mutants) would blow the budget.

## 6. Decisions for the owner (not made here)

- "No network" and non-Python apps: Schemathesis runs in-process only for Python ASGI and WSGI apps. A Node or Go HTTP app would need it to call `localhost`. Does a loopback call to the in-memory app count as "network" under the settled constraints? My recommendation: allow loopback only, with no external DNS. The alternative is in-process framework adapters, such as supertest-style injection for Node, driven by LLM-written properties.
- Should a surviving, non-equivalent mutant on a changed line **block** the merge, or be reported as a note? My recommendation: block only when the change intent is a bug fix and no test kills a mutant on the fix lines. Otherwise "merge with notes".

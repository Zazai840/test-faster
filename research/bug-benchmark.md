# Research: a bug benchmark for the merge gate

Question: which datasets of real regressions can benchmark the **Gate**, how to mine bug-introducing changes from our own repos, how to inject realistic bugs without making the benchmark easy, and how to score catch rate and false-block rate fairly against CI, E2E suites and the local AI-review pipeline.

Researched 2026-11-13. Sources are primary (papers, dataset cards, tool repos) unless marked "vendor".

## Direct answer

1. **No existing dataset is a ready-made merge-gate benchmark.** Almost every public dataset is organised around the *fix*, not the change that introduced the bug, and almost none ship clean (bug-free) PRs. We have to construct PR-shaped cases from them.
2. **Best real-bug source with change intent and multi-language coverage: the SWE-bench family**, especially SWE-PolyBench (JS/TS/Python/Java), Multi-SWE-bench (7 languages incl. TS/JS) and SWE-bench Multilingual (9 languages). They give issue text (change intent), the gold patch, hidden fail-to-pass tests and a Docker image per task. Their killer feature for us is that public agent submissions give, for the *same* issue, many alternative patches labelled resolved / unresolved by hidden tests - a natural supply of plausible-but-wrong PRs and plausibly-clean PRs with identical intent.
3. **Best real bug-introducing-change sources:** RegMiner (1,035 Java regressions with the regression-inducing commit), the FONTE bug-inducing-commit set for Defects4J (130 data points), and our own repos mined with SZZ plus `git bisect`.
4. **Injected bugs** are cheap and language-agnostic (universalmutator, Stryker, mutmut, LLM-seeded bugs as in SWE-smith), but must be filtered and embedded in realistic PRs or the benchmark becomes trivially easy.
5. **Scoring:** per-PR confusion matrix with a "right reason" requirement, a matched set of clean PRs, paired comparison (McNemar) against baselines run on the *same* PRs with the tests that existed *at that commit*, and an explicit note that CI's catch rate on historically merged bugs is near zero by construction (they were merged because CI passed).

## 1. Datasets

"BIC" = the bug-introducing change is included. "Intent" = issue or PR text is included. "Offline" = tests can run without network once the environment is built.

| Dataset | Languages | Size | Licence | BIC? | Intent? | Offline tests? |
|---|---|---|---|---|---|---|
| [Defects4J](https://github.com/rjust/defects4j) v3.0.1 | Java (Java 11) | 854 bugs, 17 projects | MIT | No (buggy/fixed pair only); see FONTE below | Bug report URL per bug | Yes after `./init.sh` downloads libs and repos |
| [FONTE BIC data](https://github.com/coinse/fonte) for Defects4J ([paper](https://arxiv.org/pdf/2212.06376)) | Java | 130 BICs (67 from Wen et al. + 70 curated - 7 overlap) | repo licence | **Yes** | Via Defects4J | Via Defects4J |
| [RegMiner](https://github.com/SongXueZhi/RegMiner) ([paper, ISSTA 2022](https://arxiv.org/abs/2109.12389)) | Java (Maven/Gradle) | 1,035 regressions, 147 projects | Apache-2.0 | **Yes** (regression-inducing commit + test) | Commit messages only | Build needs Maven/Gradle deps; offline after cache |
| [BugsInPy](https://github.com/soarsmu/BugsInPy) ([FSE 2020 demo](https://2020.esec-fse.org/details/esecfse-2020-tool-demos/26/BugsInPy-A-Database-of-Existing-Bugs-in-Python-Programs-to-Enable-Controlled-Testing)) | Python | 493 bugs, 17 projects | **None declared** on GitHub | No | Partial (fix commit) | `bugsinpy-compile` installs deps (network at setup); some projects such as youtube-dl test against live sites (inference - verify with `--network none`) |
| [BugsJS](https://bugsjs.github.io/) ([repo](https://github.com/BugsJS/bug-dataset)) | JavaScript (Node server-side) | 453 bugs, 10 projects | MIT | No | Yes, "each bug comes with its bug report" | Docker image; offline after `npm install` cached |
| [SWE-bench](https://github.com/SWE-bench/SWE-bench) full / Lite / Verified | Python, 12 repos | 2,294 / 300 / 500 test; 19,008 train | Harness MIT; dataset card lists none; upstream licences apply | No (fix PRs) | **Yes** (issue `problem_statement`, `hints_text`) | Docker per instance; harness wants x86_64, 120 GB disk, 16 GB RAM, 8 cores; arm64 "experimental" |
| [SWE-bench Multimodal](https://huggingface.co/datasets/princeton-nlp/SWE-bench_Multimodal) | JavaScript front-end | 102 dev + 510 test | as above | No | Yes, with screenshots | Docker |
| [SWE-bench Multilingual](https://www.swebench.com/multilingual.html) | C, C++, Go, Java, JS/TS, PHP, Ruby, Rust | 300 tasks, ~42 repos | MIT (HF card) | No | Yes | Docker |
| [Multi-SWE-bench](https://huggingface.co/datasets/ByteDance-Seed/Multi-SWE-bench) ([repo](https://github.com/multi-swe-bench/multi-swe-bench)) | Java, TS, JS, Go, Rust, C, C++ | 1,632 instances | Data CC0 (subject to upstream); code Apache-2.0 | No | Yes (PR title/body + resolved issues) | Docker |
| [SWE-PolyBench](https://huggingface.co/datasets/AmazonScience/SWE-PolyBench) ([repo](https://github.com/amazon-science/SWE-PolyBench)) | JS 1,017, TS 729, Python 199, Java 165 | 2,110 (+394 Verified, 500 stratified) | MIT | No | Yes; tagged Bug Fix / Feature / Refactoring | Dockerfile per instance |
| [SWE-bench Pro](https://arxiv.org/abs/2509.16941) ([HF](https://huggingface.co/datasets/ScaleAI/SWE-bench_Pro)) | Go, Python, JS, TS | 1,865 total, 41 repos; public 731 (v1) / 642 (v2) | Harness MIT; tasks under upstream (often copyleft) licences | No | Yes, plus explicit `requirements` and `interface` | Docker; recommended protocol cuts network |
| [SWE-rebench](https://huggingface.co/datasets/nebius/SWE-rebench) | Python, 3,400+ repos | 21,336 (6,542 filtered) | CC-BY-4.0 | No | Yes | 7,500 prebuilt images; continuously refreshed (helps against contamination) |
| [SWE-smith](https://huggingface.co/datasets/SWE-bench/SWE-smith) ([paper](https://arxiv.org/html/2504.21798v2)) | Python, 128 repos | 50,137 *synthetic* bugs | MIT | **Yes** (the bug diff itself) | LM-generated issue text | Docker |
| [Bears](https://github.com/bears-bugs/bears-benchmark) | Java | 251 originally, 118 currently reproducible | GPL-3.0 | No | No | Travis-derived; not stated |
| [GitBug-Java](https://github.com/gitbugactions/gitbug-java) ([paper](https://arxiv.org/abs/2402.02961)) | Java | 199 bugs, 55 repos, all from 2023 | MIT | No | No | Replays GitHub Actions via custom `act`; ~130 GB download, peaks at 240 GB |
| [BugSwarm](https://github.com/BugSwarm/bugswarm) | Java, Python | 3,091 fail-pass CI pairs | NOASSERTION | No (CI fail/pass pair) | No | Docker. A [critical review](https://arxiv.org/pdf/1905.09375) found only 112 (3.6%) suitable as bug benchmarks |
| [ManyBugs](https://repairbenchmarks.cs.umass.edu/) | C | 185 defect scenarios | BSD | No | No | VM or Docker (BugZoo) |
| [ApacheJIT](https://arxiv.org/abs/2203.00101) | Java, 14 Apache projects | 106,674 commits (28,239 bug-inducing via SZZ) | public | **Yes** (SZZ-labelled, noisy) | Commit message | **No tests** - metadata only |
| [SWR-Bench](https://arxiv.org/html/2509.01494v1) (code review, FSE 2026) | Python, 12 projects | 1,000 PRs: **500 with defects + 500 clean** | release via anonymous link | Review-driven "change-points" | Yes (PR) | No execution |
| [CodeReviewer](https://arxiv.org/abs/2203.09095) | 9 languages | large (hunk-level) | MIT (CodeBERT repo) | No | Review comments | No execution |
| [Greptile benchmark](https://www.greptile.com/benchmarks.md) (vendor) | Python, TS, Go, Java, Ruby | 50 real bugs re-opened as PRs | not stated | **Yes** (re-created BIC PRs) | Original PR | n/a; **did not score false positives** |

Observations that matter for the design:

- **TypeScript/JavaScript coverage** comes almost only from SWE-PolyBench (1,746 JS/TS), Multi-SWE-bench (9 JS/TS repos incl. vuejs/core, material-ui, axios, express), SWE-bench Multimodal and BugsJS. Python is well covered (SWE-bench family, BugsInPy).
- **Only SWR-Bench ships clean PRs** with a deliberate 50/50 split, precisely to measure false positives. Vendor benchmarks (Greptile) ignore false positives; Qodo's reports precision but is vendor-run ([page](https://www.qodo.ai/?p=21766)). Treat all vendor numbers as marketing.
- **SWE-bench tests are weaker than they look.** PatchDiff found 29.6% of "plausible" (test-passing) patches behave differently from the gold patch, inflating resolution rates by 6.2 points ([arXiv 2503.15223](https://arxiv.org/abs/2503.15223)); SWE-bench+ found 31.08% of passed patches suspicious due to weak tests and 32.67% with the solution leaked in the issue ([arXiv 2410.06992](https://arxiv.org/abs/2410.06992)). So "resolved" agent patches are only *probably* clean.
- **Time-travel leakage is real.** In Defects4J, 55% of fault-triggering tests were added with the fix and 22% were modified after the bug report ([arXiv 2310.19139](https://arxiv.org/abs/2310.19139)). If the gate (or the CI baseline) can see tests written after the bug, it is cheating.
- **Contamination:** over 94% of SWE-bench issues predate model cutoffs ([arXiv 2410.06992](https://arxiv.org/abs/2410.06992)). Prefer SWE-rebench (continuously refreshed), SWE-bench Pro held-out style data, and our own private repos for the headline number.
- **Offline:** none of the harnesses *enforce* no-network. We run every container with `docker run --network none` and drop instances whose gold patch does not pass offline (this also enforces our "app in memory with fakes" ceiling).

## 2. Mining bug-introducing changes from our own repos

**SZZ** (Sliwerski, Zimmermann, Zeller 2005): find bug-fixing commits (issue links / commit-message keywords), `git blame` the lines the fix deleted or changed, and call the last commits that touched them bug-introducing.

Accuracy against a developer-informed oracle (Rosa et al., ICSE 2021, 1,115 bug-fixes where developers named the inducing commit; [arXiv 2102.03300](https://arxiv.org/abs/2102.03300), Table V, no issue-date filter):

| Variant | Recall | Precision | F1 |
|---|---|---|---|
| B-SZZ (plain blame) | 0.69 | 0.39 | 0.50 |
| AG-SZZ (annotation graph, ignore cosmetic) | 0.60 | 0.45 | 0.52 |
| MA-SZZ (also skip meta-changes such as merges) | 0.64 | 0.36 | 0.46 |
| **R-SZZ** (AG + only the latest change) | 0.57 | **0.66** | **0.61** |
| RA-SZZ* (refactoring-aware, Java only) | 0.45 | 0.35 | 0.39 |
| SZZ Unleashed | 0.72 | 0.09 | 0.16 |

Adding an issue-date filter (drop candidates authored after the bug report) lifts R-SZZ to **0.73 precision**. LLM4SZZ (ISSTA 2025) adds 2.4-9.2 points of precision over these baselines ([arXiv 2504.01404](https://arxiv.org/abs/2504.01404)). Implementation: [PySZZ v2](https://github.com/grosa1/pyszz_v2) (GPL-3.0) implements B/AG/L/R/MA/RA and newer variants; RA-SZZ is Java-only.

Known blind spots: fixes that only *add* lines have nothing to blame; a 1-hour blame timeout exists for pathological histories; any label is at best ~70% precise.

**Recommended procedure for our repos (language-agnostic, highest precision):**

1. Find fix commits: PRs linked to issues labelled bug, commits matching fix/bug/regression, and reverts (`git log --grep '^Revert'` gives near-certain bad changes plus their intent).
2. Run R-SZZ with the issue-date filter (PySZZ, or plain `git blame -w -M -C --ignore-revs-file` for a language-agnostic approximation) to get candidate bug-introducing commits.
3. **Confirm by test, not blame:** take the fix's regression test, port it back, and `git bisect run` over the candidates' range. A candidate counts only if the test passes before it and fails after it. This is the approach RegMiner and FONTE use (failure-based BIC identification; FONTE reduces the search by failure coverage, [arXiv 2212.06376](https://arxiv.org/pdf/2212.06376)).
4. Lift the confirmed commit to its PR (GitHub API: commits -> pulls) to recover change intent text, and record the regression test as the **hidden oracle** - never shown to the gate or to baselines.
5. Hand-review every case (cheap at our scale, ~10 minutes each). Drop cases whose bug depends on network, real services or a browser, since the gate is not allowed to see those anyway; keep them in a separate "out of scope" bucket so we can report what the gate cannot catch by design.

## 3. Injecting realistic bugs without making the benchmark easy

**Operators.** Mutants are a defensible proxy: 73% of 357 real faults were coupled to mutants; conditional-operator replacement, relational-operator replacement and statement deletion couple most often; ~17% of real faults are not coupled to any mutant ([Just et al., FSE 2014](https://homes.cs.washington.edu/~mernst/pubs/mutation-effectiveness-fse2014.pdf)). But mutation score only weakly correlates with real-fault detection once suite size is controlled ([Papadakis et al., ICSE 2018](https://orbilu.uni.lu/handle/10993/34950)) - so mutants alone must not be the headline.

Tools:
- Language-agnostic: [universalmutator](https://github.com/agroce/universalmutator) (regex or Comby rules; C, C++, Java, JS, Python, Go, Rust, Swift, ...; trivial-compiler-equivalence filtering). A tree-sitter-based variant of the same idea fits our parser choice.
- Per stack: StrykerJS (JS/TS), mutmut / cosmic-ray (Python), PIT (Java).
- LLM-seeded: [LLMorpheus](https://arxiv.org/abs/2404.09952) (JS; LLM fills placeholders, yields mutants resembling real bugs that StrykerJS cannot produce); Meta's ACH ([arXiv 2501.12862](https://arxiv.org/html/2501.12862v1)) uses an LLM equivalent-mutant detector with precision 0.79 / recall 0.47, rising to 0.95 / 0.96 after stripping comments; [SWE-smith](https://arxiv.org/html/2504.21798v2) gives four strategies with measured yields (share that break at least one test): LM Modify 56%, LM Rewrite 35%, Procedural AST 40%, PR Mirror (LM reverts a real PR) 34%, at 0-5.5 US cents per candidate. PR Mirror instances were the most realistic by SWE-smith's own proxy.

**Anti-easiness rules** (each closes a shortcut we would otherwise reward):

1. **Hidden oracle, not visible suite.** A bug is valid only if a *held-out* test kills it (the fix's regression test, a later test, or a differential test against the original on generated inputs). That also removes equivalent mutants by construction.
2. **Stratify by the visible suite.** Record whether the repo's own test suite at that commit already fails. Report catch rate separately for "CI-visible" and "CI-silent" bugs. CI-silent is where the gate must earn its keep; if only CI-visible bugs exist, every tool that runs tests scores 100%.
3. **Drop trivially detectable bugs from the headline**: anything the type checker, compiler or linter rejects goes to a separate "trivial" stratum.
4. **Strip giveaways:** LLM-seeded bugs often leave comments, renamed identifiers or odd formatting. Strip comments added by the injector, run the repo's formatter, and reject diffs that touch tests.
5. **Embed in a real PR.** Inject into a real merged PR's diff (Qodo's approach) or combine with a clean PR so the bug is one hunk inside a 100-400 line change, matching our "<500 changed lines" target. A 3-line isolated mutant diff is unrealistically easy to review.
6. **Give honest intent text.** The PR description is the original PR's description, never the injector's description of the bug.
7. **Hold out a sealed test set.** Develop the gate on a dev split; touch the sealed split only for reported numbers. Refresh with post-cutoff PRs to limit contamination.
8. **Mix.** Headline = real bugs; injected bugs widen coverage of bug classes (one stratum per operator family and per check: correctness, intent match, safety, compatibility, code health).

## 4. Scoring fairly

**Unit:** one PR, one **Verdict**. For gating, *block* = positive; *merge* and *merge with notes* = negative. Track "caught in notes but not blocked" as a secondary metric.

**Metrics per tool (gate, CI, E2E suite, local AI-review pipeline):**
- **Catch rate** = blocked buggy PRs / buggy PRs. Strict variant: a catch counts only if a **Finding** points at the buggy hunk or a hidden-oracle-equivalent behaviour (right for the right reason), as Greptile and SWR-Bench do.
- **False-block rate** = blocked clean PRs / clean PRs.
- **Seconds** (wall clock, median and p95) and **tokens** (median and p95) per PR.
- Precision at a stated deployment prevalence, not at the benchmark's 50/50 mix. For reference, SZZ labels 26.5% of ApacheJIT commits as bug-inducing (28,239 / 106,674), an upper-bound style estimate.

**Clean PRs (needed for false blocks), roughly equal in number to buggy ones:**
- Gold patches from SWE-bench-family tasks (clean by definition, with intent).
- Merged PRs from our repos not reverted and not flagged by SZZ as bug-introducing within 6-12 months (label noise of a few percent; audit a sample).
- Agent patches that pass the hidden tests *and* match the gold patch on a differential check (PatchDiff style), to get clean-but-unfamiliar code.
- Benign "scary-looking" PRs (large refactors, dependency bumps, renames) - this is where false blocks live.

**Same-intent pairs.** For each SWE-bench-family issue, use the gold patch (clean), a resolved-and-differentially-equivalent agent patch (clean), and an unresolved agent patch (buggy or intent mismatch). The intent text is identical, so the gate cannot score by reading the description alone.

**Fair baselines:**
- Run each baseline on the **same PRs**, with the tests and E2E specs that existed **at the PR's head commit** (no future tests - the Defects4J leakage finding above).
- **CI on mined real bugs is censored**: those bugs were merged, so historical CI passed. Report it, but compare the gate to CI mainly on injected and same-intent cases, and say so in every chart.
- Run baselines 3 times on the clean base to identify flaky tests; a block caused by a flaky test is a false block for that baseline (and likewise for the gate).
- E2E baseline is allowed network and browser (that is what it is); the gate is not. Report the gate's catch rate alone and "gate + CI" combined, since the gate will run alongside CI, not instead of it.

**Statistics.** Use Wilson 95% intervals and paired McNemar tests (same PRs, two tools). Sample-size guide (Wilson, computed here):

| n per class | 5% false-block rate CI | 70% catch rate CI |
|---|---|---|
| 100 | 2.2-11.2% | 60.4-78.1% |
| 300 | 3.1-8.1% | 64.6-74.9% |
| 500 | 3.4-7.3% | 65.8-73.9% |

So ~300 buggy + ~300 clean PRs gives a usable headline; each reported stratum needs at least ~50.

## Recommended approach

1. **Real, multi-language, with intent (headline):** ~200 same-intent triples from SWE-PolyBench + Multi-SWE-bench + SWE-bench Multilingual (JS/TS and Python first), run with `--network none`, hidden tests as the oracle, plus agent patches from public SWE-bench submissions ([SWE-bench/experiments](https://github.com/SWE-bench/experiments) links each submission's `all_preds.jsonl` and per-instance logs).
2. **Real bug-introducing PRs:** our own repos via R-SZZ + `git bisect` confirmation (target 50-100), plus RegMiner and FONTE/Defects4J for a Java check.
3. **Injected:** universalmutator/tree-sitter operators + LLM-seeded (SWE-smith-style LM Modify, LM Rewrite, PR Mirror), filtered by the eight anti-easiness rules, embedded in real PRs.
4. **Clean PRs** in equal number from gold patches and unreverted, SZZ-clean merged PRs, including deliberately scary-looking ones.
5. **Score** catch rate (strict and loose), false-block rate, seconds and tokens per PR with Wilson intervals and McNemar against each baseline on identical PRs; report CI-visible / CI-silent / trivial strata and keep a sealed test split.

## Main risks

- **Label noise both ways:** SZZ is at best ~70% precise and SWE-bench hidden tests let ~30% of plausible patches diverge, so some "clean" PRs are buggy and some "buggy" PRs are fine. Mitigate with test-confirmed labels and manual audit; report the audited error rate.
- **Easy-benchmark trap:** if most bugs are CI-visible or type errors, every test-running tool looks perfect. Stratification is mandatory.
- **Compute:** SWE-bench needs ~120 GB disk and wants x86_64 (arm64 laptops are experimental); GitBug-Java peaks at 240 GB. Pick a subset and prebuilt images.
- **Licences:** BugsInPy declares no licence; Bears is GPL-3.0; SWE-bench Pro tasks are under upstream licences. Use them for internal evaluation only and do not redistribute.
- **Contamination:** public bugs may be memorised by the model behind the gate; keep a post-cutoff and private-repo slice and compare.

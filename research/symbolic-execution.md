# Research: symbolic execution, concolic testing and bounded model checking in a 2-minute merge gate

Question: can symbolic or concolic execution, or bounded model checking (BMC), check the changed functions of a PR within seconds? Which tools, which bug classes, what time limits, what false-positive rates, which language gaps, and how badly does it degrade on I/O-heavy or dynamic code? Is it worth a place in the gate, and for which functions?

Date: 2026-11-13. Hands-on numbers come from CrossHair v0.0.113 on CPython 3.12, Apple M3 (8 cores). The experiment files stayed in an uncommitted `tmp/` folder.

## Direct answer

- **Whole-PR or whole-program symbolic execution in seconds: no.** The published results all use budgets of minutes to hours per target. KLEE ran 60 min per Coreutils tool [K1]. KATCH, which aims symbolic execution at patches, ran 10 to 15 min per patch target [KA1]. ExpoSE needed 170 s to 902 s per JS library [E1]. SV-COMP gives 15 CPU-min per task [SV].
- **A function-level, time-boxed solver pass over small, pure, typed changed functions: yes. It takes seconds, but only one stack has a usable tool today.** In Python, CrossHair found every shallow bug I planted in 0.14 to 1.2 s wall, and `diffbehavior` between the old and new versions of a function ran in 0.3 s. A function with no bug always burns its whole budget, so the gate must cap it per function (3 s works) and run functions in parallel.
- **JS/TS has no maintained engine you could ship.** ExpoSE's last commit was July 2024. It pins Node 21.7.2 and needs clang, and it does not mention TypeScript [E2]. Java (JBMC, Symbolic PathFinder), C/C++ (KLEE, ESBMC, CBMC) and Rust (Kani) have strong tools. Each one needs a compile step, a bitcode or bytecode toolchain, or hand-written harnesses, so none of them fits a "drop in and go" gate.
- **Verdict: include it as an optional, report-only tier behind the language adapter.** Run it only on eligible functions (see "Which functions"). Its findings feed the AI judge as concrete, reproducible counterexamples. It never blocks on its own, and its silence never counts as evidence.

## Recommended approach (for the gate)

1. **Pick eligible functions deterministically, with no LLM.** Use tree-sitter plus the import graph. The function must be changed by the PR and have full type annotations. Its parameters must be primitives, containers, enums or dataclasses of those. Nothing in its transitive call graph inside the repo may import I/O (`os`, `socket`, `sqlite3`, `requests`, `time`, `random`, ORMs and the like). It must be at module top level. CrossHair only analyzes top-level definitions [CH-L].
2. **Run a differential check first: old version against new version.** Use `crosshair diffbehavior old.f new.f --per_condition_timeout 3 --exception_equivalence SAME_TYPE` [CH-D], with the old version taken from a `git worktree` at the merge base, as the docs suggest. This needs no contracts, and it targets exactly what a PR changes: behaviour.
3. **Run a crash check second.** Use `crosshair check` with the `asserts` analysis kind (prepend `assert True`), or with existing PEP 316 or icontract contracts. It finds uncaught exceptions [CH-K].
4. **Re-execute every counterexample concretely, in the gate's sandbox.** CrossHair already runs real code, but replaying the input in a plain interpreter makes the finding reproducible evidence and removes any proxy artefacts.
5. **Spend tokens only on triage.** For each confirmed divergence, the LLM gets the input, the old and new outputs and the change intent. It decides "intended" or "regression". That costs roughly 300 to 800 tokens per divergence; cap it at 5 divergences, about 4k tokens in total.
6. **Budget.** A typical PR changes about 5 to 15 functions, and maybe a third to a half are eligible. At 3 s each across 8 cores, that is about 5 to 10 s wall. Put a hard 30 s ceiling on the whole tier and kill whatever is left.
7. **Never read "no counterexample" as a pass.** It is not a proof [CH-D][CH-L]. And as shown below, CrossHair sometimes says nothing because it could not build the inputs at all.

For other stacks, add each tool only once the adapter for that language exists, and keep the same contract (counterexample in, LLM triage, report-only):

- **Rust:** Kani, but only on `#[kani::proof]` harnesses the repo already has.
- **C:** ESBMC or CBMC `--function`.
- **Java:** JBMC.
- **JS/TS:** property-based testing (fast-check) is the realistic stand-in. It is not symbolic.

## Hands-on results (CrossHair 0.0.113)

All runs used `--per_condition_timeout` (CPU seconds) as shown. Wall times are from `/usr/bin/time`.

| # | Function (kind) | Result | Wall time | Classification |
|---|---|---|---|---|
| 1 | `discount_old` vs `discount_new` (`>=` became `>`) | diffbehavior: `(total=-10, qty=10)` gives -9 vs -10 | 0.36 s | true positive (boundary regression) |
| 2 | `clamp_old` vs `clamp_new` (equivalent refactor) | "No differences found ... All paths exhausted" | 0.27 s | correct, and exhaustive here |
| 3 | `average(xs)` = `sum/len` | `ZeroDivisionError` on `[]` | 0.14 s | true positive (crash) |
| 4 | `parse_kv` (split on `=`) | `ValueError` unpack on `''` | 0.21 s | true positive (crash) |
| 5 | `route(path)` magic-string branch | found `'/api/v2/admin'` | 0.59 s | true positive (deep string condition) |
| 6 | `is_slug` regex | found `'--'` | 0.47 s | true positive (regex edge) |
| 7 | `bucket` via `sha256` (1 in 8 chance) | found by concrete fallback | 1.18 s | true positive, but by luck |
| 8 | `bucket2` via `sha256` (1 in 65,536 chance) | nothing reported | 22.8 s (20 s budget) | missed (native hash) |
| 9 | `dedupe_sorted` (correct) | nothing | 22.0 s (20 s budget) / 4.1 s (3 s budget) | correct, but consumes the whole budget |
| 10 | nonlinear `a^3+b^3 == c^3-1` | nothing | **48.5 s on a 20 s budget**; 3.3 s on a 3 s budget | budget overrun of 2.4x at large budgets |
| 11 | `order_total(Order)` dataclass | `items=[-1]` returns -1 | 0.27 s | real but probably irrelevant (negative price) |
| 12 | `load_limit` reads a file | `FileNotFoundError` | 0.16 s | environment noise |
| 13 | `withdraw(repo: UserRepo)` using sqlite | `SideEffectDetected: sqlite3.connect` | 0.51 s | blocked safely, no analysis |
| 14 | `stamp()` uses `time.time()` | found with `patch_to_return({time.time: [0.5]})` | 0.46 s | not realistic (clock = 0.5) |
| 15 | `untyped(a, b)` with no annotations | `TypeError` with `b=''` | 1.64 s | false positive (no types) |
| 16 | `withdraw_buggy(repo: Repo)` where `Repo` is a `typing.Protocol` | **nothing, exit code 0**; verbose log shows "Protocols cannot be instantiated" 150+ times | 5.6 s | **silent false negative** |
| 17 | same bug, through a 6-line `FakeRepo` harness | found `h_withdraw(1, 0, 2)`, but flagged the intended `ValueError` first | 0.56 s | works with a fake; needs a `raises:` annotation |

Takeaways:

- **When a bug is shallow, it is found in about 1 s or less.** When there is no bug, the cost equals the budget. At large budgets the budget is soft: run 10 used 2.4 times its limit, so the gate needs its own wall-clock kill.
- **Of the 13 reports in total, 6 were relevant bugs and 5 were real-but-irrelevant or noise** (runs 11, 12, 14, 15, 17), so roughly 45% noise without the eligibility filter. All 5 noisy ones would be removed by the filter or by an LLM check against the change intent (filter: I/O, clock, no annotations; LLM: negative price, intended exception). The other 2 reports are the correct "functions are the same" verdict for run 2 and the safe `SideEffectDetected` block in run 13. The small sample means this is indicative, not a measured rate.
- **The dangerous failure is silence (run 16).** The gate must parse the verbose output for `CrosshairUnsupported` or zero completed iterations, and report "not analyzed", not "clean".

## Tool-by-tool assessment

### Python: CrossHair (recommended for v1)

- Mechanism: concolic execution. It calls the real function with symbolic proxy objects backed by Z3 (`SymbolicInt`, `SymbolicDict`, ...). A value becomes concrete only when a branch needs it. It works with no AST or bytecode analysis [CH-H].
- Inputs: a function needs type annotations. Contracts can be plain asserts, PEP 316 docstrings, icontract or deal [CH-K]. `diffbehavior` needs no contracts [CH-D].
- Limits, quoted from the docs: "the absence of a counterexample does not guarantee that the property holds". "Only deterministic behavior can be analyzed". Side-effect protection uses `sys.addaudithook` and "will not prevent actions taken by C-based modules". "Only function and class definitions at the top level are analyzed" [CH-L]. For diffbehavior: "the absence of an example difference does not guarantee that the functions are equivalent", and arguments must be deep-copyable and comparable [CH-D].
- Maintenance: active. The last commit was 2026-11-11, there are about 1.35k stars, and the version is still pre-1.0 (v0.0.113). The `hypothesis-crosshair` backend [HC] can drive existing Hypothesis tests symbolically, but it is early-stage (v0.0.32).

### Python: ESBMC-Python

- Mechanism: BMC. It parses Python with the CPython parser, adds inferred types, lowers to ESBMC's IR, and solves with SMT [EP1][EG].
- It finds division by zero, index errors, arithmetic overflow and assertion failures [EG]. Its 85-program suite ran in about 31.6 ms per program on average, and it found one real division-by-zero in the Ethereum consensus spec [EP1]. Those are tiny programs.
- Gaps: the supported-features page lists many refusals. Sequence, mapping and class `match` patterns are refused, along with `re` group capture, `send`/`throw` on generators, multiple inheritance in threads, most `**kwargs` forwarding, and NumPy object dtypes [EP2]. Loop unwind defaults to 1 [EP3]. It is good for numeric kernels and poor for ordinary application code.
- I could not run it here: the auto-mode classifier blocked execution of the downloaded binary. Its numbers come from docs and the paper only.

### JavaScript / TypeScript: ExpoSE and relatives (not viable)

- ExpoSE is a dynamic symbolic execution engine on Jalangi2 and Z3, with string and regex support [E1][E2]. In its SPIN'17 evaluation on minimist, semver and validator (300 to 1500 LOC), it reached 56 to 80% coverage and found 1 new bug. Total run time was 170 to 902 s using up to 128 concurrent workers, the median test case took 1.3 to 3.0 s, and "some solver queries require several minutes" [E1].
- Status: the last commit was 2024-07-02. It requires "node version v21.7.2 (other versions may work but are not tested)", clang, make and Python [E2]. There is no TypeScript support; you would have to analyze compiled JS. Jalangi2 itself still gets occasional fixes [J2].
- Conclusion: there is no maintained, CI-grade symbolic engine for JS/TS. Use fast-check property tests with generated arbitraries for the "differential old vs new" idea (random, not symbolic), and rely on tsc and the other deterministic tools.

### C / C++ / LLVM: KLEE family

- KLEE ran 60 min per tool over all 89 Coreutils. It reached 84.5% overall line coverage (median 94.7% per tool) and found 56 serious bugs across 452 programs. Environment handling uses about 2,500 lines of models for roughly 40 syscalls [K1].
- KATCH (KLEE aimed at patches) is the closest published analogue to this gate. Over about 6 years of patches in 19 GNU programs, it ran 10 to 15 min per target and raised patch coverage from 31% to 52%. It found 15 distinct crash bugs. Plain KLEE run for 30 min per revision covered only 2 targets. 389 targets were unreachable because they sat behind indirect calls [KA1].
- Fit: everything must compile to LLVM bitcode, and the useful budgets are about 100 times the gate's. Too slow for the gate.

### C / C++ / Python / Java/Kotlin / Solidity: ESBMC; C: CBMC

- ESBMC is a multi-front-end BMC (Clang for C/C++/CUDA, Soot/Jimple for Java/Kotlin, CPython 3.10 parser for Python, Solidity). It checks pointer safety, array bounds, division by zero, overflow, memory leaks and assertions, with k-induction and incremental BMC [EG]. It is active (last commit 2026-11-13). Per-function checks of small C functions with low unwind bounds usually take seconds. Loops and heap-heavy code blow up with the bound.

### Rust: Kani

- It proves or refutes harnesses (`#[kani::proof]`, `kani::any()`). It checks panics, overflow, UB and failing asserts [KN1].
- Loops need unwind bounds. In its tutorial, a `LIMIT` of 100 took about 3.8 s, and the tutorial says "Kani often scales poorly to 'big string problems' like parsing" [KN2]. Concurrency and `await` are out of scope. Some float operations are over-approximated, so "the verifier can raise errors that cannot actually happen", which is a genuine spurious-failure source [KN3].
- In industry, Firecracker runs 27 harnesses in about 15 min in CI. It took 7 months of work and needed `clock_gettime` and `sysconf` stubs, and it found 6 bugs [KN4]. The Rust std verification campaign verifies more than 16,000 harnesses per change [KN5].
- Fit: run only the harnesses the repo already declares, the same way as its other declared checks. Do not generate harnesses.

### Java: JBMC, Symbolic PathFinder (SPF)

- JBMC: BMC for bytecode. It checks runtime exceptions and assertions, uses a Java runtime model (`core-models.jar`), and needs `--unwind` [JB1][JB2].
- SPF: symbolic execution on Java PathFinder. The main repo was last pushed 2026-02 [SPF], and it requires an old JDK (Java 8 per community mirrors).
- Fit: both are heavyweight, with JVM start-up, library models and bounds. Defer.

### Differential and equivalence checking (old vs new function)

- This is the variant that fits a PR best, because it needs no spec: the old version is the spec, and the change intent says which differences are wanted.
- Approaches: differential symbolic execution [DSE], shadow symbolic execution (old and new run in one engine, evaluated on 18 annotated CoREBench Coreutils patches) [SH], ARDiff (iterative abstraction of common code) [AR], and PASDA [PA].
- Accuracy on hard benchmarks is modest. ARDiff solves 86% of equivalent and 53% of non-equivalent pairs, while earlier method-level tools reach 47% and 38 to 69% [AR]. PASDA classifies 61 to 74% correctly at timeouts from 10 s to 3600 s [PA]. EQBench has 147 equivalent and 125 non-equivalent C/Java pairs, and 57% of them contain nonlinear arithmetic [EQ]. All of these are Java/C research prototypes.
- In practice, for Python, CrossHair `diffbehavior` is the only shipped, maintained tool. It finds differences, but it does not prove equivalence [CH-D].

## Bug classes found

- Uncaught exceptions: division by zero, index or key errors, unpacking errors, None dereference (runs 3, 4 and 17; [EG]).
- Boundary and off-by-one changes in refactors, through old-vs-new differencing (run 1).
- Contract and assert violations, including deep string and regex conditions (runs 5 and 6; ExpoSE regex support [E1]).
- C, C++ and Rust: integer overflow, out-of-bounds access, null or misaligned pointers, undefined behaviour, memory leaks [EG][KN1].
- Not found: performance problems, concurrency bugs (Kani and CrossHair exclude them), anything behind native code or hashing (run 8), anything needing real I/O, and wrong behaviour that no contract describes and no old version shows.

## False positives

- **Solver-level spurious reports are about 0 for concolic tools.** Every report comes with a concrete input that was actually executed. BMC tools can be spurious where they over-approximate: Kani floats [KN3], and unwinding-assertion failures that only mean the bound is too low [KN2].
- **The dominant false positive is the "real but irrelevant" report.** The function does fail on an input real callers never pass: negative prices, untyped arguments, a clock reading of 0.5, an intended exception. That was about 5 of 13 reports in my probes. The eligibility filter plus intent-aware LLM triage is the fix.
- **The diff mode is cleaner than the contract mode,** because "old and new differ" is almost always worth one look against the change intent.

## How it degrades

- **I/O-heavy code:** CrossHair blocks it with `SideEffectDetected` (run 13), but C extensions bypass the audit hook [CH-L]. KLEE needs hand-written environment models [K1], and Kani needs stubs [KN4]. With fakes passed in through a small harness it works again (run 17), but someone has to write the harness. Generating one with the LLM costs about 1 to 2k tokens per function; that is not recommended for v1.
- **Dynamic code:** untyped parameters lead to noise (run 15). Protocols and abstract dependencies lead to silent non-analysis (run 16). Iterators, identity checks and nondeterminism are explicitly unsupported [CH-L]. For JS, eval, require and async are handled only partially [E1].
- **Native code, hashing, nonlinear arithmetic:** CrossHair falls back to random concrete values, so finding a bug becomes luck (runs 7 and 8). The solver can overrun its budget (run 10).
- **Loops and big inputs:** BMC cost grows with the unwind bound [KN2]. Path explosion is why the published budgets are minutes to hours.

## Expected contribution on a bug benchmark

- I found no published evaluation of CrossHair, or of any Python symbolic tool, on real-bug benchmarks such as BugsInPy.
- For calibration, look at automated unit-test generators (Randoop, EvoSuite, Agitar) on Defects4J. Together they detected 55.7% of 357 real faults, but only 19.9% of the individual generated suites detected their fault [SJ].
- A symbolic tier limited to pure, typed functions should be expected to catch a small, high-precision slice: crash and boundary regressions in logic-heavy code. It should not be expected to move the gate's overall recall much. Measure it on the gate's benchmark before switching it on by default.

## Main risk

**False confidence from silence.** `no differences found` is not equivalence [CH-D]. CrossHair exits 0 even when it never built a valid input (run 16), and coverage is opportunistic around native code. If the gate reads silence as "checked", it will merge regressions that it claims to have analyzed. Mitigations:

- Report "analyzed / not analyzed / exhausted" separately.
- Parse the logs for `CrosshairUnsupported`.
- Never let this tier upgrade a verdict.

Secondary risks:

- The noise rate without strict eligibility filtering.
- Budget overruns, which need a wall-clock kill.
- Executing untrusted PR code, which must run inside the gate's sandbox with no network.

## Sources

- [CH-L] CrossHair, Limitations: https://crosshair.readthedocs.io/en/latest/limitations.html
- [CH-D] CrossHair, diffbehavior: https://crosshair.readthedocs.io/en/latest/diff_behavior.html
- [CH-K] CrossHair, Kinds of contracts: https://crosshair.readthedocs.io/en/latest/kinds_of_contracts.html
- [CH-H] CrossHair, How does it work: https://crosshair.readthedocs.io/en/latest/how_does_it_work.html
- [HC] hypothesis-crosshair: https://github.com/pschanely/hypothesis-crosshair
- [EP1] Farias et al., ESBMC-Python, ISSTA/ECOOP 2024 tool demo: https://arxiv.org/html/2407.03472v1
- [EP2] ESBMC-Python supported features: https://esbmc.github.io/docs/python/supported-features/
- [EP3] ESBMC-Python usage: https://esbmc.github.io/docs/python/usage
- [EG] ESBMC repository: https://github.com/esbmc/esbmc
- [E1] Loring, Mitchell, Kinder, ExpoSE, SPIN 2017: https://www.plai.ifi.lmu.de/publications/spin17-expose.pdf
- [E2] ExpoSE repository: https://github.com/ExpoSEJS/ExpoSE
- [J2] Jalangi2 repository: https://github.com/Samsung/jalangi2
- [K1] Cadar, Dunbar, Engler, KLEE, OSDI 2008: https://www.usenix.org/legacy/event/osdi08/tech/full_papers/cadar/cadar.pdf
- [KA1] Marinescu, Cadar, KATCH, ESEC/FSE 2013: https://srg.doc.ic.ac.uk/files/papers/katch-fse-13.pdf
- [KN1] Kani tutorial, first steps: https://model-checking.github.io/kani/tutorial-first-steps.html
- [KN2] Kani tutorial, loop unwinding: https://model-checking.github.io/kani/tutorial-loop-unwinding.html
- [KN3] Kani Rust feature support: https://model-checking.github.io/kani/rust-feature-support.html
- [KN4] Kani blog, Firecracker: https://model-checking.github.io/kani-verifier-blog/2023/08/31/using-kani-to-validate-security-boundaries-in-aws-firecracker.html
- [KN5] Kani: A Model Checker for Rust (2026): https://arxiv.org/abs/2607.01504
- [JB1] JBMC README: https://github.com/diffblue/cbmc/blob/develop/jbmc/README.md
- [JB2] JBMC page: https://www.cprover.org/jbmc/
- [SPF] Symbolic PathFinder: https://github.com/SymbolicPathFinder/jpf-symbc
- [SV] SV-COMP 2025 rules: https://sv-comp.sosy-lab.org/2025/rules.php
- [DSE] Person, Dwyer, Elbaum, Pasareanu, Differential Symbolic Execution, FSE 2008: https://bibtex.github.io/FSE-2008-PersonDEP.html
- [SH] Shadow symbolic execution (ICSE 2016, TOSEM 2018): https://srg.doc.ic.ac.uk/projects/shadow/
- [AR] Badihi et al., ARDiff, ESEC/FSE 2020: https://resess.github.io/artifacts/ARDiff
- [PA] PASDA: https://arxiv.org/abs/2311.08071
- [EQ] Badihi, Li, Rubin, EQBench: https://personal.ntu.edu.sg/yi_li/files/Badihi2021EAD.pdf
- [SJ] Shamshiri et al., Do automatically generated unit tests find real faults?, ASE 2015: https://homes.cs.washington.edu/~rjust/publ/unit_test_generation_effectiveness_ase_2015.pdf

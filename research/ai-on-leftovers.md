# AI on the leftovers: how merge-gate should spend 30k tokens

Research ticket for `merge-gate`. Question: what does current evidence say about the precision and recall of LLM code review, and how should at most 30k LLM tokens per verdict be spent so the AI only judges what deterministic tools cannot (does the change match its **change intent** and the **project intent**; logic bugs)?

Researched 2026-11-13. Settled constraints (language-agnostic, under 2 minutes, at most 30k tokens, no end-to-end testing, AI blocks only with high confidence and evidence) are taken as given.

## Answer in one paragraph

LLM code review on its own is **low-recall and noisy**. Independent benchmarks put the best models at roughly 15-36% recall against issues human reviewers flagged, and single-pass precision at about 10-25%. Repeated sampling raises recall but does little for precision, and agreement between samples is a weak signal of correctness. What does raise precision is **executable evidence checked by a separate verifier**: in the strongest study, requiring a proof-of-concept that an independent checker re-runs rejected 85-96% of false bug reports. So the AI in merge-gate should be a **narrow add-on, not the main safety net**. Deterministic preparation selects the risky hunks and packs a small labelled context. One finder pass judges intent and logic and must turn each suspected bug into a concrete test. The gate runs those tests in memory, against the base and the head. A fresh-context verifier pass then checks only the surviving candidates. A finding may **block** only if a generated test fails on the head and passes on the base (or is otherwise reproducible), and the expected behaviour is traced to a quoted source. Everything else becomes a note. This fits in about 27k tokens and about 40-60 seconds of model time with Claude Sonnet 5.5.

## 1. What the evidence says about precision and recall

### Research benchmarks (independent, reproducible)

| Benchmark | Data | Headline result | Source |
|---|---|---|---|
| SWR-Bench (2025) | 1,000 human-verified PRs from 12 Python projects; 500 change PRs and **500 clean PRs** | Best tool + model: precision 16.65 / recall 23.18 / F1 19.38 (PR-Review + Gemini-2.5-Pro); GPT-5 recall 35.93 at precision 14.69. Several tools fall below 10% precision; the paper says precision is "primarily constrained by high false positive rates". Logic-bug F1 26.2 is better than style or documentation F1 (as low as 6.05). | https://arxiv.org/abs/2509.01494, https://arxiv.org/html/2509.01494v2 |
| SWE-PRBench (2026) | 350 PRs (Python 69%, JS 11%, Go 10%, TS 6%, Java 4%), human-annotated | **No model detects more than 31%** of human-flagged issues. False-positive rate per model: 0.19 (GPT-4o) to 0.42 (Llama 3.3 70B); Claude Sonnet 4.6 0.227, Claude Haiku 4.5 0.346. All 8 models do **worse** with a 2.5k "full context" prompt than with a 2k structured diff-plus-summary prompt (Sonnet composite 0.190 -> 0.122). | https://arxiv.org/abs/2603.26130, https://arxiv.org/html/2603.26130v1 |
| LangChain ReviewBench (Jul 2026) | 59 tasks / 64 issues from trusted reviewers on a real monorepo | Strongest agent runs recover about **30%** of the issues. Misses cluster in issues that need context beyond the changed lines: implicit contracts, project safety rules such as tenant filters, API-parity regressions. | https://langchain.com/blog/evaluating-code-review-agents-with-reviewbench |
| GitHub ReviewBench (Oct 2026) | 219 PRs, 187 repos, 19 languages; golden set from humans, follow-up commits, static analysers and LLMs; 96.6% agreement on independent relabel | No absolute numbers published. In a production A/B test, a multi-model ensemble gave **+13.6% recall, +8.0% addressed rate** (a precision proxy) and **+61% comment volume**. | https://github.blog/ai-and-ml/reviewbench-an-open-benchmark-for-ai-code-review/ |
| Martian Code Review Bench | Offline: 50 PRs, 5 repos (Python, Go, TS, Ruby, Java), 173 golden comments with severity and category, 3 judge models. Online: developer fixes after bot comments on fresh PRs | Open data, judge prompts and pipeline (MIT), so another tool can be scored on it. Top-5 ranking is stable across the three judges. The authors themselves say offline and online rankings diverge. | https://github.com/withmartian/code-review-benchmark |

### Vendor numbers (treat as marketing)

Vendor-run "catch rate" tests claim 45-82% for commercial reviewers on curated bug sets, and they disagree with each other. For example, Greptile reported 82% for itself on its own test, while Augment's rerun of the same set gave 45%. These cannot be reproduced and are not used for decisions here. Pointers: https://macroscope.com/content/ai-code-review-benchmark-best-models, https://www.greptile.com/content-library/greptile-martian-code-review-benchmark.

### Industrial deployments

- Qodo PR-Agent at Beko (ICSE SEIP 2025): 73.8% of automated comments were marked resolved, but average PR close time **rose** from 5h52m to 8h20m. Developers reported faulty reviews, unnecessary corrections and irrelevant comments. https://arxiv.org/abs/2412.18531
- Meta RADAR (2026): a funnel of eligibility gates, then static heuristics, then a learned Diff Risk Score, then LLM review, then deterministic validation. It handled 535K+ reviewed diffs. Reverts ran at about 1/3 and incidents at about 1/50 of non-RADAR diffs, because only low-risk diffs are auto-landed. https://arxiv.org/abs/2605.30208 ; Diff Risk Score (a fine-tuned Llama that scores incident likelihood and highlights risky snippets): https://engineering.fb.com/2025/08/06/developer-tools/diff-risk-score-drs-ai-risk-aware-software-development-meta/

### What this means for merge-gate

1. Expect the AI layer to find **a minority of real bugs**: roughly 20-35% before filtering, and fewer after the strict evidence filter. The deterministic checks and the repo's own tests have to carry most of the gate's reliability.
2. Raw LLM findings are **mostly false positives** (SWR-Bench precision 10-17%). A gate that blocked on raw findings would block most clean PRs. Measure the **false block rate on clean PRs**; SWR-Bench's 500 clean PRs exist for this.
3. Logic bugs are where LLMs do relatively best (F1 about 26 versus about 6-16 for evolution and style issues), and this is the category merge-gate gives them.

## 2. Selecting risky hunks (deterministic, 0 tokens)

The evidence favours **small, targeted input**. SWE-PRBench found that more context hurt every model, and Chroma's "context rot" study found that all 18 models tested degrade as input grows, even on trivial tasks (https://research.trychroma.com/context-rot). So the gate should choose what the model reads.

Steps, all language-agnostic through tree-sitter and git:

1. **Drop non-reviewable files.** Remove lockfiles, generated files (`linguist-generated` in `.gitattributes`, `@generated` headers), vendored code, snapshots and binary files. *Measured:* in fastapi/fastapi#16728, `uv.lock` was 83 KB of an 86 KB diff (97%). Unfiltered, that diff is about 24k tokens; filtered, it is under 1k.
2. **Drop hunks that are fully explained elsewhere.** Remove formatting-only and comment-only hunks, and hunks whose only problem a linter or type checker already reported. The official Claude Code review command likewise lists "issues that a linter will catch" as false positives not to flag (https://github.com/anthropics/claude-plugins-official/blob/main/plugins/code-review/commands/code-review.md).
3. **Map each hunk to its enclosing symbol** (function, method or class) with tree-sitter.
4. **Score each hunk for risk** using cheap signals. These mirror the change-level defect-prediction features (size, diffusion, history, experience) from Kamei et al., TSE 2013 (https://doi.org/10.1109/TSE.2012.70), and Meta's practice of routing review effort by a risk score:
   - the hunk touches an exported or public symbol, an HTTP route, a CLI entry point or rendered output (public-facing behaviour);
   - the hunk changes conditions, loop bounds, comparisons, arithmetic, error handling, or deletes a guard or validation;
   - the hunk touches concurrency, persistence, authentication, money or a security-sensitive API;
   - no test file changed alongside the hunk, or changed lines are not covered when the repo's tests run;
   - the file is a git-history hotspot (high churn or many past bug-fix commits).
5. **Fill the diff budget** (about 6k tokens) with the highest-scoring hunks. Hunks left out appear only as a one-line list of `path:symbol` entries, so the intent check can still see the full scope of the change.
6. **Skip the AI entirely** when no hunk scores as risky and the PR description makes no behavioural claim (docs-only, tests-only or trivial changes). The official Claude Code review command uses the same trivial-PR short-circuit as its first step.

Measured sizes of typical PRs (this repo, `gh pr diff`, estimated at 3.5 chars per token; Claude's current tokenizer can use up to 1.35x more): pallets/flask#5928 with 274 changed lines is about 5.3k tokens; vitejs/vite#24079 with 198 changed lines is about 5.6k; fastapi/fastapi#16734 with 379 changed lines is about 4.0k. So a typical under-500-line PR fits in a **6k diff budget** after filtering, often without dropping any hunks.

## 3. Packing context (diff + callers + intent)

The evidence pulls two ways, and the conflict resolves cleanly:

- **Dumping whole files hurts** (SWE-PRBench; context rot; "Lost in the Middle", https://arxiv.org/abs/2307.03172).
- **Missing contract context causes misses** (LangChain: implicit contracts, project rules and API parity).

So include **targeted, labelled** context, not files:

| Slot | Budget | Content | How (deterministic) |
|---|---|---|---|
| Project intent | ~1.5k | Only rules that apply to the changed paths: CLAUDE.md, ADRs, glossary and README sections that match changed paths or symbol names. Each rule is quoted with `file:line`. | Path scoping as in the official Claude Code review command ("only consider CLAUDE.md files that share a file path with the file or parents"), plus keyword match on symbol names |
| Callers and contracts | ~3k | For each changed public symbol: its old and new signature, plus the top call sites (the call line +/-3 lines), ranked by how often they are referenced. Also the names of existing tests that cover the symbol. | tree-sitter definitions and references; aider's repo map uses the same tree-sitter tags with graph ranking and a default 1k-token budget (https://aider.chat/docs/repomap.html, https://aider.chat/2023/10/22/repomap.html) |
| Deterministic results | ~0.5k | What already ran and its outcome (tests, types, lint, coverage of changed lines), so the model does not re-derive it | Gate output |
| Change intent | ~1k | PR title and body, and the linked issue, trimmed | Platform API |
| Diff | ~6k | Selected hunks, each marked `CHANGED`, with the enclosing symbol header | Section 2 |

Ordering and format:

- Put long context first and the task last. Anthropic's docs report that "queries at the end can improve response quality by up to 30 percent". Also use XML-tagged sections and ask the model to quote the relevant lines before judging (https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices).
- Mark changed and unchanged code explicitly; SWE-PRBench names this as future work after seeing attention dilution.
- Keep the system prompt and project-intent block byte-stable so later passes reuse it from the prompt cache. The minimum cacheable prefix on current models is 512 tokens, and cache reads cost 5% of the input price on Sonnet 5.5 and Opus 5.5 (Claude API skill, `shared/prompt-caching.md`; https://platform.claude.com/docs/en/models/overview). Caching cuts latency and cost, not the token count.

## 4. Verification passes that cut false positives

| Technique | Evidence | Effect on precision |
|---|---|---|
| Multiple samples merged into one review | SWR-Bench: Gemini-2.5-Flash with 10 samples gets recall +118.8% and F1 +43.7%. GitHub ensemble: recall +13.6%, addressed rate +8%, comment volume +61% | **Raises recall, not precision**; it adds noise |
| Agreement voting as a confidence score | 265k samples: agreement correlates with correctness at only rho 0.20-0.59. For the most consistent frontier model, 48% of high-agreement answers were wrong (https://arxiv.org/abs/2607.08065) | Weak; confident errors repeat across samples |
| Consistency check plus reasoning check on a narrow yes/no question | Static-analysis alert triage: at least 98% recall and at least 94.8% specificity with mid-tier models (https://arxiv.org/abs/2607.09979) | Strong when the question is narrow ("is this specific alarm real?") |
| LLM plus static path constraints as a false-positive filter | Tencent study, 433 alarms: 94-98% of false positives removed, 75-88% of true bugs kept, 12-58 seconds per alarm (https://arxiv.org/abs/2601.18844) | Strong; hybrid beats prompt-only, and few-shot beat chain-of-thought |
| Separate validator per finding in a fresh context | Official Claude Code review command: 4 parallel reviewers, then "parallel subagents to validate the issue", then drop anything not validated | Production design; no published numbers |
| Executable proof checked by an independent checker | AnyPoC: accepted 62 of 96 true reports and **rejected 85-96% of false reports**. Plain Claude Code and Codex wrote "successful" PoCs for 85-96% of false reports, gaming the task (https://arxiv.org/abs/2604.11950) | Strongest evidence; the checker must be independent |

Takeaways:

- **Verify narrow claims, not whole reviews.** Re-asking "is this specific claim true?" in a fresh context works (alert triage, the Claude Code validators). Voting over whole reviews mostly adds recall and noise.
- Under the 30k budget, give the extra spend to **verifying a few candidates** rather than re-running the full review.

## 5. Requiring executable evidence for blocking findings

The gate already runs the app in memory with fakes, which is the right place to execute a finding. The literature supports a test-as-proof rule, with two cautions.

- **Use tests as proof.** AnyPoC treats bug validation as test generation and requires an independent checker to re-run the test in a clean workspace. AssertFlip generates a *passing* test for the current behaviour and then inverts the assertion; this beat earlier methods on SWT-Bench Verified, at 43.6% fail-to-pass (https://arxiv.org/abs/2507.17542). Property-based tests written by an agent found real bugs in NumPy and other packages, but only 56% of raw reports were valid; 86% of the top-ranked ones were (https://arxiv.org/abs/2510.09907).
- **Caution: the test-oracle problem.** A generated test can encode the model's own wrong belief about correct behaviour. Without an independent checker, agents "prove" false reports most of the time (AnyPoC baselines). An LLM writing tests against buggy code also tends to assert the buggy behaviour (https://arxiv.org/abs/2607.22883).

**Blocking rule for an AI finding.** All of the following must hold:

1. **Reproduces.** A generated test, in the repo's own test framework and run in memory with fakes, **fails on the head and passes on the base**. For a new feature with no base behaviour, it fails on the head against an expectation stated in step 2. Re-run it once to rule out flakiness.
2. **Traced oracle.** The test's expected behaviour cites a source: a quoted sentence from the PR description or linked issue, a project-intent doc line, an existing test, a type or interface contract, or the base branch's behaviour for a claimed-unchanged path. "The model thinks this is wrong" does not count.
3. **Independent confirmation.** A fresh-context verifier sees the claim, the test, its output on base and head, and the cited source. It answers yes or no to "does this test demonstrate a real defect that the cited source requires?" and is told to trust the execution output over the finder's claims (the AnyPoC checker rule).

**Intent findings** follow a separate rule, because some are not executable:

- **"Phantom change"**: the description claims X, but no changed symbol implements X. This is the most common mismatch (45.4% of 406 high-mismatch agent PRs), and such PRs were accepted 28.3% of the time versus 80.0% for consistent PRs (https://arxiv.org/abs/2601.04886). It may block when the claim is quoted, the verifier confirms no hunk addresses it, and, where possible, a generated test of the claimed behaviour fails.
- **Project-rule violation**: may block when the exact rule line is quoted, its scope covers the file, and the verifier confirms it. This is the same standard the official Claude Code review command uses ("quote the exact rule being broken").
- **Scope creep and vague intent**: notes only. Purpose-level mismatches are the hardest kind for LLMs to detect (CodeFuse-CommitEval, https://arxiv.org/abs/2511.19875).

Everything that fails these rules becomes a non-blocking note ("merge with notes"), capped at 5 to keep noise down.

## 6. Fast models that fit under 2 minutes

Current Claude lineup (https://platform.claude.com/docs/en/models/overview, checked 2026-11-13):

| Model | ID | Comparative latency | $/MTok in / out | Output speed (third party, Artificial Analysis via The Batch) |
|---|---|---|---|---|
| Claude Haiku 5.5 | `claude-haiku-5-5` | Fastest | from 0.10 / 0.50 | ~237 tok/s |
| Claude Sonnet 5.5 | `claude-sonnet-5-5` | Fast | 2 / 10 | ~127-143 tok/s |
| Claude Opus 5.5 | `claude-opus-5-5` | Moderate | 4 / 20 | ~97 tok/s (max reasoning) |

The speed figures come from https://www.deeplearning.ai/the-batch/claude-fills-out-its-model-family/ and https://artificialanalysis.ai/providers/anthropic. They depend heavily on the effort setting; measure them yourself before relying on them.

Wall-clock arithmetic (task time is roughly the number of calls times time to first token, plus output tokens divided by output speed; Claude API skill, `shared/latency-optimization.md`):

- **Finder**, Sonnet 5.5 at effort `low`, about 12k input and at most 3k output including thinking: about 2-4 s to first token plus about 22 s of output. **About 25 s.**
- **Verifiers**, up to 3 in parallel, each about 2.5k input and at most 1k output: **about 10 s.**
- **Generated tests** run in memory on base and head: about 10-30 s, depending on the repo.
- **AI-related total: about 45-65 s.** This leaves room for the deterministic checks within 2 minutes, especially if the finder starts while lint and type checks are still running.

Model choice:

- **Sonnet 5.5 is the default for both passes.** It is fast enough, and its predecessor had the lowest false-positive rate among models with top-tier detection in SWE-PRBench (0.227 vs Haiku 4.5's 0.346, at the same ~0.30 detection).
- **Opus 5.5 at effort `low`** fits the time budget at about 31 s for 3k output tokens and is what the official Claude Code command uses for bug finding. Test it as a finder swap on the bug benchmark.
- **Haiku 5.5** suits the verifier's narrow yes/no question if latency is tight. Its predecessor's higher false-positive rate argues against using it as the finder.
- Avoid high effort levels. Thinking tokens count against the 30k budget and dominate latency.

## 7. Recommended token allocation and pass structure (30k)

```
Stage 0  deterministic prep                              0 tokens   ~5 s
         filter files, tree-sitter symbols, risk-score hunks,
         callers/contracts, scoped intent docs, deterministic results
         -> no risky hunks and no behavioural claim: skip AI (0 tokens)

Pass 1   FINDER (Sonnet 5.5, effort low, structured JSON)    ~15k       ~25 s
         input  ~12k: intent docs 1.5k | callers 3k | det. results 0.5k
                      | PR text 1k | diff 6k  (context first, task last)
         output <=3k incl. thinking:
           a) claims[]   each PR claim -> implementing hunk(s) or "none"
           b) unexplained[]  hunks no claim explains (notes)
           c) rule_hits[]    quoted rule + file:line
           d) bug_candidates[] (max 3, ranked) each with hypothesis,
              cited oracle source, and a test in the repo's framework

Stage 2  EXECUTE (deterministic)                           0 tokens  ~10-30 s
         run each candidate test on head and base, in memory with fakes;
         one re-run to rule out flakiness

Pass 3   VERIFIERS (Sonnet 5.5 or Haiku 5.5, fresh context,  ~9k       ~10 s
         parallel, <=3 x ~3k)   (parallel)
         input: one claim + its hunk + oracle source + test + base/head output
         output: confirmed yes/no + one-line reason

Reserve  one repair attempt for a test that does not compile  ~3k
         or import; otherwise unused                                  ----
                                                        total <=27-30k   ~45-65 s
```

Verdict mapping:

- **Block**: a deterministic block, or an AI finding that meets every rule in Section 5.
- **Merge with notes**: unconfirmed AI candidates, unexplained hunks, vague intent (at most 5 notes).
- **Merge**: otherwise.

The AI can **only add** findings. It can never clear a deterministic failure. This also limits prompt injection: PR text is untrusted data, so injected text can at most suppress AI findings, not approve a PR (OWASP LLM01, https://genai.owasp.org/llmrisk/llm01-prompt-injection/).

Why this shape rather than the alternatives:

- **Not "several full reviews and vote".** Under 30k tokens that means two or three 12k reviews and no verification. The evidence says this buys recall and noise, not precision.
- **Not "one big full-context review".** Bulk context lowers detection, and it leaves no budget for evidence.
- **Spend on narrow verification.** The highest-precision techniques in the literature all verify narrow claims, ideally with execution.

## 8. How to prove it on the bug benchmark

- **Recall and comparability:** run the Martian offline set (50 PRs, 5 languages, open judge, many tools already scored) and GitHub ReviewBench (219 PRs, 19 languages).
- **False block rate:** run SWR-Bench's 500 clean PRs; this is the metric that matters most for a gate.
- **Report separately:** blocking precision, blocking recall, note precision, false block rate on clean PRs, tokens per verdict, and p50/p95 wall clock.
- **Ablations:** remove each stage (risk selection, context slots, execution requirement, verifier) to show that each one earns its tokens.

## Main risks

1. **The test-oracle problem.** A generated test can "prove" the model's own misunderstanding. The mitigations (a cited oracle, the base-versus-head check and an independent verifier) are supported by the evidence but have **not yet been measured inside merge-gate**.
2. **Low recall is structural.** Even good systems catch about 30% of issues that human reviewers flag. The strict evidence rule will lower recall for *blocking* findings further, probably to around 10-20% of logic bugs (an estimate, not measured). Users must not read "AI found nothing" as "no bugs".
3. **Unverified third-party numbers.** The speed figures are third-party, and several benchmarks used older models (Sonnet 4.6, Haiku 4.5). Re-measure on the current models before fixing the budget split.
4. **Execution cost varies by repo.** A slow test harness could push stage 2 past its 30 s slot. The fallback is to downgrade execution-dependent findings to notes when the time budget runs out, never to block without the evidence.

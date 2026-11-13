# Record and replay of public behaviour, in-process, as a substitute for E2E

Research ticket for the merge **gate** (see [GLOSSARY.md](../GLOSSARY.md)). Date: 2026-11-13.

**Question.** How can real public inputs and outputs (HTTP, CLI, UI, emitted events) be captured and replayed against the changed code in-process, with no network? Where do recordings come from without running E2E? How are they kept fresh? How is nondeterminism handled? How are intended behaviour changes told apart from regressions?

## Answer in one paragraph

Do not store recorded *outputs* as the oracle. Store (or generate) only public *inputs*, and use **the previous main build, loaded in memory next to the PR branch, as the oracle** (differential testing, as in Twitter's Diffy). Replay each input against main twice and against the branch once, all in-process with VCR-style fakes for outbound calls and with the network blocked. Diffs between the two main runs are noise. Diffs between main and branch that are not noise get clustered by route, JSON path and kind (added, removed, changed). Each cluster is then labelled deterministically where possible (the PR updated a golden file for it, or it is a known breaking-change shape) and otherwise by the LLM, which compares the small cluster summary with the **change intent**. Clusters the change intent does not explain become **findings**. This removes the freshness problem for outputs entirely (main is regenerated on every run), keeps the LLM input to a few hundred to a few thousand tokens, and needs no E2E run to obtain recordings: inputs come from the repo's own tests, its API/CLI schemas, generators, and (optionally) sampled request logs.

## 1. Two halves of the problem

Record/replay tools solve two different halves. The gate needs both.

| Half | What crosses the boundary | Tools that own it | Role in the gate |
|---|---|---|---|
| **Inbound** (public behaviour) | HTTP requests into the app, CLI argv/stdin/env, UI events, messages consumed | In-process test clients (WSGI/ASGI test client, supertest), trycmd, Storybook portable stories, Pact provider verification, Keploy test cases | The thing being judged: replay inputs, compare outputs |
| **Outbound** (dependencies) | HTTP calls to other services, DB/cache, queues | vcrpy, Polly.js, nock, Keploy mocks, hand-written fakes | Make the app runnable "in memory with fakes", no network |

VCR libraries (vcrpy, Polly.js, nock) are **outbound** tools: they fake the services your code calls. They do not by themselves verify your public behaviour. Keploy is the only surveyed tool that records both halves together.

## 2. Survey

### 2.1 VCR-style outbound record/replay

**vcrpy** (Python). Record modes ([usage docs](https://vcrpy.readthedocs.io/en/latest/usage.html)):
- `once` (default): records if no cassette exists, otherwise raises on new requests.
- `new_episodes`: replays known and records new.
- `none`: "guarantees that no new HTTP requests will be made". This is the gate mode.
- `all`: always re-records; with `drop_unused_requests` the cassette is rewritten to only this run's interactions. This is the refresh mode.

Scrubbing and matching ([advanced docs](https://vcrpy.readthedocs.io/en/latest/advanced.html)): `filter_headers`, `filter_query_parameters`, `filter_post_data_parameters`, `before_record_request` / `before_record_response` (return `None` to skip), custom matchers via `register_matcher`, `allow_playback_repeats`, custom persisters.

**Polly.js** (JS, Netflix). Recordings are HAR files ([README](https://raw.githubusercontent.com/Netflix/pollyjs/master/README.md)). Config ([configuration.md](https://raw.githubusercontent.com/Netflix/pollyjs/master/docs/configuration.md)): `mode` `replay` | `record` | `passthrough` (default `replay`); `recordIfMissing` (default `true`, must be `false` in the gate); `expiresIn` (e.g. `'30d'`) with `expiryStrategy` `warn` | `error` | `record`; `matchRequestsBy` toggles method, headers, body, `order`, and each URL part. `expiresIn` is the only built-in freshness mechanism found in any surveyed VCR tool.

**nock** (Node) ([README](https://github.com/nock/nock)): `nock.back` modes `wild`, `dryrun` (default), `record`, `update`, `lockdown` ("use recorded nocks, disables all http calls"). `nock.disableNetConnect()` throws `NetConnectNotAllowedError` for any unmocked host. `nock.recorder.rec({output_objects: true})` captures JSON definitions; request headers are not recorded by default because values like timestamps change between runs. Hooks `before`, `after`, `afterRecord` are the scrubbing points.

Gate settings: vcrpy `record_mode="none"`, Polly `mode: 'replay', recordIfMissing: false, expiryStrategy: 'error'`, nock `lockdown` + `disableNetConnect()`. In all three, an unrecorded outbound call is a hard failure, which is also a useful signal: the PR introduced a new external dependency.

### 2.2 Keploy (both halves, captured from a running app)

([architecture](https://keploy.io/docs/keploy-explained/how-keploy-works/)) Keploy hooks the app at socket level (eBPF on Linux, user space on macOS and Windows), records incoming HTTP calls as YAML **test cases** and outgoing DB/cache/API calls as **mocks**. In test mode it starts the app itself, sends the recorded requests, and serves mocks for the outgoing calls.

Nondeterminism handling:
- Noise detection by re-running: "Keploy re-run that API request with the captured dependency mocks ... identifies differences in API responses, marking them as random/noisy fields" ([features](https://keploy.io/docs/concepts/what-are-keploy-features/)). This is the Diffy control-replica idea.
- Declared noise: `test.globalNoise` with per-field regexes for body and header, global or per test set; `ignoreOrdering: true` for array order ([config](https://keploy.io/docs/running-keploy/configuration-file/)).
- Time freezing at replay: patched Go binaries, `LD_PRELOAD` clock library for Node, Java, Python, Ruby; not supported natively on macOS or Windows ([time freezing](https://keploy.io/docs/keploy-cloud/time-freezing/)).
- Platforms: Linux kernel 5.10+, native Apple Silicon macOS, native Windows x86-64; examples cover Go, Node.js, Java, Python ([installation](https://keploy.io/docs/server/installation/)).

Fit for the gate: good as a *source* of inputs and outbound mocks if a team already runs it. Weak as the gate engine itself: it records by running the app with real traffic (often a staging or local E2E session), it is language-runtime-specific at the capture layer, time freezing does not work on a native macOS laptop, and the `test.delay` default is 5 seconds per request, which does not fit a 2-minute budget at volume.

### 2.3 Golden / snapshot / approval testing (stored outputs as the oracle)

- **Jest** ([snapshot docs](https://jestjs.io/docs/snapshot-testing)): property matchers (`expect.any(Date)`) for generated fields; since Jest 20 snapshots are not written in CI without `--updateSnapshot`, "since new snapshots automatically pass, they should not pass a test run on a CI system"; snapshots "should be committed alongside code changes, and reviewed as part of your code review process"; the docs warn against "the habit of regenerating snapshots when test suites fail instead of examining the root causes".
- **syrupy** (pytest) ([README](https://github.com/syrupy-project/syrupy)): `--snapshot-update` rewrites and deletes unused snapshots; unused snapshots fail the run by default; `path_type`, `path_value`, `exclude=props(...)` for nondeterministic fields.
- **insta** (Rust) ([redactions](https://insta.rs/docs/redactions/)): selector-based redactions (`.id`, `[]`, `.**`), `dynamic_redaction` (validate then replace, e.g. assert it is a UUID), `sorted_redaction` for set-like ordering ("no automatic ordering can be provided without causing issues for the general case"), `rounded_redaction` for floats.
- **ApprovalTests** ([Python repo](https://github.com/approvals/ApprovalTests.Python)): `received` vs `approved` files compared against a "golden master"; a reporter opens a diff tool on mismatch.
- **trycmd** (Rust CLIs, the best model for CLI golden tests) ([docs](https://docs.rs/trycmd/latest/trycmd/)): command plus expected `.stdout`/`.stderr`/exit code in `.toml` or literate `.md`; `[..]` and `...` wildcards; `[ROOT]`, `[CWD]`, `[EXE]` and custom placeholders; `*.in/` and `*.out/` directories sandbox and verify file system effects; `TRYCMD=overwrite` to update.

Strength: the expected behaviour is in the repo and *the PR's diff to the golden files is the author's own declaration of the intended behaviour change*. Weakness: goldens only exist where someone wrote them, and blind `-u` updates launder regressions into the baseline (Jest's own warning).

### 2.4 Contract replay for events and service boundaries

**Pact** ([how Pact works](https://docs.pact.io/getting_started/how_pact_works)): consumer tests against a mock provider produce a pact file; provider verification replays each request against the real provider in-process and passes if the response "contains at least the data described" (extra fields allowed). Provider states set up data per interaction. Message pacts handle async events by having Pact "take the place of the intermediary" and testing the domain-level port, not the queue. This is the cleanest pattern for **emitted events**: capture what the app publishes through a fake broker port and compare those messages.

### 2.5 Differential testing (old main vs new branch)

- Origin: McKeeman, "Differential testing for software", *Digital Technical Journal* 10(1), 1998 (cited via [Wikipedia](https://en.wikipedia.org/wiki/Differential_testing); primary PDF not retrieved). Also called back-to-back testing.
- **Diffy** ([opendiffy/diffy](https://github.com/opendiffy/diffy)): proxy sends each request to **candidate** (new code), **primary** and **secondary** (both last known-good). "Since both run known-good code, you should expect them to agree. Where they don't, your service is exhibiting non-deterministic behavior - Diffy treats that signal as noise." Candidate-vs-primary diffs whose rate exceeds the primary-vs-secondary rate are reported as regressions.
- **Test carving** (Elbaum, Chin, Dwyer, Dokulil, FSE 2006, ["Carving differential unit test cases from system test cases"](https://researchr.org/publication/ElbaumCDD06); TSE 2009 extension): capture the state that influences a unit during a system test, replay it as a fast "differential unit test" against the new version. Reported to match system tests' fault detection while running much faster ([DTIC copy](https://apps.dtic.mil/sti/pdfs/ADA459093.pdf)). This is the academic basis for "obtain recordings from a test run, then replay cheaply".
- **SCARPE** (Joshi and Orso) ([paper](https://sites.cc.gatech.edu/home/orso/papers/joshi.orso.pdf)): selective capture/replay of field executions of deployed software into regression tests, including stubs and oracles, with attention to overhead and privacy.

Differential testing needs no stored outputs, so it has no output staleness and covers every input you can feed it, not only those someone wrote a golden for. Its cost is running the code twice (or three times) and needing a buildable main.

### 2.6 Traffic shadowing (production only, not a gate mechanism)

- **Envoy** `request_mirror_policies` is "fire and forget": Envoy "will not wait for the shadow cluster to respond", appends `-shadow` to the host header, `runtime_fraction` samples ([Envoy route API](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto)).
- **GoReplay** ([repo](https://github.com/buger/goreplay)) listens passively on a network interface ("acts like tcpdump"), can write captured traffic to a file and replay it later.

Shadowing needs a deployed candidate and live traffic, so it violates the gate's no-deploy, no-network rule. Its only role here is as an optional **input source**: a GoReplay or Keploy capture file, scrubbed, becomes replay input for the in-process gate.

### 2.7 LLM-assisted traffic-to-test (recent industrial evidence)

NL2Test (You et al., ISSTA 2026, [arXiv 2607.24000](https://arxiv.org/abs/2607.24000)): from a traffic capture plus a natural-language scenario it produces replayable API regression tests. Deterministic algorithms filter requests and validate assertion paths; the LLM only does semantic interpretation and constrained synthesis, specifically to avoid "non-deterministic fields and hallucinated paths". 82.4% exact match (42/51) and 98.0% usable drafts on 51 industrial scenarios; 3,196 tests generated in a 9-month deployment, 85.4% adoption. This supports the split the gate already assumes: deterministic tools for diffing and noise, the LLM only for meaning.

## 3. Getting recordings without running E2E

Because the oracle is main-in-memory, the gate only needs **inputs**. Ranked by cost and availability:

1. **The repo's own tests, re-run with an interceptor (carving).** Wrap the in-process test client (ASGI/WSGI test client, supertest, the CLI runner, Testing Library render) to log every public request and response the existing test suite already makes. On main this is cheap, deterministic, and needs no new tests. It is Elbaum's carving applied at the public boundary. Coverage is limited to what tests already exercise.
2. **Schemas the repo declares.** OpenAPI / GraphQL schema, CLI `--help` / argparse / commander definitions, event schemas (AsyncAPI, JSON Schema, Pact files). Generate example inputs per operation, biased toward the routes and commands the PR touches (from the change-impact ticket). Property-based generators fit here (see the property-mutation-tests ticket).
3. **Existing golden files, pact files, VCR cassettes, Storybook stories, trycmd cases.** Already inputs-with-expected-outputs; reuse them as inputs too.
4. **Sampled production or staging logs (optional, opt-in).** Access logs give method, path and query but usually not bodies. OpenTelemetry HTTP conventions make header capture opt-in and require explicit configuration of which headers to capture because "including all request headers can be a security risk"; `url.query` is stripped by default in common instrumentations; there is no standard attribute for body content ([HTTP spans spec](https://opentelemetry.io/docs/specs/semconv/http/http-spans), [migration guide](https://opentelemetry.io/docs/specs/semconv/http/migration-guide/)). So log-derived corpora are mostly GET-shaped unless the team already runs Keploy or GoReplay. Any such corpus must be scrubbed (vcrpy filters, nock `afterRecord`, insta-style redactions) and committed or cached, never fetched live by the gate.

Outbound fakes for the replay come from: hand-written fakes the repo already has (preferred, they belong to the in-memory-app ticket), VCR cassettes, or Keploy mocks. An outbound call with no fake fails the run (lockdown), which itself is a reportable finding ("PR adds an outbound call to X").

## 4. Freshness

- **Outputs:** not stored, so never stale. Main is executed in the same run. This is the main reason to prefer differential over golden as the default.
- **Golden files the repo already has:** treat as tests. Use the frameworks' own unused-snapshot detection (syrupy fails on unused by default; Jest reports obsolete) and never write snapshots in the gate (Jest CI behaviour, vcrpy `none`, Polly `recordIfMissing: false`).
- **Inputs:** regenerated from tests and schemas on each run, so they follow the code. A logged corpus is versioned with a capture date; inputs that fail to parse against the *main* schema are dropped as stale, not reported.
- **Outbound cassettes:** age them with Polly's `expiresIn` + `expiryStrategy: 'warn'` (or equivalent metadata) and refresh in a separate scheduled job that is allowed network (vcrpy `all` with `drop_unused_requests`, nock `update`). Never refresh inside the gate.

## 5. Nondeterminism

Handle it in three layers, cheapest and most reliable first:

1. **Pin the sources** before running either side, identically for main and branch: clock (Python `time-machine` / `freezegun`; JS `@sinonjs/fake-timers`, which fakes `Date`, timers, `performance`, `process.hrtime`, `nextTick`, `queueMicrotask`, `Intl` and `Temporal` by default ([repo](https://github.com/sinonjs/fake-timers)); Jest docs mock `Date.now`), RNG seeds, UUID factories, `PYTHONHASHSEED`, `TZ`, locale, temp dir paths, worker counts. Language-agnostic fallback: Keploy-style `LD_PRELOAD` faketime, Linux only.
2. **Declared redactions** for what cannot be pinned: Jest property matchers, syrupy `path_type`, insta `dynamic_redaction` and `sorted_redaction`, trycmd `[..]` and placeholders, Keploy `globalNoise` and `ignoreOrdering`. The repo may declare these once in gate config.
3. **Learned noise by control replica** (Diffy, Keploy re-run): any JSON path that differs between two main runs on the same input is noise for that route. This catches what layers 1 and 2 missed, but only probabilistically (see the experiment: one control replica missed per-process set ordering in 3 of 40 runs; pinning the hash seed made it 0 of 40).

Normalisation before diffing: parse by content type (JSON, HTML to a DOM or accessibility tree, CLI text line by line), template route ids (`/users/123` to `/users/:id`), generalise array indices (`tags[2]` to `tags[*]`). Diff at JSON-path level, not text level, so one cluster summarises thousands of requests.

## 6. Telling intended changes from regressions

Each non-noise diff cluster is `(route or command, path, kind, count, one example main/branch pair)`. Label it in this order:

1. **Declared by the PR's golden-file diff (deterministic).** If the PR edits a snapshot, cassette, pact or trycmd file and the edit covers this cluster, the author declared the change. Still pass it to step 3 in one line, because Jest's docs warn about blind regeneration.
2. **Classified by compatibility shape (deterministic).** Added field or new route: additive. Removed field, type change, status class change (2xx to 4xx/5xx), exit code change, removed CLI flag or event field: potentially breaking. This feeds the compatibility check and sets severity.
3. **Matched against change intent (LLM).** Give the LLM the PR title, description and linked issue plus the cluster list. Ask, per cluster: explained by the stated intent, contradicted by it, or not mentioned. Unexplained or contradicted clusters become findings with the example pair as evidence. Breaking-shape clusters that are explained still get a "merge with notes" finding so a human sees the API break.
4. **Default when there is no stated intent:** every behaviour diff is a finding; the verdict cannot be a clean "merge" for a PR that changes public output without saying so.

Diffy's rate rule (candidate-vs-primary rate above the control rate) is the right threshold for clusters that are partly noisy.

## 7. Recommended design for the gate

1. Build main and branch side by side (two worktrees), each loaded as "app running in memory with fakes" (from the public-behaviour-in-memory ticket), network blocked (lockdown / `none` / `disableNetConnect`).
2. Input corpus: carved from the repo's tests on main + generated from declared schemas for the routes/commands the change touches + optional committed scrubbed log corpus. Cap the corpus by time budget, routes touched by the PR first.
3. Pin time, RNG, ids, hash seed, TZ, locale for both sides. Apply repo-declared redactions.
4. Replay each input: main, main again (control), branch. Separate processes per side, identical settings, so cross-process nondeterminism appears in the control.
5. Cluster non-noise diffs; label via golden-file diff and compatibility shape; send only the unresolved cluster list plus change intent to the LLM.
6. Existing golden/snapshot/approval tests run as ordinary repo checks in non-update mode; their failures are findings without LLM involvement.

Surfaces:
- **HTTP:** framework test client, no socket.
- **CLI:** subprocess in a sandbox temp dir with fixed `HOME`, `TZ` and env, trycmd-style comparison of stdout, stderr, exit code and file system effects.
- **UI:** render components or stories in a DOM emulator (Storybook portable stories `composeStories` + `run()` in Vitest ([docs](https://storybook.js.org/docs/api/portable-stories/portable-stories-vitest)), Testing Library), serialise DOM or accessibility tree, diff. No pixels, no browser, consistent with the no-E2E constraint.
- **Events:** capture published messages at the fake broker port, compare ordered lists (or sorted lists, if the repo declares the topic unordered).

## 8. Experiment (not committed; `tmp/exp/` in the worktree)

A stdlib-only Python model: two versions of a small WSGI app called directly (no sockets). The branch has one intended change (adds `currency`) and one unstated regression (off-by-one in `/sum` for `n > 100`). Responses contain a random `request_id`, a timestamp, and a set-derived list.

- 5,000 recorded inputs x 3 executions (main, main, branch) = 15,000 in-process executions in **0.5 to 0.9 s** on the laptop, in-process or as three subprocesses. Real frameworks are slower per request; the budget should be measured per stack in the public-behaviour-in-memory ticket.
- Control replica correctly learned `request_id` and `served_at` as noise. Output: exactly **2 clusters** (`currency` added on 3,603 requests; `total` changed on 501 requests), a **460-character (~115 token)** summary for 5,000 requests. That is the whole LLM input apart from the PR text.
- **Cross-process nondeterminism:** with replicas in separate processes and Python's default random hash seed, set ordering produced a false "tags changed" finding in **3/40** runs with one control replica, **1/40** with two, **0/40** with `PYTHONHASHSEED=0`. Lesson: pin first, control replicas are only a backstop, and run all sides with identical process settings.

## 9. Key numbers

- Speed: replay cost is dominated by app startup and per-request framework overhead, not by diffing (diffing 15,000 responses took well under a second). Two builds plus one control run is roughly 3x the cost of one replay pass.
- Tokens: about 100 tokens per cluster including an example; a PR that changes public behaviour in 20 places costs about 2k tokens of cluster summary plus the PR text. Fits the 30k cap easily; cap clusters at around 50 and summarise the rest by count.
- Accuracy evidence: carved differential tests matched system-test fault detection at much lower cost (Elbaum et al. 2006). NL2Test: 82.4% exact, 98% usable with deterministic noise filtering plus LLM interpretation. No surveyed source reports bug-benchmark detection rates for differential replay as a PR gate; the bug-benchmark ticket must measure it.

## 10. Risks and open questions

- **Main risk: input coverage.** Differential replay only finds regressions on inputs it replays. Carved inputs only cover what tests already exercise; schema-generated inputs are shallow on stateful flows. Mitigation: target routes from change impact, add property-based generation, measure on the bug benchmark.
- **Stateful sequences.** Replaying single requests misses bugs that need a prior state. Need either Pact-style provider states or recorded sequences with id binding (NL2Test binds dynamic values from earlier responses).
- **Main must build and run.** If main is broken or migrations differ, fall back to golden-only mode and say so in the verdict.
- **Intended changes with silent side effects:** an intended cluster can hide an unintended one on the same path. Keep clusters per route and kind, never merge across routes.
- **Privacy** of any log-derived corpus: scrubbing is a trust boundary; default to no production data.
- **Keploy on a native macOS laptop** cannot freeze time; do not depend on it for the gate core.

## Sources

- vcrpy usage: https://vcrpy.readthedocs.io/en/latest/usage.html
- vcrpy advanced: https://vcrpy.readthedocs.io/en/latest/advanced.html
- Polly.js configuration: https://raw.githubusercontent.com/Netflix/pollyjs/master/docs/configuration.md
- Polly.js README (HAR): https://raw.githubusercontent.com/Netflix/pollyjs/master/README.md
- nock: https://github.com/nock/nock
- Keploy architecture: https://keploy.io/docs/keploy-explained/how-keploy-works/
- Keploy features (noise by re-run): https://keploy.io/docs/concepts/what-are-keploy-features/
- Keploy configuration (globalNoise, ignoreOrdering, delay): https://keploy.io/docs/running-keploy/configuration-file/
- Keploy time freezing: https://keploy.io/docs/keploy-cloud/time-freezing/
- Keploy installation: https://keploy.io/docs/server/installation/
- Jest snapshot testing: https://jestjs.io/docs/snapshot-testing
- syrupy: https://github.com/syrupy-project/syrupy
- insta redactions: https://insta.rs/docs/redactions/
- ApprovalTests.Python: https://github.com/approvals/ApprovalTests.Python
- trycmd: https://docs.rs/trycmd/latest/trycmd/
- Pact, how it works: https://docs.pact.io/getting_started/how_pact_works
- Storybook portable stories in Vitest: https://storybook.js.org/docs/api/portable-stories/portable-stories-vitest
- Diffy: https://github.com/opendiffy/diffy
- Differential testing (McKeeman 1998 citation): https://en.wikipedia.org/wiki/Differential_testing
- Elbaum et al., carving differential unit tests, FSE 2006: https://researchr.org/publication/ElbaumCDD06 and https://apps.dtic.mil/sti/pdfs/ADA459093.pdf
- Joshi and Orso, SCARPE: https://sites.cc.gatech.edu/home/orso/papers/joshi.orso.pdf
- NL2Test, ISSTA 2026: https://arxiv.org/abs/2607.24000
- Envoy request mirroring: https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto
- GoReplay: https://github.com/buger/goreplay
- @sinonjs/fake-timers: https://github.com/sinonjs/fake-timers
- time-machine: https://time-machine.readthedocs.io/en/latest/
- OpenTelemetry HTTP spans: https://opentelemetry.io/docs/specs/semconv/http/http-spans
- OpenTelemetry HTTP migration guide: https://opentelemetry.io/docs/specs/semconv/http/migration-guide/

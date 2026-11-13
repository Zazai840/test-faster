# Research: security and safety scanners for the merge gate

Question: which security and safety scanners fit a gate that must return a verdict in under 2 minutes on a laptop, stay language-agnostic, look only at the change (diff-scoped), make no network calls on the critical path, spend at most 30k LLM tokens, and ship inside a public MIT-licensed skill?

Scope: secrets, dependency vulnerabilities and supply chain, SAST/dataflow, license compliance, IaC, auth/permission changes, destructive data migrations.

Date: 2026-11-13. Measurements were taken on an Apple M3 laptop (8 cores, 16 GB) against OWASP Juice Shop (113k lines of TS/JS/HTML, 1,498 files, includes Terraform and Dockerfiles). The scripts that took them were throwaway and are not committed.

## Answer

All seven areas can be covered inside the budget, and the deterministic part needs about 10-20 s warm, as long as three rules hold:

1. **Scan the change, not the repo.** Every full-tree scan we timed took 37-112 s per tool, and a full-tree scan of a 113k-line repo returns 70 to 2,000+ old findings. Diff-scoped runs of the same tools took 0.5-7 s and returned only what the change introduced.
2. **Keep vulnerability and advisory data in a local cache that is refreshed outside the verdict.** Trivy and OSV-Scanner both scan fully offline against a pre-downloaded database. npm audit, pip-audit, Socket, GuardDog, and TruffleHog verification all need the network, so they belong in an optional "online enrichment" step, never the gate.
3. **Shell out to scanners as separate binaries. Do not bundle them, and do not ship Semgrep's or CodeQL's rule packs.** Licensing, not speed, is what rules out CodeQL, Semgrep's registry rules, Bearer and (for bundling) TruffleHog.

Two areas have no off-the-shelf diff tool, so the gate has to build them:

- **Auth/permission changes:** run route-inventory rules (route-detect, BSD-3) on the changed files at the base and at the head, and diff the two inventories. A route that loses its auth guard is a deterministic finding. No scanner we ran caught a deleted `security.isAuthorized()` middleware line, because they all look only at code that exists at the head.
- **Destructive migrations:** Squawk (Apache-2.0, offline, Postgres) handles raw SQL. For ORMs, the gate has to get SQL out first (for example Django `sqlmigrate`, Prisma `migrate diff --script`) or run framework linters such as django-migration-linter.

## Recommended stack

| Area | Pick | Diff mode | Measured, diff (11-file / 317-line real commit) | Measured, full 113k LOC | Offline | License (invoke as subprocess) |
|---|---|---|---|---|---|---|
| Secrets | **Gitleaks** | `gitleaks git --log-opts="BASE..HEAD"` | 0.47 s | `dir` 1.3 s; 300 commits of history 11.8 s | Yes | MIT |
| Dependency CVEs | **Trivy fs `--scanners vuln`** (OSV-Scanner `--offline` as an alternative) | Scan the head, subtract base findings for changed manifests/lockfiles | 0.14-0.2 s offline | 0.19 s offline (first DB pull 30 s, 1.6 GB on disk) | Yes, with `--skip-db-update` | Apache-2.0 |
| SAST/dataflow | **Opengrep or Semgrep CE engine + a ruleset the project owns** | `--baseline-commit BASE` | Semgrep 3.5 s (real commit), 6.6 s (seeded commit); Opengrep 10.9 s / 32.9 s | Semgrep 37 s; Opengrep 60 s; CodeQL 95 s (JS only) | Yes, with local rule files | Engines LGPL-2.1; Semgrep registry rules are NOT redistributable |
| License | **Trivy `--scanners license`** on changed lockfiles | Same subtract-base approach | <0.2 s | 0.15 s, 1,028 package licenses classified | Yes | Apache-2.0 |
| IaC | **Trivy config** (Checkov as a second opinion) | Pass changed IaC files/dirs | Trivy 1.3 s; Checkov 4.7 s (one file) | Trivy 1.5 s; Checkov 82 s | Yes (Trivy uses checks embedded in its binary; Checkov `--skip-download`) | Apache-2.0 / Apache-2.0 |
| Auth changes | **Build it:** route-detect rules (Semgrep format) at base and head, then diff the inventories | Changed files only, two runs | Expected to cost about the same as two Semgrep diff runs (not measured) | n/a | Yes | BSD-3 rules |
| Destructive migrations | **Squawk** for SQL; framework linter per ORM | Pass changed migration files | 3.1 s (includes `npx` start-up) | n/a | Yes | Apache-2.0 |
| Typosquat / malicious packages | **Build it:** edit distance against a bundled top-N package list for newly added dependency names, plus a "is it in the lockfile/registry snapshot" check | Newly added dependency names only | ms | n/a | Yes | own code |

Total deterministic time, warm caches, all scanners in parallel on 8 cores: about 10-20 s for a typical PR. The slowest pieces are the SAST diff run (3-33 s, depending on engine and rule count) and a cold `npx`/`uvx` start-up. That leaves well over a minute of the 2-minute budget for building and running the app in memory.

LLM tokens: none of these scanners use any. The AI should only judge findings that land on changed lines and are of a kind deterministic rules cannot settle (for example "is this SQL concatenation reachable from user input?", "is this route meant to be public?"). At about 400-700 tokens per finding (rule message plus 20 lines of context), a cap of 15 judged findings costs about 6-10k tokens, well under the 30k budget. This is an estimate, not a measurement.

## Measurements

### Setup

- Repo: `juice-shop/juice-shop` at `266448ab`, shallow-cloned with 300 commits. Lockfiles were generated with `npm install --package-lock-only --ignore-scripts`, because Juice Shop does not commit them.
- Real diff: `HEAD~1..HEAD` = 11 files, +205/-112 lines (an Angular form refactor). It contains no security issues, so it measures cost and noise.
- Seeded diff: one commit, 6 files, +40/-1, adding:
  - a GitHub PAT, an AWS key ID and an AWS secret key in a new TS route;
  - a Sequelize string-concatenation query and `eval(req.body…)`;
  - a Flask route with `subprocess(..., shell=True)` and `pickle.loads(request.data)`;
  - deletion of `app.use('/api/BasketItems', security.isAuthorized())` from `server.ts`;
  - a SQL migration with `DROP COLUMN`, `DROP TABLE` and `ALTER COLUMN TYPE`;
  - `requirements.txt` with `flask==0.12.2`, `django==3.2.0` and the typosquat `reqeusts==2.0`;
  - a Terraform S3 bucket with `acl = "public-read-write"`.
- Tool versions: Gitleaks 8.30.1, TruffleHog 3.99.9, detect-secrets 1.5.0, OSV-Scanner 2.7.1, Trivy 0.76.2, Checkov 3.4.5, Semgrep 1.185.0, Opengrep 1.33.1, CodeQL 2.27.4, Squawk (npm `squawk-cli@2`), GuardDog (latest via uvx), pip-audit 2.10.1.
- Python tools ran through `uvx` with a warm cache. All times are wall clock.

### Timings (seconds)

| Tool / mode | Full tree | Diff-scoped |
|---|---|---|
| Gitleaks `dir` / `git --log-opts` | 1.27 | 0.47 (real), 0.66 (seeded) |
| Gitleaks `git`, 300 commits of history | 11.77 | - |
| TruffleHog `filesystem` / `git --since-commit`, `--no-verification` | 4.06 | 3.42 (real), 4.89 (seeded) |
| detect-secrets `scan` (tracked files / changed files) | 100.8-111.7 | 0.99 (real), 2.07 (seeded) |
| Semgrep CE `p/default` (1,074 rules) | 37.3 | 3.49 (real), 6.63 (seeded) |
| Semgrep CE, local semgrep-rules JS+TS+generic/secrets | 74.4 | - |
| Opengrep, same local rules | 60.0 | 10.85 (real), 32.89 (seeded, + Python rules) |
| CodeQL JS/TS: `database create` + `analyze` (code-scanning suite) | 35 + 60 = 95 | Diff-informed mode still builds the whole database (see below) |
| Trivy `fs --scanners vuln`, first run (DB download) / offline | 30.0 / 0.19 | 0.14 (seeded `pyapp/`) |
| Trivy `--scanners license` offline | 0.15 | - |
| Trivy `config` (online check bundle / embedded checks) | 5.79 / 1.48 | 1.29 |
| Checkov | 82.1 (all frameworks) / 2.9 (`infrastructure/` only) | 4.74 (single file) |
| OSV-Scanner online / offline (npm DB is a 210 MB zip) | 3.4 / 16.0 | - |
| OSV-Scanner offline DB download | 33.9 | - |
| npm audit (network) | 5.8 | - |
| pip-audit `--no-deps` on NetBox `requirements.txt` (network) | 5.3 | - |
| Squawk | - | 3.1 |
| GuardDog `pypi verify` (network, downloads each package) | - | 15.2 |

### What each tool caught on the seeded diff

| Seeded problem | Caught by | Missed by |
|---|---|---|
| GitHub PAT, AWS key ID + secret | Gitleaks (3 findings), TruffleHog (2), detect-secrets (5), Semgrep `p/default` (3) | - |
| SQL concat + `eval` (TS) | Semgrep (`express-sequelize-injection`, `eval-detected`), Opengrep (+ `tainted-sql-string`) | - |
| `shell=True` + `pickle.loads` (Py) | Semgrep, Opengrep (5-6 findings each, several duplicates per issue) | - |
| Public S3 bucket | Trivy config (9 findings, 6 HIGH), Checkov (9), Semgrep `s3-public-rw-bucket` | - |
| Vulnerable Flask/Django pins | Trivy offline (42 CVEs) | OSV-Scanner `--offline`: failed, because only the npm database had been pre-downloaded. Its offline cache is per ecosystem. |
| `DROP COLUMN` / `DROP TABLE` / type change | Squawk (`ban-drop-column`, `ban-drop-table`, `changing-column-type`, plus 6 lock/robustness warnings) | - |
| Removed `isAuthorized()` middleware | **nobody** | Every SAST tool: they only report code present at the head |
| Typosquat `reqeusts` | **nobody** | GuardDog: "Package/Version reqeusts not on PyPI … No risks found". GuardDog only analyses names that exist in the registry. |

### Noise on full-tree scans (why diff scoping is mandatory)

Juice Shop is deliberately vulnerable, so these counts are not false-positive rates. They do show how many old findings a full scan would hand the gate:

- Gitleaks `dir`: 75 findings (60 `generic-api-key`, mostly in test files).
- Gitleaks over 300 commits: 756 findings.
- detect-secrets: 2,175 findings across 132 files.
- TruffleHog without verification: 6 findings.
- Trivy secret: 5 findings.
- Semgrep `p/default`: 71 findings.
- CodeQL: 80 findings (36 of them `js/missing-rate-limiting`).
- Opengrep with the whole JS/TS rule directories: 152 findings, led by code-quality rules such as `missing-template-string-indicator` and `useless-assignment`. Rule selection matters as much as the engine.
- Trivy misconfig: 89 findings.
- Checkov: 113 Terraform + 13 GitHub Actions + 6 Dockerfile failures.

## Per area

### Secrets

- **Gitleaks** (MIT). It has `git` (uses `git log -p`, with `--log-opts` for commit ranges), `dir` and `stdin` modes, a baseline file (`--baseline-path`), inline `gitleaks:allow`, and `.gitleaksignore`. It makes no network calls. ([README](https://github.com/gitleaks/gitleaks))
  - In the only peer-reviewed head-to-head we found (Basak et al., ESEM 2023, SecretBench of 818 repos), Gitleaks had the best recall (88%) and the second-best precision (46%) of 9 tools. TruffleHog scored 6% precision / 52% recall, and GitHub's secret scanner 75% precision / 6% recall. ([arXiv 2307.00714](https://arxiv.org/abs/2307.00714), Table III)
  - The authors trace false positives to "generic regular expressions and ineffective entropy calculation".
  - Our numbers agree: 60 of 75 Gitleaks hits came from the generic rule. Recommendation: block on Gitleaks' specific-provider rules and send `generic-api-key` hits to AI judgment, or report them as notes.
- **TruffleHog** (AGPL-3.0). It is diff-capable (`git --since-commit <base> --branch HEAD`). Its selling point is verification: "Verification eliminates false positives". But verification means calling each provider's API with the found credential, which is network access and breaks the no-network rule. With `--no-verification` it loses that edge (6% precision in the ESEM study). ([README](https://github.com/trufflesecurity/trufflehog))
  - Calling it as a separate binary from an MIT tool is fine. Bundling or linking it would pull in AGPL obligations.
  - Verdict: optional, not default.
- **detect-secrets** (Yelp, Apache-2.0). Its baseline design flags only *new* secrets, and it has a `detect-secrets-hook` for staged or listed files. ([README](https://github.com/Yelp/detect-secrets))
  - Diff mode was fast (1-2 s), but full scans took 100+ s with 2,175 findings, and it was not in the ESEM comparison.
  - Last push April 2026, so maintenance is slowing. Verdict: no advantage over Gitleaks.

### Dependency vulnerabilities and supply chain

- **Trivy** (Apache-2.0).
  - It scans lockfiles for vulnerabilities, licenses, secrets and misconfigurations from one binary.
  - Offline: the vulnerability DB is pulled once (30 s here, 1.6 GB on disk) and then used with `--skip-db-update`. Misconfiguration checks are "embedded in the Trivy binary (at build time)" as a fallback when offline. ([air-gap docs](https://trivy.dev/latest/docs/advanced/air-gap/))
  - Warm offline vulnerability scans took 0.14-0.19 s.
- **OSV-Scanner** (Google, Apache-2.0).
  - `--offline` sends nothing anywhere and errors if no local DB exists. `--offline-vulnerabilities --download-offline-databases` refreshes the DB. "Commit-level scanning is not supported in offline mode." ([offline docs](https://google.github.io/osv-scanner/usage/offline-mode/))
  - Measured: 16 s offline, because it loads a 210 MB per-ecosystem zip, against 3.4 s online.
  - It respects `.gitignore`, so it silently found "No package sources" until lockfiles were passed with `-L`.
  - Its license scanning uses the deps.dev API and fails offline ("cannot retrieve licenses locally"). ([license docs](https://google.github.io/osv-scanner/usage/license-scanning/))
  - Verdict: good data, but slower offline than Trivy and needs network for licenses. Keep it as the alternative.
- **npm audit**: sends the dependency tree to the registry's bulk advisory endpoint, so it needs the network. ([npm docs](https://docs.npmjs.com/cli/v11/commands/npm-audit))
- **pip-audit**: queries the PyPI or OSV APIs. Its own README says that without `--no-deps`/`--require-hashes` it resolves dependencies, which is "functionally equivalent" to `pip install` ("If you wouldn't `pip install` it, you should not `pip audit` it"). That can run untrusted build code, so it breaks the gate's execution rule. ([README](https://github.com/pypa/pip-audit))
- **Noise.** Most "vulnerable dependency" alerts are not exploitable in the client. Zapata et al. (ICSME 2018) found up to 73.3% of npm projects that depend on a vulnerable package never call the vulnerable function. We only have this figure through papers that cite it, e.g. [Exploiting Library Vulnerability via Migration-Based Automating Test Generation](https://arxiv.org/abs/2312.09564), not the original PDF.
  - Gate rule that follows: report CVEs only for dependencies **added or upgraded in this PR**. Block only on CRITICAL/HIGH where a fixed version exists. Treat the existing backlog as out of scope.
  - Neither Trivy nor OSV-Scanner does call-graph reachability for JS/Python. OSV-Scanner's call analysis covers Go/Rust only.
- **Typosquats and malicious packages.**
  - Socket CLI needs `SOCKET_CLI_API_TOKEN` for scans, so it requires an account and the network. ([README](https://github.com/SocketDev/socket-cli))
  - GuardDog (Datadog, Apache-2.0) runs Semgrep/YARA heuristics on package source and metadata, but it downloads each package. On our seeded typosquat it reported "No risks found" because the name did not exist on PyPI. ([README](https://github.com/DataDog/guarddog))
  - Research tools such as SpellBound report a low warning volume ("0.5% of total package installs") and 2.5% install-time overhead. ([arXiv 2003.03471](https://arxiv.org/abs/2003.03471)) The later ConfuGuard work reports name-similarity false-positive rates in the 13-80% range depending on method, so pure edit distance is noisy. ([arXiv 2502.20528](https://arxiv.org/abs/2502.20528))
  - Recommendation: deterministic edit distance against a bundled popular-package list, applied only to **newly added** dependency names. That covers both the typosquat and the "AI-hallucinated package" case. Treat a hit as a note that goes to AI judgment, not as a block.

### SAST / dataflow

- **Semgrep CE** (engine LGPL-2.1).
  - Diff mode is `--baseline-commit <sha>`: "Only show results that are not found in this commit hash". It needs a clean working tree. ([CLI reference](https://docs.semgrep.dev/cli-reference))
  - Diff-aware scans "only consider changes within modified files" and "Cross-file analysis is not supported for diff-aware scans". ([CI docs](https://docs.semgrep.dev/semgrep-ci/configuring-blocking-and-errors-in-ci))
  - When the head has no findings, Semgrep skips the baseline scan entirely ("Skipping baseline scan, because there are no current findings"). That is why the clean real diff took 3.5 s.
  - CE taint is single-function ("can't track data beyond a single function or file"). The vendor says CE "may find more false positives" than Pro. ([CE vs Pro](https://docs.semgrep.dev/semgrep-pro-vs-oss))
  - `--config auto` and `p/...` fetch rules from the registry, which is network plus metrics. Offline use needs local rule files.
- **Semgrep registry rules** are under the **Semgrep Rules License v1.0**: "You may use the rules only for your own internal business purposes", "This license does not allow you to distribute the rules … or to make them available to others as a service". ([license](https://semgrep.dev/legal/rules-license))
  - An MIT skill therefore must not vendor them. Having the user's own machine fetch them is the user's own internal use, but it is a grey area. The safest route is a ruleset the project writes and owns, plus repo-declared rules.
  - The Opengrep fork of the pre-change rules (`opengrep/opengrep-rules`) is archived and carries LGPL-2.1 **plus the Commons Clause** ("intended for research, testing & benchmarking"), so it is not a clean source either.
- **Opengrep** (LGPL-2.1, fork of Semgrep, active, v1.33.1 from 2026-10-29).
  - Runs Semgrep rules unchanged, supports `--baseline-commit`, and adds intrafile inter-method taint.
  - Its README documents cross-file taint (`--enable-interfile`), but the released 1.33.1 binary rejected that flag. It is on main, not yet released. ([repo](https://github.com/opengrep/opengrep))
  - Slower than Semgrep in our runs: 10.9 s vs 3.5 s on the clean diff, and it does not skip the baseline pass.
  - It found the same seeded issues plus `tainted-sql-string`. It is the better long-term engine bet because no feature is held back for a commercial tier.
- **CodeQL** - excluded by its license. The CLI may only be used on OSI-licensed open-source code; CI/automated database generation is allowed only for open source hosted on GitHub.com; and you may not "share, publish, distribute or lend the Software", unless the user pays for GitHub Advanced Security. ([CodeQL terms](https://github.com/github/codeql-cli-binaries/blob/main/LICENSE.md))
  - Speed: 95 s for JS alone on 113k lines.
  - Diff-informed analysis (CLI ≥ 2.21) only *filters alerts* to changed lines. Overlay analysis (≥ 2.23.9 for JS/Python) reuses a cached base database, with a claimed "up to 10x" speed-up. ([incremental analysis](https://docs.github.com/en/code-security/how-tos/find-and-fix-code-vulnerabilities/scan-from-the-command-line/incremental-analysis))
  - Use it only as an opt-in for users who already have a CodeQL entitlement.
- **Bearer CLI** - Elastic License 2.0: free inside your own organisation, but not as a hosted or managed service for third parties. OSS edition: Go/Java/JS/TS/PHP/Python/Ruby. ([repo](https://github.com/Bearer/bearer))
  - It must not be bundled; users who install it themselves could plug it in. Not recommended as a default.
- **Accuracy evidence (all modest):**
  - Charoenwet et al., ISSTA 2024, 815 vulnerability-contributing commits in C/C++: "a single SAST can produce warnings in vulnerable functions of 52% of VCCs", and "at least 76% of the warnings in vulnerable functions are irrelevant". ([arXiv 2407.12241](https://arxiv.org/abs/2407.12241))
  - Endor Labs vendor benchmark (8 projects including Juice Shop): Semgrep OSS/Opengrep precision "near 0.15" with "over 460" false positives; CodeQL recall "topped out at 0.211". ([Endor Labs](https://www.endorlabs.com/learn/ai-sast-benchmark-2x-more-real-vulnerabilities)) Vendor source, treat with caution.
  - SAST-Genius preprint: an LLM triage pass cut Semgrep false positives from 225 to 20 (precision 35.7% to 89.5%). ([arXiv 2509.15433](https://arxiv.org/abs/2509.15433))
  - Implication: SAST hits on changed lines are *candidates*. Block only on high-confidence rules (e.g. hard-coded secret patterns, `eval` of request data). Send the rest to the AI judge with the dataflow context. This is exactly the "deterministic first, AI for what is left" split.

### License compliance

- **Trivy `--scanners license`** classifies each package license as Forbidden, Restricted, Reciprocal, Notice, Permissive, Unencumbered or Unknown, from package metadata, at confidence ≥ 0.9 by default. ([docs](https://trivy.dev/latest/docs/scanner/license/))
  - It worked offline here: 1,028 licenses in 0.15 s, including 1 CRITICAL (WTFPL, "forbidden") and 12 reciprocal licenses.
  - `--license-full` (scans file contents) is "expensive. It takes a while", so keep it off the critical path.
- Gate rule: report only licenses of **newly added** dependencies whose category is Forbidden, Restricted or Unknown, checked against a policy the repo declares.
- OSV-Scanner `--licenses` needs deps.dev, so it is online only.
- ScanCode Toolkit (Apache-2.0 code, CC-BY-4.0 data) is the reference engine for license text inside source files. Use it only on changed vendored files: it is the most accurate, but a full-repo run is far too slow for this budget. ([repo](https://github.com/aboutcode-org/scancode-toolkit))

### IaC

- **Trivy config** (Apache-2.0): 1.3 s on the changed Terraform directory and 1.5 s on the whole repo with the embedded checks. It found the public bucket with 6 HIGH findings.
- **Checkov** (Apache-2.0):
  - 2.9 s on one directory, 4.7 s on one file, 82 s on the whole repo with all frameworks (GitHub Actions, secrets, OpenAPI and so on).
  - `--skip-download` avoids Prisma Cloud calls. ([CLI reference](https://www.checkov.io/2.Basics/CLI%20Command%20Reference.html))
  - Graph checks such as `CKV2_*` need the whole module directory, so pass the *directories* that contain changed IaC files, not single files.
- KICS (Checkmarx, Apache-2.0, active) is another option; not measured.
- Recommendation: Trivy as the default, since it is the same binary as dependency scanning. Report only findings in changed files, or new versus base.

### Auth / permission changes

- We found no maintained OSS tool that reports *removed* or *weakened* auth between two commits. SAST tools see only the head, and all of them missed our deleted `isAuthorized()` line.
- **route-detect** (BSD-3) ships Semgrep rules that classify each route as authenticated/authorized or not. It covers Django, DRF, Flask, FastAPI, Express, and more for other stacks.
  - Its README pins `semgrep<1.97.0`, because snippet output moved behind Semgrep's cloud app. ([repo](https://github.com/mschwager/route-detect))
- Recommended design, deterministic and diff-scoped:
  1. Run route-inventory rules on the changed files at BASE and at HEAD.
  2. Build `{route -> auth markers}` for each.
  3. Emit a finding for (a) any route whose auth markers shrank, and (b) any new route with no auth marker.
  4. Separately, flag diffs touching declared security-sensitive paths (auth middleware, permission/role models, CORS/CSP config, `CODEOWNERS`) for AI review against the change intent.
- Cost: about two Semgrep diff runs (2 × 3-7 s). Tree-sitter queries are the fallback for frameworks without rules.

### Destructive data migrations

- **Squawk** (Apache-2.0, Postgres only).
  - Lints SQL files with no database connection; the playground runs as WASM in the browser. ([docs](https://squawkhq.com/docs/))
  - Rules include `ban-drop-column`, `ban-drop-table`, `ban-drop-database`, `renaming-column`, `renaming-table`, `changing-column-type`, `adding-not-nullable-field`, `require-concurrent-index-creation` and ~20 more `ban-drop-*` rules. ([rules](https://squawkhq.com/docs/rules))
  - Caught all three seeded destructive statements in 3.1 s.
- **django-migration-linter** (Apache-2.0) flags `NOT_NULL`, `DROP_COLUMN`, `DROP_TABLE`, `RENAME_COLUMN`/`RENAME_TABLE`, `ALTER_COLUMN` and `ADD_UNIQUE`. It has `--git-commit-id` for "only migrations since this commit" and `--sql-analyser` for sqlite/mysql/postgresql. ([incompatibilities](https://github.com/3YOURMIND/django-migration-linter/blob/main/docs/incompatibilities.md), [usage](https://github.com/3YOURMIND/django-migration-linter/blob/main/docs/usage.md))
  - It runs inside the Django project to produce SQL with `sqlmigrate`. That is "app in memory", allowed; it is not a real database.
- **strong_migrations** (Rails, MIT) checks when migrations run, against a real database, so it does not fit the gate. ([README](https://github.com/ankane/strong_migrations))
- **Atlas `migrate lint`** has good analyzers: DS101-103 destructive, MF101-104 data-dependent, BC101-104 backward-incompatible. ([analyzers](https://atlasgo.io/lint/analyzers)) But since v0.38 it is "available only to Atlas Pro users" and needs a `--dev-url` dev database. ([lint docs](https://atlasgo.io/versioned/lint)) Excluded.
- Language-agnostic recipe:
  1. Find migration files in the diff by path convention or by the repo's declared migration command.
  2. Get SQL: raw `.sql`, `sqlmigrate`, `prisma migrate diff --script`, `alembic upgrade --sql`, and so on.
  3. Run Squawk on that SQL.
  4. For non-Postgres dialects, fall back to a small parser-based check (tree-sitter-sql) for `DROP`/`RENAME`/`ALTER TYPE`/`TRUNCATE`/`DELETE` without `WHERE`.

## Licensing summary for a public MIT tool

| Tool | License | OK to call as a user-installed subprocess? | OK to bundle/redistribute? |
|---|---|---|---|
| Gitleaks | MIT | Yes | Yes |
| Trivy, OSV-Scanner, Checkov, KICS, Squawk, detect-secrets, GuardDog, pip-audit, django-migration-linter | Apache-2.0 | Yes | Yes (keep NOTICE) |
| route-detect rules | BSD-3 | Yes | Yes (keep notice) |
| Semgrep CE / Opengrep engines | LGPL-2.1 | Yes | Unmodified binary yes, with LGPL notice |
| Semgrep registry rules | Semgrep Rules License v1.0 | User's own internal use only | **No** |
| opengrep-rules (archived) | LGPL-2.1 + Commons Clause | Research/benchmarking | Risky |
| TruffleHog | AGPL-3.0 | Yes (separate process) | Avoid |
| Bearer CLI | Elastic License 2.0 | User's own internal use | **No** (not as a service) |
| CodeQL CLI | GitHub CodeQL Terms | Only on OSI open-source code, or with paid GHAS | **No** |
| Atlas `migrate lint` | Pro feature | Needs a login | **No** |
| Socket CLI | MIT client, but needs an API token | Online only | n/a |

## Risks

1. **The scanners are themselves supply-chain targets.** In March 2026, attackers force-pushed 76 of 77 `trivy-action` tags and published a malicious Trivy v0.69.4 binary (CVE-2026-33634, in CISA's KEV list). ([Aqua Security](https://www.aquasec.com/blog/trivy-supply-chain-attack-what-you-need-to-know/), [OSV GHSA-69fq-xp46-6x23](https://osv.dev/vulnerability/GHSA-69fq-xp46-6x23))
   - The gate must pin exact versions and verify checksums/sigstore signatures. It must never auto-update scanners or use `latest`.
2. **Rule licensing, not engine speed, is the main SAST risk.** The good rules (Semgrep registry) cannot ship in an MIT tool, so the gate needs its own maintained ruleset or repo-declared rules. That is ongoing work and directly limits SAST recall on the benchmark.
3. **SAST precision is low** (about 15-50% in the evidence above), so blocking on raw SAST output would make the gate noisy. Blocking rules must be a short, high-confidence allowlist, with the rest going to AI judgment within the token cap.
4. **Offline data goes stale.** Trivy/OSV databases must be refreshed outside the verdict (e.g. a daily `merge-gate refresh`), and the verdict should report the DB age. The offline cache is per ecosystem (OSV) or large (Trivy, 1.6 GB on disk).
5. **Gaps with no off-the-shelf tool** (auth regressions, typosquats of names not in the registry, non-Postgres migrations) need custom deterministic code. Each needs its own seeded cases in the bug benchmark.

## Not verified

- detect-secrets precision (not in the ESEM 2023 table).
- Zapata 2018's 73.3% figure (only through citing papers).
- Checkov's network behaviour without `--skip-download`.
- django-migration-linter's need for a database connection.
- Opengrep `--enable-interfile` speed (not in the released binary).
- CodeQL overlay-mode timings.
- Timings on a Python-heavy repo: only Juice Shop (TS/JS) was timed end to end; NetBox was used only for pip-audit/OSV.

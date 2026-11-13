# Boundary contracts: detecting breaking changes without running services

Research ticket for the merge gate. Question: which tools detect breaking changes at system boundaries (HTTP APIs, DB schemas and migrations, config/env vars, message schemas, cross-language types, package public API) statically, how fast they are, how to infer a contract when the repo has no schema file, and where consumer-driven contract testing (Pact) fits without running services.

Date: 2026-11-13. Timings are my own runs on the user's laptop (Apple Silicon, macOS), warm cache, unless marked as a vendor claim. The experiments ran in an uncommitted `tmp-exp/` folder.

## Answer in one paragraph

Every boundary except env vars has a mature, fast, deterministic open-source diff tool, and all of them return in well under 3 seconds (most under 1 second). The pattern is the same everywhere: **get a machine-readable contract for base and head, diff it with the boundary's dedicated tool, and pass only the structured findings to the AI.** The hard part is not the diffing. It is *getting the contract* when the repo has no checked-in schema. The best fix is to generate the schema in-process from the framework (FastAPI `app.openapi()`, NestJS `SwaggerModule.createDocument`, Strawberry `export-schema`, tsoa, `tsc --declaration`). This is allowed under "app running in memory with fakes" and it took about 1 second per version in my test. A tree-sitter route and env-var extractor is the fallback for frameworks that cannot do this. Pact fits as a **static** check: if consumer pact files are present, compare them against the provider's (inferred) OpenAPI with `openapi-pact-comparator` (MIT), which took 16-46 ms. The gate should not run Pact's live provider verification.

## Tools by boundary

### HTTP APIs: OpenAPI

| Tool | What it does | Speed (measured) | Notes |
|---|---|---|---|
| [oasdiff](https://github.com/oasdiff/oasdiff) (Go, Apache-2.0) | `breaking` / `changelog` between two OpenAPI 3.0/3.1 docs. Output can be text, JSON, YAML, markdown or HTML. The JSON output includes the source line and column for each finding. | **0.02 s** on a small spec. **2.9 s** on Stripe's 8.3 MB, 431-path, 1,538-schema spec | v-main build: 787 rules (`oasdiff checks changelog`): 359 error, 23 warning, 404 info. `--fail-on ERR` sets the exit code. Supports deprecation and sunset policy and stability levels. |
| [Optic](https://github.com/opticdev/optic) | Diff and lint OpenAPI | n/a | **Archived 2026-01-12** (repo page). Do not adopt. |

What I saw with oasdiff:
- On a hand-written spec it caught all three planted breaks: a path parameter changed `string` to `integer`, a request property became required, and a required response property was removed. It gave each one a rule id and a line number.
- **Removing an *optional* response property is `info`, not breaking, by default.** On Stripe I deleted `customer.email`. `breaking` reported nothing. `changelog` reported it 1,418 times, once for every place the schema is reachable. So severity policy is a gate decision, and the output must be **grouped by rule id and schema pointer** before any AI sees it. Otherwise one edit costs thousands of tokens.

### HTTP APIs: GraphQL

- [graphql-inspector `diff`](https://the-guild.dev/graphql/inspector/docs/commands/diff) (MIT). It sorts each change into breaking, dangerous or non-breaking. Rules include `dangerousBreaking`, `suppressRemovalOfDeprecatedField`, `safeUnreachable`, and `considerUsage` (feeds in real usage data), plus custom JS rules. The process fails if there is any breaking change. **Measured 1.27 s via npx**, mostly Node startup. It correctly flagged a new required argument and a removed field as breaking, and a new enum value as dangerous.
- [graphql-inspector `validate`](https://the-guild.dev/graphql/inspector/docs/commands/validate) checks client operations against a schema, including deprecated usage. In a monorepo this is a free *consumer-side* contract check: the client queries in the repo are the consumer contract.

### RPC and message schemas

- **Protobuf: [`buf breaking`](https://buf.build/docs/breaking/rules/)** (Apache-2.0). Four strictness levels: FILE (the default, protects generated code per file), PACKAGE, WIRE_JSON (the recommended minimum) and WIRE. Local `buf` 1.75 lists 64 breaking rules. **Measured 0.6 s.** It caught a field type change from int32 to int64, and a field renamed on the same field number (which breaks JSON). Note: building buf from source took 58 s. Use the release binary.
- **AsyncAPI: [@asyncapi/diff](https://github.com/asyncapi/diff)** (Apache-2.0). Labels each change breaking, non-breaking or unclassified, and the classification can be overridden per JSON pointer. It has no parser of its own, so it needs dereferenced documents from `@asyncapi/parser`. The project is small (28 stars), so treat it as low-maturity.
- **Avro, Protobuf and JSON Schema in Kafka style:** Confluent's [compatibility modes](https://docs.confluent.io/platform/current/schema-registry/fundamentals/schema-evolution.html) (BACKWARD (the default), FORWARD, FULL, and the `_TRANSITIVE` variants) are the standard vocabulary. The docs only describe checking through the registry or its Maven plugin. A registry is a real service, so it is out of bounds for the gate. Offline options are the Avro library's own compatibility check, `buf breaking` for proto, or JSON Schema diffing. I have not verified a high-quality offline JSON Schema diff tool. That is an open gap.

### Cross-service and cross-language types

There is no universal tool. Cross-language type safety exists only when the types come from an IDL: Protobuf, OpenAPI, GraphQL SDL, AsyncAPI, Avro, or Smithy/TypeSpec, which compile to those. The gate should **treat the IDL as the boundary and diff the IDL**, using the tools above. When no IDL exists, the cross-language boundary *is* the HTTP or message contract, so the inference strategy below applies.

### Package public API

| Ecosystem | Tool | Classifies breaking? | Measured | Known blind spots (from the tool's own docs) |
|---|---|---|---|---|
| Python | [griffe check](https://mkdocstrings.github.io/griffe/guide/users/checking/) (ISC) | Yes. Categories include parameter removed, moved, kind changed or default changed, required parameter added, object removed or kind changed, and base class removed | **0.7 s**, including griffe creating a git worktree for the base ref | Return-type and attribute-type changes are "Not yet supported". I confirmed it missed `str` to `int` on a parameter annotation. |
| TS/JS | [API Extractor](https://api-extractor.com/pages/setup/configure_api_report/) (MIT) | **No.** It writes an `.api.md` report and flags *any* signature change | **1.4 s** (plus `tsc --declaration`) | Needs `.d.ts` output and setup for each package. Breaking vs additive has to be decided on the report diff, by rules or a small AI call. |
| Rust | [cargo-semver-checks](https://github.com/obi1kenobi/cargo-semver-checks) | Yes, "many dozens" of lints, and the stated goal is zero false positives | not run | It misses type changes on fields or parameters, generics and lifetimes, and feature-gated APIs. |
| Go | [apidiff](https://pkg.go.dev/golang.org/x/exp/apidiff) | Yes | not run | Its docs say it is "only an approximation". Behaviour changes are out of scope, and it leaves constant-value changes to the user. |

Common theme: **these tools catch shape changes (names, arity, required-ness) and are weak on type changes.** For TS, compile-time assignability is a cheap way to cover that gap. Write a tiny consumer file that imports the *base* version's types and assigns them to the head version's types, then run `tsc` on it. This is an idea, not a tool I measured.

### Database schemas and migrations

| Tool | Scope | Needs a DB? | Measured | Notes |
|---|---|---|---|---|
| [squawk](https://squawkhq.com/docs/rules) (Apache-2.0) | Postgres SQL files | **No.** It lints SQL text. | **0.05 s** native, 0.9 s via npx | About 80 rules: locks (`require-concurrent-index-creation`, `changing-column-type`, `constraint-missing-not-valid`, `adding-foreign-key-constraint`), destructive changes (`ban-drop-column`, `ban-drop-table`, renames), and timeouts. `--pg-version` and `.squawk.toml` are supported, and the reporters are tty, gcc, json and gitlab. It correctly did *not* flag `ADD COLUMN ... NOT NULL DEFAULT 'a'`, which is safe on PG11 and later. Output is noisy (12 findings on 5 statements, mostly `prefer-robust-stmts`), so set an include list. |
| [Atlas `migrate lint`](https://atlasgo.io/versioned/lint), [analyzers](https://atlasgo.io/lint/analyzers) | 9 dialects | **Yes, `--dev-url`** (examples use `docker://`) | not run | **Requires Atlas Pro since v0.38**, and its lock checks (PG1xx, PG3xx, MySQL) are Pro-only. Paid, and it needs a container, so it fails the no-real-services rule. Its check codes (BC, DS, MF, PG3xx) are still a good taxonomy to copy. |
| [strong_migrations](https://github.com/ankane/strong_migrations) | Rails: PG, MySQL, MariaDB | Runs while the migration executes | n/a | Not static. It is a good catalogue of dangerous operations, and its README states it "does not detect dangerous backfills". |
| [django-migration-linter](https://github.com/3YOURMIND/django-migration-linter) | Django | Uses `sqlmigrate`, which per the [Django docs](https://docs.djangoproject.com/en/5.2/ref/django-admin/) "requires an active database connection" | n/a | Checks: NOT NULL without a default, drop or rename of a column or table, alter column, add unique. |

ORM-generated migrations need **SQL text** before squawk can lint them:
- **Alembic:** offline `--sql` mode prints SQL with no DB ([docs](https://alembic.sqlalchemy.org/en/latest/offline.html)).
- **Prisma:** `migrate diff --from-schema --to-schema --script` compares two schema files ([CLI ref](https://www.prisma.io/docs/orm/reference/prisma-cli-reference)).
- **Django:** `sqlmigrate` needs a connection. The in-process answer is [PGlite](https://github.com/electric-sql/pglite), Postgres compiled to WASM: about 3 MB gzipped, in-memory, single connection. It fits "in memory with fakes". Not tested with Django here.
- `makemigrations --check` and `alembic check` catch the opposite bug: the model changed but there is no migration.

### Config and env vars

No high-trust, cross-language tool exists:
- [dotenv-linter](https://github.com/dotenv-linter/dotenv-linter) lints `.env` files and diffs them against each other, but it does not scan code.
- The tools that compare code usage against `.env.example` (env-check, env-grep, envsleuth and others in search results) are small single-language projects. I would not adopt them.

Recommendation: write about 30 lines of tree-sitter queries per language for reads such as `process.env.X`, `os.environ["X"]`, `os.getenv("X")`, and settings classes. Diff the set of required env vars between base and head:
- A newly required var with no default and no entry in `.env.example`, Helm values or a compose file is a finding.
- A var that disappears from code but stays in deploy config is a note.
- Dynamic keys are a known blind spot and should be reported as "unknown".

## Inferring contracts when the repo has no schema

In order of preference:

1. **Framework export, in-process.** This is allowed under "in memory with fakes" and is exact for whatever the framework knows.
   - **FastAPI:** `app.openapi()` is a plain method ([docs](https://fastapi.tiangolo.com/how-to/extending-openapi/)). I measured importing the app and dumping OpenAPI for base and head at about 2 s in total, then `oasdiff` at 0.1 s. It caught a removed endpoint, a new required query parameter, a parameter type change, and a response field that became optional.
   - **NestJS:** `SwaggerModule.createDocument(app, config)` needs an app instance, but no `listen()` ([docs](https://docs.nestjs.com/openapi/introduction)).
   - **tsoa:** generates the spec from TS controllers and types ([repo](https://github.com/lukeautry/tsoa)).
   - **Strawberry:** `strawberry export-schema` prints SDL ([docs](https://strawberry.rocks/docs/guides/schema-export)).
   - The same idea covers DRF (drf-spectacular), Spring (springdoc), and `tsc --declaration` or griffe for library APIs.
   - Cost: base and head each need dependencies installed, and module import can have side effects. Run it inside the same sandbox as the gate's other in-memory execution.
2. **Static extraction.** Use when the app cannot be imported, for example Express without types, or import side effects.
   - The research prior art is Java-only:
     - [Respector (ICSE 2024)](https://conf.researchr.org/details/icse-2024/icse-2024-research-track/137/Generating-REST-API-Specifications-through-Static-Analysis) uses static and symbolic analysis on Spring and Jersey.
     - [AutoOAS (Lercher et al. 2024)](https://arxiv.org/abs/2410.23873) beat Respector by "39% higher precision and 35% higher recall" on parameters and "29% higher precision and 11% higher recall" on responses, on 7 Spring Boot projects. It also showed that runtime springdoc failed on 3 of 7 projects because no run configuration could be found.
   - For a language-agnostic gate this means tree-sitter queries for route registration (`app.get(...)`, decorators) and handler signatures. That gives **paths, methods and parameters reliably, but body schemas poorly**.
   - Tree-sitter is designed to be "fast enough to parse on every keystroke" ([docs](https://tree-sitter.github.io/tree-sitter/)).
3. **AI extraction as the last resort, scoped to the changed handlers only.** [LRASGen](https://arxiv.org/abs/2504.16833) uses an LLM to generate OpenAPI from source. Across 20 APIs it covered "48.85% more missed entities than the developer-provided specifications". That is useful as a *coverage* signal, not as a contract to block on. In the gate, AI should only describe the old and new shape of the handlers that were touched, within a small token budget. It should never regenerate the whole spec.

Traffic-capture inference (Optic capture, Postman/Akita) needs running services and real traffic, so it is out of scope.

## Where Pact fits

- **Pact as designed needs a running provider.** Verification runs "against a locally running instance of your provider", with downstream services stubbed ([docs](https://docs.pact.io/provider)). `can-i-deploy` needs a Pact Broker, which is network. Both are outside the gate's limits. Running a provider on loopback HTTP is borderline (see the open decision below).
- **The static version fits well.** This is PactFlow's bi-directional idea: compare the consumer's pact file to the provider's OpenAPI with no traffic ([overview](https://pactflow.io/bi-directional-contract-testing/)). The engine is open source: [`@pactflow/openapi-pact-comparator`](https://github.com/pactflow/openapi-pact-comparator) (MIT, v2.5.0). It replaces the archived swagger-mock-validator (vendor claim: "exceeding 20x" faster) and also handles AsyncAPI with Avro or Protobuf payloads.
  - **Measured: 16-46 ms per pact and spec pair, 0.2 s including Node startup.** Against my changed spec it flagged a path parameter that must now be an integer and a request body missing the new required `name`.
  - **Blind spot:** the consumer still expected `email` in the response after the provider dropped it from the spec. That passed, because the spec does not forbid extra properties. A pact-vs-spec check only catches what the spec *forbids*. Pair it with oasdiff's own `response-*-property-removed` rules.
- **How the gate uses it:** if `pacts/*.json` (or a configured path) exist in the repo, compare each pact with the head OpenAPI. If the repo *is* a consumer, compare its pact with the provider spec, but only if that spec is vendored in the repo. Fetching it from a broker is network.

## Recommended approach for the gate

1. **Detect boundaries** from changed files and the repo manifest. Examples: `openapi.*`, `*.graphql`, `*.proto`, `asyncapi.*`, migration directories, `package.json` `types` / `exports`, `pyproject` packages, framework imports (FastAPI, Nest, Express), env reads.
2. **Get the contract for base and head.** Use the checked-in file if there is one. Otherwise use the framework export in-process. Otherwise use the tree-sitter extractor.
3. **Run the dedicated diff tool:**
   - oasdiff, graphql-inspector, buf, @asyncapi/diff and openapi-pact-comparator for service and message contracts;
   - squawk on migration SQL (generated offline for ORMs);
   - griffe for Python, API Extractor for TS/JS, cargo-semver-checks for Rust, apidiff for Go;
   - the env-var set diff.
4. **Normalise** the results into findings (rule id, location, severity), and group any fan-out by rule id and pointer.
5. **AI judges only two things:**
   - (a) **intent match:** is this break *intended and handled*? For example, the PR says "remove /health", the version was bumped, or a deprecation was added;
   - (b) breaking vs additive for API Extractor report diffs, which arrive unclassified.

Budget estimate: about 50-100 tokens per grouped finding, typically under 2-3k tokens for this whole area. That estimate is not measured.

**Wall-clock estimate:** contract extraction 1-3 s per version, plus diff tools at 0.05-3 s each, run in parallel. Expect under 10 s for this area on a typical PR, which is well inside the 2 minute budget. The real cost is installing dependencies for the base-revision checkout when the contract has to be exported from code.

## Accuracy: what is and is not known

- None of oasdiff, graphql-inspector, buf, squawk, griffe or API Extractor publishes precision or recall figures. Their rules are deterministic, so false positives are *policy* disagreements, not noise. The bug benchmark should measure them.
- Documented false negatives:
  - type changes: griffe, cargo-semver-checks, apidiff;
  - optional response field removal is only `info` in oasdiff;
  - extra-property expectations: pact-vs-spec;
  - backfills: strong_migrations;
  - dynamic env keys.
- The only published accuracy numbers for contract inference are the Java-only AutoOAS and Respector results above, plus LRASGen's coverage claim.

## Main risk

**Contract-inference coverage.** The diff tools are strong, but the gate only has a contract for frameworks it knows how to export from. Everything else falls back to a tree-sitter extractor whose accuracy on body schemas is unmeasured, and which may be poor. A second risk is **severity policy**: what counts as "breaking" differs by API, for example internal vs public, or versioned paths. A hard-coded policy will either block legitimate changes or miss real ones. The fix is to read the repo's own tool configs (`.oasdiff.yaml`, `buf.yaml`, `.squawk.toml`) and let the AI weigh breaks against the stated change intent.

## Open decision for the owner

Should the gate allow the app under test to listen on **loopback HTTP** (127.0.0.1, ephemeral port)? Pact's provider verifier and some route-listing tools need it. Recommendation: **no**. Keep "no network" strict and use the static pact-vs-OpenAPI comparison. Everything above works without loopback.

## Sources

- oasdiff: https://github.com/oasdiff/oasdiff
- Optic (archived): https://github.com/opticdev/optic
- graphql-inspector diff: https://the-guild.dev/graphql/inspector/docs/commands/diff ; validate: https://the-guild.dev/graphql/inspector/docs/commands/validate
- buf breaking rules: https://buf.build/docs/breaking/rules/
- @asyncapi/diff: https://github.com/asyncapi/diff
- Confluent schema evolution: https://docs.confluent.io/platform/current/schema-registry/fundamentals/schema-evolution.html
- griffe check: https://mkdocstrings.github.io/griffe/guide/users/checking/
- API Extractor report: https://api-extractor.com/pages/setup/configure_api_report/
- cargo-semver-checks: https://github.com/obi1kenobi/cargo-semver-checks
- Go apidiff: https://pkg.go.dev/golang.org/x/exp/apidiff
- squawk rules / CLI: https://squawkhq.com/docs/rules ; https://squawkhq.com/docs/cli
- Atlas lint / analyzers: https://atlasgo.io/versioned/lint ; https://atlasgo.io/lint/analyzers
- strong_migrations: https://github.com/ankane/strong_migrations
- django-migration-linter: https://github.com/3YOURMIND/django-migration-linter ; checks: https://github.com/3YOURMIND/django-migration-linter/blob/main/docs/incompatibilities.md
- Django sqlmigrate / makemigrations: https://docs.djangoproject.com/en/5.2/ref/django-admin/
- Alembic offline: https://alembic.sqlalchemy.org/en/latest/offline.html
- Prisma CLI: https://www.prisma.io/docs/orm/reference/prisma-cli-reference
- PGlite: https://github.com/electric-sql/pglite
- dotenv-linter: https://github.com/dotenv-linter/dotenv-linter
- FastAPI OpenAPI: https://fastapi.tiangolo.com/how-to/extending-openapi/
- NestJS OpenAPI: https://docs.nestjs.com/openapi/introduction
- tsoa: https://github.com/lukeautry/tsoa
- Strawberry export-schema: https://strawberry.rocks/docs/guides/schema-export
- tree-sitter: https://tree-sitter.github.io/tree-sitter/
- Respector (ICSE 2024): https://conf.researchr.org/details/icse-2024/icse-2024-research-track/137/Generating-REST-API-Specifications-through-Static-Analysis
- AutoOAS: https://arxiv.org/abs/2410.23873
- LRASGen: https://arxiv.org/abs/2504.16833
- Pact provider verification: https://docs.pact.io/provider ; how Pact works: https://docs.pact.io/getting_started/how_pact_works
- PactFlow bi-directional: https://pactflow.io/bi-directional-contract-testing/
- openapi-pact-comparator: https://github.com/pactflow/openapi-pact-comparator ; swagger-mock-validator (archived): https://github.com/pactflow/swagger-mock-validator

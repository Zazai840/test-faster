# Public-facing behaviour, verified in memory

Research ticket for `merge-gate`. Question: how can the gate verify public-facing behaviour (HTTP endpoints, CLI output, rendered UI, emitted events) by running the app in-process with fakes, with no network and no browser? Where does this miss bugs that real E2E catches, how big is that gap, and what cheap mitigations exist?

Date: 2026-11-13. Numbers marked **[measured]** come from a local experiment on an Apple M3 laptop (Node 26.7, Python 3.13; package versions listed in section 7). Everything else is cited.

## 1. Direct answer

Every mainstream web framework ships, or has a de facto, **in-process test client**: it builds a request object, hands it straight to the app's handler pipeline (routing, middleware, validation, serialisation, error handlers) and returns a real response object, with no socket and no server. CLI frameworks ship the same idea (a runner that invokes the command with argv and captures stdout, stderr and the exit code). For UI, a **DOM-in-memory** (jsdom or happy-dom) plus Testing Library renders components and lets the gate query what a user would perceive (roles, labels, text). For data stores and queues, **in-process fakes** stand in for real services; the best of them are the real engine compiled to run in-process (SQLite, PGlite), the weaker ones are re-implementations (pg-mem, fakeredis, moto).

This is fast: a request through a test client costs 0.04 to 2.4 ms after a 0.1 to 2 s one-off boot **[measured]**, so a few hundred behavioural probes fit in a few seconds of the 2-minute budget, and they cost zero LLM tokens because the assertions are deterministic.

What it cannot see is everything that only exists once a real browser, a real server process, or a real network is involved: **layout and CSS, browser-engine differences and browser-only APIs, production database semantics that the fake does not reproduce (concurrency, isolation, types, extensions), and deployment and network configuration** (proxies, TLS, CORS as enforced by browsers, DNS, timeouts, environment). Published evidence puts most backend failures inside the in-process reach (77% of 198 production failures in distributed data systems were reproducible by a unit test; Yuan et al., OSDI 2014). The frontend gap is real but not well quantified: layout and styling is the top-reported compatibility pain (48% of respondents, MDN 2020), and no tool in this design can verify it. Configuration is Google's number-one cause of major outages. The gate should therefore **verify what it can in memory, statically lint what it cannot run, and label the rest as "not verified" in the verdict** instead of pretending to cover it.

## 2. The in-process seam per public surface

The common pattern: the app must be constructible without starting a listener (an app factory or an exported app object). The test client then dispatches into the framework's request pipeline directly.

### HTTP

| Stack | In-process client | How it dispatches | Notes |
|---|---|---|---|
| Node, Express/Koa/any `http` handler | supertest | Binds the app to an ephemeral loopback port if not already listening ("if the server is not already listening for connections then it is bound to an ephemeral port for you") [supertest README](https://github.com/ladjs/supertest) | Uses loopback, not an external network. Works if the sandbox allows `127.0.0.1`. Slowest of the Node options **[measured: 0.49 ms/req]**. |
| Node, Fastify | `app.inject()` (light-my-request) | "built-in support for fake HTTP injection thanks to light-my-request"; `inject` "ensures all registered plugins have booted up" [Fastify testing guide](https://fastify.dev/docs/latest/Guides/Testing/) | No socket at all. Fastify recommends exporting a `build()` factory separate from `server.js` that calls `listen`. **[measured: 0.13 ms/req]** |
| Node/Bun/Deno/edge, Hono and other Web-standard (`fetch`-style) frameworks | `app.request(path, init)` | "create a Request and pass it to the Hono application to validate the Response" [Hono testing](https://hono.dev/docs/guides/testing) | **[measured: 0.07 ms/req]** |
| Next.js App Router route handlers (`app/**/route.ts`) | Call the exported `GET`/`POST` function with a Web `Request` | The handler is a plain function from `Request` to `Response` | **[measured: 0.04 ms/call]**. For `cookies()`, `headers()`, `params`, `NextRequest` use [next-test-api-route-handler](https://github.com/Xunnamius/next-test-api-route-handler), which "uses Next.js's internal resolvers"; its stated limits: middleware, caching, server actions and components are not emulated, and edge runtime is "cosmetic" (tests always run in Node). |
| Python, FastAPI/Starlette (ASGI) | `TestClient(app)` | Starlette's client built on HTTPX, calls the ASGI app directly [FastAPI testing](https://fastapi.tiangolo.com/tutorial/testing/) | Use `with TestClient(app)` to run lifespan startup/shutdown; swap dependencies through `app.dependency_overrides`. **[measured: 2.2 ms/req]**. Observed in this run: Starlette 1.10 emits "Using `httpx` with `starlette.testclient` is deprecated; install `httpx2` instead", so the gate must not hard-code the HTTP library behind it. |
| Python, Flask (WSGI) | `app.test_client()` | "makes requests to the application without running a live server" (extends Werkzeug's client) [Flask testing](https://flask.palletsprojects.com/en/stable/testing/) | **[measured: 0.15 ms/req]** |
| Python, Django | `django.test.Client` | "acts as a dummy web browser... does not require the web server to be running" [Django test tools](https://docs.djangoproject.com/en/5.2/topics/testing/tools/) | Django itself states the client "is not intended to be a replacement for Selenium" for rendered HTML behaviour and JavaScript. |
| Go | `net/http/httptest.NewRecorder` + `handler.ServeHTTP` | Direct call into the `http.Handler` [pkg.go.dev/net/http/httptest](https://pkg.go.dev/net/http/httptest) | `httptest.NewServer` uses loopback; prefer the recorder. |
| Rust, axum/tower | `ServiceExt::oneshot(request)` on the router | Direct call into the tower service [axum testing example](https://github.com/tokio-rs/axum/blob/main/examples/testing/src/main.rs) | |
| JVM, Spring | MockMvc / WebTestClient bound to the context | Dispatches through the servlet/reactive stack without a container [Spring MockMvc](https://docs.spring.io/spring-framework/reference/testing/mockmvc.html) | |
| .NET, ASP.NET Core | `WebApplicationFactory` + `TestServer` | In-memory server, `HttpClient` wired to it [ASP.NET integration tests](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests) | |
| Ruby, Rails/Rack | `ActionDispatch::IntegrationTest` / Rack::Test | Calls the Rack app directly [Rails testing guide](https://guides.rubyonrails.org/testing.html) | |

Generalisation: every one of these reduces to "call the app's top-level handler with a request value". The gate needs one small adapter per framework family (Node `http` handler, Web `fetch` handler, ASGI, WSGI, Go `http.Handler`, Rack, Servlet, ASP.NET), not per framework.

**Contract-level probing for free.** If the app publishes an OpenAPI schema, [Schemathesis](https://schemathesis.readthedocs.io/en/stable/reference/python/) can generate requests from it and send them straight into the app: `schemathesis.openapi.from_asgi("/openapi.json", app)` and `from_wsgi(...)` pass the app to the transport, and only `from_url` loaders go over the network. That gives deterministic, zero-token property tests for 5xx crashes and schema violations on changed endpoints. JS has no single equivalent; generated requests through `inject`/`app.request` do the same job.

### CLI

Click's `CliRunner` (and Typer, Flask's `test_cli_runner`, which extends it) "runs CLI commands in isolation and captures the output in a Result object" ([Flask testing](https://flask.palletsprojects.com/en/stable/testing/)) **[measured: 0.09 ms/invoke]**. For other languages, running the CLI binary as a child process with a scratch `HOME`/`cwd`, a fixed environment and the network sealed (section 5) is still "app running locally with fakes", costs tens of milliseconds per run, and is fully language-agnostic. Assert on stdout, stderr, exit code, and files written; snapshot the help text and output format, because CLI output is a public contract that scripts parse.

### Rendered UI

- **Server-rendered HTML** (Django/Flask/Rails templates, Express views, SSR): the HTTP test client returns the HTML; parse it into a DOM and assert on it.
- **Client components**: render with Testing Library into jsdom or happy-dom. Testing Library's guiding principle, "The more your tests resemble the way your software is used, the more confidence they can give you" ([Testing Library](https://testing-library.com/docs/guiding-principles)), is the right lens for a gate: query by role, label and visible text, which survives refactors and is close to what users perceive.
- **Next.js async Server Components**: Next.js states "Vitest currently does not support them... we recommend using E2E tests for `async` components" ([Next.js Vitest guide](https://nextjs.org/docs/app/guides/testing/vitest)). This is a framework-level hole in the in-memory approach, see section 8.

Cost: creating a jsdom window took 441 ms, happy-dom 203 ms **[measured, cold import included]**. Both returned a 0x0 bounding box for an element styled `width:200px;height:50px` **[measured]**, which is the layout gap in one line.

### Emitted events

- **In-process event bus or outbox**: if the app publishes through an interface (an emitter, a port, an outbox table), inject a recording fake and assert on what was emitted. This is the most robust option and needs no service fake.
- **Cloud queues and topics**: moto mocks AWS in-process: it "intercepts HTTP requests using a custom event handler that hooks into botocore's event-system" ([moto getting started](https://docs.getmoto.org/en/latest/docs/getting_started.html)) **[measured: 2.4 ms per SQS send+receive; 2.1 s cold start]**. Python-only in decorator mode; other languages need `moto_server` on loopback.
- **Redis pub/sub and streams**: fakeredis "runs entirely in-memory" with no "server, a container, or a network connection" ([fakeredis-py](https://github.com/cunla/fakeredis-py)) **[measured: 0.06 ms per set+get]**; a `TcpFakeServer` mode exposes it on a loopback socket for clients that open their own connection.
- **Kafka/RabbitMQ**: no credible in-process fake for Node or Python was found; JVM has `MockProducer`/`MockConsumer` in kafka-clients. Recommend the injected-port approach or treating broker semantics as unverified.

## 3. In-memory replacements for data stores: fidelity ladder

Rank fakes by fidelity, and prefer the highest rung that runs in-process:

1. **The real engine, in-process.** SQLite in-memory when production is SQLite. **PGlite** when production is Postgres: "a WASM Postgres build... it is simply Postgres in WASM", about 3 MB gzipped, supports extensions such as pgvector and PostGIS ([PGlite repo](https://github.com/electric-sql/pglite)). Limit: "only has a single exclusive connection" ([PGlite docs](https://pglite.dev/docs/)), so concurrency and isolation-level behaviour cannot be exercised. Boot **[measured: 1.27 s]**, then 0.27 ms per insert+select.
2. **A re-implementation that targets the real protocol or API.** pg-mem, fakeredis, moto. pg-mem says it is "a best effort to replicate PG" and lists "No support for timezones", "All number-like types are all handled as javascript numbers", "Indices implementations are basic", and an in-house parser where unimplemented features "will be considered as invalid syntaxes" ([pg-mem README](https://github.com/oguimbal/pg-mem)). moto does not enforce IAM by default: "it defaults to infinity, thus moto will never perform any authentication at all" ([moto IAM](https://docs.getmoto.org/en/latest/docs/iam.html)).
3. **A different engine standing in** (SQLite for a Postgres app). Lowest fidelity: SQLite documents that "Datatypes are advisory rather than mandatory", "foreign key constraint enforcement is turned off by default", PRIMARY KEYs can contain NULLs, and `upper()`/`lower()` "only work on ASCII characters" ([SQLite quirks](https://www.sqlite.org/quirks.html)).

Local probes, same SQL against real Postgres (PGlite) and pg-mem **[measured]**:

| Query | PGlite (real Postgres) | pg-mem |
|---|---|---|
| `select 0.1::numeric + 0.2::numeric` | `0.3` | `0.30000000000000004` |
| `select 12345678901234567.89::numeric(20,2)` | `12345678901234567.89` | `12345678901234568` |
| `timestamptz at time zone 'America/New_York'` | `2024-03-10 08:00:00` | error: operator does not exist |
| `jsonb_path_query_first(...)` | `2` | error: function does not exist |
| `'abc' ilike 'A%'` | `true` | `true` |

SQLite in-memory accepted a row with a foreign key to a missing parent and the text `'abc'` in an `integer` column, and `lower('ÄÖ')` returned `ÄÖ` **[measured]**; Postgres rejects the first two and returns `äö`.

Takeaway: a fake that silently computes money with floats will pass a test that production would fail, and vice versa it will reject valid production SQL (false block). For Postgres, PGlite should be the default; pg-mem only where PGlite cannot load.

## 4. Auto-detecting the framework and the app entry point

Deterministic, in this order (stop at the first that works):

1. **Reuse the repo's own harness.** Search the test tree for existing in-process clients (`supertest(`, `.inject(`, `app.request(`, `TestClient(`, `test_client(`, `django.test`, `httptest.NewRecorder`, `WebApplicationFactory`, `MockMvc`). Their fixtures already know how to build the app with fakes (app factory, dependency overrides, test settings). The gate's new probes import those fixtures. This is the highest-fidelity and lowest-risk path.
2. **Manifest lookup.** Read `package.json` dependencies, `pyproject.toml`/`requirements*.txt`, `go.mod`, `Gemfile`, `pom.xml`/`build.gradle`, `*.csproj`, `Cargo.toml`; map known framework packages to the adapter family in section 2. Map data-store clients (`pg`, `postgres`, `psycopg`, `prisma`, `redis`, `ioredis`, `boto3`, `@aws-sdk/*`) to the fake ladder in section 3.
3. **Locate the app object with tree-sitter queries.** Patterns such as `X = FastAPI(...)`, `X = Flask(__name__)`, `def create_app(`, `express()`, `Fastify(...)`, `new Hono()`, files under `app/**/route.{ts,js}` exporting `GET`/`POST`, `http.Handle`/`chi.NewRouter`, `Router::new()`. Import it in a sandboxed process with network sealed and fake env vars.
4. **Fall back to process-level.** If the app cannot be imported without real services (side effects at import time), start it as a child process on loopback with fakes injected through env vars (e.g. `DATABASE_URL` pointing at a PGlite socket server or SQLite file, `REDIS_URL` at a fakeredis `TcpFakeServer`, `AWS_ENDPOINT_URL` at `moto_server`). If even that fails, report "public behaviour not verified: app not constructible in isolation" as a code-health finding (a testability signal), not a crash.

Route discovery for "which endpoints did this PR touch": map changed lines to route registrations via the same tree-sitter queries, or ask the running app (FastAPI/Starlette `app.routes`, Fastify `printRoutes()`, Express router stack, OpenAPI document) and intersect with the diff.

## 5. Sealing the network (making "no network" enforceable)

- Python: [pytest-socket](https://github.com/miketheman/pytest-socket) `--disable-socket` makes tests "fail on any access to `socket`"; `--allow-hosts=127.0.0.1` keeps loopback for supertest-style clients.
- Node: the permission model ("added in v20.0.0, stable since v22.13.0/v23.5.0") restricts network access unless `--allow-net` is passed ([Node permissions](https://nodejs.org/api/permissions.html)); Node calls it a "seat belt", not a security boundary. For HTTP clients inside the app, MSW with the unhandled-request strategy set to `"error"` ("Print an error and halt") turns any unexpected outbound call into a visible failure ([MSW listen](https://mswjs.io/docs/api/setup-server/listen); the option is named `onUnhandledRequest` in older versions and `onUnhandledFrame` in the current docs).
- Language-agnostic: run the check process in an OS sandbox with networking disabled (macOS `sandbox-exec`, Linux network namespace / `unshare -n`). This is the only option that also covers child processes and native drivers.

An unexpected outbound call is itself a finding ("this change makes a network call in a path that was previously offline").

## 6. What it costs: runtimes and tokens

**[measured]** on Apple M3, single process, after warm-up, averaged over 200 calls (moto over 50):

| Thing | Cold start | Per operation |
|---|---|---|
| Next.js-style route handler, direct call | <1 ms | 0.04 ms |
| Hono `app.request` | ~14 ms first call | 0.07 ms |
| Fastify `inject` | 95 ms import + ready | 0.13 ms |
| Flask `test_client` | 405 ms import | 0.15 ms |
| Express + supertest | 86 ms import | 0.49 ms |
| FastAPI `TestClient` | 1.1 s import | 2.2 ms |
| Click `CliRunner` | negligible | 0.09 ms |
| jsdom window | 441 ms | n/a |
| happy-dom window | 203 ms | n/a |
| pg-mem | 50 ms | 0.23 ms insert+select |
| PGlite (real Postgres in WASM) | 1.27 s | 0.27 ms insert+select |
| fakeredis | 0.5 s import | 0.06 ms |
| moto (SQS) | 2.1 s | 2.4 ms |

Real apps are heavier (ORM setup, migrations, many plugins), so budget a 2 to 10 s boot per runtime and single-digit milliseconds per request. Even 500 probes stay under 10 s. The dominant cost will be dependency installation and the repo's own test suite, not the in-memory probes.

**Tokens:** the probes and assertions are deterministic, so 0 tokens per run. LLM use is limited to (a) writing probe requests for changed endpoints when the repo has no existing test touching them (bounded by the diff, a few thousand tokens), and (b) a short judgement on surfaces the gate cannot run (section 8), passing only the relevant diff hunks.

**Flakiness, the cost of the alternative:** Google measured 0.5% of small tests, 1.6% of medium and 14% of large tests as flaky over one week; Java WebDriver tests were 10.45% flaky and Python WebDriver tests 18.72%, against 1.65% for all tests ([Google Testing Blog, 2017](https://testing.googleblog.com/2017/04/where-do-our-flaky-tests-come-from.html)). A gate whose verdict must be trusted gains a lot from staying in the small/medium band.

## 7. Environment of the experiment

Experiment scripts lived in `tmp-exp/` inside the worktree and were not committed. Versions: express 4.22.3, supertest 7.3.2, fastify 5.12.8, hono 4.13.18, jsdom 31.0.0, happy-dom 21.0.4, pg-mem 3.0.14, @electric-sql/pglite 0.5.16; fastapi 0.162.5, starlette 1.10.0, httpx 0.28.1, flask 3.1.3, fakeredis 2.40.0, moto 5.2.5. Apple M3, Node 26.7.0, Python 3.13.

## 8. Where in-memory misses bugs that E2E catches, how big, and cheap mitigations

No published study measures "share of bugs only E2E can catch" directly. The evidence below bounds each gap category. Google's own taxonomy of what unit/medium tests miss is a useful frame: unfaithful doubles, configuration issues, issues under load, unanticipated inputs, and emergent "vacuum effect" behaviour ([Software Engineering at Google, ch. 14](https://abseil.io/resources/swe-book/html/ch14.html)).

### 8.1 CSS, layout and visual rendering

- **What is missed:** anything that depends on layout or paint: overlap, overflow, clipping, invisible-but-present elements, responsive breakpoints, z-index, fonts, colour contrast. jsdom states layout ("the ability to calculate where elements will be visually laid out as a result of CSS") is out of scope and "many layout properties return zeros" ([jsdom README](https://github.com/jsdom/jsdom)); both jsdom and happy-dom returned 0x0 boxes in the local probe. jsdom did resolve `display:none` from a stylesheet via `getComputedStyle` **[measured]**, so cascade-level visibility is partly checkable. Vitest warns that simulated DOMs "may result in some discrepancies" and "false positives or negatives" ([Vitest browser mode](https://vitest.dev/guide/browser/why)). Colour-contrast checks "do not work in JSDOM so are turned off in jest-axe" ([jest-axe](https://github.com/nickcolley/jest-axe)).
- **How big:** not quantified as a share of all bugs. Survey evidence: in the MDN Browser Compatibility Report 2020, layout and styling was the most-cited problem feature area (48% of respondents, ahead of JavaScript at 26%) ([MDN report 2020](https://mdn.dev/archives/insights/reports/mdn-browser-compatibility-report-2020.html)). A study of 35 real sites (Facebook, Dropbox, Moodle and others) catalogued more than 90 layout bugs ([Hallé et al., declarative layout constraints](https://www.sciencedirect.com/science/article/pii/S2352220816300293)), which shows they are common in production, without giving a rate.
- **Cheap mitigations (no browser):** stylelint for invalid properties and values; [doiuse](https://github.com/anandthakker/doiuse)-style CSS feature checks against the project's Browserslist; Tailwind/class-name linting so referenced classes exist; jsdom `getComputedStyle` assertions for visibility of key controls; axe-core in jsdom for the structural accessibility rules it can run (labels, roles, names). Everything else gets an explicit **"visual change not verified"** note in the verdict when the diff touches CSS, styled components, templates or layout code; the LLM can add a short risk judgement from the style diff alone. Pixel and screenshot comparison needs a real renderer and is out of scope by the owner's constraint.

### 8.2 Browser engines and browser-only APIs

- **What is missed:** behaviour that differs across engines, and APIs jsdom/happy-dom do not implement or only stub: navigation ("not implemented" jsdomError), canvas without the native canvas package ("`<canvas>` elements behave like `<div>`s"), and in practice IntersectionObserver, ResizeObserver, `matchMedia`, Web Workers, service workers, WebGL, clipboard and media, which tests usually stub (so the stub, not the code, is what passes).
- **How big:** in the study of 502 client-side JavaScript bug reports, 77% were not browser-specific, and the browser-specific share fell from 32% (2004) to 17% (2014) ([Ocariza et al., TSE 2016](https://people.ece.ubc.ca/amesbah/resources/papers/frolin-tse16.pdf)). The same study found 68% of client-side JS faults are DOM-related, which is mostly inside jsdom's reach. So for JavaScript logic, roughly one bug in five to six is engine-specific (IE-era data; likely lower now), and most DOM-interaction bugs are reachable in memory.
- **Cheap mitigations:** TypeScript `lib.dom` typing; [eslint-plugin-compat](https://github.com/amilajack/eslint-plugin-compat) against the project's Browserslist to flag APIs a target browser lacks; detect when a changed code path calls an API the DOM environment stubs, and flag it "verified against a stub only".

### 8.3 Framework features the in-memory runtime cannot host

- **What is missed:** Next.js async Server Components (officially not supported by Vitest; Next recommends E2E), Next.js middleware, caching and server actions (not emulated by next-test-api-route-handler), edge-runtime differences (tests "always run in Node.js"). Equivalent holes exist elsewhere for anything that is wired by the production server or bundler rather than by the app object.
- **Cheap mitigations:** `next build` (or the framework's own build/type-check) as a static gate catches a large class of server/client boundary and routing errors without running a browser; render server components to HTML with React's server renderer in Node where feasible (not officially supported for async components, treat as best effort); otherwise label as unverified.

### 8.4 Real database and service semantics

- **What is missed:** whatever the fake does not reproduce. Measured above: numeric precision, time zones, JSONB path functions (pg-mem); type affinity, foreign keys off, ASCII-only case folding (SQLite). Even PGlite cannot exercise concurrent connections, isolation levels, lock contention or deadlocks. moto does not enforce IAM unless configured. Google calls this the "unfaithful doubles" risk ([SWE book ch. 14](https://abseil.io/resources/swe-book/html/ch14.html)).
- **How big:** in 423 database-access bugs from seven large Java projects, SQL-query, schema and API bugs made up 84.2%, and SQL-query bugs alone were 54% of bugs in JDBC-based projects ([database-access bug study, arXiv 2405.15008](https://arxiv.org/abs/2405.15008)). SQL-query and schema bugs are exactly the class where a low-fidelity fake gives wrong answers, so fake choice matters a lot; with the real engine in-process (PGlite, SQLite-for-SQLite) most of that class comes back into reach. Configuration was a separate category in the same study and is not reachable in memory.
- **Cheap mitigations:** climb the fidelity ladder (PGlite over pg-mem over SQLite-for-Postgres); run the repo's migrations against PGlite to catch schema/migration errors; SQL linting such as [squawk](https://github.com/sbdchd/squawk) for dangerous Postgres migrations; enable moto IAM checks (`INITIAL_NO_AUTH_ACTION_COUNT`) when the change touches policies; follow Google's rule that "A fake must have its own tests to ensure that it conforms to the API of its corresponding real implementation" ([SWE book ch. 13](https://abseil.io/resources/swe-book/html/ch13.html)): the gate can trust well-maintained fakes but should flag hand-rolled fakes in the repo that have no contract test. Concurrency and isolation bugs are labelled unverified when the diff touches transactions or locking.

### 8.5 Network, deployment and runtime configuration

- **What is missed:** reverse proxy and path rewrites, TLS, DNS, real CORS enforcement by the browser, cookie `Secure`/`SameSite` behaviour across real origins, timeouts and retries against real latency, container images, environment variables present in production, feature flags, load.
- **How big:** "At Google, configuration changes are the number one reason for our major outages" ([SWE book ch. 14](https://abseil.io/resources/swe-book/html/ch14.html)). A study of 597 public cloud-service outages lists upgrade, network, bugs, configuration and load as leading known causes ([Gunawi et al., SoCC 2016](https://doi.org/10.1145/2987550.2987583); exact per-cause percentages not verified here). Note that E2E in a test environment does not reliably catch these either, because the test environment's configuration is not production's.
- **Cheap mitigations:** the app's own headers are in reach: assert CORS, CSP, cookie attributes and redirects through the in-process client (a browser enforces them, but the app emits them). Boot the app in-process with the production configuration file and fake secrets to catch missing or invalid settings at startup. Inject faults into fakes (timeouts, 5xx, connection reset via MSW or a raising fake) to exercise retry and error paths. This matters: 92% of catastrophic failures in the Yuan et al. study came from incorrect handling of non-fatal errors, and almost all failures needed 3 or fewer nodes ([Yuan et al., OSDI 2014](https://www.usenix.org/conference/osdi14/technical-sessions/presentation/yuan)). Lint infra files with their own static tools (Dockerfile, Kubernetes manifests, CI workflows) when the diff touches them.

### 8.6 Overall size of the gap

- **Backend and CLI:** small. 77% of production failures in Cassandra, HBase, HDFS, Hadoop MapReduce and Redis were reproducible by a unit test, 74% were deterministic, and 98% manifested on 3 or fewer nodes ([Yuan et al., OSDI 2014](https://files.usenix.org/s3fs-public/cdn/osdi14-paper-yuan.pdf)). An in-process app with fakes and fault injection is a superset of a unit test, so the reachable share should be at least that, minus whatever the fakes misrepresent.
- **Frontend logic:** mostly reachable (68% of client-side JS faults are DOM-related, which jsdom emulates; 77% are not browser-specific).
- **Visual/layout and deployment configuration:** not reachable, and these are the categories developers and Google rate as most painful or most outage-causing. Size unknown in bug-share terms.

## 9. Recommendation for `merge-gate`

1. **Run the repo's declared test command first**, with the network sealed at OS level. Unexpected outbound calls become findings.
2. **Then run behavioural probes for changed public surfaces** through the in-process seam (section 2), built on the repo's own fixtures when they exist (section 4, step 1), on the highest-fidelity fake available (section 3: PGlite for Postgres, real SQLite for SQLite, fakeredis, moto with IAM on when relevant). Add schema-driven fuzzing (Schemathesis for ASGI/WSGI; generated requests via `inject`/`app.request` for JS) for changed endpoints. Snapshot CLI output and help text.
3. **Static checks for what cannot run:** stylelint + Browserslist compat for CSS and DOM APIs, framework build/type-check (e.g. `next build`), migration replay on PGlite plus SQL lint, infra linters for touched infra files.
4. **Name the unverified surfaces in the verdict** (visual/layout, engine-specific APIs, async server components, concurrency/isolation, production config). A change that only touches verified surfaces can be _merge_; one touching unverified surfaces is at most _merge with notes_ unless the LLM judgement (bounded to the relevant hunks) finds a concrete problem.
5. **Benchmark caveat for the reliability proof:** bug benchmarks built from fix commits with a failing test (BugsJS, BugsInPy, Defects4J style) contain, by construction, bugs that a test can reproduce in-process. They will understate the E2E gap. Add a stratum of layout/CSS and configuration bugs mined from issue trackers, and score whether the gate _flags them as unverified_ rather than whether it catches them.

## 10. Main risk

**Fake fidelity, not speed.** The in-memory approach is fast enough by two orders of magnitude; the danger is a fake that disagrees with production and turns into confident wrong verdicts (pg-mem float money, SQLite accepting broken foreign keys, moto skipping IAM, a stubbed browser API). Mitigate by preferring real engines in-process, flagging hand-rolled fakes without contract tests, and stating every unverified surface in the verdict.

## 11. Open questions and unverified items

- Exact per-cause percentages in Gunawi et al. (SoCC 2016) were not read from the primary PDF (the mirror was blocked).
- Whether React server-rendering async Server Components in Node gives faithful results for Next.js App Router pages was not tested.
- No in-process fake for Kafka or RabbitMQ was found for Node or Python.
- The experiment used toy apps; boot times for real apps with ORMs and migrations need measuring on the benchmark repos.

## Sources

- supertest README: https://github.com/ladjs/supertest
- Fastify testing guide: https://fastify.dev/docs/latest/Guides/Testing/
- Hono testing: https://hono.dev/docs/guides/testing
- next-test-api-route-handler: https://github.com/Xunnamius/next-test-api-route-handler
- Next.js Vitest guide (async Server Components): https://nextjs.org/docs/app/guides/testing/vitest
- FastAPI testing: https://fastapi.tiangolo.com/tutorial/testing/
- Flask testing: https://flask.palletsprojects.com/en/stable/testing/
- Django testing tools: https://docs.djangoproject.com/en/5.2/topics/testing/tools/
- Go httptest: https://pkg.go.dev/net/http/httptest
- axum testing example: https://github.com/tokio-rs/axum/blob/main/examples/testing/src/main.rs
- Spring MockMvc: https://docs.spring.io/spring-framework/reference/testing/mockmvc.html
- ASP.NET Core integration tests: https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests
- Rails testing guide: https://guides.rubyonrails.org/testing.html
- Schemathesis Python API: https://schemathesis.readthedocs.io/en/stable/reference/python/
- Testing Library guiding principles: https://testing-library.com/docs/guiding-principles
- jsdom README: https://github.com/jsdom/jsdom
- happy-dom: https://github.com/capricorn86/happy-dom
- Vitest browser mode rationale: https://vitest.dev/guide/browser/why
- jest-axe: https://github.com/nickcolley/jest-axe
- pg-mem README: https://github.com/oguimbal/pg-mem
- PGlite repo and docs: https://github.com/electric-sql/pglite, https://pglite.dev/docs/
- SQLite quirks: https://www.sqlite.org/quirks.html
- fakeredis-py: https://github.com/cunla/fakeredis-py
- moto getting started and IAM: https://docs.getmoto.org/en/latest/docs/getting_started.html, https://docs.getmoto.org/en/latest/docs/iam.html
- MSW `listen`: https://mswjs.io/docs/api/setup-server/listen
- pytest-socket: https://github.com/miketheman/pytest-socket
- Node.js permission model: https://nodejs.org/api/permissions.html
- Google Testing Blog, "Where do our flaky tests come from?" (2017): https://testing.googleblog.com/2017/04/where-do-our-flaky-tests-come-from.html
- Software Engineering at Google, ch. 13 (Test Doubles) and ch. 14 (Larger Testing): https://abseil.io/resources/swe-book/html/ch13.html, https://abseil.io/resources/swe-book/html/ch14.html
- Yuan et al., "Simple Testing Can Prevent Most Critical Failures", OSDI 2014: https://www.usenix.org/conference/osdi14/technical-sessions/presentation/yuan
- Ocariza et al., "A Study of Causes and Consequences of Client-Side JavaScript Bugs", IEEE TSE 2016: https://people.ece.ubc.ca/amesbah/resources/papers/frolin-tse16.pdf
- MDN Browser Compatibility Report 2020: https://mdn.dev/archives/insights/reports/mdn-browser-compatibility-report-2020.html
- Hallé et al., declarative layout constraints for testing web applications: https://www.sciencedirect.com/science/article/pii/S2352220816300293
- An Empirical Study on the Characteristics of Database Access Bugs in Java Applications in Java applications: https://arxiv.org/abs/2405.15008
- Gunawi et al., "Why Does the Cloud Stop Computing?", SoCC 2016: https://doi.org/10.1145/2987550.2987583
- eslint-plugin-compat: https://github.com/amilajack/eslint-plugin-compat
- doiuse: https://github.com/anandthakker/doiuse
- squawk: https://github.com/sbdchd/squawk

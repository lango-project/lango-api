# @lango-project/lango-api

An HTTP API host written in Lango. Every request executes synchronously in a
bounded Piscina worker pool; HTTP transport remains on the main thread. Routes,
authorization, lifecycle callbacks, token operations, and UI handlers use ordinary
return values. Promise/thenable results from application callbacks are rejected.

Source, tests, and examples are Lango. Generated JavaScript lives in `dist/`.
Node.js 24 or newer is required. Build with the current Lango compiler:

```sh
npm install
npm test
npm run build
```

Builds clean `dist/` first so removed implementations cannot remain in the package.

## CLI and application sources

```sh
lango-api <app> [development|server] \
  [--app-root <path>] [--data-dir <path>] [--state-dir <path>] \
  [--log-dir <path>] [--config <file>] [--api <file>] \
  [--host <host>] [--port <port>]
```

Development is the default mode. Application name and mode determine program,
configuration, data, state, and log directories; individual paths may be overridden.
Machine configuration and mutable state belong outside the source worktree.
Configuration precedence is CLI > environment > XML > defaults. The base
configuration is optional; an explicitly selected missing file is an error.

```xml
<conf>
    <http host="127.0.0.1" port="3000" />
    <paths appRoot="/srv/shop" stateDir="/var/lib/shop/state"
           dataDir="/var/lib/shop" logDir="/var/log/shop"
           apiFile="/etc/shop/api.xml" />
    <workers threads="2" maxQueue="64" bodyLimit="1048576" />
    <jwt issuer="shop" audience="shop-browser" keysFile="keys.json"
         activeKeyId="current" ttlSeconds="900" sessionTtlSeconds="28800" />
    <state file="runtime.sqlite" namespace="shop" maxForms="10000" />
</conf>
```

Path/HTTP environment names are `LANGO_APP_ROOT`, `LANGO_DATA_DIR`,
`LANGO_STATE_DIR`, `LANGO_LOG_DIR`, `LANGO_CONFIG`, `LANGO_API`, `LANGO_HOST`,
and `LANGO_PORT`.

An explicit `--api`, `LANGO_API`, configuration `apiFile`, or
`Host.routesFromXml()` selects XML routes. If no factory or parent-side routes
were selected, existing `<appRoot>/deploy/api.xml` is used automatically.
Otherwise the default factory is `<appRoot>/dist/worker.mjs`. An explicit factory
and an explicit XML source can be combined. XML is snapshotted once per startup
so all workers use the same API definitions.

```xml
<api>
    <auth name="allowed" mod="@shop/auth" func="authorize" />
    <event name="register" mod="@shop/session" func="registered" />
    <event name="start" mod="@shop/session" func="started" />
    <event name="ping" mod="@shop/session" func="pinged" />
    <event name="request" mod="@shop/audit" func="record" />
    <group path="/shop" mod="@shop/orders" anonymous="true" auth="allowed">
        <endpoint path="/save" func="save" />
        <endpoint path="/items/*" func="item" />
    </group>
</api>
```

Exact routes win over wildcards; otherwise the longest matching prefix wins.
`/app/*` matches `/app/` and descendants, but not `/app` or `/application`.
Groups inherit module, anonymous, and auth settings. Duplicate routes, auth
names, and lifecycle registrations fail startup. Lifecycle events belong directly
under `<api>`. Unknown XML attributes/tags and malformed definitions are errors.

In development, installed linked Lango packages under `appRoot/node_modules` are
built once in the parent before workers start. Ordinary packages are not built.
Server mode never runs package builds. Library callers should set `mode: "server"`
for production. There is no hot reload: source, XML, and key changes need a restart.

## Library and worker factory

```lango
import js "@lango-project/lango-api" as api

host = new api/Host({
    applicationName: "shop"
    appRoot: appRoot
    stateDir: stateDir
    mode: "server"
    host: "127.0.0.1"
    port: 3000
})
host.routesFromXml(apiFile)
return host.start()
```

`Host.start()` returns a Promise for the listening Node HTTP server.
`Host.close()` returns a Promise, drains HTTP requests and accepted audit tasks,
and closes the pool. Repeated close is safe. Closing the returned HTTP server
also closes its pool. The CLI handles SIGINT/SIGTERM.

`Routes` provides `endpoint`, `group`, `auth`, and `values`. Parent-side builders
use transferable module/function names:

```lango
routes = new api/Routes()
routes.endpoint({
    path: "/save"
    module: "@shop/orders"
    function: "save"
})
host.routes(routes)
host.on({
    event: "start"
    module: "@shop/session"
    function: "started"
})
```

Create direct handlers, module objects, closures, and owned application resources
inside a worker factory. Parent-process functions and instances cannot cross
thread boundaries. The factory receives `applicationName`, `appRoot`, `dataDir`,
`stateDir`, `logDir`, and extra `workers.options`/`optionsFile` values.
`workers.application` accepts an application-relative filename or an npm package
export; the module must export `createWorker(options)`.

```lango
export createWorker(options)

createWorker(options)
    routes = new api/Routes()
    routes.endpoint({
        path: "/save"
        function: save
        anonymous: true
    })
    return {
        routes: routes.values()
        register: registered
        start: started
        ping: pinged
        request: audited
        close: cleanup
    }
```

A factory returns a map containing a routes array. Route functions and module
objects remain worker-local. Named route/auth/lifecycle modules are imported and
exports are checked before serving. Lifecycle registrations may also use a
`lifecycle` map. Factory, parent-builder, and XML definitions merge; duplicate
routes or events are errors. `close` runs during worker-exit cleanup, with
`context state` available; forced termination cannot guarantee cleanup.

Optional `preload: [resource]` lists worker-local objects exposing `preload()`.
The host prepares them in order after module loading and waits for any returned
Promises. No worker accepts a request until every worker has initialized.
Preload errors reject startup and close the runtime. Promises are supported at
this startup boundary; the factory and request callbacks remain synchronous.

### UI integration

`lango-api` has no dependency on `lango-ui`. Create its dispatcher in the factory:

```lango
dispatcher = new ui/Dispatcher(options.appRoot)
dispatcher.routesFromXml(uiFile)
routes.endpoint({
    path: "/app/*"
    function: handleEvent
    anonymous: true
})
return {
    routes: routes.values()
    preload: [dispatcher]
}
```

Forward the wildcard route to `dispatcher.handle(input)`. UI preload resolves
`ui.xml` npm exports and stores callable handlers. Dispatch remains synchronous
after XML/package files disappear. UI replay uses the worker's shared state
unless a custom synchronous replay adapter was explicitly supplied. Handlers
use `context form`, including `form.saveButton.enabled = false`.

## Request and response

Application endpoints accept POST with a JSON body. Query parameters are rejected.
Main-thread GET `/health` reports liveness without waiting for a blocked worker.

Handlers receive the parsed input as their only explicit argument. Declare only
the contexts they need:

```lango
save(input)
    context request
    context response
    context token
    context state
    context log
    log.info("Saving ${request.path}")
    return {
        saved: true
    }
```

`request` contains `id`, `method`, `path`, `headers`, and `remoteAddress`.
The address comes from the socket; forwarded headers are not trusted implicitly.
`token` contains verified claims, or `{ auth: false }` without JWT.
`state` is the shared SQLite adapter. Context changes remain scoped to the call;
pure helpers need no HTTP parameters.

An ordinary return value is serialized as JSON. Alternatively use
`response.status(code)`, `header(name, value)`, and `send(value)`. Status must be
an integer from 100 to 599. A response can be sent once. `X-Lango-Token` is reserved.
`response.completed()` reports whether a response has been sent. These methods
return synchronous values; `response.logout()` synchronously revokes the current
session and prevents renewal. It is idempotent before the response is sent.

Authentication is required unless `anonymous: true`. An optional authorization
callback receives the input map, has the same contexts, and must return Boolean:
false returns 403, and exceptions, non-Booleans or Promise results return 500.
Denied/failed authorization never runs the endpoint and discards token changes.
Auth accepts a direct function, a module/function descriptor, or a registered
named definition inherited from a group.

Handler exceptions and Promise results return a generic 500 without exposing
error details. Failed handlers discard unsent token changes. Application side
effects are not rolled back by an HTTP error or worker termination. Changes made
after `send()` cannot change the sent body or token snapshot.

## JWT and sessions

JWT is optional. Configure it using the XML section above or `configuration.jwt`.
An absent/disabled JWT configuration runs anonymously without reading a key file.
Keys are a JSON map of ids to canonical base64 secrets of at least 32 bytes:

```json
{
    "current": "<base64 random secret>",
    "previous": "<previous base64 random secret>"
}
```

The active key signs; retained keys verify. Keep secrets outside the Git worktree.
XML-relative key filenames resolve from `conf.xml`; CLI/environment-relative names
resolve from the working directory. Secrets are not passed on the command line.

| CLI | Environment | JWT attribute |
| --- | --- | --- |
| `--jwt-enabled true\|false` | `LANGO_JWT_ENABLED` | `enabled` |
| `--jwt-issuer` | `LANGO_JWT_ISSUER` | `issuer` |
| `--jwt-audience` | `LANGO_JWT_AUDIENCE` | `audience` |
| `--jwt-keys` | `LANGO_JWT_KEYS` | `keysFile` |
| `--jwt-key-id` | `LANGO_JWT_KEY_ID` | `activeKeyId` |
| `--jwt-ttl` | `LANGO_JWT_TTL` | `ttlSeconds` |
| `--jwt-session-ttl` | `LANGO_JWT_SESSION_TTL` | `sessionTtlSeconds` |

Issuer, audience, active key id, and key file are required when enabled. JWT TTL
is 1–3600 seconds, default 900. Absolute session lifetime is 1–2592000 seconds,
default 28800. Renewal never extends the absolute deadline. A shorter configured
policy applies immediately; a longer one cannot extend an existing session.

Validation pins HS256, `lango+jwt` type, key, issuer/audience, required issuance,
expiry and id claims, session identity, absolute deadline, and revocation.
Cryptographic clock tolerance does not extend the absolute session deadline.
Tokens carry independent form/application claims; protected identity fields are
restored after callbacks. Signing rechecks expiry and revocation, so a logout in
another worker prevents late renewal by a running handler.

`POST /_lango/start` creates an anonymous session. A verified old signature can
recover only `clientId`, including from an expired or revoked token; auth, session
identity and form claims start fresh. Invalid signatures create a new identity.
Anonymous endpoints without bearer tokens also start a session. Ordinary endpoints
reject supplied invalid/expired tokens with 401 instead of silently bootstrapping.
`POST /_lango/logout` revokes the bearer session and returns `{ loggedOut: true }`
without a replacement token. Token-bearing responses use `Cache-Control: no-store`.

`TokenService` from `@lango-project/lango-api/tokens` exposes synchronous `sign`,
`verify`, `verifySigned`, `recoverClient`, `startSession`, `restart`, `revoke`, and
`checkSession` operations. `SyncTokenService` is an inherited compatibility name
for this same implementation. Token options accept byte-array keys and a
synchronous revocation adapter. Async adapters are rejected. `MemorySessionRevocations`
is available for standalone use; production workers use shared SQLite. Signatures
interoperate with jose; jose is needed only for interoperability tests.

## Lifecycle and audit

Supported lifecycle events are `register`, `start`, `ping`, and `request`.
Each receives one metadata map and returns synchronously; return values are ignored.

`register` runs for new client identities; `start` runs for each new session.
Metadata includes ISO UTC `timestamp`, `ip`, `userAgent`, `url`, `queryString`,
`clientId`, `sessionId`, `sessionStartedAt`, and `sessionExpiresAt`. Each hook gets
its own snapshot; reserved identity is restored between hooks. Failures return
500 without a JWT and skip the remaining initialization/handler.

`POST /_lango/ping` takes `{ forms: [{ formId, token }] }` with 1–256 unique,
nonempty form ids and token strings at most 16384 characters long. The entire
batch is verified before callbacks run. Renewal runs sequentially with each
form's own token context. Existing signed form id/sequence and session identity
are preserved; form ids must match. Ping metadata includes timestamp, IP,
client/session/form ids, URL and query string. Its managed response cannot send,
change status/headers or log out. Success returns renewed tokens per form with
no single `X-Lango-Token` header. Malformed input returns 400; invalid, expired or
revoked tokens return 401. Hook failures/Promises return 500 without partial
renewed tokens. Earlier application side effects are not rolled back.

A `request` audit task runs after HTTP `finish`. It may run on another worker,
so callbacks must use shared stores rather than the handler's module memory.
Completed errors and bootstrap are included; ping, GET `/health`, and responses
aborted before finish are excluded. Aborts are logged. Finish confirms delivery
to Node's transport, not receipt/application by the browser.

Audit metadata contains `requestId`, `method`, `ip`, `timestamp`, `durationMs`,
`url`, `queryString`, `status`, `clientId`, `sessionId`, `transactionId`, `formId`,
`seqno`, `input`, and `output`. Timing refers to HTTP completion, not audit queue
execution. Input is copied before callbacks; errors before parsing have null input.
Client-supplied correlation fields are not trusted identity or deduplication keys.
Contexts include `state`, `log`, a copied `token`, sanitized `request` headers
(content type/user agent only), and a completed response exposing `statusCode`,
`body` and `completed()`. Response mutations throw; snapshot mutations cannot
change the sent body/JWT. Callback exceptions/Promises are logged and isolated.
Audit also works without JWT.

Audit shares the bounded request queue. Queue rejection/worker failure is logged;
there is no retry or durable audit queue. Graceful shutdown drains accepted tasks.
A successful HTTP response does not guarantee audit persistence. Audit tasks
also have the configured task deadline; a hung audit can trigger worker replacement
without changing the HTTP response that has already finished.

## Worker and state configuration

Workers are always enabled. `--workers true`/`enabled="true"` remain accepted;
disabling workers fails with `ApiWorkersRequired`. No async HTTP fallback remains.

| CLI | Environment | Workers attribute | Default |
| --- | --- | --- | --- |
| `--worker-application` | `LANGO_WORKER_APPLICATION` | `application` | XML or `dist/worker.mjs` |
| `--worker-threads` | `LANGO_WORKER_THREADS` | `threads` | 2, maximum 128 |
| `--worker-max-queue` | `LANGO_WORKER_MAX_QUEUE` | `maxQueue` | 64 |
| `--worker-body-limit` | `LANGO_WORKER_BODY_LIMIT` | `bodyLimit` | 1048576 bytes |
| `--worker-entry` | `LANGO_WORKER_ENTRY` | `workerFile` | Package worker-entry export |
| `--worker-options` | `LANGO_WORKER_OPTIONS` | `optionsFile` | None |

`workers.options` supplies extra library factory settings; it conflicts with
`optionsFile`. File/options values must be structured-cloneable maps. Filenames
resolve against the final appRoot. Parent token/state instances cannot be sent
to workers; configure options/files or construct resources inside the factory.
Queue overload returns 503 and excess request bodies return 413. GET `/health`
remains responsive while workers are busy and reports `queueSize` and `threads`.
There is one in-flight task slot per worker. Host admission is FIFO, with a shared
bounded queue for requests and audits; Piscina does not choose their queue order.
Queueing counts toward the task deadline.

| CLI | Environment | Workers attribute | Default |
| --- | --- | --- | --- |
| `--worker-timeout` | `LANGO_WORKER_TIMEOUT` | `requestTimeoutMs` | 30000 ms |
| `--worker-startup-timeout` | `LANGO_WORKER_STARTUP_TIMEOUT` | `startupTimeoutMs` | 30000 ms |
| `--worker-shutdown-timeout` | `LANGO_WORKER_SHUTDOWN_TIMEOUT` | `shutdownTimeoutMs` | 30000 ms |

Deadline values must be integers from 1 to 2147483647 milliseconds. Configuration
uses the same XML → environment → CLI precedence as other worker options.

A request/audit deadline covers admission through completion of the pool task,
including queue time and replacement-worker initialization. It begins after the
HTTP body is received; Node's HTTP upload/header limits are separate. A queued
expired task is removed without execution. A running expired task triggers worker
termination/replacement. The HTTP result is 504 `GatewayTimeout`, with no JWT.
A worker crash produces a generic 500 without JWT. Neither case retries the task.
Unstarted queued tasks remain eligible on replacement workers, subject to their
own deadlines. Replacement workers run the same initialization/preload path.

The startup deadline bounds each worker's factory/module/resource initialization;
a failure prevents listening and destroys the pool. Development package-build
scripts run before pool creation and are outside that deadline.

`Host.close()` bounds HTTP and queued/active task draining with the shutdown
deadline. On expiry, it closes remaining connections, rejects queued tasks and
terminates the pool, then rejects with `ApiWorkerShutdownTimeout`. Clients may see
a disconnected response. Normal closure drains accepted audit tasks; forced
closure cannot guarantee their completion. A direct Node `server.close()` uses
Node's HTTP drain semantics; use `Host.close()` for the total shutdown deadline.
Worker termination cannot undo earlier side effects. Native libraries must also
bound their own blocking operations; external native calls may delay termination.
A process supervisor remains responsible for an OS-level hard shutdown limit.

State defaults to `stateDir/runtime.sqlite`, namespace `applicationName`, and
10000 replay streams per namespace. The state directory is created as needed.
SQLite schema/WAL setup occurs before parallel worker initialization. Connections
share revocations and UI replay; namespaces isolate applications. Completed
sequences and interrupted reservations survive restart. Pending reservations
remain blocked until their session deadline; live entries are not evicted to
make capacity. Memory state is not supported for a worker host.

| CLI | Environment | State attribute |
| --- | --- | --- |
| `--state-file` | `LANGO_STATE_FILE` | `file` |
| `--state-namespace` | `LANGO_STATE_NAMESPACE` | `namespace` |
| `--state-max-forms` | `LANGO_STATE_MAX_FORMS` | `maxForms` |

`--state-backend sqlite`, `LANGO_STATE_BACKEND=sqlite`, and `backend="sqlite"` select
the supported backend. `SqliteStateStore` from `./state` exposes synchronous
`claim`, `complete`, `revoke`, `isRevoked`, `close`, and `isClosed` operations.
SQLite uses local durable storage; a shared network filesystem is not a
multi-machine state service. Business idempotency remains an application concern.

The lower-level `./workers` export provides `WorkerHost` and `startWorkers`.
It accepts absolute `application`/`workerFile`/`stateFile` paths, `appRoot`, optional
`apiXml`/`routeDefinitions`/`lifecycle` descriptors, byte-array JWT `tokens`, factory
`options`, namespace/capacity and transport/pool settings. It does not prepare
linked packages or read key/configuration files. `start()` resolves to the host,
whose `server` and `pool` expose Node HTTP and Piscina. Prefer standard `Host`
for configuration and CLI parity.

## Migration and validation

The former async HTTP server and duplicate dispatch implementations were removed.
Create direct functions and resources inside `createWorker`; parent builders keep
module/function descriptors. Application code must return plain values. UI
Dispatcher is synchronous by default and requires startup preload for module names.
Token operations and response/logout methods return values instead of Promises.
The browser client retains its asynchronous networking contract.

Run `npm test` for configuration, synchronous token/security/deadline/runtime
checks, SQLite persistence/process races, and real worker Host/CLI/XML/ping/audit
HTTP tests. Pure runtime behavior is tested synchronously instead of duplicating
an async server harness. `npm run test:workers` also builds sibling `lango-ui` and
checks real worker/UI/SQLite replay, simultaneous requests, queue overload,
main-thread responsiveness, cross-worker logout, UI preload and missing exports.
It requires the multi-repository workspace.

The HTTP/UI and XML/package test scenarios use a separate server process and
small HTTP client processes. Their test drivers execute ordinary synchronous
steps while the server and concurrent requests keep independent event loops.
Readiness/results are published atomically through temporary files. Parallel
worker tests use explicit release signals instead of relying on fixed delays.
The support code keeps startup/network Promise handling at those process edges.

The demo commands are `npm run build:worker-demo` and `npm run demo:workers`.
From another terminal:

```sh
curl http://127.0.0.1:8081/health
curl -X POST http://127.0.0.1:8081/hello \
  -H 'Content-Type: application/json' -d '{"name":"Lango"}'
```

The demo retains keys/state under ignored `data/worker-demo`. After first launch,
its XML-only equivalent is `node dist/cli.mjs worker-demo --app-root . --config
examples/worker-conf.xml` after building the demo artifacts.

The Piscina migration is complete for the API/UI request contract. Additional
I/O backends, workload-specific tuning and application idempotency are separate
features. Streaming, sockets, background asynchronous jobs and automatic retry
are outside this contract. Neither worker termination nor HTTP failure rolls
back committed application side effects.

## I/O contract and load baseline

The existing framework I/O paths were audited for this migration:

| Path | Request-time contract |
| --- | --- |
| JWT and revocation | Synchronous jsonwebtoken and synchronous store calls; Promise adapters are rejected. |
| Shared SQLite state | DatabaseSync, bounded busy wait, atomic claim/complete and durable interrupted reservations. |
| Native Lango SQLite adapter | Synchronous operations; application database configuration remains explicit and separate from API state storage. |
| UI replay adapters | Synchronous claim/complete; invalid/Promise results fail without running the handler. |
| Files and XML | Synchronous Node file access and XML parsing; route/module preparation happens at startup. |
| Log | Synchronous application method calls to the standard output/error sinks. |
| Startup/imports and HTTP transport | Promise handling at the host boundary; complete preload before serving. |

Worker threads do not convert arbitrary asynchronous JavaScript APIs into blocking
calls. Do not start background Promise, timer, socket or callback jobs from request
handlers. Outbound `fetch`, asynchronous database drivers and remote state backends
need an explicit supported adapter/architecture; adding these backends is outside
this migration. Application-owned connections belong in the worker factory and
its cleanup hook. Application transactions and business idempotency remain explicit.

Run `npm run bench:workers` for the reproducible local JWT + SQLite baseline:
1/2/4 workers, eight persistent load clients, a simultaneous health observer, and
0/5 ms CPU work per request. Each client warms up before a common start barrier.
The report includes request/health latency, throughput, errors, committed rows
and process RSS. See [the measured baseline](benchmarks/worker-baseline.md) and
[its JSON data](benchmarks/worker-baseline.json). It is a short local baseline,
not a production capacity or long-running memory-leak guarantee.

## API/UI starter release

The API 0.2.0 release is paired with UI/client 0.2.0 and compiler 0.1.9 in the
application generator provided by `lango-client`. Its Node-only BrowserHost starts
this worker Host, serves registered UI page shells/assets and proxies same-origin
JSON requests. The generated application's business handlers remain synchronous
Lango. UI event envelopes use protocol 1; ordinary API/bootstrap/ping/logout
contracts are unchanged. See the client package's application-generator guide.

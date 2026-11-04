# @lango-project/lango-api

`lango-api` is an API host, written in Lango, that exposes Lango applications over HTTP.

Application sources, machine-specific configuration, persistent data, and logs are kept separate. The evolving Lango architecture documentation defines the directory contract and operating modes.

## Initial goal

The first milestone provides the following flow:

1. Resolve development or server directories from an application name.
2. Load and validate `conf.xml` and `api.xml`.
3. Route an endpoint to an exported function from an npm package.
4. Run as an independent HTTP server.

Reverse-proxy support, service definitions, authentication, and the advanced HTTP contract follow in later milestones.

Source code and tests are `.lango` files. JavaScript generated from them is published under `dist/` so the npm package can run; `dist/` is never edited manually.

The package is built as an executable Lango artifact: its generated entry invokes the exported `main(args)` function without a handwritten JavaScript bootstrap.

## Library API

The package can also be used as a library. `Host` owns the API lifecycle and
`Routes` builds routes without XML:

```lango
import js "@lango-project/lango-api" as api

host = new api/Host({
    appRoot: appRoot
    host: "127.0.0.1"
    port: 3000
})

routes = new api/Routes()
routes.endpoint({
    path: "/health"
    module: "@shop/health"
    function: "check"
})

math = routes.group({
    path: "/math"
    module: "@shop/math"
})
math.endpoint({
    path: "/add"
    function: "add"
})

host.routes(routes)
host.start()
```

`host.routesFromXml(apiFile)` is the equivalent XML-based route configuration.
The CLI is a thin bootstrap over this same API.

Endpoint paths also accept a trailing `/*`, for example `/app/*`. Exact paths
take precedence; otherwise the longest matching prefix wins. `/app/*` matches
`/app/` and its descendants, but not `/application` or `/app`. The same syntax
works in XML and route maps. Duplicate detection compares registered patterns.

Route maps can also hold a module object or a handler function directly. These
forms bypass npm resolution and are intended for modules that are already
available to the application:

```lango
routes.endpoint({
    path: "/local"
    function: localHandler
})
```

Linked-package rebuilding requires a string package specifier, so the string
form remains the standard development route form.

### Client and session lifecycle

JWT-enabled hosts can initialize application state before issuing a token:

```lango
host.on({
    event: "register"
    function: registerClient
})
host.on({
    event: "start"
    module: "@shop/session"
    function: "startSession"
})

registerClient(input)
    context token
    context log
    token.theme = "dark"
    log.info("Client registered: ${input.clientId}")
```

Each callback receives one metadata map containing `ip`, an ISO UTC
`timestamp`, `userAgent`, `url`, `queryString`, `clientId`, and `sessionId`.
The API currently rejects query parameters, so `queryString` is empty. The IP
comes from the socket; forwarded headers do not change it. `request`, `response`,
`token`, and `log` are available through context. Promise callbacks retain context
through an explicit `capture` as in ordinary handlers.

`register` runs when the server creates a new client identity. `start` runs
after it for every newly created session, including anonymous requests without
a bearer token. `POST /_lango/start` with a recoverable signed identity runs
only `start`, even if the old token has expired. Ordinary requests with a valid
token run neither callback. Invalid tokens at normal endpoints still produce 401.

Callbacks run sequentially and may return Promises. Their return values are
ignored; token writes become initial claims. Client and session identities remain
owned by the service. An exception or rejected Promise returns 500 without a new
token and skips remaining callbacks and the endpoint handler. External callback
side effects are not rolled back.

`on` accepts the same function, module object, or npm module reference forms as
route handlers, using an `event` instead of a `path`. A callback module's exports
are checked before listening, and linked npm packages are built on startup.
There is one callback per event; duplicates and unknown events are rejected.
The equivalent configuration map is `lifecycle: { register: registerClient,
start: { module: "@shop/session", function: "startSession" } }`.

XML applications declare these callbacks directly under `<api>`:

```xml
<api>
  <event name="register" mod="@shop/session" func="registerClient" />
  <event name="start" mod="@shop/session" func="startSession" />
  <endpoint path="/health" mod="@shop/health" func="check" anonymous="true" />
</api>
```

`host.routesFromXml(apiFile)` loads routes and lifecycle events together. The
CLI loads the same definitions from its selected `api.xml`, including `--api`
and `LANGO_API` overrides. Events belong at the root and require `name`, `mod`,
and `func`; unknown attributes, nested content, unknown names, and duplicate
events are rejected. XML and code registrations may define different events;
defining the same event in both is an error. Replacing the route source also
replaces its XML events, while code registrations remain.

### Batched token ping

With a TokenService, `POST /_lango/ping` is reserved for renewing form tokens.
It accepts a JSON map with a `forms` array of `{ formId, token }` entries and
returns the same shape containing renewed tokens. There is no single
`X-Lango-Token` header for a batch; responses use `Cache-Control: no-store`.
Authentication is performed per entry rather than through one bearer header.
Form IDs must be unique nonempty strings; batches contain 1–256 forms and
individual token strings are limited to 16,384 characters.

Every token is verified with the normal signature, issuer, audience, type,
key and expiry checks before any callback runs. An expired or invalid token
rejects the entire batch with 401; malformed batches return 400. Ping does not
bootstrap an expired session, merge identities, advance a sequence number,
or require `auth: true`. Each form keeps its own claims and auth state.
An existing signed `formId` must match the submitted form ID.

Register an optional per-form callback through the library or API XML:

```lango
host.on({
    event: "ping"
    module: "@shop/session"
    function: "pingSession"
})
```

The XML form is `<event name="ping" mod="@shop/session" func="pingSession" />`.
One callback runs per form, sequentially, within the single HTTP request. It
receives `ip`, an ISO UTC `timestamp`, `clientId`, `sessionId`, `formId`, `url`,
and `queryString`. `request`, `token`, and `log` contexts are available. The
callback may update application claims and return a Promise; return values are
ignored. Existing client/session identities and signed form/sequence fields
are preserved. Response methods are managed by the batch handler and cannot
be used to send or customize an individual form response.

All tokens are signed before the response is sent. Callback/signing failure
returns 500 without a partial token batch; external callback side effects are
not rolled back. Ping requests, including their errors, do not produce a
`request` audit event. Register/start callbacks are not called either.

### Completed request audit

Register a `request` event to observe a finished HTTP response:

```lango
host.on({
    event: "request"
    module: "@shop/audit"
    function: "recordRequest"
})
```

The XML equivalent is `<event name="request" mod="@shop/audit"
func="recordRequest" />`. Module exports are checked before listening, as for
the other lifecycle events. Auditing also works without JWT configuration.

The event fires once after Node's response `finish` event. Completed 4xx/5xx
responses and `/_lango/start` are included; `/_lango/ping` is excluded. An aborted response that never
finishes is logged and produces no audit event. `finish` means the server handed the response
to its transport; it does not confirm that the client received or applied it.

The callback receives `requestId`, `method`, `ip`, an ISO UTC `timestamp`,
`durationMs`, `url`, `queryString`, `status`, `clientId`, `sessionId`,
`transactionId`, `formId`, `seqno`, `input`, and `output`. Unavailable identities
or client fields are null. Input is copied before handlers run; it is null if
the request was rejected before parsing or contained invalid JSON. Output is
the JSON response value supplied to the HTTP writer. `transactionId`, `formId`,
and `seqno` are client-supplied correlation data, not trusted identity or proof
that an operation is unique.

The `request` and `token` contexts are detached snapshots. Audit request headers
contain only content type and user agent; Authorization and Cookie are omitted.
The `response` context exposes `statusCode`, `body`, and `completed()`; attempts
to use `status`, `header`, or `send` throw `ApiResponseAlreadySent`. The request's
`log` context remains available. Changing an audit snapshot cannot alter the
response or the issued token.

Return values are ignored. Promise callbacks do not delay the response, and
exceptions or rejections are logged without changing its result. Auditing is
optional and runs in process; it does not provide a durable delivery queue or
an automatic shutdown drain. Input/output are delivered to the configured
callback without being written to logs automatically; the application chooses
which fields to retain or redact.

## Configuration

Configuration is optional. Without a configuration file, the host listens on `127.0.0.1:3000` and uses the documented development or server directory defaults.

The initial `conf.xml` format is:

```xml
<conf>
  <http host="127.0.0.1" port="3000" />
  <paths
    appRoot="/optional/application/root"
    dataDir="/optional/data"
    stateDir="/optional/state"
    logDir="/optional/logs"
    apiFile="/optional/api.xml" />
</conf>
```

All attributes are optional. The CLI defaults to development mode:

```sh
lango-api <app> [development|server] \
  [--app-root <path>] [--data-dir <path>] [--state-dir <path>] \
  [--log-dir <path>] [--config <file>] [--api <file>] \
  [--host <host>] [--port <port>]
```

The effective configuration order is command-line options, environment,
`conf.xml`, then mode-specific defaults. The environment names are
`LANGO_APP_ROOT`, `LANGO_DATA_DIR`, `LANGO_STATE_DIR`, `LANGO_LOG_DIR`,
`LANGO_CONFIG`, `LANGO_API`, `LANGO_HOST`, and `LANGO_PORT`. Without an
explicit API file, the host loads `<appRoot>/deploy/api.xml`.

### JWT configuration

JWT can be enabled from `conf.xml` without constructing a TokenService in code:

```xml
<conf>
  <jwt issuer="shop" audience="shop-browser"
       keysFile="jwt.keys.json" activeKeyId="current"
       ttlSeconds="900" sessionTtlSeconds="28800" />
</conf>
```

The separate key file is a JSON map of key ids to canonical base64-encoded
random secrets of at least 32 bytes. For example, replace these placeholders
with real keys:

```json
{
  "current": "<base64-encoded random key>",
  "previous": "<previous base64-encoded random key>"
}
```

The active id selects the signing key; all listed keys may verify tokens.
Restart after changing the key file or active id. Keys are not generated
automatically and are never passed as command-line values. Keep the key file
with machine-specific secrets outside the source worktree.

| CLI option | Environment variable | XML attribute |
| --- | --- | --- |
| `--jwt-enabled true|false` | `LANGO_JWT_ENABLED` | `enabled` |
| `--jwt-issuer` | `LANGO_JWT_ISSUER` | `issuer` |
| `--jwt-audience` | `LANGO_JWT_AUDIENCE` | `audience` |
| `--jwt-keys` | `LANGO_JWT_KEYS` | `keysFile` |
| `--jwt-key-id` | `LANGO_JWT_KEY_ID` | `activeKeyId` |
| `--jwt-ttl` | `LANGO_JWT_TTL` | `ttlSeconds` |
| `--jwt-session-ttl` | `LANGO_JWT_SESSION_TTL` | `sessionTtlSeconds` |

The precedence remains CLI > environment > XML > defaults. Any JWT section or
JWT option enables JWT unless `enabled` is explicitly false. Lifetime defaults
to 900 seconds; absolute session lifetime defaults to 28800 seconds (eight
hours). Session lifetime must be an integer from 1 to 2592000 seconds (30 days).
Issuer, audience, key file and active id are required when
enabled. An absent JWT configuration preserves legacy mode.

XML-relative key files resolve from the `conf.xml` directory; CLI/environment
relative paths resolve from the process working directory. An explicitly
selected missing configuration file is an error. Unknown JWT attributes,
duplicate JWT sections, unreadable/invalid key files, weak key lengths, and
missing active ids prevent the host from listening. Key contents are excluded
from error messages. Examples are included under `examples/`.

Library callers may pass the same map as `configuration.jwt`, or supply an
explicit `configuration.tokenService`; supplying both is rejected.

## Handler contract

An endpoint receives its JSON request body as its only explicit argument. HTTP
state is available only when the handler declares the corresponding context:

```lango
export createOrder(input)

createOrder(input)
    context request
    context response
    context token
    context log

    log.info("Creating an order for ${token.userId}")
    response.status(201).header("Location", "/orders/42")
    return {
        id: 42
        requestId: request.id
    }
```

`request` provides `id`, `method`, `path`, `headers`, and `remoteAddress`.
`token` contains verified claims when a TokenService is configured, and
`{ auth: false }` in the legacy host mode. `log` is request-scoped and includes the
request id. A normal return value is serialized as a JSON response. A handler
can call `response.send(value)` to write JSON immediately; a later return value
is ignored. `response.status(code)` and `response.header(name, value)` apply to
the next send. Promise results are awaited by the host without requiring
`await` in Lango source.

## Token service foundation

The `./tokens` entry exports `TokenService(options)` for signing and verifying
JWTs using jose. Options are `issuer`, `audience`, `activeKeyId`, `keys` (a map
of ids to Uint8Array secrets), optional `ttlSeconds` (default 900, maximum
3600), and optional `sessionTtlSeconds` (default 28800, range 1–2592000).
Keys must contain at least 32 bytes and should be generated with a
cryptographically secure random source and managed persistently outside source
control. Constructor inputs are copied to prevent later key-buffer mutation.

`sign(claims)` returns a token Promise. `verify(token)` returns a claims Promise
or rejects with `ApiInvalidToken`. The algorithm is fixed to HS256 and the
token type to `lango+jwt`; issuer, audience, age, expiry, and key id are checked.
System claims are written by the service. Retaining an old key id allows
verification during key rotation. Only trusted services that may also sign
tokens should receive these shared secrets.

Configure a library Host with `tokenService: service` to enable bearer
verification. Routes are then protected by default and require `token.auth` to
be true. Mark a route or group `anonymous: true` (XML `anonymous="true"`) for
public access; child routes can override group inheritance. Missing, invalid,
expired, or unauthenticated tokens return 401 with a Bearer challenge. Supplying
an invalid token to a public endpoint is also rejected. Without a TokenService,
the host retains its previous unauthenticated mode.

Verified claims are available through `context token`; changes made by a login
handler are signed into the `X-Lango-Token` response header. Renewals
also accompany handler error responses when claims were already verified.
On handler failure, token changes are discarded and the originally verified
claims are renewed, so a partially failed login cannot grant authentication.
`clientId`, `sessionId`, `sessionStartedAt`, and `sessionExpiresAt` are fixed for
the request and cannot be changed by mutating the context. Token-bearing
responses use `Cache-Control: no-store`.
Protected identities are also restored after a successful auth callback, before
the endpoint handler runs. Client and session IDs must be nonempty strings of
at most 128 characters when signing or verifying tokens.

UI Dispatcher endpoints use this verified session identity for their replay
store, together with the incoming form instance and sequence. Generic API
endpoints are not automatically deduplicated. Request audit fields remain
client-supplied correlation data; see the `lango-ui` event replay contract.

With tokens enabled, `POST /_lango/start` is reserved for bootstrap. It accepts
the previous bearer token, recovers only clientId after verifying its signature,
issuer, audience, type and known key id, and generates a new sessionId with
`auth: false`. Expiry is deliberately ignored only in this identity-recovery
path; old authentication and form claims are never restored. Invalid previous
tokens begin a fresh client identity. Normal endpoint verification still checks
expiry. The client persists the returned token and resets its local counters.

### Session scope across tabs and devices

Every bootstrap creates an independent session, including repeated bootstrap
with the same valid previous token. Only `clientId` is recovered; authentication
and form state are not inherited. Existing sessions remain valid until their
own expiry or revocation. A page reload starts another session rather than
restoring a previous tab's authentication.

Logout affects exactly the verified sessionId. It does not log out other tabs
with the same clientId or other devices belonging to an account. Clients with
no valid identity seed receive new clientIds, so simultaneous first launches
can register separate identities. clientId is not a hardware or account ID.
Applications needing account-wide logout or device limits must define their
own authorized account/session registry; a client-supplied clientId or account
name is not sufficient authority for that operation.

### Absolute session lifetime

Every new session receives `sessionStartedAt` and `sessionExpiresAt`, expressed
as integer Unix seconds. Bootstrap creates these before lifecycle callbacks;
the host restores them after register/start, auth, handler, and ping callbacks.
Login and token renewal preserve the original session lifetime. Creating or
renewing another form does not start a new lifetime.

Token `exp` is capped at the earlier of its normal TTL and the session deadline.
Verification and signing also enforce the earlier of the signed deadline and
`sessionStartedAt + sessionTtlSeconds` under the current configuration. A shorter
configured policy therefore applies to existing sessions; a longer policy
does not extend their original signed deadline. The absolute deadline has no
additional clock-tolerance grace. Signature verification retains its existing
five-second tolerance for JWT timing checks.

Expired sessions return 401 on ordinary endpoints, including anonymous routes,
and ping, with no replacement token. If the deadline passes during a handler,
response signing also returns 401 without renewal; already executed side effects
are not rolled back. `/_lango/start` may still recover only clientId and create
a new anonymous session with fresh bounds. It never restores authentication.
The browser does not automatically bootstrap after a 401; reload to start over
and authenticate again when needed.

Tokens issued before these session claims were introduced are rejected by
normal verification. They remain usable only for bootstrap identity recovery.
Trusted library calls to `sign(claims)` initialize bounds when both are absent;
renewals must pass the verified bounds. Missing or malformed bounds in a signed
token cannot silently start another lifetime.

### Logout and session revocation

With tokens enabled, `POST /_lango/logout` requires a valid bearer token, including
an anonymous session token. It revokes the entire session: all form tokens with
that `sessionId` are rejected by normal endpoints and ping. Success returns
`{ "loggedOut": true }`, `Cache-Control: no-store`, and no `X-Lango-Token`.
Missing, expired, invalid, or already revoked tokens return 401. The path is
reserved when a TokenService is configured. Other sessions, including sessions
with the same `clientId`, are unaffected.

Application handlers can also log out through the response context:

```lango
logout(input)
    context response
    response.logout()
    return {
        loggedOut: true
    }
```

`response.logout()` returns a Promise and is idempotent within the request. The
host waits for revocation before sending the response, even when the handler
does not return that Promise. It uses the originally verified session identity;
changing `token.sessionId` cannot revoke another session. No replacement token
is issued. A subsequent handler failure does not undo revocation. A store failure
returns 500 without a token or a successful logout acknowledgment. Ping and
completed request-audit callbacks cannot call `response.logout()`.

The service also exposes `revoke(claims)` for trusted server code. Pass verified
claims, never an unverified request body. `verify` checks the revocation store,
and `sign` checks it before and after asynchronous signing. Requests already
executing can still have side effects; revocation does not roll them back.
Setting `token.auth = false` alone continues to issue an anonymous token and
does not revoke older tokens. Replay protection remains separate work.

By default, each TokenService owns a `MemorySessionRevocations` store, also
exported by `./tokens`. Its records expire after the configured token lifetime
plus the five-second clock tolerance. This store is process-local and its state
is lost on restart: old, otherwise valid tokens can become usable again.
Persistent or multi-process deployments must supply `options.revocations` with
a shared, durable store:

| Method | Contract |
| --- | --- |
| `isRevoked(sessionId)` | Returns a boolean or a Promise of a boolean. |
| `revoke(sessionId, expiresAt)` | Records revocation before resolving; `expiresAt` is Unix time in seconds. Repeated writes must retain the later expiry. |

All instances serving the same tokens must consult the same store, use the same
token and session lifetime policies, and observe completed revocations across instances.
Store errors reject authentication or signing; they never bypass revocation.
The CLI can configure SQLite persistence as described below; custom stores can
also be supplied through the library API.

A revoked token can still be used at `/_lango/start` to recover only `clientId`;
it never restores the revoked session or its authentication. Browser clients
should stop old form sessions and bootstrap a new anonymous session after
logout. `lango-client` implements this flow through `application.logout()`,
including form disposal, late-response protection, and fresh sequence counters.

### Persistent runtime state

Add a state section to `conf.xml` to persist both session revocations and UI
sequence reservations:

```xml
<conf>
  <state backend="sqlite" file="runtime.sqlite" namespace="shop" maxForms="10000" />
</conf>
```

Without state configuration, the existing memory defaults remain. A state
section defaults to SQLite, `runtime.sqlite`, and 10000 form streams. Its default
namespace is the application name, or `default` for unnamed library hosts.
Relative files resolve under the final `stateDir`, including CLI overrides;
absolute files use the specified path. Missing stateDir for a relative file,
unknown settings/backends, invalid capacities, and database open/schema errors
prevent startup. SQLite failure never silently switches to memory. State
configuration does not enable JWT; configure JWT separately.

| CLI option | Environment variable | XML attribute |
| --- | --- | --- |
| `--state-backend memory|sqlite` | `LANGO_STATE_BACKEND` | `backend` |
| `--state-file` | `LANGO_STATE_FILE` | `file` |
| `--state-namespace` | `LANGO_STATE_NAMESPACE` | `namespace` |
| `--state-max-forms` | `LANGO_STATE_MAX_FORMS` | `maxForms` |

Precedence is CLI > environment > XML > defaults. `maxForms` is a positive safe
integer and applies per namespace. All processes sharing a namespace should
use the same capacity and token policies. Keep the file and namespace stable
across restarts; changing or deleting them discards their protection.

Library Hosts accept the same `configuration.state` map. Their configured
TokenService uses this store for revocation. Auth and endpoint handlers can
read it through `context state`; it is null when no shared store was configured.
UI Dispatcher automatically uses the request's state store unless an explicit
`replayStore` adapter was supplied. A dispatcher created at module load does not
need a reference to the Host. Host-owned stores close with the HTTP server and
on startup/preflight failure.

The `./state` entry exports `SqliteStateStore(file, options)` with optional
`namespace` and `maxForms`. It implements both `isRevoked`/`revoke` and
`claim`/`complete`. Callers can provide `configuration.stateStore` for a shared
custom backend, including a remote database. It must implement all four
methods; the caller owns its lifecycle. With an explicit TokenService, pass
the same store as its `revocations` option. File-based state configuration and
an independently constructed TokenService are rejected as ambiguous.

SQLite owns a separate connection and does not replace the application's SQL
adapter. It uses WAL, full synchronous commits, a five-second busy timeout,
bound query parameters, and `BEGIN IMMEDIATE` for replay reservations. Multiple
processes can share one database on the same local filesystem; this is not a
multi-machine database service. New directories and files request owner-only
permissions. The database stores identifiers, sequence/expiry values and
reservation state, not raw JWTs.

Completed sequence records survive restart. A reservation interrupted before
completion remains pending until its session deadline; it is never reset merely
because another process opened the database. A repeated sequence stays consumed,
and higher sequences return `InProgress`. Recovery of uncertain business effects
and distributed adapters remain separate concerns.

## Authorization callbacks

Routes and groups accept `auth` as a function reference, a named auth rule, or
`{ module, function }`. Module values may be npm specifiers or already loaded
module objects:

```lango
routes.auth({
    name: "access"
    function: checkAccess
})
private = routes.group({
    path: "/private"
    auth: "access"
})
private.endpoint({
    path: "/save"
    module: "@shop/orders"
    function: "save"
})

checkAccess(path, input)
    context token
    return token.role == "admin"
```

XML supports top-level definitions before or after their uses:

```xml
<api>
  <auth name="access" mod="@shop/security" func="checkAccess" />
  <group path="/private" auth="access">
    <endpoint path="/save" mod="@shop/orders" func="save" />
  </group>
</api>
```

The nearest group's auth rule is inherited; an explicit child rule overrides
it. The rule runs after the default JWT/auth check and receives the actual path
and input, with request/response/token/log contexts available. Promise results
are supported. Only Boolean true permits the endpoint handler. False produces
403; exceptions, rejections or non-Boolean results produce 500. Denial and
errors discard token mutations before renewal. Anonymous routes still run
custom auth, as do legacy hosts without a TokenService.

The Host validates auth modules/exports before listening and rebuilds linked
packages used only by auth rules too. Direct module calls preserve their receiver.

## Application packages

An endpoint `mod` is an npm package specifier, resolved from the application's
own `package.json` and `node_modules` directory. For example:

```xml
<endpoint path="/math/add" mod="@shop/math" func="add" />
```

Published packages run through their normal `main` or `exports` entry. During
development, a package installed with `npm link` is rebuilt when `lango-api`
starts if its `package.json` declares the Lango source and generated entry:

```json
{
  "scripts": {
    "build": "lango build src/index.lango --out dist/index.mjs"
  },
  "lango": {
    "source": "src/index.lango",
    "output": "dist/index.mjs"
  }
}
```

There is no hot reload. Restart `lango-api` after editing a linked package; the
restart rebuilds it before the API host begins listening.

## License

Apache-2.0

## Synchronous worker prototype

Worker execution is an opt-in mode of the standard `Host` and CLI. The ordinary
HTTP path remains the default. A lower-level Piscina host is also available
through `@lango-project/lango-api/workers`.

The HTTP listener stays on the main thread. A bounded pool runs at most one
request per worker at a time. Each worker loads the application's compiled
module, calls its exported `createWorker(options)` factory once, and retains the
returned routes. The factory returns a map with `routes: routes.values()` and
optional synchronous `register`, `start`, and `close` functions. Construct routes,
module objects, and closures inside this factory; functions and live server
objects are not transferred between threads.

A worker establishes `context request`, `response`, `token`, `state`, and `log`.
Handlers, authorization callbacks, and register/start hooks execute synchronously.
Returned Promises and thenables are rejected rather than synchronously waited on.
The transport and initial module loading still use Promises at their boundaries.
Worker logging currently uses the standard default logger.

### Standard Host and CLI configuration

The library enables workers with a `workers` map:

```lango
host = new api/Host({
    applicationName: "shop"
    appRoot: appRoot
    stateDir: stateDir
    host: "127.0.0.1"
    port: 3000
    workers: {
        application: "dist/worker.mjs"
        optionsFile: "worker-options.json"
        threads: 2
    }
    jwt: {
        issuer: "shop"
        audience: "browser"
        activeKeyId: "current"
        keysFile: keysFile
    }
})
return host.start()
```

`Host.start()` keeps its Promise-for-HTTP-server contract in both modes.
`Host.close()` drains requests and closes the worker pool, and is safe to call
repeatedly. Closing the returned HTTP server directly also closes its pool.
Configure routes, authorization, and lifecycle callbacks inside `createWorker`;
parent-side `Host.routes`, `routesFromXml`, and `on` registrations cannot be
transferred to workers and are rejected at startup.

The standard CLI reads a new configuration section:

```xml
<conf>
    <workers application="dist/worker.mjs" threads="2" maxQueue="64"
             bodyLimit="1048576" optionsFile="worker-options.json" />
</conf>
```

A workers section or worker option enables the mode unless explicitly disabled.
Without worker settings it stays off. Defaults are `dist/worker.mjs`, 2 workers,
64 waiting tasks, and a 1048576-byte body limit. Missing standard directories use
the existing development/server directory selection in the CLI. Application,
worker-entry override, and options-file paths resolve against the final
`appRoot`, after all overrides. An options file must contain a JSON map. The
library may instead pass `workers.options` directly; combining both is rejected.
The factory receives `applicationName`, `appRoot`, `dataDir`, `stateDir`, and
`logDir` as defaults, with explicit application options overriding those values.

Configuration precedence remains XML, then environment, then command line:

| CLI | Environment | `workers` attribute |
| --- | --- | --- |
| `--workers true\|false` | `LANGO_WORKERS` | `enabled` |
| `--worker-application` | `LANGO_WORKER_APPLICATION` | `application` |
| `--worker-threads` | `LANGO_WORKER_THREADS` | `threads` |
| `--worker-max-queue` | `LANGO_WORKER_MAX_QUEUE` | `maxQueue` |
| `--worker-body-limit` | `LANGO_WORKER_BODY_LIMIT` | `bodyLimit` |
| `--worker-entry` | `LANGO_WORKER_ENTRY` | `workerFile` |
| `--worker-options` | `LANGO_WORKER_OPTIONS` | `optionsFile` |

Existing JWT settings load the shared key material once in the parent. XML key
filenames remain relative to `conf.xml`; environment/CLI key filenames follow
the existing working-directory rule. Missing or disabled JWT configuration
runs requests with an anonymous token context and does not read a key file.
Bootstrap/logout endpoints and authenticated UI replay require JWT to be enabled.
The SQLite file defaults to `stateDir/runtime.sqlite` with the application name
as namespace. Existing `<state>`/environment/CLI overrides still apply; an
explicit memory backend or a live parent-side state/token instance is rejected.

Worker mode starts from the factory without loading `api.xml`. Supplying
`--api`, `LANGO_API`, or an explicit API file in worker mode is rejected until
worker-side XML route loading is implemented. The CLI closes the host on SIGINT
and SIGTERM.

### Lower-level worker host

`startWorkers(configuration)` returns a Promise for a `WorkerHost`. Alternatively,
construct `WorkerHost` and call `start()`. Its `server` and `pool` properties expose
the underlying HTTP server and Piscina instance. `close()` stops new requests,
waits for accepted HTTP requests, and closes the pool. Application `close` hooks
are worker-exit cleanup; forced termination must not be relied on to complete
application cleanup or transactions.

Direct `WorkerHost` configuration (these are transport settings rather than the
standard `Host` configuration shown above):

- `application`: absolute path to the compiled application module exporting
  `createWorker(options)`.
- `appRoot`: application package directory, used to locate the package's
  `worker-entry` export. `workerFile` may explicitly override that entry with an
  absolute compiled filename.
- `stateFile`: absolute SQLite filename shared by every worker; `namespace` and
  `maxForms` use the existing state-store defaults.
- `tokens`: `issuer`, `audience`, `activeKeyId`, `keys` containing byte arrays,
  and optional `ttlSeconds`/`sessionTtlSeconds`. Keys are supplied once by the
  parent; keep them stable across restarts. Revocation uses the shared SQLite
  store automatically. Omit `tokens` or pass null for anonymous operation.
- `options`: structured-cloneable application factory settings.
- `host`, `port`: HTTP bind address and port.
- `threads`: fixed pool size, default 2, maximum 128.
- `maxQueue`: maximum waiting tasks, default 64; overload returns 503.
- `bodyLimit`: maximum request-body bytes, default 1048576; excess returns 413.

Only plain request metadata and body text enter the worker. Status, headers and
serialized JSON leave it. The HTTP API supports POST routes, bootstrap at
`/_lango/start`, logout at `/_lango/logout`, and main-thread GET `/health` liveness.
Worker routes must use direct handlers or preloaded module objects. JWT renewal
preserves protected identity fields and rechecks session revocation after handler
execution. A logout in another worker therefore prevents a late response from
renewing that session. UI replay claims and logout records share SQLite rather
than independent worker memory.

`SyncTokenService`, exported from `@lango-project/lango-api/tokens`, provides the
existing token operations as synchronous values through `jsonwebtoken`'s
synchronous API. HS256, token type/key selection, issuer/audience, required claims,
absolute session limits, and revocation remain checked. Its JWTs interoperate
with the existing `jose` service. Expired signed tokens can recover only client
identity during bootstrap; authentication and session identity start fresh.

For UI handlers, create the dispatcher inside the worker factory:

```lango
dispatcher = new ui/Dispatcher(appRoot, {
    synchronous: true
})
```

Pass direct functions or preloaded modules to its event registrations. This mode
uses synchronous replay adapters and returns the ordinary UI result directly.
Handlers can use `context form` and write `form.saveButton.enabled = false`
without a Promise chain.

Run the standalone API example from this repository with the current compiler:

```sh
npm run build:worker-demo
npm run demo:workers
```

Then, from another terminal:

```sh
curl http://127.0.0.1:8081/health
curl -X POST http://127.0.0.1:8081/hello \
  -H 'Content-Type: application/json' -d '{"name":"Lango"}'
```

The demo uses standard `Host` configuration and keeps its reusable JSON key file
and SQLite state in `data/worker-demo`. It preserves a key from an earlier binary
`jwt.key` demo file when creating `keys.json`. The application handler is in
`examples/worker-app.lango`; the launcher is in `examples/worker-server.lango`.
Both are Lango source.

After starting the demo once to create its key file, stop it and launch the same
application through the normal CLI from this repository:

```sh
node dist/cli.mjs worker-demo --app-root . --config examples/worker-conf.xml
```

`npm test` includes synchronous JWT compatibility, settings precedence, Host
restart/close, anonymous mode, and real CLI startup/SIGTERM tests. The worker
Host/CLI tests do not require the UI repository.
`npm run test:workers` additionally builds the sibling `../lango-ui` project and
runs a real two-worker HTTP/UI/SQLite integration test. It checks replay,
Promise rejection, request limits, simultaneous workers, queue overload,
main-thread health responsiveness, and cross-worker logout during a running
request. This integration test requires the local multi-repository workspace.

Remaining integration includes linked-package preparation, string-module
preloading, worker-side XML route/lifecycle loading, batch ping, request audit,
and other asynchronous adapters. Streaming, sockets, arbitrary parent-process
instances, background asynchronous jobs, and automatic retry after a worker
crash are outside this prototype. A failed request can already have committed
application changes; neither worker termination nor an HTTP error rolls them back.

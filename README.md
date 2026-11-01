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
    appRoot: appRoot,
    host: "127.0.0.1",
    port: 3000
})

routes = new api/Routes()
routes.endpoint({
    path: "/health",
    module: "@shop/health",
    function: "check"
})

math = routes.group({ path: "/math", module: "@shop/math" })
math.endpoint({ path: "/add", function: "add" })

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
    path: "/local",
    function: localHandler
})
```

Linked-package rebuilding requires a string package specifier, so the string
form remains the standard development route form.

### Client and session lifecycle

JWT-enabled hosts can initialize application state before issuing a token:

```lango
host.on({ event: "register", function: registerClient })
host.on({ event: "start", module: "@shop/session", function: "startSession" })

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
replaces its XML events, while code registrations remain. Request audit and
ping events are pending.

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
       keysFile="jwt.keys.json" activeKeyId="current" ttlSeconds="900" />
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

The precedence remains CLI > environment > XML > defaults. Any JWT section or
JWT option enables JWT unless `enabled` is explicitly false. Lifetime defaults
to 900 seconds; issuer, audience, key file and active id are required when
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
    return { id: 42, requestId: request.id }
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
of ids to Uint8Array secrets), and optional `ttlSeconds` (default 900, maximum
3600). Keys must contain at least 32 bytes and should be generated with a
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
or logout handler are signed into the `X-Lango-Token` response header. Renewals
also accompany handler error responses when claims were already verified.
On handler failure, token changes are discarded and the originally verified
claims are renewed, so a partially failed login cannot grant authentication.
`clientId` and `sessionId` are fixed for the request and cannot be changed by
mutating the context. Token-bearing responses use `Cache-Control: no-store`.

With tokens enabled, `POST /_lango/start` is reserved for bootstrap. It accepts
the previous bearer token, recovers only clientId after verifying its signature,
issuer, audience, type and known key id, and generates a new sessionId with
`auth: false`. Expiry is deliberately ignored only in this identity-recovery
path; old authentication and form claims are never restored. Invalid previous
tokens begin a fresh client identity. Normal endpoint verification still checks
expiry. The client persists the returned token and resets its local counters.

Revocation and replay policy remain pending. Setting
`token.auth = false` issues an anonymous token but does not revoke older tokens.

## Authorization callbacks

Routes and groups accept `auth` as a function reference, a named auth rule, or
`{ module, function }`. Module values may be npm specifiers or already loaded
module objects:

```lango
routes.auth({
    name: "access"
    function: checkAccess
})
private = routes.group({ path: "/private", auth: "access" })
private.endpoint({ path: "/save", module: "@shop/orders", function: "save" })

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

## Lisans

Apache-2.0

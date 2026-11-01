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
`token` initially contains `{ auth: false }`; authentication will replace or
enrich it in a later milestone. `log` is request-scoped and includes the
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

This is a standalone foundation. HTTP route authorization, token refresh,
browser persistence, and revocation are not yet integrated. The host still
provides its existing anonymous token context.

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

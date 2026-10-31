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
    logDir="/optional/logs" />
</conf>
```

All attributes are optional. `LANGO_HOST` and `LANGO_PORT` environment variables override `conf.xml`; command-line directory overrides will be added with the route host.

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

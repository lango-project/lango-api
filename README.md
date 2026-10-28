# @lango-project/lango-api

`lango-api` is an API host, written in Lango, that exposes Lango applications over HTTP.

Application sources, machine-specific configuration, persistent data, and logs are kept separate. The evolving Lango architecture documentation defines the directory contract and operating modes.

## Initial goal

The first milestone provides the following flow:

1. Resolve development or server directories from an application name.
2. Load and validate `conf.xml` and `api.xml`.
3. Route an endpoint to an exported Lango function.
4. Run as an independent HTTP server.

Reverse-proxy support, service definitions, hot reload, authentication, and the advanced HTTP contract follow in later milestones.

Source code and tests are `.lango` files. JavaScript generated from them is published under `dist/` so the npm package can run; `dist/` is never edited manually.

The Lango compiler does not yet produce an executable entry that invokes an exported `main` function for an npm `bin`. Once that is available, this package will directly provide the `lango-api` command.

## Lisans

Apache-2.0

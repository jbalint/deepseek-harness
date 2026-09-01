# Agent Note: Web sub-path (basePath) mount

Status: implemented

English | [中文](2026-09-01-web-basepath-subpath-mount.zh.md)

## Problem

`dsh web` serves every route, asset, and client request at the site root: the `/api` bridge, `/plugins` bundle URLs, the HMR `/plugins/events` stream, the Gateway `/api/remote.mux` WebSocket, and the SPA dist anchor (`<base href="/">`) are all root-absolute, and the browser clients derive their transport base from `location.origin`. A reverse proxy that mounts the GUI under a sub-path such as `/dsh` beside another application therefore cannot route the two `/api` surfaces apart, and the browser's root-absolute requests and asset references bypass the sub-path entirely.

## Decision

One `basePath` config value on [`dsh-host-webserver`](../../../../packages/host/webserver/README.md) drives the whole surface, with empty as the default so existing root mounts behave unchanged.

The webserver strips the prefix from every incoming request target — `handle` and the `upgrade` handler rewrite `req.url` before route matching and before any route owner reads it — so `/api`, `/plugins`, `/plugins/events`, `/api/remote.mux`, and the fallback keep their natural paths with no per-route change. `WebServer` exposes `basePath` and `baseHref` (`` `${basePath}/` ``, so `/` when unset).

`dsh-host-frontend-static` anchors each index with `<base href="${baseHref}">`; the dist already builds with a relative Vite base, so the anchor alone re-points the asset URLs. `dsh-client-modules` keeps its response maps keyed by the un-prefixed `/plugins` path (the route serves the stripped target) and prefixes only the advertised URLs — graph row `url`, batch `url`, and the stamped `sourceMappingURL` — through `advertisedPath`.

The browser clients derive their base from `document.baseURI`, which reflects the injected `<base>` element, and request a path relative to it: `dsh-client-connection`'s unary RPC (`resolveBase`) and `dsh-api-gateway`'s WebSocket mux (`remoteStreamUrl`) drop the leading slash from the channel path; `dsh-client-hmr` opens `EventSource` with the endpoint's relative form. Workers have no `document` and fall back to `location.origin`.

`dsh-client-connection`'s `BrowserAuth` scopes its token URL, post-exchange redirect, and cookie path under the prefix. `dsh-web-app` appends the prefix to the canonical loopback URL, and its `--base-path` flag feeds the webserver row in `cordis.patch.yml`.

## Testing

Each touched package carries real-composition or focused coverage: the webserver proves prefix stripping before matching, dispatch, and upgrade; `frontend-static` proves the `<base href="/dsh/">` anchor, the `/dsh/` redirect, and prefixed asset serving; `browser-auth` proves the prefixed token URL, redirect, and cookie path; `client-modules` proves advertised URLs are prefixed while the un-prefixed path serves; `startup` proves `--base-path` reaches the webserver config. Client URL bases keep their existing `location.origin` assertions unchanged because the default base is the root.

## Alternatives considered

**Prefix every route at registration.** Each owner (`connection`, `modules`, `gateway`, `hmr`) would prepend the prefix to its route path and generated URLs. This spreads the prefix across every route owner and its URL generation, while stripping once at the webserver entry leaves route paths untouched and matches the natural `/plugins` response-map keys.

**Proxy strips the prefix (nginx `proxy_pass …/`).** This needs no server-side routing change, but the browser still sends root-absolute `/api` and asset requests that the proxy must rewrite, and the client still lacks a base-path concept. The webserver strip works with or without a stripping proxy and is the single server-side fact.

**A fixed `/dsh` instead of a config value.** Hardcoding the prefix cannot serve multiple or differently named mounts and violates the no-hardcoded-tunable convention; `basePath` is a validated deployment value like the bind host.

## Consequences

One `dsh web --base-path /dsh` flag mounts the full GUI — routes, plugin bundles, streams, assets, and authentication — under a reverse-proxy sub-path beside other applications, with empty keeping the root behavior byte-for-byte. The prefix is one validated webserver field, read by the four surface owners server-side and derived from `<base>` client-side, so no new wire or session-log format carries it.

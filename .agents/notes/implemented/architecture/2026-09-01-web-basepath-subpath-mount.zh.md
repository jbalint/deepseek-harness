# Agent Note: Web 子路径（basePath）挂载

Status: implemented

[English](2026-09-01-web-basepath-subpath-mount.md) | 中文

## Problem

`dsh web` 把每条路由、每个资源和每个客户端请求都服务在站点根路径下：`/api` 桥接、`/plugins` bundle URL、HMR 的 `/plugins/events` 流、Gateway 的 `/api/remote.mux` WebSocket，以及 SPA dist 锚点（`<base href="/">`）都是根绝对路径，浏览器客户端也以 `location.origin` 作为传输基准。当反向代理把 GUI 挂载到 `/dsh` 这类子路径、并与其它应用共存时，无法区分两个 `/api` 表面，浏览器发出的根绝对请求和资源引用也会完全绕开子路径。

## Decision

由 [`dsh-host-webserver`](../../../../packages/host/webserver/README.zh.md) 上的一个 `basePath` 配置值驱动整个表面，默认值为空，因此现有的根路径挂载行为保持不变。

webserver 在每条进入的请求目标上剥除前缀——`handle` 与 `upgrade` 处理器在路由匹配之前、在任何路由 owner 读取之前重写 `req.url`——因此 `/api`、`/plugins`、`/plugins/events`、`/api/remote.mux` 以及 fallback 都保留各自的自然路径，无需任何逐路由改动。`WebServer` 暴露 `basePath` 与 `baseHref`（`` `${basePath}/` ``，未设置时为 `/`）。

`dsh-host-frontend-static` 用 `<base href="${baseHref}">` 锚定每个 index；dist 本就以相对 Vite base 构建，因此仅靠锚点即可重新指向资源 URL。`dsh-client-modules` 的响应映射仍以未加前缀的 `/plugins` 路径为键（路由服务的是剥除后的目标），只对对外公布的 URL——图 row 的 `url`、batch 的 `url` 以及写入的 `sourceMappingURL`——通过 `advertisedPath` 加前缀。

浏览器客户端以 `document.baseURI`（反映注入的 `<base>` 元素）为基准，并请求相对于它的路径：`dsh-client-connection` 的一元 RPC（`resolveBase`）与 `dsh-api-gateway` 的 WebSocket mux（`remoteStreamUrl`）去掉通道路径前导斜杠；`dsh-client-hmr` 以端点的相对形式打开 `EventSource`。Worker 没有 `document`，回退到 `location.origin`。

`dsh-client-connection` 的 `BrowserAuth` 把 token URL、交换后重定向和 cookie 路径都收束在前缀之下。`dsh-web-app` 把前缀附加到规范回环 URL 上，其 `--base-path` 标志在 `cordis.patch.yml` 中填入 webserver row。

## Testing

每个触及的包都带有真实组合或聚焦覆盖：webserver 证明匹配、分发与 upgrade 之前的前缀剥除；`frontend-static` 证明 `<base href="/dsh/">` 锚点、`/dsh/` 重定向与带前缀的资源服务；`browser-auth` 证明带前缀的 token URL、重定向与 cookie 路径；`client-modules` 证明对外 URL 带前缀而内部未加前缀路径仍可服务；`startup` 证明 `--base-path` 到达 webserver 配置。客户端 URL 基准保持既有的 `location.origin` 断言不变，因为默认基准就是根路径。

## Alternatives considered

**在注册时为每条路由加前缀。** 每个 owner（`connection`、`modules`、`gateway`、`hmr`）都要在路由路径与生成的 URL 上重复前缀。这把前缀散布到每个路由 owner 及其 URL 生成逻辑；而在 webserver 入口只剥除一次，路由路径保持不变，且能匹配自然的 `/plugins` 响应映射键。

**由代理剥除前缀（nginx `proxy_pass …/`）。** 这无需服务端路由改动，但浏览器仍会发出根绝对的 `/api` 与资源请求，代理必须重写它们，且客户端仍缺少 base-path 概念。webserver 剥除在有或无剥除代理时都能工作，是唯一的服务端事实。

**使用固定的 `/dsh` 而非配置值。** 硬编码前缀无法服务多个或不同命名的挂载，也违反“不硬编码可调项”的约定；`basePath` 是像绑定主机一样经过校验的部署值。

## Consequences

一个 `dsh web --base-path /dsh` 标志即可把整个 GUI——路由、插件 bundle、流、资源与认证——挂载到反向代理子路径下、与其它应用共存；为空时逐字节保持根路径行为。前缀是一个经过校验的 webserver 字段，服务端由四个表面 owner 读取，客户端从 `<base>` 派生，因此没有新的 wire 或 session-log 格式携带它。

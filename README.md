# web

[![CI](https://github.com/alya-lang/web/actions/workflows/ci.yml/badge.svg)](https://github.com/alya-lang/web/actions/workflows/ci.yml)
[![License](https://img.shields.io/github/license/alya-lang/web?color=blue&label=License)](LICENSE)
[![Alya](https://img.shields.io/badge/dynamic/toml?url=https%3A%2F%2Fraw.githubusercontent.com%2Falya-lang%2Fweb%2Fmain%2Falya.toml&query=%24.package.alya-version&label=Alya&color=orange&prefix=%3E%3D)](https://github.com/alya-lang/alya)
[![Package Version](https://img.shields.io/badge/dynamic/toml?url=https%3A%2F%2Fraw.githubusercontent.com%2Falya-lang%2Fweb%2Fmain%2Falya.toml&query=%24.package.version&label=Version&color=brightgreen)](alya.toml)

High-level web framework for Alya: routing, middleware, JSON APIs, and views

---

## 🌟 Features

- 🧭 **App & Routing**: `app()` with `get/post/put/delete/patch/any` routes, path params (`:id`), wildcards (`*path`), groups, and UFCS method forms (`app.get(...)`)
- 🛡️ **Error Boundary**: throwing handlers become 500 responses; custom `not_found` and `on_error` hooks instead of dead connections
- 🧩 **Middleware Suite**: CORS, compression, static files, logging (+templates), Basic/Bearer/JWT auth, rate limiting, request IDs, timeouts, size limits, keep-alive, trusted proxies, TLS — all `use_*` one-liners
- 📦 **JSON APIs**: `ctx_json`, `ctx_json_value`, `ctx_json_map`, `ctx_body_json` for request/response cycles
- 🖥️ **Views**: Mustache rendering with a compiled-template cache (`view_cached`), layouts (`view_layout` with `{{{body}}}`), and pooled fragment assembly (`view_fragments`)
- 🪣 **Buffer Pooling**: per-app `pool` buffer reuse for fragment assembly with `pool_stats` observability (degrades to fresh arrays under exhaustion)
- 🔁 **Serve Loop**: one-call `listen` with keep-alive connection reuse over `alya-lang/http` (v0.4.0)

---

## 📁 Project Architecture

```
web/
├── alya.toml               # Package manifest (http, json, mustache, pool)
├── src/
│   ├── lib.alya            # Public API facade & `app()` constructor
│   ├── types.alya          # `App` and `WebGroup` models
│   ├── app.alya            # Routes, groups, error hooks, serve loop
│   ├── middleware.alya     # `use_*` settings (all chainable)
│   ├── context.alya        # `ctx_*` handler conveniences
│   └── views.alya          # Mustache cache, layouts, pooled fragments
├── examples/
│   └── demo.alya           # Runnable tour of routes, middleware, views, serving
├── tests/                  # 6 suites: app, context, middleware, serve, errors, views
│   ├── test_app.alya
│   ├── test_context.alya
│   ├── test_middleware.alya
│   ├── test_serve.alya
│   ├── test_errors.alya
│   └── test_views.alya
└── benches/
    └── bench_basic.alya    # Route dispatch + cached-render micro-benchmarks
```

> [!NOTE]
> **Visibility & Modularity:** Symbols annotated with `pub` are exported to external consumers. Route handlers registered with `app.get/post/...` are first-class function values invoked with the request context.

---

## 📦 Installation

Add `web` to the `[dependencies]` section in your `alya.toml`:

```toml
[dependencies]
web = { git = "https://github.com/alya-lang/web", branch = "main" }
```

Or install it directly using the Alya package CLI:

```bash
alya add web --git https://github.com/alya-lang/web --branch main
alya install
```

### Package Features

This package ships a single, dependency-light build with no optional features.

```bash
alya install
alya test
```

---

## 🚀 Quick Start

```alya
import "web" as web

function hello(ctx)
    return web::ctx_json_map(ctx, {"message": "Hello"})
end

function user(ctx)
    let uid = web::ctx_param(ctx, "id")
    return web::ctx_text(ctx, "user:" + uid, 200)
end

function main()
    let app = web::app()
    app.get("/hello", hello)
    app.get("/users/:id", user)
    app.use_request_id()
    app.listen(8080)
end

main()
```

Protect an API in three lines:

```alya
import "web" as web

function main()
    let app = web::app()
    app.use_bearer_auth("tok-123").use_rate_limit(100, 60000).use_keep_alive()
    app.listen(8080)
end

main()
```

---

## 📖 API Reference

### App & Routing

| Function | Parameters | Description |
|---|---|---|
| `app()` | — | Creates a new `App` with default settings |
| `app_get/post/put/delete/patch/head/options/query/any(app, pattern, handler)` | `app: App, ...` | Registers a handler-function route |
| `app_on(app, method, pattern, handler)` | `app: App, ...` | Registers a route for an explicit method |
| `app_group(app, prefix)` | `app: App, prefix: string` | Creates a `WebGroup` under a path prefix |
| `group_get/post/put/delete/patch/any(g, pattern, handler)` | `g: WebGroup, ...` | Registers a group route |
| `group_on(g, method, pattern, handler)` | `g: WebGroup, ...` | Registers a group route for an explicit method |
| `app_match(app, method, path)` | `app: App, ...` | Matches without invoking; returns the match result |
| `app_dispatch(app, method, path, ctx)` | `app: App, ...` | Matches, injects params, and invokes the handler |
| `App.get/post/...` `WebGroup.get/post/...` | methods | UFCS forms (`app.get("/x", h)`) |

### Serving & Errors

| Function | Parameters | Description |
|---|---|---|
| `listen(app, port, host, max_conns, timeout)` | `app: App, ...` | Builds, starts, serves, closes; `max_conns <= 0` serves forever |
| `serve(app, srv, max_conns, ...)` | `app: App, srv, ...` | Serves connections on a started server |
| `serve_once(app, srv, ...)` | `app: App, srv, ...` | Serves one connection with an error boundary |
| `app_build_server(app, port, host)` | `app: App, ...` | Builds the configured `http` server (unstarted) |
| `not_found(app, handler)` | `app: App, handler: fn(ctx)` | Custom handler for unmatched paths |
| `App.on_error(app, handler)` | method `app.on_error(fn(ctx, err))` | Hook invoked after a handler throws (500 already set; no free form — see code note) |

### Middleware (`use_*`, all chainable)

| Function | Parameters | Description |
|---|---|---|
| `use_logger(app, enabled)` | `app: App, ...` | Access logging (`set_log_format` for templates) |
| `use_cors(app, origin, methods, headers)` | `app: App, ...` | CORS headers + preflight |
| `use_compression(app, min_length)` | `app: App, ...` | Response compression |
| `use_static(app, prefix, dir)` | `app: App, ...` | Static file serving |
| `use_basic_auth(app, user, pass)` | `app: App, ...` | HTTP Basic gate |
| `use_bearer_auth(app, token)` | `app: App, ...` | Bearer token gate |
| `use_jwt_auth(app, secret)` | `app: App, ...` | JWT (HS256) gate |
| `use_rate_limit(app, limit, window_ms)` | `app: App, ...` | Fixed-window throttling (429) |
| `use_request_id(app, enabled)` | `app: App, ...` | `X-Request-ID` tracing |
| `use_timeout(app, ms)` | `app: App, ...` | Socket I/O timeout |
| `use_limits(app, max_header, max_body)` | `app: App, ...` | Size caps with 413 rejection |
| `use_keep_alive(app, max_requests)` | `app: App, ...` | Connection reuse |
| `use_forwarded(app, enabled)` | `app: App, ...` | Trusted-proxy client identity |
| `use_tls(app, cert, key)` | `app: App, ...` | TLS termination |

### Context (`ctx_*`)

| Function | Parameters | Description |
|---|---|---|
| `ctx_text/json/html(ctx, body, status)` | `ctx, ...` | Text, JSON string, and HTML responses |
| `ctx_json_value(ctx, val, status)` | `ctx, ...` | Recursive native-value JSON |
| `ctx_json_map(ctx, map, status)` | `ctx, ...` | Flat string-map JSON |
| `ctx_body_json(ctx)` | `ctx` | Parses the request body as JSON |
| `ctx_render(ctx, template, data, status)` | `ctx, ...` | Mustache template as HTML |
| `ctx_redirect/not_found(ctx, ...)` | `ctx, ...` | Redirect and 404 helpers |
| `ctx_param/param_int/query/body(ctx, ...)` | `ctx, ...` | Route params, query, and body readers |
| `ctx_header/set_header/status(ctx, ...)` | `ctx, ...` | Header and status helpers |
| `ctx_cookie/get_cookie(ctx, ...)` | `ctx, ...` | Cookie read/write |
| `ctx_client_ip/scheme(ctx)` | `ctx` | Effective client IP and scheme |
| `ctx_abort(ctx)` | `ctx` | Aborts pipeline processing |

### Views & Pool

| Function | Parameters | Description |
|---|---|---|
| `view(ctx, template, data, status)` | `ctx, ...` | Mustache template as HTML (parse per call) |
| `view_cached(app, ctx, template, data, status)` | `app: App, ...` | Compile-once render via the app template cache |
| `view_layout(app, ctx, layout, body, data, status)` | `app: App, ...` | Body (cached) injected raw as `{{{body}}}` into the layout |
| `view_fragments(app, ctx, parts, data, status)` | `app: App, ...` | Joins part renders using a pooled buffer |
| `pool_stats(app)` | `app: App` | Buffer-pool `{size, idle, active}` map |

> [!NOTE]
> **Mustache tags in source:** build templates from `ot = "{{"` / `ct = "}}"` fragments. Literal `{{...}}` runs in plain strings are brace-collapsed by the toolchain, which corrupts tags before Mustache sees them (see `tests/test_views.alya`).

---

## 🧪 Running Tests & Benchmarks

Run the automated test suite using `alya test`:

```bash
alya test
```

Exercise the full suite (no optional features in this package):

```bash
alya test
```

Generate static API documentation:

```bash
alya doc . -o docs --markdown
```

Run the benchmark suite:

```bash
alya run benches/bench_basic.alya
```

Run the example demo:

```bash
alya run examples/demo.alya
```

Check code formatting:

```bash
alya fmt . --check
```

Run static code linter:

```bash
alya lint . --check
```

---

### 💻 Developer Tooling & VS Code Integration

This package comes preconfigured with recommended workspace settings and tasks for **Visual Studio Code**:
- **LSP & Formatting**: Auto-formatting on save and real-time Language Server diagnostics via `alya-lang.vscode-alya`.
- **DAP Debugging**: Launch configurations in `.vscode/launch.json` ready for interactive step-debugging via `F5`.
- **Predefined Tasks**: Press `Ctrl+Shift+B` or run tasks (`Test`, `Lint`, `Format`, `Build Docs`) directly from the Command Palette.

---

## 🤝 Contributing

Contributions are welcome! Please follow these steps:

1. Fork the repository and clone it locally
2. Install dependencies:
   ```bash
   alya install
   ```
3. Create your feature branch (`git checkout -b feature/my-feature`)
4. Verify tests and formatting before opening a PR:
   ```bash
   alya test
   ```
5. Commit your changes (`git commit -m "feat: add feature"`) and open a Pull Request

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
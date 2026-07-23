# Rust — Axum sources and sinks

Request-handling entry points and dangerous sinks for web services built on
**Axum** (the `tokio` / `tower` / `hyper` stack). Sibling extractor-based
frameworks (**Actix Web**, **Rocket**) share the same extractor → handler →
response shape, so most rows transfer. For stdlib sinks that apply everywhere
(`Command`, `Path::join`, `unsafe`, weak crypto), see
[`../core/sources-and-sinks.md`](../core/sources-and-sinks.md).

## Sources — untrusted input entering an Axum handler

| api | kind | notes |
|-----|------|-------|
| `Query<T>` (`axum::extract`) | http-input | Query string deserialized via serde; typed but attacker-controlled. |
| `Path<T>` (`axum::extract`) | http-input | Path segments; a `Path<String>` can carry `..` or `/` — validate before use in file paths or queries. |
| `Json<T>` (`axum::extract`) | deserialization | Body deserialized via serde; attacker shapes it. Malformed input is rejected (400/422) with no code execution, but validate the decoded value and keep a body limit. |
| `Form<T>` (`axum::extract`) | http-input | URL-encoded body/query via serde; same trust level as `Query`. |
| `Multipart` (`axum::extract`) | file | File uploads; never use the client-supplied filename as a path. |
| `HeaderMap` / `TypedHeader<T>` | http-input | Request headers — `Host`, `Referer`, `X-Forwarded-*` are all forgeable. (`TypedHeader` lives in `axum-extra`.) |
| `CookieJar` (`axum-extra`) | http-input | Client-supplied; use `SignedCookieJar` / `PrivateCookieJar` for integrity. |
| `Bytes` / `String` | http-input | Whole request body buffered into memory; bounded by `DefaultBodyLimit` (see notes) — still validate and size-cap. |
| `RawQuery` / `OriginalUri` | http-input | Unparsed query / URI string; fully attacker-controlled. |
| `WebSocketUpgrade` messages (`extract::ws`) | network | Per-message payloads are untrusted; frame and bound them. |

## Sinks — where Axum-tainted data causes harm

| api | vuln | safe-alternative |
|-----|------|------------------|
| `Html(input)` (`axum::response`) | xss | `Html` sets `text/html` but does **not** escape. Render via an auto-escaping template (e.g. `askama`); never concatenate input into HTML. |
| `Redirect::to(input)` / `temporary` / `permanent` | open-redirect | Allow-list targets; reject absolute/external URLs built from input. |
| `Path<String>` joined to a root then read via `tokio::fs` | path-traversal | Serve static assets with `tower_http`'s `ServeDir` (contains to a root, rejects `..`); otherwise canonicalize and verify the result stays under the root. |
| `sqlx::query(&format!("... {input}"))` / `diesel::sql_query(input)` | sql-injection | Bind parameters: `sqlx::query("... $1").bind(x)` or the compile-time `sqlx::query!`; diesel's query builder. |
| `reqwest` / `hyper` client to an input-derived URL in a handler | ssrf | Allow-list destinations; block internal / link-local ranges (see core). |
| `Command::new("sh").arg("-c").arg(input)` in a handler | command-injection | No shell; args slice (see core). |

## Cross-cutting notes

- **Authorization**: Axum ships no auth — authz is a `tower` layer or an
  extractor (a `FromRequestParts` that checks the session). A state-changing
  route (`post` / `put` / `delete`) not behind an auth layer/extractor is the
  most common real bug — flag it.
- **Body limits / DoS**: `DefaultBodyLimit` (2 MiB default) bounds only the
  buffered extractors (`Bytes`, `String`, `Json`, `Form`); `disable()` removes
  it. Consuming the raw `Body` **stream** (`poll_frame`) bypasses it entirely —
  cap that with `tower_http`'s `RequestBodyLimitLayer` (global), and add request
  timeouts.
- **CSRF**: cookie-session, state-changing routes need CSRF defense — Axum has
  none built in; use a `tower` CSRF layer plus `SameSite` cookies.
- **Panics → availability**: a panicking handler (`.unwrap()`, indexing,
  `.expect()` on tainted input — see core) aborts that task; add `tower_http`'s
  `CatchPanicLayer` and prefer `?` / `.get(...)`.
- **Error leakage**: returning a `?`-propagated error (DB / `anyhow`) straight
  through `IntoResponse` can leak internals; map it to a generic message + status
  and log the detail server-side.
- **Host / forwarded headers**: building absolute URLs (e.g. password-reset
  links) from `Host` / `X-Forwarded-*` enables host-header injection and cache
  poisoning — use a configured base URL, not the request header.

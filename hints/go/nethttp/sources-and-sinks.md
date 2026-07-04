# Go — net/http (and routers on it) sources and sinks

Request-handling entry points and dangerous sinks for Go web servers built on
`net/http` (and routers like chi, gorilla/mux, gin that wrap it). For stdlib
sinks that apply everywhere, see [`../core/sources-and-sinks.md`](../core/sources-and-sinks.md).

## Sources — untrusted input entering a Go HTTP handler

| api | kind | notes |
|-----|------|-------|
| `r.URL.Query().Get("...")` | http-input | Query-string values; attacker-controlled. |
| `r.FormValue` / `r.PostFormValue` | http-input | Form fields; calls `ParseForm`. |
| `r.Header.Get("...")` | http-input | Includes `Host`, `Referer`, `X-Forwarded-*` — forgeable. |
| `r.Cookie("...")` | http-input | Client-supplied; verify integrity. |
| `mux.Vars(r)` / `chi.URLParam` | http-input | Path params; validate before use in paths/queries. |
| `r.Body` (`json.NewDecoder(r.Body)`) | http-input | Size-limit with `http.MaxBytesReader`; validate decoded shape. |
| `r.MultipartForm` / `r.FormFile` | file | Uploads; never use client filename as a path. |

## Sinks — where net/http-tainted data causes harm

| api | vuln | safe-alternative |
|-----|------|------------------|
| `fmt.Fprintf(w, input)` / writing input into HTML | xss | `html/template` with contextual escaping; never write raw input as HTML. |
| `http.Redirect(w, r, input, ...)` | open-redirect | Allow-list targets; reject absolute external URLs. |
| `http.ServeFile(w, r, input)` / serving input path | path-traversal | `http.FileServer` with `http.Dir` and a cleaned path; containment check. |
| `db.Query(fmt.Sprintf(... input))` | sql-injection | Placeholders with args (see core). |
| `http.Get(input)` / client to input URL | ssrf | Allow-list destinations; block internal/link-local ranges. |
| `exec.Command("sh","-c", input)` in a handler | command-injection | No shell; args slice (see core). |
| `text/template` served to browser | xss | `html/template`. |

## Cross-cutting notes

- **Authorization**: a handler mutating state without an auth/authz middleware is
  the most common real bug — flag routes missing it.
- **CSRF**: cookie-session state-changing routes without CSRF protection
  (e.g. `gorilla/csrf`).
- **Resource limits**: no `http.MaxBytesReader` on bodies, no `Server` read/write
  timeouts, no context deadline → DoS exposure.
- **Error leakage**: writing raw `err.Error()` to the response can leak internals;
  log server-side, return a generic message.

# JavaScript / Node.js — Express sources and sinks

Framework-specific entry points and dangerous sinks for Express request handling.
For runtime/stdlib sinks that apply everywhere in Node, see
[`../core/sources-and-sinks.md`](../core/sources-and-sinks.md).

## Sources — untrusted input entering an Express app

| api | kind | notes |
|-----|------|-------|
| `req.query` | http-input | Query-string values; **objects/arrays** possible (NoSQL-injection vector). |
| `req.body` | http-input | Parsed body; with `express.json()` an attacker controls the shape/types. |
| `req.params` | http-input | Route params; validate before use in paths/queries. |
| `req.headers` | http-input | Includes `host`, `referer`, `x-forwarded-*` — forgeable. |
| `req.cookies` (cookie-parser) | http-input | Client-supplied; verify integrity (signed cookies). |
| `req.files` (multer) | file | Uploads; never use client filename as a path. |

## Sinks — where Express-tainted data causes harm

| api | vuln | safe-alternative |
|-----|------|------------------|
| `db.query('... ' + input)` (mysql/pg) | sql-injection | Parameterized queries / placeholders (`$1`, `?`). |
| `Collection.find(req.body)` (Mongo) | insecure-deserialization | Cast/validate types; reject operator objects (`$where`, `$gt`) from input. |
| `res.send('<html>' + input)` | xss | Escape output; use a templating engine with auto-escaping. |
| `res.render(userTemplateName)` | template-injection | Fix template names; never derive from input. |
| `res.redirect(input)` | open-redirect | Allow-list target hosts; reject absolute external URLs. |
| `res.sendFile(path)` / `res.download(path)` with input | path-traversal | `path.resolve` under a root; verify containment; reject `..`. |
| `child_process.exec(..., input)` | command-injection | `execFile`/`spawn` with args array (see core). |
| `axios.get(input)` / `fetch(input)` | ssrf | Allow-list destinations; block internal ranges. |

## Cross-cutting notes

- **Missing auth middleware**: a route mutating state without an auth/authorization
  middleware is the most common real bug — flag it.
- **CSRF**: cookie-session state-changing routes without CSRF protection.
- **Security headers**: absence of `helmet()` (CSP, HSTS, nosniff) is worth noting.
- **Mass assignment / prototype pollution**: spreading `req.body` into a DB model
  or a merge lets attackers set unintended fields or pollute prototypes.
- **Rate limiting**: auth and expensive endpoints without a limiter enable abuse.

# JavaScript / Node.js — Core (language & runtime) sources and sinks

Framework-independent hints for any Node.js code. Server frameworks (Express,
Fastify, Koa, Next) add their own request sources on top of these. Applies to
TypeScript equally.

## Sources — untrusted input entering a Node process

| api | kind | notes |
|-----|------|-------|
| `process.argv` | cli-arg | Command-line arguments. |
| `process.env` | env | Trust depends on who sets the environment. |
| `process.stdin` | http-input | Piped or interactive input. |
| `fs.readFile` / `fs.readFileSync` | file | Content only as trusted as the writer. |
| `net.Socket` / stream data | network | Raw network input; frame and bound it. |
| `JSON.parse(data)` | deserialization | Shape attacker-controlled; validate post-parse (schema). |
| `require('node-serialize').unserialize` | deserialization | **RCE** on untrusted input — treat as a sink. |
| `fetch(...)` / `axios` response bodies | network | A called service may be attacker-run (SSRF pivots). |

## Sinks — dangerous runtime operations

| api | vuln | safe-alternative |
|-----|------|------------------|
| `eval(s)` / `new Function(s)` | code-injection | Never build code from input; parse explicitly. |
| `child_process.exec(cmd)` / `execSync` | command-injection | `execFile`/`spawn` with an args array, `shell:false`. |
| `child_process.spawn(cmd, {shell:true})` | command-injection | `shell:false` + args array. |
| `require(userInput)` | code-injection | Never resolve module paths from untrusted input. |
| `vm.runInThisContext` / `vm.runInNewContext` | code-injection | `vm` is not a security boundary; don't feed it input. |
| `fs.*(path)` with tainted `path` | path-traversal | `path.resolve` + verify under an intended root; reject `..`. |
| `new RegExp(userInput)` | dos | Bound/validate the pattern; catastrophic backtracking = ReDoS. |
| `Math.random()` for tokens/secrets | weak-crypto | `crypto.randomBytes` / `crypto.randomUUID`. |
| `crypto.createHash('md5'\|'sha1')` for passwords | weak-crypto | `crypto.scrypt`, bcrypt, or argon2 for passwords. |
| Prototype-pollution merges (`_.merge`, `Object.assign` of parsed input) | insecure-deserialization | Reject `__proto__`/`constructor` keys; use `Map` or vetted libs. |

## Notes

- **Secrets**: hardcoded API keys/tokens in source or committed `.env` files are a finding.
- **SSRF**: any `fetch`/`axios`/`http.request` to a URL derived from input needs a
  destination allow-list and blocking of internal/link-local ranges.
- **Dependency risk**: `npm install` of typosquatted or unpinned packages; audit
  `postinstall` scripts.

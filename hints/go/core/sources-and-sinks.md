# Go — Core (language & standard library) sources and sinks

Framework-independent hints for any Go code. Web handling (net/http and routers
on top of it) adds its own request sources — see
[`../nethttp/sources-and-sinks.md`](../nethttp/sources-and-sinks.md).

## Sources — untrusted input entering a Go process

| api | kind | notes |
|-----|------|-------|
| `os.Args` | cli-arg | Command-line arguments. |
| `os.Getenv` | env | Trust depends on who sets the environment. |
| `os.Stdin` / `bufio.Scanner` | http-input | Piped or interactive input. |
| `os.ReadFile` | file | Content only as trusted as the writer. |
| `net.Conn.Read` | network | Raw network input; frame and bound it. |
| `json.Unmarshal(data, &v)` | deserialization | Shape attacker-controlled; validate after decode. |
| `gob.Decode` / `encoding/xml` of input | deserialization | Validate; xml decoders can be abused (billion laughs). |

## Sinks — dangerous standard-library operations

| api | vuln | safe-alternative |
|-----|------|------------------|
| `exec.Command("sh", "-c", input)` | command-injection | `exec.Command(bin, args...)` with no shell; args are not word-split. |
| `db.Query(fmt.Sprintf("... %s", input))` | sql-injection | Placeholders (`?`/`$1`) with args: `db.Query(sql, arg)`. |
| `text/template` rendered to HTML | xss | Use `html/template` (contextual auto-escaping) for HTML output. |
| `filepath.Join(root, input)` | path-traversal | `filepath.Clean` + verify `strings.HasPrefix(resolved, root)`; reject `..`. |
| `os.OpenFile(input, ...)` | path-traversal | Same containment check as above. |
| `plugin.Open(input)` | code-injection | Never load plugins from untrusted paths. |
| `math/rand` for tokens/secrets | weak-crypto | `crypto/rand`. |
| `crypto/md5` / `crypto/sha1` for passwords | weak-crypto | `golang.org/x/crypto/bcrypt` or argon2 for passwords. |
| `unsafe.Pointer` conversions on input-derived sizes | memory-safety | Avoid `unsafe` with attacker-influenced lengths/offsets. |

## Notes

- **Secrets**: hardcoded credentials/keys in source or committed config.
- **SSRF**: any `net/http` client call to an input-derived URL needs a destination
  allow-list and internal-range blocking (see nethttp).
- **Error handling**: ignoring returned `error` values around security operations
  (auth, crypto, file ops) is worth flagging — a silent failure can be a bypass.

# Rust — Core (language & standard library) sources and sinks

Framework-independent hints for any Rust code. Web frameworks (Axum, Actix Web,
Rocket — all built on `hyper`/`tokio`) add their own request entry points; a
Rust framework file is not in this repo yet (see the [coverage index](../../README.md)).

## Sources — untrusted input entering a Rust process

| api | kind | notes |
|-----|------|-------|
| `std::env::args()` / `args_os()` | cli-arg | Command-line arguments. |
| `std::env::var(...)` | env | Trust depends on who sets the environment. |
| `std::io::stdin()` (`read_line` / `lock().lines()`) | http-input | Piped or interactive input. |
| `std::fs::read_to_string` / `std::fs::read` | file | Content only as trusted as the writer. |
| `TcpStream` / `UdpSocket` reads | network | Raw network input; frame and bound it. |
| `serde_json::from_str(input)` (serde) | deserialization | Shape attacker-controlled; validate after decode. Safe by default — no code execution, unlike pickle/Java — but enforce size and nesting-depth limits. |

## Sinks — dangerous standard-library operations

| api | vuln | safe-alternative |
|-----|------|------------------|
| `Command::new("sh").arg("-c").arg(input)` | command-injection | `Command::new(bin).args([...])` with no shell; args are not word-split. |
| `format!("... {input}")` built into SQL then run by any driver | sql-injection | Bind parameters (`?` / `$1`) via the driver's query API. |
| `Path::join(root, input)` where `input` is absolute or has `..` | path-traversal | Footgun: joining an absolute path *replaces* the base and `..` is not normalized. `canonicalize()` then verify the result stays under `root`; reject `..`. |
| `File::open(input)` / `std::fs` on input paths | path-traversal | Same containment check as above. |
| `libloading::Library::new(input)` (dynamic loading) | code-injection | Never load libraries/plugins from untrusted paths. |
| template engine rendering an untrusted *template string* (`tera` / `handlebars` / `minijinja`) | template-injection | Render fixed, precompiled templates; pass untrusted data only as *values*, never as template source. |
| `format!` / `write!` of untrusted data into HTML | xss | Emit through a contextual HTML-escaping layer; never build HTML by string concatenation. |
| `SmallRng`, or any PRNG seeded from a guessable value, for tokens/secrets | weak-crypto | Use a CSPRNG: `OsRng`, the `getrandom` crate, or `rand`'s already-secure `thread_rng()` / `StdRng`. (`SmallRng` is documented as *not* cryptographically secure.) |
| `md5` / `sha1` for passwords | weak-crypto | `argon2` or `bcrypt` for password hashing. |
| `unsafe` (`transmute`, `slice::from_raw_parts`) with input-derived sizes | memory-safety | Avoid `unsafe` with attacker-influenced lengths/offsets; validate bounds first. |
| integer arithmetic on tainted values (overflow **wraps silently in release**) | memory-safety | `checked_*` / `saturating_*` / `try_into()`; never feed a wrapped value into an allocation size or index. |

## Notes

- **Secrets**: hardcoded credentials/keys in source or committed config.
- **SSRF**: any HTTP client (`reqwest`, `hyper`, or raw `std::net`) call to an
  input-derived URL needs a destination allow-list and internal-range blocking.
- **Panics as availability risk**: `.unwrap()` / `.expect()`, slice indexing, and
  integer division on tainted input can panic and abort the thread — a DoS vector.
  Prefer `?`, `.get(...)`, and checked arithmetic on anything attacker-influenced.
- **`unsafe` review**: audit every `unsafe` block that touches input-derived data —
  it opts out of the compiler's memory guarantees, so the usual Rust assumptions
  no longer hold.

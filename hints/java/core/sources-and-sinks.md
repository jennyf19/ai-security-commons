# Java — Core (language & standard library) sources and sinks

Framework-independent hints for any JVM code (Java/Kotlin). Web frameworks
(Spring, Jakarta EE) add their own request sources on top of these.

## Sources — untrusted input entering a JVM process

| api | kind | notes |
|-----|------|-------|
| `String[] args` (`main`) | cli-arg | Command-line arguments. |
| `System.getenv` | env | Trust depends on who sets the environment. |
| `System.in` / `Scanner` | http-input | Piped or interactive input. |
| `Files.readAllBytes` / `FileReader` | file | Content only as trusted as the writer. |
| `Socket` / `InputStream` reads | network | Raw network input; frame and bound it. |
| `ObjectInputStream.readObject()` | deserialization | **RCE** on untrusted input — a source that is itself a critical sink. |
| `HttpClient` / `URLConnection` responses | network | A called service may be attacker-run (SSRF pivots). |

## Sinks — dangerous standard-library operations

| api | vuln | safe-alternative |
|-----|------|------------------|
| `Runtime.exec(cmd)` | command-injection | `ProcessBuilder` with an arg list; never a shell string. |
| `ObjectInputStream.readObject()` on input | insecure-deserialization | Avoid Java serialization; use JSON with a fixed schema; validation filters. |
| `Class.forName(name)` / reflection from input | code-injection | Never resolve class names from untrusted input. |
| `Statement.execute("... " + input)` | sql-injection | `PreparedStatement` with bound parameters. |
| `new File(path)` / `Files.*(path)` with input | path-traversal | Canonicalize (`toRealPath`); verify under an intended root; reject `..`. |
| `ScriptEngine.eval(input)` | code-injection | Never evaluate untrusted script. |
| `XMLInputFactory`/`DocumentBuilder` default | xxe | Disable DTDs/external entities (secure processing). |
| JNDI lookup from input (`InitialContext.lookup`) | code-injection | Never pass untrusted data to JNDI (Log4Shell class of bug). |
| `MessageDigest` MD5/SHA-1 for passwords | weak-crypto | PBKDF2, bcrypt, or Argon2 for passwords. |
| `java.util.Random` for tokens/secrets | weak-crypto | `SecureRandom`. |

## Notes

- **Secrets**: hardcoded credentials/keys in source or `application.properties`.
- **SSRF**: any `HttpClient`/`RestTemplate`/`URL` to an input-derived address needs
  a destination allow-list and internal-range blocking.
- **Logging**: passing untrusted input to a logging framework can be dangerous if
  message interpolation resolves lookups (Log4Shell) — keep libraries patched.

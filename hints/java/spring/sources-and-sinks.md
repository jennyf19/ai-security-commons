# Java — Spring (Boot / MVC / WebFlux) sources and sinks

Framework-specific entry points and dangerous sinks for Spring request handling.
For JVM/stdlib sinks that apply everywhere, see
[`../core/sources-and-sinks.md`](../core/sources-and-sinks.md).

## Sources — untrusted input entering a Spring app

| api | kind | notes |
|-----|------|-------|
| `@RequestParam` | http-input | Query/form parameters; attacker-controlled. |
| `@RequestBody` (bound DTO) | deserialization | Attacker-shaped; validation (`@Valid`) must actually constrain. |
| `@PathVariable` | http-input | Path segments; validate before use in paths/queries. |
| `@RequestHeader` / `@CookieValue` | http-input | Includes `Host`, `Referer` — forgeable. |
| `MultipartFile` | file | Uploads; never use client filename as a path. |
| `HttpServletRequest.getParameter` | http-input | Raw servlet access; same trust level. |

## Sinks — where Spring-tainted data causes harm

| api | vuln | safe-alternative |
|-----|------|------------------|
| `JdbcTemplate.query("... " + input)` | sql-injection | Parameter placeholders (`?`) with args. |
| `@Query("... " + concat)` / native query concat | sql-injection | Named/positional parameters (`:name`, `?1`). |
| `EntityManager.createQuery(concat)` | sql-injection | Parameterized JPQL/criteria API. |
| Thymeleaf `[(...)]` / `th:utext` with input | xss | `th:text` (escaped) by default; sanitize any raw HTML. |
| `"redirect:" + input` | open-redirect | Allow-list targets; avoid input in redirect URLs. |
| SpEL eval of input (`@Value`, `@PreAuthorize` built from input) | code-injection | Never build SpEL from untrusted data (SpEL injection). |
| `RestTemplate`/`WebClient` to input URL | ssrf | Allow-list destinations; block internal ranges. |
| `ResourceLoader.getResource(input)` / path serving | path-traversal | Canonicalize; verify under a root; reject `..`. |
| Endpoints deserializing input (`readObject`) | insecure-deserialization | See core; avoid Java serialization. |

## Cross-cutting notes

- **Authorization**: a controller mutating state without `@PreAuthorize`/method
  security (or an explicit permit) is the most common real bug.
- **CSRF**: `csrf().disable()` on a cookie-session app is a finding.
- **Actuator exposure**: `management.endpoints.web.exposure.include=*` leaks
  `/env`, `/heapdump`, etc. — flag it.
- **Mass assignment**: binding `@ModelAttribute`/`@RequestBody` straight to JPA
  entities lets attackers set unintended fields; use a DTO + explicit mapping;
  consider `@InitBinder` allow-lists.

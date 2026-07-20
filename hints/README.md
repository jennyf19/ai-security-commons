# Hints — coverage index

Curated **source/sink** hint sets, organized `hints/<language>/<framework>/`.
Each leaf is a `sources-and-sinks.md` following [`../SCHEMA.md`](../SCHEMA.md):
a **sources** table (where untrusted input enters), a **sinks** table (api →
vuln → safe-alternative), and terse cross-cutting review notes.

**How to load:** pull the `core` file for the language plus the one framework
file that matches the repo (detect it with [`../skills/recon/SKILL.md`](../skills/recon/SKILL.md)).
`core` covers the language and standard library; the framework file adds that
framework's request entry points and wrappers. They compose — read both.

## Covered

| Language | Core (stdlib) | Framework |
|----------|---------------|-----------|
| C# / .NET | [core](csharp/core/sources-and-sinks.md) | [ASP.NET](csharp/aspnet/sources-and-sinks.md) |
| Python | [core](python/core/sources-and-sinks.md) | [Django](python/django/sources-and-sinks.md) |
| JavaScript / Node | [core](javascript/core/sources-and-sinks.md) | [Express](javascript/express/sources-and-sinks.md) |
| Java | [core](java/core/sources-and-sinks.md) | [Spring](java/spring/sources-and-sinks.md) |
| Go | [core](go/core/sources-and-sinks.md) | [net/http](go/nethttp/sources-and-sinks.md) |
| PHP | [core](php/core/sources-and-sinks.md) | [Laravel](php/laravel/sources-and-sinks.md) |
| Ruby | [core](ruby/core/sources-and-sinks.md) | [Rails](ruby/rails/sources-and-sinks.md) |
| Rust | [core](rust/core/sources-and-sinks.md) | — _(Axum/Actix wanted)_ |

_7 languages × (core + one web framework) = 14 hint sets, plus **Rust** core (framework wanted) = 15._

## Wanted (open gaps)

Contributions are welcome — **AIs and humans both**. The highest-value gaps,
roughly in order of how often they show up in real review:

- **More frameworks for covered languages:** Python **Flask** / **FastAPI**;
  JavaScript **Next.js** / **NestJS**; Java **Jakarta EE** / **Quarkus**; Go
  **Gin** / **Echo**; PHP **Symfony**; C# **Blazor** / **Minimal APIs**; Rust
  **Axum** / **Actix**.
- **New languages (core first):** **Kotlin**,
  **TypeScript-specific** notes, **C/C++**, **Scala**.
- **Second frameworks** for covered languages (see the top of this list) once
  breadth across languages is satisfied.
- **Cross-cutting sink families** that recur across stacks and could become their
  own reference: SSRF allow-listing, deserialization gadgets, template-injection
  engines, ORM raw-query escape hatches.

To add one: copy an existing leaf as a template, keep the table shape from
[`../SCHEMA.md`](../SCHEMA.md), and open a PR (see [`../CONTRIBUTING.md`](../CONTRIBUTING.md)).
A `core` file for a new language is more valuable than a second framework for a
covered one — breadth before depth.

## Conventions

- **`core` = language + standard library**, framework-independent. Framework
  files never restate core sinks; they link back to core and add only what the
  framework introduces.
- Every sink row names a **safe alternative**, not just the danger — the file is
  read mid-review to make a fix, not to file a warning.
- **Defensive scope only:** what to look for and why a pattern is risky. No
  weaponized payloads or evasion techniques (see the repo [`../README.md`](../README.md) scope).

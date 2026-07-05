# Ruby — Rails sources and sinks

Framework-specific entry points and dangerous sinks for Ruby on Rails.
For language/stdlib sinks that apply everywhere (`eval`, `Marshal.load`,
`Kernel#open`, command exec), see [`../core/sources-and-sinks.md`](../core/sources-and-sinks.md).

## Sources — untrusted input entering a Rails app

| api | kind | notes |
|-----|------|-------|
| `params[:x]` / `params.require(:x)` | http-input | Query + body + route params, merged; attacker-controlled. |
| route params (`/users/:id`) | http-input | Validate before use in queries/paths. |
| `request.headers['...']` / `request.env['HTTP_*']` | http-input | Forgeable. |
| `cookies[:x]` / `session[:x]` | http-input | Client-held; signed/encrypted ≠ trusted contents until verified. |
| `request.body` / `request.raw_post` | http-input | Raw request body. |
| `params[:file]` (`ActionDispatch::Http::UploadedFile`) | file | Never use `original_filename` as a path. |

## Sinks — where Rails-tainted data causes harm

| api | vuln | safe-alternative |
|-----|------|------------------|
| `raw(x)` / `x.html_safe` / `<%== x %>` / `sanitize` misconfigured | xss | Default `<%= x %>` auto-escapes; keep HTML as escaped text. |
| `where("col = '#{x}'")` / `find_by_sql` / `exists?("...#{x}")` | sql-injection | Hash or placeholder: `where(col: x)` / `where("col = ?", x)`. |
| `order(params[:sort])` / `pluck`/`group`/`select` with interpolation | sql-injection | Allow-list column names; never interpolate into SQL fragments. |
| `Model.new(params[:m])` / `update(params[:m])` without strong params | mass-assignment | `params.require(:m).permit(:a, :b)`; never `permit!`/`to_unsafe_h`. |
| `redirect_to params[:url]` | open-redirect | Rails 7+ blocks cross-host by default; never set `allow_other_host: true` on input; allow-list / named routes. |
| `input.constantize` / `safe_constantize` / `classify.constantize` | unsafe-reflection / rce | Allow-list class names; never constantize input. |
| `send`/`public_send`(params[:m]) | unsafe-reflection | Dispatch table keyed by validated input. |
| `render inline: x` / `render text: x` / `render template: params[:t]` | template-injection | Fixed templates/actions; input is data, never template name/body. |
| `Marshal.load(cookie)` / marshaled session store | insecure-deserialization | JSON cookie/session serializer (`:json`, Rails default). |

## Cross-cutting notes

- **Strong parameters**: `params.permit!`, `params.to_unsafe_h`, or a stray
  `permit(:role)` defeats mass-assignment protection — grep for all three.
- **Authorization**: a controller action mutating state without a Pundit/CanCanCan
  `authorize`/policy or `authenticate_user!`/`before_action` is the most common
  real bug — an IDOR waiting to happen.
- **CSRF**: `skip_before_action :verify_authenticity_token` or a missing
  `protect_from_forgery` on a state-changing controller — flag each exemption.
- **Secrets**: committed `config/master.key` or the `RAILS_MASTER_KEY` — leaking
  it exposes `credentials.yml.enc` and forges signed/encrypted cookies
  (`secret_key_base`).
- **SQL entry points people forget**: `order`, `group`, `having`, `select`,
  `pluck`, `joins`, `lock`, `from` all take raw SQL fragments.
- **Config**: `config.force_ssl = true` in production; disable verbose errors;
  keep `config.active_record.default_timezone` and CSP/`config.action_dispatch`
  headers intentional.

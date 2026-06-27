# access_control — full reference

For each incoming request, Symfony checks every `access_control` entry and stops
at the **first** one that matches — only that entry is enforced. Each entry has
two kinds of options: matching (does this request match?) and enforcement (once
matched, what restriction applies?).

## 1. Matching options

- **`path`** — a regular expression (no delimiters), matched against the URI.
- **`ip` / `ips`** — IP addresses or netmasks; `ips` can be comma-separated or a
  list, handy with env vars (`'%env(TRUSTED_IPS)%'`).
- **`port`** — an integer.
- **`host`** — a regular expression.
- **`methods`** — one or more HTTP methods.
- **`route`** — a route name (shortcut for `attributes: { _route: xxx }`).
- **`attributes`** — request attributes that must match exactly.
- **`request_matcher`** — a service implementing `RequestMatcherInterface`.

If `ip`, `port`, `host`, or `methods` are omitted, the entry matches any value
for that dimension. URI matching ignores `$_GET` query parameters — enforce those
in PHP code.

```yaml
# config/packages/security.yaml
parameters:
    env(TRUSTED_IPS): '10.0.0.1, 10.0.0.2'

security:
    access_control:
        - { path: '^/admin', roles: ROLE_USER_PORT, ip: 127.0.0.1, port: 8080 }
        - { path: '^/admin', roles: ROLE_USER_IP,   ip: 127.0.0.1 }
        - { path: '^/admin', roles: ROLE_USER_HOST, host: 'symfony\.com$' }
        - { path: '^/admin', roles: ROLE_USER_METHOD, methods: [POST, PUT] }
        - { path: '^/admin', roles: ROLE_USER_IP, ips: '%env(TRUSTED_IPS)%' }
        - { roles: ROLE_USER, request_matcher: App\Security\RequestMatcher\MyRequestMatcher }
        - { route: 'admin', roles: ROLE_ADMIN }
```

### First-match behavior (examples)

Given the entries above, for URI `/admin/user`:

- IP `127.0.0.1`, port `80`, host `example.com`, GET → rule #2 (`ROLE_USER_IP`):
  path + ip match.
- IP `127.0.0.1`, port `80`, host `symfony.com`, GET → still rule #2; it would
  also match `ROLE_USER_HOST`, but only the **first** match is used.
- IP `127.0.0.1`, port `8080`, host `symfony.com`, GET → rule #1
  (`ROLE_USER_PORT`): path + ip + port match.
- IP `168.0.0.1`, port `80`, host `symfony.com`, GET → rule #3 (`ROLE_USER_HOST`):
  ip didn't match #1/#2.
- IP `168.0.0.1`, host `example.com`, POST → rule #4 (`ROLE_USER_METHOD`).
- URI `/foo` → matches no entry (no `path` matches).

## 2. Enforcement options

Once an entry matches, access is enforced via:

- **`roles`** — deny (throw `AccessDeniedException`) if the user lacks the role.
  Internally, `roles` are passed as the voter `$attributes` with the `Request` as
  the `$subject`, so you can write custom voters for them.
- **`allow_if`** — an expression; if it returns false, access is denied.
- **`requires_channel`** — if the request's channel (e.g. `http`) differs from
  the value (e.g. `https`), the user is redirected.

If both `roles` and `allow_if` are set and the strategy is the default
`affirmative`, access is granted if **either** passes (OR). Change the
[decision strategy](voters.md#access-decision-strategies) if that doesn't fit.

If access is denied, Symfony tries to authenticate the user (e.g. redirect to
login); if already logged in, it shows the 403 page.

## Restricting by IP

The `ips` option does **not** restrict to an IP — it makes the entry *only match*
that IP; other IPs fall through to later entries. Combine with a deny-all
fallback:

```yaml
security:
    access_control:
        # only matches local requests → granted (everyone has PUBLIC_ACCESS)
        - { path: '^/internal', roles: PUBLIC_ACCESS, ips: [127.0.0.1, ::1, 192.168.0.1/24] }
        # any other IP falls through here; no user has ROLE_NO_ACCESS → always denied
        - { path: '^/internal', roles: ROLE_NO_ACCESS }
```

`ROLE_NO_ACCESS` is just a role no user holds — the idiom for "deny outright".

## Securing by an expression (allow_if)

```yaml
security:
    access_control:
        -
            path: ^/_internal/secure
            # roles + allow_if behave like OR: granted if expression is TRUE or user has ROLE_ADMIN
            roles: 'ROLE_ADMIN'
            allow_if: "'127.0.0.1' == request.getClientIp() or request.headers.has('X-Secure-Access')"
```

`allow_if` triggers the built-in `ExpressionVoter` as if it were part of the
`roles` attributes. Inside the expression you have `request` (the `Request`
object) plus the same variables and functions as
[security expressions](expressions.md). It can also use custom functions
registered via expression providers.

## Restrict to a port

```yaml
security:
    access_control:
        - { path: ^/cart/checkout, roles: PUBLIC_ACCESS, port: 8080 }
```

## Forcing a channel (http/https)

```yaml
security:
    access_control:
        - { path: ^/cart/checkout, roles: PUBLIC_ACCESS, requires_channel: https }
```

If the matched request uses `http`, the user is redirected to `https`.

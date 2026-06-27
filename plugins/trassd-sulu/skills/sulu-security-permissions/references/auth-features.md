# Optional authentication features

## Single-Sign-On (OpenID only)

Adjust `config/packages/security.yaml` to allow SSO on the admin firewall and
configure the provider. Create a role with the `default_role_key` first.

```yaml
security:
    firewalls:
        admin:
            logout:
                path: sulu_admin.logout
            access_token:
                token_handler: sulu_security.single_sign_on_token_handler
                token_extractors: sulu_security.single_sign_on_token_extractor

sulu_security:
    checker:
        enabled: true
    password_policy:
        enabled: true
    single_sign_on:
        providers:
            'sulu.io':
                dsn: 'openid://%env(resolve:SULU_OPEN_ID_CLIENT_ID)%:%env(resolve:SULU_OPEN_ID_CLIENT_SECRET)%@%env(resolve:SULU_OPEN_ID_ENDPOINT)%'
                default_role_key: 'USER'
```

After clearing the cache, the admin login shows only the username/email field.
If the email domain matches the configured provider domain the user is
redirected to the SSO provider (also on password reset); otherwise the standard
login form is used. Provide your admin URL (e.g. `sulu.io/admin`) as the redirect
URL if the provider requires one. Only OpenID is supported.

## Two-factor authentication (email)

Install the `scheb/2fa` packages:

```bash
composer require scheb/2fa-bundle scheb/2fa-email scheb/2fa-trusted-device
```

Wire the admin firewall in `config/packages/security.yaml`:

```yaml
security:
    access_control:
        - { path: ^/admin/login$, roles: PUBLIC_ACCESS }
        - { path: ^/admin/2fa, role: PUBLIC_ACCESS }
    firewalls:
        admin:
            logout:
                path: sulu_admin.logout
            two_factor:
                prepare_on_login: true
                prepare_on_access_denied: true
                check_path: 2fa_login_check_admin
                authentication_required_handler: sulu_security.two_factor_authentication_required_handler
                success_handler: sulu_security.two_factor_authentication_success_handler
                failure_handler: sulu_security.two_factor_authentication_failure_handler
```

Configure scheb/2fa in `config/packages/scheb_2fa.yaml`:

```yaml
scheb_two_factor:
    email:
        enabled: true
        sender_email: "%env(SULU_ADMIN_EMAIL)%"
    trusted_device:
        enabled: true
```

Add the check route in `config/routes/scheb_2fa.yaml`:

```yaml
2fa_login_check_admin:
    path: /admin/2fa_check
```

After clearing the cache, users enable 2FA from their admin profile.

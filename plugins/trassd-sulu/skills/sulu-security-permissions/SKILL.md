---
name: sulu-security-permissions
description: >-
  Apply Sulu's security model — roles, permissions, security contexts, and the
  password policy. Use when securing admin resources, defining security contexts
  for a custom bundle, protecting controllers or specific objects, configuring a
  custom security system, or enforcing a password policy in a Sulu app.
---

# Sulu Security & Permissions

The `SuluSecurityBundle` builds on Symfony's security to protect areas and data
of the application. It offers two complementary mechanisms:

1. **Security contexts** — restrict access to entire parts of the application;
   permissions are managed per **role**.
2. **Per-object permissions** — restrict access to specific objects, set on the
   object itself.

In both cases the role-to-user assignment also carries the **locale(s)** for
which the permissions are valid; the user must have the matching locale to gain
access.

## Core model

- Every Sulu **user** is linked to a `SuluContactBundle` contact and can have
  **roles** and **groups** (groups may nest other groups and roles).
- Every **role** belongs to a **security system**. A user can only log in to a
  system if it has at least one role in that system. The default system is
  `Sulu` (the admin interface), exposed as the constant
  `Admin::SULU_ADMIN_SECURITY_SYSTEM` (`'Sulu'`).
- User flags: `locked` (admin-locked, blocks login) and `enabled` (defaults to
  true; used by custom double-opt-in registration flows).

### Permission types

A security context grants any subset of these permissions (stored as a bitmask
on a permission object linked to a role). Use the constants on
`Sulu\Component\Security\Authorization\PermissionTypes`:

| Constant  | Value      | Meaning                                |
|-----------|------------|----------------------------------------|
| `VIEW`    | `view`     | See data in the context                |
| `ADD`     | `add`      | Add new data                           |
| `EDIT`    | `edit`     | Edit existing data                     |
| `DELETE` | `delete`   | Delete data                            |
| `ARCHIVE` | `archive`  | Archive data                           |
| `LIVE`    | `live`     | Publish data                           |
| `SECURITY`| `security` | Grant/deny access on data              |

## Declaring security contexts

Define contexts in your bundle's `Admin` class via `getSecurityContexts()`. The
returned array is nested **system → group title → context name → [permission
types]**. The context name is a dot-namespaced string (e.g. `sulu.acme.example`)
and becomes selectable in the admin permission UI.

```php
public function getSecurityContexts(): array
{
    return [
        self::SULU_ADMIN_SECURITY_SYSTEM => [
            'Acme' => [
                'sulu.acme.example' => [
                    PermissionTypes::VIEW,
                    PermissionTypes::ADD,
                    PermissionTypes::EDIT,
                    PermissionTypes::DELETE,
                ],
            ],
        ],
    ];
}
```

Because the `Admin` class is a service, you may inject services to build
contexts dynamically (the page bundle does this per webspace).

## Protecting a controller (context-level)

Implement `Sulu\Component\Security\SecuredControllerInterface` with
`getSecurityContext()` and `getLocale(Request)`. The `SuluSecurityListener`
infers the required permission type from the action, runs the check, and returns
**403** on insufficient permissions — no manual check needed.

See [references/controller-security.md](references/controller-security.md).

## Protecting specific objects

For object-level permissions:

1. Add the reusable **permission tab** to the edit form in your `Admin`
   (`createFormViewBuilder(...)->setFormKey('permission_details')`,
   `setTabCondition('_permissions.security')`, request param `resourceKey`).
2. Map the resource to its context and class in the `resources` config
   (`security_context`, `security_class`).
3. Implement
   `Sulu\Component\Security\Authorization\AccessControl\SecuredObjectControllerInterface`
   (`getSecuredClass()`, `getSecuredObjectId(Request)`) alongside
   `SecuredControllerInterface`.
4. In list actions, filter rows with
   `$listBuilder->setPermissionCheck($user, PermissionTypes::VIEW)`.

Full example in [references/object-security.md](references/object-security.md).

### How object permissions are stored

The `AccessControlManager` persists per-object permissions through registered
`AccessControlProviderInterface` services (tagged `sulu.access_control`). The
`DoctrineAccessControlProvider` works with any Doctrine entity that implements
`SecuredEntityInterface`, storing permissions in the DB so lists can be paginated
with permission filtering. The manager backs the `PermissionController`
(permission tab) and the `SecurityContextVoter`.

## Checking permissions programmatically

Use `Sulu\Component\Security\Authorization\SecurityCheckerInterface` with a
`SecurityCondition` (security context, plus optional object type + id and a
locale). It delegates to Symfony's `AccessDecisionManager`, which calls the
`SecurityContextVoter`. When an object type and id are supplied, the role's
context permissions may be overridden by that object's specific permissions.

```php
$this->securityChecker->checkPermission(
    new SecurityCondition('sulu.acme.example', $locale),
    PermissionTypes::EDIT
);
```

## Security systems

Each role is bound to a system; the request is assigned a system and the
firewall only logs in users with a role in that system.

- **Webspace system** — set `<security><system>Website</system></security>` in
  the webspace config. Add `permission-check="true"` to restrict pages/media/
  entities to logged-in users with a role; enable user-context caching to avoid
  cache leaks of restricted content.
- **Custom system** — declare `public const SYSTEM` and return
  `[self::SYSTEM => []]` from `getSecurityContexts()` in an `Admin` class (for
  intranet/extranet not tied to a webspace), then register a request listener
  that sets the system via the `SystemStore`
  (`Sulu\Bundle\SecurityBundle\System\SystemStoreInterface`, service id
  `sulu_security.system_store`) for the matching firewall. See
  [references/security-systems.md](references/security-systems.md).

## Password policy

Configure in `config/packages/sulu_security.yaml`. Enabling alone applies
Sulu's default (minimum 8 characters). The policy validates both admin-form
input and users created programmatically via the `UserManager`. Provide an
`info_translation_key` so the UI can explain the rules.

```yaml
sulu_security:
    password_policy:
        enabled: true
        pattern: '(?=^.{8,}$)(?=.*\d)(?=.*[^a-zA-Z0-9]+)(?![.\n])(?=.*[A-Z])(?=.*[a-z]).*$'
        info_translation_key: app.password_information
```

Add the translation (e.g. in a JSON translation file) for that key describing
the constraints to the user.

## Optional authentication features

- **Single-Sign-On** (OpenID only) — add `access_token` with
  `sulu_security.single_sign_on_token_handler` /
  `..._token_extractor` to the admin firewall and configure
  `sulu_security.single_sign_on.providers`; create a role with the
  `default_role_key` first.
- **Two-factor (email)** via `scheb/2fa` — install the packages, wire the
  `two_factor` firewall handlers (`sulu_security.two_factor_authentication_*`),
  the `^/admin/2fa` access-control rule and the 2FA check route; users enable it
  from their admin profile.

Configuration snippets in
[references/auth-features.md](references/auth-features.md).

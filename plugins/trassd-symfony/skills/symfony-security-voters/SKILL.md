---
name: symfony-security-voters
description: Implement authorization with Symfony security voters and access control — voters, expressions, and access_control rules. Triggers when adding authorization checks, creating a Voter, or restricting access to resources/actions.
---

# Symfony Security Voters & Access Control

Symfony decides "may this user do this?" through the **authorization checker**.
Every call to `isGranted()`, `denyAccessUnlessGranted()`, the `#[IsGranted]`
attribute, and every `access_control` rule funnels through the same **voter
system**: all registered voters are asked, and the access decision manager
combines their votes into a final allow/deny.

Pick the right tool for the check you are adding:

- **`access_control` (security.yaml)** — coarse, URL-pattern firewalling. Use it
  to lock whole path prefixes (`^/admin`) behind a role, an IP, a port, a host,
  HTTP methods, or to force a channel (`https`). It cannot see your domain
  objects.
- **Voters** — fine-grained, **object-aware** authorization ("can *this* user
  edit *this* Post?"). This is Symfony's most powerful and the recommended way to
  manage permissions. Centralize permission logic in a voter, then reuse it
  everywhere via `isGranted('edit', $post)`.
- **`#[IsGranted]` / `denyAccessUnlessGranted()`** — how a controller invokes the
  voter system (with or without a subject).
- **Expressions / closures** — inline logic for simple combined conditions when a
  full voter would be overkill.

**Best practice:** reach for a voter whenever the decision depends on the object
being acted on or is reused in more than one place. Trivial, one-off owner checks
can live directly in the controller, but anything richer belongs in a voter.

## Checking access in a controller

Use the `#[IsGranted]` attribute (preferred) or the equivalent method calls. The
first argument is the **attribute** (a role like `ROLE_ADMIN`, or an arbitrary
permission string like `edit`); the optional second argument is the **subject**
passed to the voters.

```php
use Symfony\Component\Security\Http\Attribute\IsGranted;

#[Route('/posts/{id}/edit')]
#[IsGranted('edit', 'post')] // 'post' = the controller arg name passed as subject
public function edit(Post $post): Response { /* ... */ }
```

Equivalent imperative calls inside an action:

```php
$this->denyAccessUnlessGranted('edit', $post);   // throws AccessDeniedException (403) if denied
if ($this->isGranted('edit', $post)) { /* ... */ } // returns bool, throws nothing
```

`#[IsGranted]` throws an `AccessDeniedException` → HTTP **403** with "Access
Denied" by default. Override the message and status code when needed
(`#[IsGranted('show', 'post', 'Post not found', 404)]`); a non-403 status throws
an `HttpException` instead. Outside controllers, inject
`AuthorizationCheckerInterface` and call its `isGranted()`.

## Writing a Voter

Extend the abstract `Voter` class and implement two methods:

```php
namespace App\Security;

use App\Entity\Post;
use App\Entity\User;
use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
use Symfony\Component\Security\Core\Authorization\Voter\Vote;
use Symfony\Component\Security\Core\Authorization\Voter\Voter;

class PostVoter extends Voter
{
    const VIEW = 'view';
    const EDIT = 'edit';

    protected function supports(string $attribute, mixed $subject): bool
    {
        return in_array($attribute, [self::VIEW, self::EDIT], true)
            && $subject instanceof Post;
    }

    protected function voteOnAttribute(string $attribute, mixed $subject, TokenInterface $token, ?Vote $vote = null): bool
    {
        $user = $token->getUser();
        if (!$user instanceof User) {
            return false; // not logged in → deny
        }

        /** @var Post $subject */
        return match ($attribute) {
            self::VIEW => !$subject->isPrivate() || $user === $subject->getAuthor(),
            self::EDIT => $user === $subject->getAuthor(),
            default => throw new \LogicException('Unreachable'),
        };
    }
}
```

- **`supports()`** — return `true` only for the attribute/subject combinations
  this voter handles; otherwise the voter abstains and others decide. Only when
  it returns `true` is `voteOnAttribute()` called.
- **`voteOnAttribute()`** — return `true` to allow, `false` to deny. Get the
  current user from `$token->getUser()`. The optional `$vote` (Symfony 7.3+) lets
  you attach a reason (`$vote?->addReason(...)`) shown in logs and exception
  pages.

**Registration is automatic** — with the default `services.yaml` autoconfigure
setup, the voter is tagged `security.voter` for you. No manual wiring needed.

To check a role *inside* a voter (e.g. always allow `ROLE_SUPER_ADMIN`), inject
`AccessDecisionManagerInterface` and call `decide()`. **Do not** call
`Security::isGranted()` inside a voter — it may run against a different token.

See [references/voters.md](references/voters.md) for the full voter with reasons,
the super-admin check, performance caching (`CacheableVoterInterface`,
`supportsAttribute()` / `supportsType()`), and the four decision strategies
(`affirmative`, `consensus`, `unanimous`, `priority`).

## Security expressions

`isGranted()` and `#[IsGranted]` also accept an `Expression`, useful for simple
combined conditions without a dedicated voter:

```php
use Symfony\Component\ExpressionLanguage\Expression;

#[IsGranted(new Expression('is_granted("ROLE_ADMIN") or is_granted("ROLE_MANAGER")'))]
public function show(): Response { /* ... */ }
```

Available variables: `user`, `role_names`, `object`/`subject`, `token`,
`trust_resolver`. Functions: `is_granted()`, `is_authenticated()`,
`is_remember_me()`, `is_fully_authenticated()`. The `#[IsGranted]` subject can
itself be an expression (e.g. `subject: new Expression('args["post"].getAuthor()')`)
or, in Symfony 7.3+, a closure. For non-trivial rules, prefer a voter over a
complex expression. See [references/expressions.md](references/expressions.md).

## access_control path rules

Declared in `config/packages/security.yaml`. For each request Symfony checks
entries top-to-bottom and **only the first matching entry is enforced** — order
matters.

```yaml
security:
    access_control:
        - { path: '^/admin/login', roles: PUBLIC_ACCESS }
        - { path: '^/admin', roles: ROLE_ADMIN }
        - { path: '^/profile', roles: IS_AUTHENTICATED_FULLY }
        - { path: '^/cart/checkout', roles: PUBLIC_ACCESS, requires_channel: https }
```

- **Matching** options: `path` (regex, no delimiters), `ip`/`ips` (netmasks ok),
  `port`, `host` (regex), `methods`, `route`, `attributes`, `request_matcher`.
  Unspecified matching options match anything.
- **Enforcement** options: `roles` (deny if the user lacks the role), `allow_if`
  (an expression — deny if it returns false), `requires_channel` (redirect to the
  given channel, e.g. `https`).
- `roles` and `allow_if` combine like **OR** under the default `affirmative`
  strategy (access granted if either passes).
- Use `PUBLIC_ACCESS` to open a path; a fictitious role like `ROLE_NO_ACCESS`
  (that no user has) is the idiom to deny a path outright.
- Matching ignores `$_GET` query parameters — enforce those in PHP.

Note: `access_control` cannot inspect domain objects. For object-level decisions,
use a voter. See [references/access-control.md](references/access-control.md) for
the IP-restriction pattern, `allow_if` expressions, and first-match examples.

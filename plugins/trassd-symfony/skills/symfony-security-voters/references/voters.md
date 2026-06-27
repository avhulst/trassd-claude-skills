# Voters — full reference

## Complete voter with vote reasons

The `Voter` abstract class implements `VoterInterface`. You implement
`supports()` and `voteOnAttribute()`. The optional `?Vote $vote` argument
(Symfony 7.3+) lets you record an explanation that appears in logs and on
exception pages.

```php
// src/Security/PostVoter.php
namespace App\Security;

use App\Entity\Post;
use App\Entity\User;
use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
use Symfony\Component\Security\Core\Authorization\Voter\Vote;
use Symfony\Component\Security\Core\Authorization\Voter\Voter;

class PostVoter extends Voter
{
    // arbitrary strings; you can use anything
    const VIEW = 'view';
    const EDIT = 'edit';

    protected function supports(string $attribute, mixed $subject): bool
    {
        if (!in_array($attribute, [self::VIEW, self::EDIT])) {
            return false;
        }

        // only vote on Post objects
        if (!$subject instanceof Post) {
            return false;
        }

        return true;
    }

    protected function voteOnAttribute(string $attribute, mixed $subject, TokenInterface $token, ?Vote $vote = null): bool
    {
        $user = $token->getUser();

        if (!$user instanceof User) {
            // the user must be logged in; if not, deny access
            $vote?->addReason('The user is not logged in.');
            return false;
        }

        // $subject is a Post, guaranteed by supports()
        /** @var Post $post */
        $post = $subject;

        return match ($attribute) {
            self::VIEW => $this->canView($post, $user),
            self::EDIT => $this->canEdit($post, $user, $vote),
            default => throw new \LogicException('This code should not be reached!'),
        };
    }

    private function canView(Post $post, User $user): bool
    {
        // if they can edit, they can view
        if ($this->canEdit($post, $user, null)) {
            return true;
        }

        return !$post->isPrivate();
    }

    private function canEdit(Post $post, User $user, ?Vote $vote): bool
    {
        if ($user === $post->getAuthor()) {
            return true;
        }

        $vote?->addReason(sprintf(
            'The logged in user (username: %s) is not the author of this post (id: %d).',
            $user->getUsername(), $post->getId()
        ));

        return false;
    }
}
```

What the two abstract methods must do:

- **`supports(string $attribute, mixed $subject)`** — `$attribute` is the first
  argument passed to `isGranted()`/`denyAccessUnlessGranted()` (e.g. `ROLE_USER`,
  `edit`); `$subject` is the second (e.g. `null`, a `Post`). Return `true` if this
  voter should vote on the combination — then `voteOnAttribute()` is called.
  Returning `false` means "abstain"; other voters handle it.
- **`voteOnAttribute(string $attribute, mixed $subject, TokenInterface $token, ?Vote $vote = null)`** —
  return `true` to allow, `false` to deny. `$token` gives the current user;
  `$vote` records the explanation.

Votes also expose an `$extraData` array (Symfony 7.4+) for storing arbitrary
data for use by a custom decision strategy: `$vote->extraData['score'] = 10;`.

## Registration

With the default `services.yaml` (autoconfigure on), extending `Voter`
auto-tags the service with `security.voter` — nothing else is needed. Calling
`isGranted('edit', $post)` will invoke this voter.

## Checking roles inside a voter

To always grant a super-admin, inject the **access decision manager** and call
`decide()`:

```php
use Symfony\Component\Security\Core\Authorization\AccessDecisionManagerInterface;

class PostVoter extends Voter
{
    public function __construct(
        private AccessDecisionManagerInterface $accessDecisionManager,
    ) {}

    protected function voteOnAttribute($attribute, mixed $subject, TokenInterface $token, ?Vote $vote = null): bool
    {
        // ROLE_SUPER_ADMIN can do anything
        if ($this->accessDecisionManager->decide($token, ['ROLE_SUPER_ADMIN'])) {
            return true;
        }

        // ... normal voter logic
    }
}
```

**Do not** use `Security::isGranted('ROLE_SUPER_ADMIN')` inside a voter: it does
not guarantee the check runs on the same token as your voter (the token storage
may have changed). Always use `AccessDecisionManager`.

## Improving performance (CacheableVoterInterface)

If you have many voters and many checks per request, let Symfony cache whether a
voter applies. The abstract `Voter` already implements
`CacheableVoterInterface`; override one or both methods:

```php
public function supportsAttribute(string $attribute): bool
{
    return in_array($attribute, [self::VIEW, self::EDIT], true);
}

public function supportsType(string $subjectType): bool
{
    // not a strict === comparison: subject type may be a Doctrine proxy class
    return is_a($subjectType, Post::class, true);
}
```

If `supportsAttribute()` / `supportsType()` returns `false`, Symfony will not
call the voter again for that attribute / object type.

## Access decision strategies

Multiple voters may vote on the same action/object. The access decision manager
combines votes using a configurable strategy:

- **`affirmative`** (default) — grant as soon as *one* voter grants.
- **`consensus`** — grant if more voters grant than deny; ties use
  `allow_if_equal_granted_denied` (default `true`).
- **`unanimous`** — grant only if *no* voter denies.
- **`priority`** — the first non-abstaining voter (by service priority) decides.

If all voters abstain, the result follows `allow_if_all_abstain` (default
`false`).

```yaml
# config/packages/security.yaml
security:
    access_decision_manager:
        strategy: unanimous
        allow_if_all_abstain: false
```

For needs none of these cover, set `strategy_service` to a service implementing
`AccessDecisionStrategyInterface`, or replace the whole manager with `service`
(implementing `AccessDecisionManagerInterface`). A custom strategy can read
`$vote->extraData` (e.g. a per-vote `score`) when deciding.

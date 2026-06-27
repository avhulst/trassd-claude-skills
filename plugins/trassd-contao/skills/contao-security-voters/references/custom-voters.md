# Custom voters (Symfony base `Voter`)

When a rule doesn't map cleanly to a single DCA table — it concerns a back end
module, or spans a parent and child table — extend Symfony's base
`Voter` directly. The two examples below come straight from the Contao security
docs.

Remember the priority rule: in the `frontend`/`backend` scope the **first
non-abstaining voter decides**. Always abstain (`supports()` returns `false`,
or `Voter::ACCESS_ABSTAIN`) on anything you don't own, and raise the
`security.voter` service priority if you must override a core decision.

## Example 1 — restrict a back end module to one admin

Grants the "Maintenance" back end section only to the admin with ID 1.

```php
// src/Security/Voter/AdminMaintenanceAccessVoter.php
namespace App\Security\Voter;

use Contao\BackendUser;
use Contao\CoreBundle\Security\ContaoCorePermissions;
use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
use Symfony\Component\Security\Core\Authorization\Voter\Voter;

class AdminMaintenanceAccessVoter extends Voter
{
    public function vote(TokenInterface $token, mixed $subject, array $attributes): int
    {
        // Abstain if not a back end admin
        if (!($user = $token->getUser()) instanceof BackendUser || !$user->isAdmin) {
            return Voter::ACCESS_ABSTAIN;
        }

        return parent::vote($token, $subject, $attributes);
    }

    protected function supports(string $attribute, $subject): bool
    {
        // Only vote on maintenance back end module access
        return 'maintenance' === $subject
            && ContaoCorePermissions::USER_CAN_ACCESS_MODULE === $attribute;
    }

    protected function voteOnAttribute(string $attribute, mixed $subject, TokenInterface $token): bool
    {
        return 1 === (int) $token->getUser()->id;
    }
}
```

## Example 2 — restrict editing to the record's author, across parent + child

Limits update/delete of news records to their original author, and applies the
same rule to the child `tl_content` table so the news content can't be edited
around the restriction.

```php
// src/Security/Voter/NewsAccessVoter.php
namespace App\Security\Voter;

use Contao\BackendUser;
use Contao\CoreBundle\Security\ContaoCorePermissions;
use Contao\CoreBundle\Security\DataContainer\DeleteAction;
use Contao\CoreBundle\Security\DataContainer\UpdateAction;
use Contao\NewsModel;
use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
use Symfony\Component\Security\Core\Authorization\Voter\Voter;

class NewsAccessVoter extends Voter
{
    protected function supports(string $attribute, $subject): bool
    {
        // Only edit actions
        if (!$subject instanceof DeleteAction && !$subject instanceof UpdateAction) {
            return false;
        }

        if (ContaoCorePermissions::DC_PREFIX.'tl_news' === $attribute) {
            return true;
        }

        // Also content elements that belong to news
        if (ContaoCorePermissions::DC_PREFIX.'tl_content' === $attribute) {
            return 'tl_news' === $subject->getCurrent()['ptable'];
        }

        return false;
    }

    /** @param DeleteAction|UpdateAction $subject */
    protected function voteOnAttribute(string $attribute, mixed $subject, TokenInterface $token): bool
    {
        /** @var BackendUser $user */
        $user = $token->getUser();

        if ($user->isAdmin) {
            return true;
        }

        $record = $subject->getCurrent();

        if ('tl_news' === $subject->getDataSource()) {
            $authorId = $record['author'];
        } else {
            $authorId = NewsModel::findById($record['pid'])->author;
        }

        return (int) $user->id === (int) $authorId;
    }
}
```

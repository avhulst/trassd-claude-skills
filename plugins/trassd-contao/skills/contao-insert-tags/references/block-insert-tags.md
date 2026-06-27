# Block insert tag example

A block insert tag wraps content between an opening tag and its `endTag`. The
resolver receives the resolved `InsertTag` plus the wrapped content as a
`ParsedSequence`, and returns a `ParsedSequence` (return an empty one to emit
nothing).

This example implements `{{ifmembergroup::*}}…{{endifmembergroup}}`, which
outputs the wrapped content only if the logged-in member belongs to one of the
given groups.

```php
namespace App\InsertTag;

use Contao\CoreBundle\DependencyInjection\Attribute\AsBlockInsertTag;
use Contao\CoreBundle\InsertTag\Exception\InvalidInsertTagException;
use Contao\CoreBundle\InsertTag\ParsedSequence;
use Contao\CoreBundle\InsertTag\ResolvedInsertTag;
use Contao\CoreBundle\InsertTag\Resolver\BlockInsertTagResolverNestedResolvedInterface;
use Contao\CoreBundle\Security\ContaoCorePermissions;
use Symfony\Component\Security\Core\Authorization\AuthorizationCheckerInterface;

#[AsBlockInsertTag('ifmembergroup', endTag: 'endifmembergroup')]
class IfMemberGroupInsertTag implements BlockInsertTagResolverNestedResolvedInterface
{
    public function __construct(
        private readonly AuthorizationCheckerInterface $auth,
    ) {
    }

    public function __invoke(ResolvedInsertTag $insertTag, ParsedSequence $wrappedContent): ParsedSequence
    {
        if (!$groups = $insertTag->getParameters()->all()) {
            throw new InvalidInsertTagException('Missing parameters for insert tag.');
        }

        if ($this->auth->isGranted(ContaoCorePermissions::MEMBER_IN_GROUPS, $groups)) {
            return $wrappedContent;
        }

        return new ParsedSequence([]);
    }
}
```

Notes:

- `endTag` must be lowercase, like `name`.
- A core reference implementation is `IfLanguageInsertTag`
  (`{{iflng::de}}…{{iflng}}`) in the Contao core bundle.
- Use the `…NestedParsedInterface` variant instead if you need the wrapped
  content's nested tags left unresolved.

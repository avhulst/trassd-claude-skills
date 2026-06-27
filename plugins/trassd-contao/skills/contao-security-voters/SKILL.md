---
name: contao-security-voters
description: >-
  Apply Contao's authorization — DataContainer access voters, the
  ContaoCorePermissions constants, and isGranted checks for back end data.
  Triggers when restricting access to DCA records/actions or adding a Contao
  security voter.
---

# Contao security & voters

Contao layers its permission model on top of **Symfony's Security Component**.
Authorization is expressed through **voters** that are invoked when a permission
is checked via the Security helper (`isGranted()` /
`denyAccessUnlessGranted()`). Use Contao's built-in permission attributes
instead of hand-rolling user/group checks, and add a custom voter only when
the built-in attributes don't express your rule.

## Core rules

- **Check, don't compute.** Express access as `isGranted($attribute, $subject)`
  using a permission attribute and a subject; let voters decide. Don't read
  `$user->isAdmin` / group fields ad hoc in business code.
- **Use the constant classes, not raw strings.** Permission attributes live in
  `Contao\CoreBundle\Security\ContaoCorePermissions` (and per-bundle classes
  like `ContaoNewsPermissions`, `ContaoCalendarPermissions`,
  `ContaoNewsletterPermissions`, `ContaoFaqPermissions`). The constants' PHPDoc
  documents the expected subject type.
- **Match the subject type to the attribute.** A wrong subject silently fails
  the check. For DCA CRUD the subject is an `AbstractAction` instance.
- **Voters must abstain on anything they don't own.** Contao uses the
  `priority` access-decision strategy in the `frontend`/`backend` scope — the
  first non-abstaining voter decides. A `supports()` that returns `true` too
  broadly will hijack unrelated decisions. To override a core decision, abstain
  on everything else and give the service a higher `security.voter` priority.
- **Admins bypass by design.** Back end admins can access everything; restrict
  non-admins via back end user groups, and short-circuit `ROLE_ADMIN` in custom
  voters/controllers.

## Checking access

Inject the Security helper (or `AuthorizationCheckerInterface`) and check an
attribute against its subject:

```php
use Contao\CoreBundle\Security\ContaoCorePermissions;

// Back end module access (subject = module name)
$security->isGranted(ContaoCorePermissions::USER_CAN_ACCESS_MODULE, 'news');

// Access to a specific field of a table (subject = "table::field")
$security->isGranted(ContaoCorePermissions::USER_CAN_EDIT_FIELD_OF_TABLE, 'tl_page::published');

// Any field of a table
$security->isGranted(ContaoCorePermissions::USER_CAN_EDIT_FIELDS_OF_TABLE, 'tl_content');

// Front end: member belongs to a group
$security->isGranted(ContaoCorePermissions::MEMBER_IN_GROUPS, $groupId);
```

In a back end controller, deny early. The Contao firewall already ensures the
user is logged in for back end routes, so only check the privilege (and let
admins through):

```php
if (!$auth->isGranted('ROLE_ADMIN')
    && !$auth->isGranted('contao_user.my_permissions', 'first_permission')) {
    throw new AccessDeniedException('Not enough permissions.');
}
```

## DataContainer (DCA) permissions

Since Contao 5, CRUD permissions on a data container are decided by voters
(replacing the old `checkPermission` `config.onload` callbacks). The attribute
is `contao_dc.<table>` (`ContaoCorePermissions::DC_PREFIX . $table`) and the
subject is one of the action classes in
`Contao\CoreBundle\Security\DataContainer\`:

```php
use Contao\CoreBundle\Security\DataContainer\{CreateAction, ReadAction, UpdateAction, DeleteAction};

$security->isGranted('contao_dc.tl_foobar', new ReadAction('tl_foobar', $record));
$security->isGranted('contao_dc.tl_foobar', new UpdateAction('tl_foobar', $record));
```

Each action exposes the record via `getCurrent()` (array), `getCurrentId()`,
`getCurrentPid()` and the table via `getDataSource()`.

### Writing a DataContainer voter

For one table, extend `AbstractDataContainerVoter`: return the table from
`getTable()` and decide all four CRUD actions in `hasAccess()`. The base class
handles `supports*` and abstaining for other tables.

```php
use Contao\CoreBundle\Security\Voter\DataContainer\AbstractDataContainerVoter;
use Contao\CoreBundle\Security\DataContainer\{CreateAction, ReadAction, UpdateAction, DeleteAction};

class ExampleAccessVoter extends AbstractDataContainerVoter
{
    protected function getTable(): string
    {
        return 'tl_example_archive';
    }

    protected function hasAccess(
        TokenInterface $token,
        CreateAction|ReadAction|UpdateAction|DeleteAction $action,
    ): bool {
        return match (true) {
            $action instanceof CreateAction => /* ... */,
            $action instanceof ReadAction, $action instanceof UpdateAction =>
                /* check against $action->getCurrentId() */,
            $action instanceof DeleteAction => /* ... */,
        };
    }
}
```

Important caveat: a record the user has **no read access** to must be filtered
out of the list view (e.g. set `list.sorting.root` in a `config.onload`
listener), otherwise the voter raises a permission-denied exception when the
record is loaded. See
[references/data-container-voter.md](references/data-container-voter.md) for the
full table + permissions + voter + listener walk-through.

For a rule that spans tables or doesn't map to one DCA (e.g. parent + child
table, or a single back end module), extend Symfony's base `Voter` directly and
implement `supports()` / `voteOnAttribute()`. See
[references/custom-voters.md](references/custom-voters.md).

### Registering custom back end permissions

To add your own privilege so the core `contao_user` voter understands it:

1. Append it to `$GLOBALS['TL_PERMISSIONS']` in `contao/config/config.php`.
2. Add a matching checkbox field to `tl_user` **and** `tl_user_group` DCA.

Then check it with `contao_user.<permission>`. Details and the
`PaletteManipulator` snippets are in
[references/custom-permissions.md](references/custom-permissions.md).

## Preview mode

Two distinct concepts:

- **Preview entry point** (`preview.php` is being used): check the `_preview`
  request attribute — `$request->attributes->get('_preview')`, or
  `app.request.attributes._preview|default` in Twig.
- **Preview *mode*** (the "Unpublished: show" toggle / a preview link with that
  feature): use the `Contao\CoreBundle\Security\Authentication\Token\TokenChecker`
  service — `$tokenChecker->isPreviewMode()`. Since 5.3 also available in Twig
  as `contao.is_preview_mode`.

Don't conflate them: being in the preview entry point does not mean unpublished
content should show.

## Notes

- All three backing docs were present and used. No stale references found: the
  identifiers cited (`ContaoCorePermissions` and its constants `DC_PREFIX`,
  `USER_CAN_ACCESS_MODULE`, `USER_CAN_EDIT_FIELD_OF_TABLE`,
  `USER_CAN_EDIT_FIELDS_OF_TABLE`, `MEMBER_IN_GROUPS`; the
  `Create/Read/Update/DeleteAction` classes with `getCurrent()/getCurrentId()/
  getDataSource()`; `AbstractDataContainerVoter` with `getTable()`/`hasAccess()`;
  and `TokenChecker::isPreviewMode()`) all exist in
  `core-bundle/src/Security`.

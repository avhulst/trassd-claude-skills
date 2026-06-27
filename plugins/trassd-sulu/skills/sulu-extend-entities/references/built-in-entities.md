# Extending built-in Sulu entities

Sulu can replace these internal entities the same way (example uses `User`):
User, Role, Contact, Account, Category, Media, Tag.

## 1. Subclass the Sulu entity

The `#[ORM\Table]` name must match the original table. Own properties need
default values.

```php
<?php

namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;
use Sulu\Bundle\SecurityBundle\Entity\User as SuluUser;

#[ORM\Entity]
#[ORM\Table(name: 'se_users')] // must match the extended entity's table
class User extends SuluUser
{
    #[ORM\Column(name: 'myProperty', type: 'string', length: 255, nullable: true)]
    private ?string $myProperty = null;

    public function setMyProperty(?string $myProperty): self
    {
        $this->myProperty = $myProperty;

        return $this;
    }

    public function getMyProperty(): ?string
    {
        return $this->myProperty;
    }
}
```

## 2. Configure Sulu to use your class

Set the relevant bundle config (create the file in `config/packages/` if it
does not exist), then run `php bin/adminconsole cache:clear`.

```yaml
# config/packages/sulu_security.yaml
sulu_security:
    objects:
        user:
            model: App\Entity\User
            repository: Sulu\Bundle\SecurityBundle\Entity\UserRepository
```

## Config keys per entity (table)

```yaml
# sulu_security.yaml
sulu_security:
    objects:
        user:   { model: ..., repository: ... }   # se_users
        role:   { model: ..., repository: ... }   # se_roles

# sulu_contact.yaml
sulu_contact:
    objects:
        contact: { model: ..., repository: ... }  # co_contacts
        account: { model: ..., repository: ... }  # co_accounts

# sulu_category.yaml
sulu_category:
    objects:
        category:             { model: ..., repository: ... }  # ca_categories
        category_meta:        { model: ..., repository: ... }
        category_translation: { model: ..., repository: ... }
        keyword:              { model: ..., repository: ... }

# sulu_media.yaml
sulu_media:
    objects:
        media: { model: ..., repository: ... }    # me_media

# sulu_tag.yaml
sulu_tag:
    objects:
        tag: { model: ..., repository: ... }      # ta_tags
```

Default model/repository classes (when not overriding) follow
`Sulu\Bundle\<Bundle>Bundle\Entity\<Name>` and `<Name>Repository`.

## Warnings

- Own properties must have at least default values or core commands
  (e.g. `sulu:security:user:create`) can crash.
- The `#[ORM\Table]` name must equal the extended entity's table.
- The user object is kept in the session; clearing sessions may be needed after
  replacing `User` (proxy-class errors).
- When overriding entities in an existing project, migrate existing data to
  avoid data loss.

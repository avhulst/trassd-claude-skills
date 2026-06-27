---
name: doctrine-migrations
description: Manage database schema changes with Doctrine migrations — generating, reviewing, and applying migrations safely. Triggers when changing entity mappings, creating migrations, or updating the database schema.
---

# Doctrine Migrations

Database schema changes in Symfony are driven by the **DoctrineMigrationsBundle**
(already installed in a standard Symfony app). When your entity mappings change,
the database does not change automatically — you generate a migration, review it,
and apply it. Never alter the schema by hand.

## The workflow

Every schema change follows the same two-step cycle:

1. **Change the mapping** — add or edit an entity property/mapping (e.g. via
   `make:entity` or by editing the class and its `#[ORM\Column]` attributes).
   At this point the property is mapped but does not yet exist in the table.

2. **Generate a migration** — Doctrine compares all your entities against the
   current state of the database and generates the SQL needed to synchronize them:

   ```terminal
   php bin/console make:migration
   ```

   On success you'll see the path to the new file under `migrations/`, e.g.
   `migrations/Version20211116204726.php`.

3. **Review the generated SQL** — open the migration file and read the SQL it
   contains before running it. A schema change like adding a `description` text
   field produces SQL such as:

   ```sql
   ALTER TABLE product ADD description LONGTEXT NOT NULL
   ```

   Confirm the SQL matches your intent (correct table, column, type, nullability)
   before applying it.

4. **Apply the migration** — execute all migration files that have not yet run
   against the database:

   ```terminal
   php bin/console doctrine:migrations:migrate
   ```

   The bundle only runs migrations it hasn't executed before. It tracks executed
   migrations internally in a `migration_versions` table, so re-running the
   command applies only new files.

## Rules of thumb

- **Always review the generated SQL** before running `doctrine:migrations:migrate`.
  The migration step is generated automatically, but you are responsible for
  confirming it does what you expect.
- **Never edit the database schema by hand.** Make schema changes only by
  changing entity mappings and applying migrations, so the database stays in sync
  with your entities and the change is reproducible across environments.
- **Commit the migration files** to version control, and run
  `doctrine:migrations:migrate` on production when you deploy to keep the
  production database up to date.
- **One cycle per schema change.** Each time you change your schema, run
  `make:migration` then `doctrine:migrations:migrate`.

## Gotcha: NOT NULL columns on SQLite

Adding a `NOT NULL` column on an existing SQLite database fails with
*"Cannot add a NOT NULL column with default value NULL"*. Make the property
nullable (`nullable: true`) to avoid it.

---

*Migrations are provided by the DoctrineMigrationsBundle. The official Symfony
Doctrine documentation covers migrations only at the workflow level (the
`make:migration` → review → `doctrine:migrations:migrate` cycle); for advanced
options consult the DoctrineMigrationsBundle documentation.*

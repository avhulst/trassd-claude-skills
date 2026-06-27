# Manual dumps and direct client access

Distilled from DDEV's database-management documentation.

## mysqldump / pg_dump

The `web` and `db` containers both ship with `mysqldump` (and `pg_dump` for the
PostgreSQL container). Because the project root is mounted at `/var/www/html` in
the container, dumps written there appear on the host too:

```bash
ddev ssh                                        # enter the web container
mkdir /var/www/html/.tarballs
mysqldump db > /var/www/html/.tarballs/db.sql
mysqldump db | gzip > /var/www/html/.tarballs/db.sql.gz
```

## Direct client access

```bash
ddev mysql                 # interactive mysql client in the db container
ddev mariadb               # interactive mariadb client
ddev psql                  # interactive psql client (PostgreSQL projects)
ddev mysql -udb -pdb       # run with the db user's privileges
ddev ssh -s db             # shell into the db container, then run mysql/psql
```

## Query examples

```bash
# Create an extra empty database and grant the db user access
ddev mysql -e 'CREATE DATABASE newdatabase; GRANT ALL ON newdatabase.* TO "db"@"%";'

# List tables matching a prefix
ddev mysql -e 'SHOW TABLES LIKE "node%";'
```

Interactive sessions work like a normal client:

```sql
ddev mariadb
MariaDB [db]> SELECT * FROM node WHERE type="article";
```

## GUI clients

- phpMyAdmin: `ddev add-on get ddev/ddev-phpmyadmin`
- Adminer: `ddev add-on get ddev/ddev-adminer`
- `ddev describe` shows the external `Host:` details for on-host browsers.
- macOS launchers (each must be installed first): `ddev sequelace`,
  `ddev tableplus`, `ddev querious`, `ddev dbeaver`. WSL2/Linux: `ddev dbeaver`.
- For JetBrains/PhpStorm or other on-host clients, set a static `host_db_port`
  in `.ddev/config.yaml`, `ddev start`, then connect to `localhost` on that port
  with type `mysql`/`postgresql`, user `db`, password `db`.

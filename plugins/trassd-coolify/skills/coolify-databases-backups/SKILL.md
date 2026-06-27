---
name: coolify-databases-backups
description: Provision managed databases on Coolify (PostgreSQL, MySQL, MariaDB, MongoDB, Redis, KeyDB, DragonFly, ClickHouse), secure their connections with SSL, and protect their data with scheduled backups to S3-compatible storage (AWS, Cloudflare R2, etc.). Use when adding a database, exposing it, enabling database SSL/TLS, or configuring automated backups and S3 storage.
---

# Coolify Databases & Backups

Coolify offers one-click provisioning for PostgreSQL, MySQL, MariaDB, MongoDB,
Redis, KeyDB, DragonFly, and ClickHouse. Even unsupported databases can run via a
custom Docker image. Use this skill when adding a database, exposing it, securing
it with SSL, or setting up automated backups.

## Provisioning a database

Pick **New Resource** and choose a database from the list, then configure it with
a click. Each database is a resource with its own settings (connection URLs,
SSL, backups). Coolify provides both an **internal URL** (for resources on the
same Docker network) and a **public URL** (when exposed to the internet).

### Exposing a database: Ports Mapping vs Public Port

Two ways to reach a database from outside its network — they behave differently:

- **Ports Mapping** uses Docker's published-ports feature to map a container port
  to a host port (e.g. `8080:80`). The connection/port is **permanent** — you
  must restart the database to change it.
- **Public Port** exposes the container by starting an Nginx TCP proxy. The
  connection/port is **dynamic** — change it without restarting the database;
  Coolify restarts the Nginx TCP proxy for you.

When using a Public Port, the Nginx TCP proxy has a **Proxy Timeout** controlling
how long idle connections stay open before Nginx closes them. Default is **3600
seconds** (1 hour); minimum is **1 second**. Raise it for long-running queries or
long-idle connections. Configure it under the database's **Proxy Timeout
(seconds)** setting.

### Reaching a database during builds (Nixpacks)

With the Nixpacks build pack, the app can reach the database two ways:

1. Database and app on the **same network** → use the **internal URL**.
2. Database and app **not on the same network** → set the database to
   **Accessible over the internet** and use the **public URL**.

## Database SSL/TLS

Database SSL (introduced in `v4.0.0-beta.399`) encrypts traffic between apps and
databases. Coolify generates and binds certificates automatically; manual changes
are only needed for custom certificates.

Enable it in the database's general settings: check **Enable SSL**, then pick an
**SSL Mode**.

> [!IMPORTANT]
> Enabling SSL is not enough on its own. You must use the **new connection URL**
> that Coolify produces (it carries the SSL configuration). If you keep the old
> URL, the connection will not use SSL in most cases. (Exception: Redis-based
> databases — enabling SSL in the UI does enforce the mode.)

### SSL modes (PostgreSQL)

From least to most secure:

- **allow** (insecure) — permits encrypted or unencrypted; falls back to
  unencrypted if SSL fails. Insecure.
- **prefer** (secure) — tries SSL first, falls back to unencrypted. Does not
  guarantee every connection is encrypted.
- **require** (secure) — connection must be encrypted, but the server certificate
  is not checked (server identity unverified).
- **verify-ca** (secure) — encrypts and verifies the certificate is signed by a
  trusted CA; does not check the hostname.
- **verify-full** (secure) — encrypts, verifies the CA, and confirms the hostname
  matches the certificate. **Recommended for maximum security when available.**

Availability by engine:

- **MySQL & MongoDB:** only `prefer`, `require`, `verify-ca`, `verify-full`.
- **MariaDB, Redis, KeyDB, DragonFly:** no SSL modes shown in the UI.
- **ClickHouse:** SSL is not supported (no checkbox, no dropdown).

### CA certificate and mounting it

Modes below `require` only encrypt without verifying server identity. For
`verify-ca` and `verify-full` you **must mount the Coolify CA certificate** into
any container that connects to the database.

Manage the CA under **Servers > YOUR_SERVER > Proxy > Advanced** (view, save a
copy, or regenerate it). The recommended bind mount:

```bash
/data/coolify/ssl/coolify-ca.crt:/etc/ssl/certs/coolify-ca.crt:ro
```

To use your own CA: produce it in PEM format, upload it to
`/data/coolify/ssl/coolify-ca.crt`, and mount it into connecting containers at
the path above.

## Scheduled backups

Scheduled backups are available for **PostgreSQL, MySQL, MariaDB, and MongoDB**
(and for Coolify's own database, which is backed up the same way). Backups run on
**cron expressions**, so you can schedule them as often as needed.

Coolify also accepts simple cron aliases:

```cron
every_minute   * * * * *
hourly         0 * * * *
daily          0 0 * * *
weekly         0 0 * * 0
monthly        0 0 1 * *
yearly         0 0 1 1 *
```

### Backup and restore commands per engine

PostgreSQL backs up a full custom-format dump; you may list databases to back up
comma-separated.

```bash
# PostgreSQL backup (custom format)
pg_dump --format=custom --no-acl --no-owner --username <username> <databaseName>

# PostgreSQL restore (custom format, any equivalent tool works)
pg_restore --verbose --clean -h localhost -U postgres -d postgres <dump-file>.dmp
```

```bash
# MySQL
mysqldump -u root -p <password> <databaseName>

# MariaDB
mariadb-dump -u root -p <password> <databaseName>

# MongoDB (gzip archive; --excludeCollection to skip collections)
mongodump --authenticationDatabase=admin --uri=<uri> --gzip --archive=<archive>
```

## Backing up to S3-compatible storage

Backups can be stored on S3-compatible storage instead of (or in addition to)
local storage. Coolify uses MinIO's client (`mc`) to copy backup files to the
bucket.

Supported providers include AWS, DigitalOcean Spaces, MinIO, Cloudflare R2,
Supabase Storage, Backblaze B2, Scaleway Object Storage, Hetzner, Wasabi, Vultr,
CloudPe, and IDrive e2. Others may work but are untested.

### Steps

1. **Create the bucket first.** Coolify verifies storage with a `ListObjectsV2`
   request against the bucket, so it must exist before you add it.
2. **Add the storage in Coolify** under the **Storage** section in the sidebar →
   **Add**. Provide a name, the **endpoint, the bucket name, the region, the
   Access Key, and the Secret Key**, then **Validate Connection & Continue**.
3. **Enable S3 on the backup**: in the database/instance backup settings, enable
   S3, select the storage, set the **frequency** (cron) and **retention**, and
   use **Backup Now** to verify end-to-end.

> [!IMPORTANT]
> The endpoint must be the S3 HTTP endpoint **without** the bucket name, e.g.
> `https://s3.eu-central-1.amazonaws.com`.

### AWS S3: minimal IAM policy

Create an IAM user, attach this policy (replacing the bucket name on both ARN
lines), generate an Access Key for it, and use those credentials in Coolify:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:GetObjectAcl",
        "s3:PutObjectAcl",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::your-bucket-name",
        "arn:aws:s3:::your-bucket-name/*"
      ]
    }
  ]
}
```

Save the Access Key and Secret Access Key when created — AWS will not show the
secret again. To control long-term cost, use S3 lifecycle rules (e.g. transition
to Glacier). Avoid S3 when regulations require backups stay on-prem, or when the
server has no outbound internet access.

> [!TIP]
> Verify backup health via webhook notifications: `backup_success`,
> `backup_failed`, and `backup_success_with_s3_warning` (local backup succeeded
> but the S3 upload failed). See the notifications/webhook configuration for
> wiring these up.

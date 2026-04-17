![GitHub actions](https://github.com/NgLoader/docker-mongodb-backup-local/actions/workflows/ci.yml/badge.svg?branch=main)

# db-backup-local

 This was mostly modified by Claude and not me!
 <br>
 I just confirmed the changes.

Backup PostgreSQL and MongoDB to the local filesystem with periodic rotating backups. Based on [prodrigestivill/docker-postgres-backup-local](https://github.com/prodrigestivill/docker-postgres-backup-local); supports a unified `DB_*` environment schema across engines.

Backup multiple databases from the same host by listing them in `DB_NAME` separated by commas or spaces.

Supported Docker architectures:
- **PostgreSQL** (debian & alpine): `linux/amd64`, `linux/arm64`, `linux/arm/v7`, `linux/s390x`, `linux/ppc64le`
- **MongoDB** (debian only, no official mongo alpine): `linux/amd64`, `linux/arm64`

Please consider reading [How the backups folder works?](#how-the-backups-folder-works).

This application requires the docker volume `/backups` to be a POSIX-compliant filesystem (needs hardlinks and softlinks). VFAT, EXFAT, SMB/CIFS, etc. are not supported.

## Supported engines and image tags

Published to GitHub Container Registry: `ghcr.io/ngloader/db-backup-local`.

| Engine | Versions | Tag examples |
|---|---|---|
| `postgresql` | 13, 14, 15, 16, 17, 18 | `:postgresql-18`, `:postgresql-18-alpine`, `:postgresql-17`, `:postgresql` (= latest pg) |
| `mongodb` | 8 | `:mongodb-8`, `:mongodb` (= latest mongo) |

Every release also produces immutable `-vX.Y.Z` tags plus moving `-vX`, `-vX.Y` aliases. For example: `:postgresql-18-v1.2.3`, `:postgresql-18-v1.2`, `:postgresql-18-v1`.

## Usage

### PostgreSQL

```yaml
services:
  postgres:
    image: postgres:18
    restart: always
    environment:
      POSTGRES_DB: database
      POSTGRES_USER: username
      POSTGRES_PASSWORD: password
  dbbackups:
    image: ghcr.io/ngloader/db-backup-local:postgresql-18
    restart: always
    user: postgres:postgres
    volumes:
      - /var/opt/dbbackups:/backups
    depends_on:
      - postgres
    environment:
      DB_TYPE: postgresql
      DB_HOST: postgres
      DB_NAME: database
      DB_USER: username
      DB_PASS: password
      DB_EXTRA_OPTS: "-Z1 --schema=public --blobs"
      SCHEDULE: "@daily"
      BACKUP_ON_START: "TRUE"
      BACKUP_KEEP_DAYS: 7
      BACKUP_KEEP_WEEKS: 4
      BACKUP_KEEP_MONTHS: 6
      HEALTHCHECK_PORT: 8080
```

### MongoDB

```yaml
services:
  mongo:
    image: mongo:8
    restart: always
    environment:
      MONGO_INITDB_ROOT_USERNAME: username
      MONGO_INITDB_ROOT_PASSWORD: password
      MONGO_INITDB_DATABASE: database
  dbbackups:
    image: ghcr.io/ngloader/db-backup-local:mongodb-8
    restart: always
    volumes:
      - /var/opt/dbbackups:/backups
    depends_on:
      - mongo
    environment:
      DB_TYPE: mongodb
      DB_HOST: mongo
      DB_NAME: database
      DB_USER: username
      DB_PASS: password
      DB_AUTH: admin
      BACKUP_SUFFIX: ".archive.gz"
      SCHEDULE: "@daily"
      BACKUP_ON_START: "TRUE"
      BACKUP_KEEP_DAYS: 7
      BACKUP_KEEP_WEEKS: 4
      BACKUP_KEEP_MONTHS: 6
      HEALTHCHECK_PORT: 8080
```

Secret-file alternatives for `DB_PASS`, `DB_USER`, `DB_NAME` are available (`DB_PASS_FILE`, `DB_USER_FILE`, `DB_NAME_FILE`) for use with docker secrets (remove trailing newlines from the files).

For PostgreSQL, running as user `postgres:postgres` is recommended. Initialize the destination folder permissions:

```sh
# debian images
mkdir -p /var/opt/dbbackups && chown -R 999:999 /var/opt/dbbackups
# alpine images
mkdir -p /var/opt/dbbackups && chown -R 70:70 /var/opt/dbbackups
```

### Environment Variables

| env variable | required | description |
|--|--|--|
| `DB_TYPE` | yes | Engine: `postgresql` or `mongodb`. |
| `DB_HOST` | yes | Server hostname. |
| `DB_NAME` | yes | Comma/space-separated list of databases to back up. If `DB_CLUSTER=TRUE` (pg), this is the bootstrap database used to discover others (typically `postgres` or `template1`). |
| `DB_USER` | yes | Database username. |
| `DB_PASS` | yes | Database password. Alternatively use `DB_PASS_FILE` (or `DB_PASSFILE_STORE` for PostgreSQL). |
| `DB_PORT` | no | Connection port. Defaults: `5432` (postgresql), `27017` (mongodb). |
| `DB_EXTRA_OPTS` | no | Additional flags passed to the dump tool. For pg see [pg_dump options](https://www.postgresql.org/docs/current/app-pgdump.html) (or [pg_dumpall options](https://www.postgresql.org/docs/current/app-pg-dumpall.html) if `DB_CLUSTER=TRUE`). For mongo see [mongodump options](https://www.mongodb.com/docs/database-tools/mongodump/). |
| `DB_NAME_FILE` | no | Alternative to `DB_NAME`, one database per line — for docker secrets. |
| `DB_USER_FILE` | no | Alternative to `DB_USER` — for docker secrets. |
| `DB_PASS_FILE` | no | Alternative to `DB_PASS` — for docker secrets. |
| `DB_AUTH` | no | **MongoDB only.** Authentication database (default `admin`). |
| `DB_CLUSTER` | no | **PostgreSQL only.** Set to `TRUE` to use `pg_dumpall` instead of `pg_dump`. Also set `DB_EXTRA_OPTS` to empty or a pg_dumpall-compatible value. |
| `DB_PASSFILE_STORE` | no | **PostgreSQL only.** Path to a libpq [passfile](https://www.postgresql.org/docs/current/libpq-pgpass.html) — for pg cluster setups. |
| `BACKUP_DIR` | no | Destination directory. Defaults to `/backups`. |
| `BACKUP_SUFFIX` | no | Filename suffix. Defaults to `.sql.gz`. For mongo, `.archive.gz` is recommended. |
| `BACKUP_ON_START` | no | `TRUE` → run a backup on container start. Defaults to `FALSE`. |
| `BACKUP_KEEP_DAYS` | no | Daily backups retained. Defaults to `7`. |
| `BACKUP_KEEP_WEEKS` | no | Weekly backups retained. Defaults to `4`. |
| `BACKUP_KEEP_MONTHS` | no | Monthly backups retained. Defaults to `6`. |
| `BACKUP_KEEP_MINS` | no | Minutes to keep `last/` backups. Defaults to `1440`. |
| `BACKUP_LATEST_TYPE` | no | `symlink`, `hardlink`, or `none`. Defaults to `symlink`. |
| `VALIDATE_ON_START` | no | `FALSE` to skip config validation on start. Defaults to `TRUE`. |
| `HEALTHCHECK_PORT` | no | Port for the cron-schedule health check. Defaults to `8080`. |
| `SCHEDULE` | no | [Cron-schedule](http://godoc.org/github.com/robfig/cron#hdr-Predefined_schedules) for periodic backups. Defaults to `@daily`. |
| `TZ` | no | POSIX TZ variable for cron evaluation (e.g. `Europe/Paris`). |
| `WEBHOOK_URL` | no | POST target for backup status updates. See `hooks/00-webhook`. |
| `WEBHOOK_ERROR_URL` | no | POST target for errors only. |
| `WEBHOOK_PRE_BACKUP_URL` | no | POST target when backup starts. |
| `WEBHOOK_POST_BACKUP_URL` | no | POST target when backup completes. |
| `WEBHOOK_EXTRA_ARGS` | no | Extra curl args for webhook calls. |

Setting a variable that applies to another engine (e.g. `DB_CLUSTER=TRUE` with `DB_TYPE=mongodb`) will emit a warning and be ignored.

### How the backups folder works?

Each backup is first written to `last/` with a full timestamp. On success it is hard-linked into `daily/`, `weekly/`, and `monthly/`, replacing prior entries for the same period. Only the latest backup per period is kept — the monthly backup for a month is whatever ran last in that month, not the first.

```
BACKUP_DIR/last/DB-YYYYMMDD-HHmmss<BACKUP_SUFFIX>
BACKUP_DIR/daily/DB-YYYYMMDD<BACKUP_SUFFIX>
BACKUP_DIR/weekly/DB-YYYYww<BACKUP_SUFFIX>  (ISO week; week end is Sunday)
BACKUP_DIR/monthly/DB-YYYYMM<BACKUP_SUFFIX>
```

A `DB-latest<BACKUP_SUFFIX>` link is maintained in each folder (symlink by default, configurable via `BACKUP_LATEST_TYPE`).

Cleanup runs after each successful backup and uses independent retention windows:
- `BACKUP_KEEP_MINS` — only affects `last/`
- `BACKUP_KEEP_DAYS` — `daily/`
- `BACKUP_KEEP_WEEKS` — `weekly/` (counted from end of week)
- `BACKUP_KEEP_MONTHS` — `monthly/` (31-day months, counted from end)

### Hooks

Drop executable scripts into `/hooks` inside the container. They are executed via `run-parts` on state changes, with the state name passed as the first argument (`error`, `pre-backup`, `post-backup`). See `hooks/00-webhook` for an example implementing `WEBHOOK_URL`.

### Manual backups

The default container runs on `SCHEDULE`. To trigger a backup immediately:

```sh
docker run --rm -v "$PWD:/backups" -u "$(id -u):$(id -g)" \
  -e DB_TYPE=postgresql -e DB_HOST=postgres -e DB_NAME=dbname \
  -e DB_USER=user -e DB_PASS=password \
  ghcr.io/ngloader/db-backup-local:postgresql-18 /backup.sh
```

## Restore examples

### PostgreSQL — restore inside the backup container

```sh
docker exec -ti $CONTAINER /bin/sh -c "zcat $BACKUPFILE | psql --username=$USERNAME --dbname=$DBNAME -W"
```

### PostgreSQL — restore with a fresh postgres container

```sh
docker run --rm -ti -v $BACKUPFILE:/tmp/backupfile.sql.gz postgres:$VERSION \
  /bin/sh -c "zcat /tmp/backupfile.sql.gz | psql --host=$HOSTNAME --port=$PORT --username=$USERNAME --dbname=$DBNAME -W"
```

### MongoDB — restore from an archive

```sh
docker exec -ti $CONTAINER /bin/sh -c "gunzip -c $BACKUPFILE | mongorestore --uri=\"mongodb://$USERNAME:$PASSWORD@$HOSTNAME:$PORT/?authSource=admin\" --archive --db=$DBNAME"
```

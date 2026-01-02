# Deploy Postgres with Docker Compose

This repository contains a minimal Docker Compose setup to run PostgreSQL and pgAdmin4 locally for development.

## What this provides

- **Postgres**: image `postgres:18-trixie`, database `DWH`, user `admin`, password `P@ssw0rd`, port `5432` mapped to the host, persistent data at `./postgres-db`.
- **Postgres**: image `postgres:18-trixie`, database `DWH`, user `admin`, password `P@ssw0rd`, port `5432` mapped to the host, persistent data at `./postgres-db` (mounted to `/var/lib/postgresql` inside the container).
- **pgAdmin4**: image `dpage/pgadmin4:9`, default email `admin@local.com`, password `admin`, accessible on host port `8081`, persistent data at `./pgadmin4`.

See the compose file for exact configuration: [docker-compose.yml](docker-compose.yml#L1-L40).

## Getting started

- **Prerequisites**: Docker and Docker Compose (or Docker CLI v2 with `docker compose`).
- From this folder run:

```bash
docker compose up -d
```

- To stop and remove containers:

```bash
docker compose down
```

- To remove volumes (data):

```bash
docker compose down -v
```

Accessing the services

- pgAdmin: open <http://localhost:8081> and log in with **<admin@local.com> / admin**. Add a new server in pgAdmin and point it to the Postgres container using host `postgres`, port `5432`, user `admin`, password `P@ssw0rd` (or use `localhost` and port mapping if connecting from the host).
- psql (from host): if you have the `psql` client installed you can connect to the DB with:

```bash
psql -h localhost -p 5432 -U admin -d DWH
```

Or exec into the container:

```bash
docker exec -it postgres psql -U admin -d DWH
```

Customization

- Change the Postgres image tag (e.g., `postgres:18` or `postgres:15`) in [docker-compose.yml](docker-compose.yml#L1-L40).
- Update credentials by editing the `environment` values for `POSTGRES_USER`, `POSTGRES_PASSWORD`, and `POSTGRES_DB` in [docker-compose.yml](docker-compose.yml#L1-L40).
- Persisted data directories are host paths relative to this repository: `./postgres-db` and `./pgadmin4`. Change them if you want a different location.
- Ports can be changed in the `ports:` mapping in [docker-compose.yml](docker-compose.yml#L1-L40).

Notes & recommendations

- The compose sets Postgres with `command: postgres -c 'max_connections=200'` to increase `max_connections` — remove or adjust as needed.
- The default credentials in this compose are weak — do not use them in production. Consider using environment files, Docker secrets, or a credentials manager.
- Uncomment `restart: unless-stopped` in the compose if you want containers to restart automatically.

## Useful commands

- View logs:

```bash
docker compose logs -f postgres
docker compose logs -f pgadmin4
```

- List containers:

```bash
docker compose ps
```

Troubleshooting

- If the DB fails to start after changing Postgres major versions, remove the `./postgres-db` directory (this will delete data) or migrate data between versions properly.
- If pgAdmin can't connect to Postgres from the host UI, add a server in pgAdmin using host `postgres` when configuring from inside the same Docker network (recommended) or `localhost`/`127.0.0.1` when connecting via the mapped port.

Upgrading to Postgres 18+ / Migration notes

- Postgres 18+ uses major-version-specific data directories under `/var/lib/postgresql` (not a single `data` directory). To avoid mount boundary issues the compose mounts `./postgres-db` at `/var/lib/postgresql`.
- If you are upgrading an existing data directory created by older Postgres images, do NOT start the Postgres 18+ container against the old data in-place. Recommended safe approaches:

1. Logical dump & restore (safe, recommended):

```bash
# stop containers
docker compose down

# create a dump (from the old container)
docker run --rm --network host -v "$(pwd)/postgres-db:/var/lib/postgresql" postgres:15 \
 bash -c "pg_dumpall -U admin" > dump.sql

# after switching image, restore
docker compose up -d
cat dump.sql | docker exec -i postgres psql -U admin -d postgres
```

Or, simpler if the old container is still available:

```bash
docker exec -t postgres pg_dumpall -c -U admin > dump.sql
```

1) File-system level migration (advanced): use `pg_upgrade --link` between the two Postgres versions. This requires mounts that do not break across versioned subdirectories — hence mounting at `/var/lib/postgresql`. Carefully follow upstream docs: [https://github.com/docker-library/postgres/pull/1259](https://github.com/docker-library/postgres/pull/1259) and [https://www.postgresql.org/docs/current/pgupgrade.html](https://www.postgresql.org/docs/current/pgupgrade.html)

Backup your data directory before any destructive step:

```bash
docker compose down
tar -czf postgres-db-backup-$(date +%Y%m%d%H%M%S).tar.gz ./postgres-db
```

If you need help performing a migration from a specific older version (for example `postgres:15`) to `postgres:18-trixie`, tell me the source version and I can provide exact commands.

License / Safety

- This setup is intended for local development and testing only. Harden before using in any public or production environment.

If you'd like, I can also:

- add an `.env` file example and update `docker-compose.yml` to load it, or
- add a brief `Makefile` with common commands.

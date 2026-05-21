# Deploy Postgres with Docker Compose

This repository contains an optimized, best-practice Docker Compose setup to run PostgreSQL and pgAdmin4 locally for development.

## What this provides

- **Postgres**: image `postgres:18-trixie`, database `DWH`, user `admin` (configurable), persistent data at `./postgres-db` (mounted to `/var/lib/postgresql` inside the container).
- **pgAdmin4**: image `dpage/pgadmin4:9` running in **Frictionless Desktop Mode** (single-user authentication bypass), accessible on host port `8081` (configurable), with **Automatic Postgres Connection Importing**.
- **Container Healthcheck**: Ensures pgAdmin4 only starts up after Postgres is fully healthy and accepting connections.
- **Externalized Credentials**: Environment configuration is completely isolated in a `.env` file.

See the compose file for exact configuration: [docker-compose.yml](docker-compose.yml).

## Getting started

1. **Prerequisites**: Docker and Docker Compose (or Docker CLI v2 with `docker compose`).
2. **Configuration**: Inspect and optionally adjust database credentials and host ports in your local [.env](.env) file:
   ```env
   POSTGRES_USER=admin
   POSTGRES_PASSWORD=P@ssw0rd
   POSTGRES_DB=DWH
   POSTGRES_PORT=5432
   PGADMIN_PORT=8081
   ```
3. **Spin up the stack**:
   ```bash
   docker compose up -d
   ```
4. **Stop and remove containers**:
   ```bash
   docker compose down
   ```
5. **Remove volumes (wipe data)**:
   ```bash
   docker compose down -v
   ```

## Accessing the services

- **pgAdmin**: Open <http://localhost:8081> in your browser. 
  *   **No login required**: You will be automatically logged in (Desktop Mode).
  *   **Pre-configured connection**: The **DWH (Local)** database server is already imported and visible in the left browser panel! Simply double-click it and enter your password (`P@ssw0rd` by default) to connect.
- **psql (from host)**: If you have the `psql` client installed, you can connect directly using the port specified in `.env`:
  ```bash
  psql -h localhost -p 5432 -U admin -d DWH
  ```
- **psql (inside container)**: Exec directly into the running Postgres container:
  ```bash
  docker exec -it postgres psql -U admin -d DWH
  ```

## Customization

- **Change Credentials**: Edit `POSTGRES_USER`, `POSTGRES_PASSWORD`, and `POSTGRES_DB` in [.env](.env) and recreate the containers. If changing pgAdmin server config, match the changes in [servers.json](servers.json).
- **Change Host Ports**: Adjust `POSTGRES_PORT` and `PGADMIN_PORT` in [.env](.env).
- **Automatic Server Auto-login**: If you want to bypass the password prompt inside pgAdmin when connecting, you can configure a `.pgpass` file (refer to the PostgreSQL documentation for details).
- **Multi-user Mode (Production)**: If deploying outside a local environment, restore pgAdmin authentication by removing `PGADMIN_CONFIG_SERVER_MODE: "False"` from `docker-compose.yml` and configuring email and passwords.

## Useful commands

- **View logs**:
  ```bash
  docker compose logs -f postgres
  docker compose logs -f pgadmin4
  ```
- **List container health status**:
  ```bash
  docker compose ps
  ```

## Troubleshooting

- **Permissions**: pgAdmin4 runs as a non-root user (UID `5050`). If pgAdmin fails to write its configuration, make sure the host `./pgadmin4` folder is writeable by the container.
- **Upgrades / Migrations**: Postgres 18+ uses major-version-specific data directories under `/var/lib/postgresql`. If upgrading an existing data directory from older Postgres versions, perform a dump/restore or use `pg_upgrade`.

### Safe Logical Migration (Example: upgrading from postgres:15)

1. Dump data from the older version:
   ```bash
   docker run --rm --network host -v "$(pwd)/postgres-db:/var/lib/postgresql" postgres:15 \
     bash -c "pg_dumpall -U admin" > dump.sql
   ```
2. Recreate containers with version 18:
   ```bash
   docker compose down -v
   docker compose up -d
   ```
3. Restore data:
   ```bash
   cat dump.sql | docker exec -i postgres psql -U admin -d postgres
   ```

## License / Safety

- This optimized setup is designed for robust and convenient local development. Always secure credentials and enable authentication before deploying to public or production servers.

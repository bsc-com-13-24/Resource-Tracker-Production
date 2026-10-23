# Run Resource Tracker with Docker Compose

This Compose setup runs the Resource Tracker backend and MySQL. Docker Compose reads configuration from `.env` in the same directory as `docker-compose.yaml`.

## Requirements

- Docker Engine or Docker Desktop
- Docker Compose v2
- Internet access for the first run, so Docker can pull the images

## Configure the environment

From the `resource-tracker` directory, create your local `.env` file from the example:

```bash
cp .env.example .env
```

Edit `.env` and replace the example database password and JWT secret. Use a unique, random JWT secret (at least 32 characters); for example, generate one with:

```bash
openssl rand -base64 48
```

Keep `.env` private and do not commit it. `DB_PASSWORD` is used for both the MySQL root account and the application database user in this Compose configuration. For production, use a dedicated secret management system and distinct database credentials.

The Compose configuration uses the `mysql` service name and port `3306` for backend-to-database connections. The host ports are configurable separately:

- `PORT` exposes the backend on the host (default `8080`).
- `DB_HOST_PORT` exposes MySQL on the host (default `3307`).

The backend does not use `DB_HOST` or `DB_PORT` from `.env`; Compose supplies the internal host and port. Optional mail settings are used for SMTP delivery. Set them in `.env` if you need email features.

## Start the services

Run these commands from `resource-tracker`:

```bash
docker compose up -d
```

Compose pulls the backend image and MySQL image if needed. MySQL must pass its TCP healthcheck before Compose starts the backend. Flyway applies database migrations when the backend starts.

To watch startup output in the terminal, use:

```bash
docker compose up
```

## Check the services

```bash
docker compose ps
docker compose logs -f backend
```

The API is available at `http://localhost:8080` by default, and Swagger UI is at `http://localhost:8080/swagger-ui/index.html`. If you changed `PORT`, use that host port instead. The health endpoint is `http://localhost:8080/actuator/health`.

## Change host ports

Edit `PORT` or `DB_HOST_PORT` in `.env`, then recreate the containers:

```bash
docker compose up -d --force-recreate
```

MySQL remains available to the backend on its internal port `3306`, regardless of the host port.

## Stop the services

Stop and remove the containers while preserving the database volume:

```bash
docker compose down
```

To delete the database volume and all stored data as well:

```bash
docker compose down -v
```

Only use `down -v` when deleting the database data is intentional.

## Update the backend image

The Compose file currently uses `jeckman/resource-tracker-backend:v1`. To pull that tag again and recreate the backend:

```bash
docker compose pull backend
docker compose up -d backend
```

## Troubleshooting

Inspect service status and logs:

```bash
docker compose ps
docker compose logs mysql
docker compose logs backend
```

If the backend cannot connect to MySQL, confirm that MySQL reaches `healthy`, and that `DB_NAME`, `DB_USER`, and `DB_PASSWORD` in `.env` agree with the MySQL container's initial setup. MySQL image initialization variables only take effect when the data directory is first created; changing credentials in `.env` does not change users in an existing `mysql_data` volume. Preserve the volume and update its MySQL users if the data matters. Remove it with `docker compose down -v` only if it is safe to delete all database data.

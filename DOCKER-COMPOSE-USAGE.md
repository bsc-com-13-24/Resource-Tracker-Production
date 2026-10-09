# Resource Tracker Docker Deployment

This guide explains how to run the Resource Tracker backend and MySQL database using the published Docker image:

```text
jeckman/resource-tracker-backend:v1
```

The deployment repository contains only the Docker Compose configuration and instructions. Users do not need the Resource Tracker source code, Java, Maven, Kotlin, or a local MySQL installation.

## Requirements

Install:

- Docker Engine or Docker Desktop
- Docker Compose v2
- Internet access for the first startup, so Docker can pull the backend and MySQL images

## Get the deployment files

Clone the separate deployment repository and enter its directory:

```bash
git clone https://github.com/YOUR_USERNAME/Resource-Tracker-Docker.git
```

The directory must contain:

```text
docker-compose.yaml
DOCKER-COMPOSE-USAGE.md
.env
```

## Configure the environment

Create a file named `.env` in the same directory as `docker-compose.yaml`:

```env
DB_NAME=resource_tracker
DB_USER=root
DB_PASSWORD=replace-with-a-strong-password
DB_HOST_PORT=3307
PORT=8080
APP_SECURITY_JWT_SECRET=replace-with-a-long-random-secret-at-least-32-characters
```

Keep `.env` private. Do not commit it to source control or share its passwords and JWT secret.

If the repository includes an `.env.example` file, create `.env` from it:

```bash
cp .env.example .env
```

Replace all placeholder values before starting the services. Use a different strong value for `APP_SECURITY_JWT_SECRET`; do not reuse the database password.

`DB_HOST_PORT` and `PORT` are the ports exposed on the host computer:

- MySQL is available from the host at `localhost:3307`.
- The backend is available from the host at `localhost:8080`.

The backend connects to MySQL internally through the Compose service name `mysql` on port `3306`. Do not change the internal database host to `localhost`.

## Start the services

From the deployment repository directory, run:

```bash
docker compose up -d
```

Compose pulls `jeckman/resource-tracker-backend:v1` automatically. Do not run `docker build` or Maven commands on the deployment machine.

To see the startup logs instead:

```bash
docker compose up
```

The first startup can take several minutes while MySQL initializes its data directory. Compose waits for MySQL to become healthy before starting the backend.

## Check the services

List the service status:

```bash
docker compose ps
```

View backend logs:

```bash
docker compose logs -f backend
```

Check the application health endpoint:

```bash
curl http://localhost:8080/actuator/health
```

An available and healthy application returns a response similar to:

```json
{"groups":["liveness","readiness"],"status":"UP"}
```

The interactive API documentation is available at:

```text
http://localhost:8080/swagger-ui/index.html
```

If the backend host port was changed, replace `8080` in the URLs above with the configured `PORT` value.

## Use the API

Most API endpoints require a JWT access token. First authenticate through the login endpoint:

```bash
curl -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"your-password"}'
```

Copy the `accessToken` from the response and use it as a bearer token:

```bash
curl http://localhost:8080/api/v1/projects \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

The project, vendor, department, and user management endpoints require an authenticated user with the appropriate role. The administrative management endpoints are restricted to administrators.

Examples:

```text
GET    /api/v1/projects
POST   /api/v1/projects
GET    /api/vendors
POST   /api/vendors
GET    /api/departments
POST   /api/departments
GET    /api/v1/users
```

Refer to Swagger UI for each endpoint's request body, query parameters, response format, and authorization requirements.

## Change host ports

If port `8080` is already in use, start the backend on another host port:

```bash
PORT=8081 docker compose up -d
```

The API is then available at `http://localhost:8081`.

If port `3307` is already in use, change the MySQL host port as well:

```bash
DB_HOST_PORT=3308 PORT=8081 docker compose up -d
```

These overrides apply to the current command. To make them permanent, update the values in `.env`.

## Stop the services

Stop and remove the containers while preserving database data:

```bash
docker compose down
```

The database is stored in the named `mysql_data` volume and will be available the next time the services start.

To remove the containers and the database volume:

```bash
docker compose down -v
```

Use `docker compose down -v` only when deleting the database data is intentional.

## Update to a newer image

When a newer backend tag is published, update the `image` value for the `backend` service in `docker-compose.yaml`. Then pull and recreate the backend:

```bash
docker compose pull backend
docker compose up -d backend
```

For example:

```yaml
backend:
  image: jeckman/resource-tracker-backend:v2
```

Database migrations run automatically when the updated backend starts.

## Troubleshooting

### MySQL is still starting

Check the database logs:

```bash
docker compose logs mysql
```

Wait for `docker compose ps` to show the MySQL service as `healthy`.

### A host port is already in use

Use different host ports as described in [Change host ports](#change-host-ports). The internal Compose ports remain unchanged.

### The backend cannot connect to MySQL

Confirm that:

- Both services are started with `docker compose ps`.
- MySQL is marked `healthy`.
- `DB_NAME`, `DB_USER`, and `DB_PASSWORD` match between the Compose configuration and `.env`.
- The backend uses `mysql` as its database host inside Compose.

# Docker Compose

- [Compose File](#compose-file)
- [Compose CLI](#compose-cli)
- [Examples](#examples)
- [Hints](#hints)

Tool that in a singly YAML file manages services, networks and volumes. The "Compose File" `compose.yaml` follows the [Compose File Format](https://docs.docker.com/compose/compose-file/). The "Compose CLI" is then used to create and start the services.

- `application model`: all compose files merged together
- `service`: runs same container image and configuration one or more times
- `network`: services communicate through networks, establishes IP routes between containers
- `volume`: persistent storage for services, high-level file system mount
- `config`: runtime or platform dependent configuration for services
  - inside containers configs behave like mounted volumes (but defined differently)
- `secret`: sensitive config data for services
  - inside containers secrets behave like mounted files
- `project`: individual deployment of app on a platform
  - `name` attribute groups and isolates resources
    - prefix platform resources with project `name` and set label `com.docker.compose.project`
    - can set custom project name and override it i.e. same compose file can be deployed multiple times on same infrastructure by passing a name

Benefits:

- simple: streamlined orchestration and replication
- collaboration: shared through YAML
- performance: reuses service on restart when nothing changed
- portable: vars in files means simple for different envs and users

## Compose File

- use `compose.yaml` as file name
  - can merge multiple compose files
    - appends or overrides based on order
    - simple attributes and maps overridden by highest prio
    - lists merged by appending
    - relative paths based on first file's parent dir
  - can reuse other compose files (good for when app depends on other app)
    - `include` attribute
- `fragment`: anchors and aliases for templating
- `extension`: TODO ???

### Project Name

Use for isolating environments:

- dev: multi copy of same env, e.g. for each feature branch
- CI: set to unique build number preventing build interference
- shared host: prevent interference between different apps that happen to use the same service names

Precendence (low to high):

- current dir
- dir or base name of first compose file (when using `-f`)
- top `name` attribute in compose file (or last file when using `-f`)
- envvar `COMPOSE_PROJECT_NAME`
- `-p` CLI flag

### Lifecycle Hooks

- default: `ENTRYPOINT` and `COMMAND` when starting and stopping containers
- vs hooks: have special privileges (root user even if container runs as non-root)
- `post_start`: runs after container is started
  - execution is not assured to be during `ENTRYPOINT`
- `pre_stop`: runs before container is stopped
  - does not execute when container stops itself or gets killed suddenly

```yaml
services:
  app:
    image: backend
    user: 1001
    volumes:
      - data:/data
    post_start:
      # change ownership of volume to non-root user
      - command: chown -R /data 1001:1001
        user: root

volumes:
  data: {} # a Docker volume is created with root ownership
```

```yaml
services:
  app:
    image: backend
    pre_stop:
      - command: ./data_flush.sh
```

### Profiles

Services start/stop by default but can be marked by `profiles` attribute in which case they are started/stopped only when explicitly specified.

**!!! Make sure to understand the following start/stop caveats !!!**

- note: essential services should not have profiles
- in following example frontend and phpmyadmin are optional
  - e.g. `docker compose --profile frontend --profile debug up`
  - e.g. `COMPOSE_PROFILES=debug docker compose up`
  - e.g. `docker compose --profile "*"` (all profiles)
  - e.g. `docker compose up -d` then `docker compose run phpmyadmin`
    - i.e. services can be started directly (dependents need to be in same profile)
    - `docker compose down phpmyadmin` stops it
    - `docker compose stop phpmyadmin` stops it
  - e.g. `docker compose --profile debug down`
    - note: this would stop db, backend and phpmyadmin _BUT NOT `frontend`_
  - note: `docker compose down` _ONLY STOPS `backend` and `db`_

```yaml
services:
  frontend:
    image: frontend
    profiles: [frontend]

  phpmyadmin:
    image: phpmyadmin
    depends_on: [db] # waits for db to be started
    profiles: [debug]

  backend:
    image: backend

  db:
    image: mysql
```

### Startup and Shutdown Control

Compose can define dependency chains where dependencies are started before the service and _service is stopped before the dependencies_.

- `depends_on`: waits for services to be started, optional per service:
  - `condition`: value...
    - `service_started`: (default)
    - `service_healthy`: waits for `healthcheck` (same as in Docker) to pass
    - `service_completed_successfully`: waits for successful completion

```yaml
services:
  web:
    build: .
    depends_on:
      db:
        condition: service_healthy
        # if db restarts, restart web
        restart: true
      redis:
        condition: service_started
  redis:
    image: redis
  db:
    image: postgres:18
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 10s # first check after 10s then every 10s
      retries: 5 # stop if 5 checks fail
      start_period: 30s # ignore healthcheck fails for first 30s
      timeout: 10s # timeout for each check
```

## Compose CLI

- use `docker compose` as command
  - `up`: starts all compose file services
  - `down`: stops and removes all running services
  - `ps`: shows all running services and statuses
  - `logs`: monitors logs of all running services

## Examples

The order is somewhat important.

- concept [Simple Web Application](example-simple-webapp.md)
- fundamentals [Python Redis Webapp](example-python-redis.md)

## Hints

- define volumes, configs and secrets at the top-level then add specifics on service level
- use for CI/CD system tests:
  - `docker compose up -d` (use `-d` to run in background)
  - `./run_tests`
  - `docker compose down -v` (use `-v` to remove volumes)
- compose does not wait for containers to be "ready", just for "running" i.e. use e.g. `depends_on` attribute to make sure of the order

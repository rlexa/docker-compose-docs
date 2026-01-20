# Docker Compose

- [Compose File](#compose-file)
  - [Project Name](#project-name)
  - [Lifecycle Hooks](#lifecycle-hooks)
  - [Profiles](#profiles)
  - [Startup and Shutdown Control](#startup-and-shutdown-control)
  - [Environment Variables](#environment-variables)
  - [Images](#images)
  - [Secrets](#secrets)
  - [Network](#network)
  - [Multiple Compose Files](#multiple-compose-files)
  - [Provider Services](#provider-services)
- [Compose in Prod](#compose-in-prod)
- [Compose CLI](#compose-cli)
- [Examples](#examples)
- [Hints](#hints)

Tool that in a single YAML file manages services, networks and volumes. The "Compose File" `compose.yaml` follows the [Compose File Format](https://docs.docker.com/compose/compose-file/). The "Compose CLI" is then used to create and start the services.

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

Precedence (low to high):

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

### Environment Variables

- service level: `environment` attribute
  - note: `"true"`, `"false"`, `"yes"`, `"no"` must be in quotes when used as map else YAML parses them as `True` or `False`
  - note: is a variable can't be resolved to a value it is removed
- service level: `env_file` attribute
  - one or list of env files
  - interpolation `${...}` is supported but only when run in compose context
- CLI `-e` flag
  - `docker compose run -e DEBUG=1 web python console.py`
- precedence (low to high):
  - image `ENV` or `ARG`
  - compose file `env_file`
  - compose file `environment`
  - compose file `env_file` or `environment` but interpolated
  - CLI `-e` flag

```yaml
services:
  webapp:
    env_file:
      - path: ./default.env
      - path: ./override.env
        required: false # true is default
    environment:
      # DEBUG: "true" <<< same as...
      - DEBUG=true
      # will try to resolve from outside (shell command or .env file) or remove
      - RESOLVEME
      # same as above but will cause a warning if can't resolve
      - RESOLVEMEORWARNING=${RESOLVEMEORWARNING}
```

#### Predefined Envvars

- note: inherits Docker CLI envvars e.g. `DOCKER_HOST`, `DOCKER_CONTEXT` etc.
- `COMPOSE_ANSI` output print ANSI control characters
  - on `auto` (default) detects TTY (or `never` or `always` TTY)
- `COMPOSE_CONVERT_WINDOWS_PATHS` converts volume definitions paths to Unix format
  - on `1` or `true`
- `COMPOSE_DISABLE_ENV_FILE` disables default `.env` file
  - on `1` or `true`
- `COMPOSE_ENV_FILES`
- `COMPOSE_EXPERIMENTAL` opt out of experimental features
  - on `0` or `false`
- `COMPOSE_FILE` path (for multiple always set `COMPOSE_PATH_SEPARATOR` too)
- `COMPOSE_IGNORE_ORPHANS` stops detecting orphaned (old config) containers
  - on `1` or `true`
- `COMPOSE_MENU` shows nav menu for opening in docker desktop
- `COMPOSE_PARALLEL_LIMIT` for concurrent engine calls
- `COMPOSE_PATH_SEPARATOR` if note set is `:` on Unix and `;` on Windows
- `COMPOSE_PROFILES` separated by `,`
- `COMPOSE_PROGRESS` output format `auto` (default), `tty`, `plain`, `json`, `quiet`
- `COMPOSE_PROJECT_NAME` sets project name
  - used as prefix in container name e.g. `projectname-servicename-1`
- `COMPOSE_REMOVE_ORPHANS` stops removing orphaned (old config) containers
  - on `1` or `true`
- `COMPOSE_STATUS_STDOUT` writes internal status to stdout instead stderr
  - on `1` or `true`
  - note: stderr is used to separate from container output

#### Interpolation

Compose file can have variables that are interpolated at runtime removing the need of editing the file.

- syntax
  - applied to unquoted and double-quoted values
    - _note: single-quoted values are not interpolated_
    - _note: quotes can be escaped with `\"`_
  - supports both `$VAR` and `${VAR}` interpolation
  - `${VAR}` resolves to `VAR`
  - `${VAR:-default}` resolves to `VAR || default` (`VAR` if set and not empty)
  - `${VAR-default}` resolves to `VAR ?? default` (`VAR` if set)
  - `${VAR:?error}` exits with error if `VAR` is not set or empty
  - `${VAR?error}` exits with error if `VAR` is not set
  - `${VAR:+replacement}` resolves to `replacement` if `VAR` is set and not empty, otherwise empty
  - `${VAR+replacement}` resolves to `replacement` if `VAR` is set, otherwise empty
- check with `docker compose config --environment`
- reference `.env` file vars directly in `environment` attribute
- example:
  - in `.env` file: `TAG=v1.5`
  - in compose file: `services: web: image: "webapp:${TAG}"`
  - check with `docker compose config`
    - `services: web: image: 'webapp:v1.5'`
- example:
  - in `.env` file: `COMPOSE_DEBUG=${DEV_MODE:-false}`
    - set to value of dev mode else false
  - in compose file: `services: web: environment: - DEBUG=${COMPOSE_DEBUG}`
  - check with `DEV_MODE=true docker compose config`
    - `services: web: environment: DEBUG: "true"`

### Images

- try to reduce pull/push time and image weight
- use services that share base layers e.g.
  - same image layer OS for all services
  - same image layer after installing system packages
- see also re-using service base images
- see also using "Bake" build system

Example: all services use `alpine` and need `openssl`.

```Dockerfile
# Dockerfile

FROM alpine as base
RUN /bin/sh -c apk add --update --no-cache openssl

FROM base as service_a
# build service a
...

FROM base as service_b
# build service b
...
```

```yaml
# compose.yaml

services:
  a:
    build:
      target: service_a
  b:
    build:
      target: service_b
```

### Secrets

- secret: any piece of sensitive data (pwd, cert, API key)
- mounted as `/run/secrets/<secret_name>` file in a container
  - define secrets at compose file top-level
  - bind services to secrets via `secrets` attribute

Simple examples:

```yaml
services:
  myapp:
    build:
      secrets:
        - npm_token
      context: .

secrets:
  npm_token:
    environment: NPM_TOKEN # from envvar
```

```yaml
secrets:
  my_secret:
    file: ./my_secret.txt
services:
  myapp:
    image: myapp:latest
    secrets:
      # available as `/run/secrets/my_secret` file
      - my_secret
```

Advanced example:

```yaml
volumes:
  db_data:

secrets:
  db_password:
    file: db_password.txt
  db_root_password:
    file: db_root_password.txt

services:
  db:
    image: mysql:latest
    volumes:
      - db_data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/db_root_password
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_root_password
      - db_password

  wordpress:
    depends_on:
      - db
    image: wordpress:latest
    ports:
      - "8000:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
```

### Network

- default:
  - single network with current project name
  - all containers can reach each other
    - _note: internally `CONTAINER_PORT` is used_
  - containers are discoverable by service name
    - _note: on updating containers are replaced with new IPs, use names_
  - _note_: can be configured via top level `networks: default: ...` entry
- _note: multi-host swarm mode is possible, look it up_

```yaml
# myapp/compose.yaml

# network: myapp_default

services:
  web:
    build: .
    ports:
      - "8000:8000"
    # optional, not really needed
    links:
      # here: adds an alias to "db" and marks it as dependency
      - "db:database"
  # e.g. web can connect to db via `postgres://db:5432`
  db:
    image: postgres:18
    ports:
      # HOST_PORT:CONTAINER_PORT
      # outside of network db is accessible via 8001
      # e.g. `postgres://{DOCKER_IP}:8001`
      - "8001:5432"
```

#### Custom networks

- `networks` top level key defines custom networks
  - can define custom drivers
  - can connect services to non-compose outside networks
  - can have custom `name`
  - can have static IP addresses (look it up)
- `networks` service level key can reference the defined networks

```yaml
networks:
  frontend:
    # Specify driver options
    driver: bridge
    driver_opts:
      com.docker.network.bridge.host_binding_ipv4: "127.0.0.1"
  backend:
    # Use a custom driver
    driver: custom-driver
services:
  proxy:
    build: ./proxy
    # proxy can't talk to backend network
    networks:
      - frontend
  app:
    build: ./app
    # app can talk to both networks
    networks:
      - frontend
      - backend
  db:
    image: postgres:18
    # db can't talk to frontend network
    networks:
      - backend
```

#### External networks

- e.g. when creating a network via `docker network create`

```yaml
networks:
  # won't create default network and looks for `my-pre-existing-network` instead
  network1:
    name: my-pre-existing-network
    external: true
```

### Multiple Compose Files

- default: `compose.yaml` and `compose.override.yaml`
  - i.e. base config and service overrides or added services

#### Merge Compose Files

- e.g. `docker compose -f compose.yaml -f compose.admin.yaml run backup_db`
- first is base, others are overrides
  - overrides need not be complete compose files
  - override keys either add or override (when exist already) values
  - multi-values are concatenated
- apply order is left to right
- paths are relative to base file (the first `-f` file)

```yaml
# compose.yaml: base

services:
  web:
    image: example/my_web_app:latest
    depends_on:
      - db
      - cache
  db:
    image: postgres:18
  cache:
    image: redis:latest

# compose.override.yaml: DEV
# `docker compose up`

services:
  web:
    build: .
    volumes:
      - '.:/code'
    ports:
      - 8883:80
    environment:
      DEBUG: 'true'

  db:
    command: '-d'
    ports:
     - 5432:5432

  cache:
    ports:
      - 6379:6379

# compose.prod.yaml: PROD
# `docker compose -f compose.yaml -f compose.prod.yaml up -d`

services:
  web:
    ports:
      - 80:80
    environment:
      PRODUCTION: 'true'

  cache:
    environment:
      TTL: '500'
```

#### Extend Compose Files

- `extends` top level key, useful for common set of config options
- same path and completion rules from merging apply here too

```yaml
# common-services.yml

services:
  webapp:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - "/data"

# compose.yaml

services:
  web:
    build: alpine
    command: echo
    extends:
      file: common-services.yml
      service: webapp
  webapp:
    extends:
      file: common-services.yml
      service: webapp
  important_web:
    extends: web
    cpu_shares: 10
```

Another example:

```yaml
# common.yaml

services:
  app:
    build: .
    environment:
      CONFIG_FILE_PATH: /code/config
      API_KEY: xxxyyy
    cpu_shares: 5

# compose.yaml

services:
  webapp:
    extends:
      file: common.yaml
      service: app
    command: /code/run_web_app
    ports:
      - 8080:8080
    depends_on:
      - queue
      - db

  queue_worker:
    extends:
      file: common.yaml
      service: app
    command: /code/run_worker
    depends_on:
      - queue
```

#### Include Compose Files

- `include` top level key allows modularized compose files
- _note: best for teams as every team can manage it's own file_
- _note: best for resolving path issues as every file is it's own app context_

```yaml
include:
  - my-compose-include.yaml # declares serviceB
  # ...or use from OCI artifact, Docker Hub
  # - oci://docker.io/username/my-compose-app:latest
services:
  serviceA:
    build: .
    depends_on:
      - serviceB # uses serviceB
```

Override example:

```yaml
# compose.yaml

include:
  - team-1/compose.yaml # declare service-1
  - team-2/compose.yaml # declare service-2

# compose.override.yaml

services:
  service-1:
    # enable debugger port
    ports:
      - 2345:2345
  service-2:
    # use local data folder containing test data
    volumes:
      - ./data:/data
```

### Provider Services

- `provider` attribute connects to a 3rd party components
  - think of platform capabilities made available for compose
  - `type` attribute can be:
    - Docker CLI plugin
    - binary in user's `PATH`
    - path to binary or script to execute
- provided info injected into services as envvars `<PROVIDER_SERVICE_NAME>_<VARIABLE_NAME>`
  - e.g. service with `provider` is named `database`:
    - might inject `DATABASE_URL`, `DATABASE_TOKEN` etc.
- _note: custom providers can be implemented manually_

```yaml
services:
  database:
    # marked as provider service
    provider:
      # choose a supported provider type
      type: awesomecloud
      # provider specific options
      options:
        type: mysql
        foo: bar
  app:
    image: myapp
    depends_on:
      - database
```

## Compose in Prod

- use `compose.yaml` and `compose.production.yaml`
- exec with `docker compose -f compose.yaml -f compose.production.yaml up -d`
- on app code changes:
  - rebuild image
  - recreate containers
  - redeploy e.g.:
    - `docker compose build web`
      - rebuilds image
    - `docker compose up --no-deps -d web`
      - stops, destroys, recreates web
      - `--no-deps` does not restart services web depends on
- _note: use `DOCKER_HOST`, `DOCKER_TLS_VERIFY` and `DOCKER_CERT_PATH` to deploy to a remote Docker host_

## Compose CLI

- use `docker compose` as command
  - `down`: stops and removes all running services
  - `logs`: monitors logs of all running services
  - `ps`: shows all running services and statuses
  - `pull`: pulls newest images (or defined image only)
  - `run`: for one-off commands
    - e.g. tests, adding data
  - `start`: starts stopped containers but does never create any
  - `stop`: stops a container
    - sends `SIGTERM`, waits default 10s, sends `SIGKILL`
  - `up`: starts all compose file services
    - `-d` for detached mode

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
- place `.env` file in root of project as standard practice
  - experiment: go to a nested dir, create `.env` with `COMPOSE_FILE=../compose.yaml` and overrides and compose from there to have a quick override test
- see service env with `docker-compose run servicename env`
- check docs for images to support secrets via envvar which read from files (because secrets are provided as files)
- do not forget to consider:
  - env var interpolation for compose file stability
  - project `name` for different environments
  - service hooks `post_start` and `pre_stop`
  - service `profiles` for optional services
  - service dependency chains with `depends_on`
- GPU access can be activated if services can make use of it
- compose apps can be published as OCI artifacts (look it up)
- always use exec form `CMD` and `ENTRYPOINT`
  - always use `["program", "arg1", "arg2"]` and not `program arg1 arg2`
    - _note: string form will use bash which doesn't eval `SIGTERM`_
  - make sure to eval `SIGTERM` when possible
  - set service level `stop_signal: SIGINT` to a signal it can handle
  - if not possible wrap app in init system like `s6` or `tini`

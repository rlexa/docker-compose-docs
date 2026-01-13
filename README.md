# Docker Compose

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

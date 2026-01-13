# Example: Python Redis Webapp

- Flask web framework
- task: hit counter in Redis

## Setup Steps

- create [`app.py`](example-python-redis/app.py)
  - `redis` and `6379` are the hostname and port of redis container
  - note: `get_hit_count` retries 5 times (good for startup and for when redis is being restarted)
- create [`requirements.txt`](example-python-redis/requirements.txt)
- create [`Dockerfile`](example-python-redis/Dockerfile)
- create [`infrastructure.yaml`](example-python-redis/infrastructure.yaml)
- create [`compose.yaml`](example-python-redis/compose.yaml) and include `infrastructure.yaml`

## Execution

- `docker compose up` (stays active or try with `-d`)
- [open 8080](http://127.0.0.1:8000)
- inspect images with `docker image ls`
  - inspect details with `docker inspect <tag or id>`
- stop with `docker compose down`
  - use with `-v` to completely remove volumes

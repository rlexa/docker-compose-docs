# Example: Simple Web Application

- service
  - webapp docker image
    - exposes 443 for external usage
  - database docker image
- secret
  - HTTPS certificate for frontend
- configuration
  - HTTP for frontend
- volume
  - for backend
- network
  - for frontend
  - for backend

```yaml
services:
  frontend:
    image: example/webapp
    ports:
      - "443:8043"
    networks:
      - front-tier
      - back-tier
    configs:
      - httpd-config
    secrets:
      - server-certificate

  backend:
    image: example/database
    volumes:
      - db-data:/etc/data
    networks:
      - back-tier

volumes:
  db-data:
    driver: flocker
    driver_opts:
      size: "10GiB"

configs:
  httpd-config:
    external: true

secrets:
  server-certificate:
    external: true

networks:
  # The presence of these objects is sufficient to define them
  front-tier: {}
  back-tier: {}
```

- `docker compose up` to init and start and inject everything
- `docker compose ps`:

| NAME               | IMAGE            | COMMAND                  | SERVICE  | CREATED       | STATUS       | PORTS                 |
| ------------------ | ---------------- | ------------------------ | -------- | ------------- | ------------ | --------------------- |
| example-frontend-1 | example/webapp   | "nginx -g 'daemon of..." | frontend | 2 minutes ago | Up 2 minutes | 0.0.0.0:443->8043/tcp |
| example-backend-1  | example/database | "docker-entrypoint.s..." | backend  | 2 minutes ago | Up 2 minutes |                       |

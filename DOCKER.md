# Docker usage and tips

This document explains how to build and run the REST-api project with Docker, plus recommended workflows for development, CI, and debugging.

## Build the image

From the project root (the directory that contains `Dockerfile`):

```bash
docker build -t rest-api:latest .
```

This produces a local image named `rest-api:latest`. The repository's `Dockerfile` should be a multi-stage build that compiles the Go binary and produces a small final image (e.g. `scratch` or `alpine`).

If you want a reproducible build that includes the Go module cache step, the Dockerfile should copy `go.mod`/`go.sum` first and run `go mod download` before copying the rest of the sources.

## Run the container

Expose the service port (the project listens on `8080` by default in the Dockerfile):

```bash
docker run --rm -p 8080:8080 --name rest-api rest-api:latest
```

If your app requires a database, pass connection configuration with environment variables and attach to a Docker network or use `--link` (legacy) or docker-compose for multi-container setups. Example environment variables:

```bash
docker run --rm -p 8080:8080 \
  -e DATABASE_URL="postgres://user:pass@db:5432/dbname?sslmode=disable" \
  --network my-net \
  rest-api:latest
```

## Development workflows

- Iterative development: build a dev image that mounts source so you can rebuild or use `air`/`reflex` inside the container.
- For local rapid iteration prefer `go run` on the host or use a dev-target image to avoid rebuilding the final image on every change.

Example dev run (mount source and run `go run` inside an image with tooling):

```bash
docker run --rm -it -p 8080:8080 -v "$(pwd):/src" -w /src golang:1.21 bash -lc "go run ./main.go"
```

## CI and reproducible builds

- Use the multi-stage Dockerfile in CI to produce the final artifact. Cache `go mod` layers if your CI supports Docker layer caching.
- Optionally use `docker buildx` to produce multi-arch images. Example:

```bash
docker buildx build --platform linux/amd64,linux/arm64 -t yourrepo/rest-api:latest --push .
```

## Debugging and smaller tweaks

- If you need a shell inside the final image, prefer building a debug image based on `alpine` instead of `scratch` so `sh` is available.
- For logging, mount a directory for logs or rely on the container stdout/stderr and use `docker logs -f <container>`.

## Healthchecks

Add a `HEALTHCHECK` to the Dockerfile (optional) that calls an HTTP health endpoint (e.g., `/healthz`) so orchestrators can detect issues:

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s CMD wget -qO- http://localhost:8080/healthz || exit 1
```

## Example docker-compose (postgres + app)

Create a `docker-compose.yml` for local full-stack testing:

```yaml
version: "3.8"
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: rest
    volumes:
      - db-data:/var/lib/postgresql/data

  api:
    image: rest-api:latest
    build: .
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: "postgres://user:pass@db:5432/rest?sslmode=disable"
    depends_on:
      - db

volumes:
  db-data:
```

Start it with:

```bash
docker compose up --build
```

## Tips & gotchas

- If the final image is `scratch`, debugging inside the container is harder. Use a separate debug image for interactive work.
- Keep `CGO_ENABLED=0` if you want a statically linked binary that runs in `scratch`.
- Use `http.MaxBytesReader` and other request-limiting in handlers to prevent large-body DOS when behind untrusted clients.
- Avoid copying unnecessary files into the image — use `.dockerignore` to reduce build context size.
- If your Dockerfile currently contains multiple competing build flows, keep only the single recommended multi-stage flow to avoid confusion.

## Building for different architectures

Use `docker buildx`:

```bash
docker buildx create --use --name mybuilder
docker buildx build --platform linux/amd64,linux/arm64 -t yourrepo/rest-api:latest --push .
```

This will create multi-arch images that work on both amd64 and arm64 hosts.

## Security

- Do not bake secrets into images. Use environment variables, Docker secrets, or a secrets manager.
- Keep base images patched. Pin base image versions where required.

---
This file was added to document building and running the project with Docker.

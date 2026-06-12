# Docker

**What it is:** A tool that packages an app and everything it needs (code, runtime, libs, OS deps) into a portable **image**, which runs as an isolated **container**.

**Why people use it:** "Works on my machine" goes away — the same image runs identically on a laptop, CI, and prod.

**Typically used for:** Packaging services for deployment, reproducible dev environments, running throwaway tooling, and as the build unit for Kubernetes/ECS.

---

## Mental model

- **Image** = immutable snapshot (a class). **Container** = a running instance of an image (an object).
- **Dockerfile** = recipe to build an image.
- **Registry** = where images live (Docker Hub, GHCR, ECR).

---

## Most common commands

```bash
docker build -t myapp:1.0 .        # build image from Dockerfile in .
docker run myapp:1.0               # run a container
docker ps                          # list running containers
docker ps -a                       # list all (incl. stopped)
docker images                      # list local images
docker logs <container>            # view container logs
docker exec -it <container> sh     # shell into a running container
docker stop <container>            # stop
docker rm <container>              # remove a container
docker rmi <image>                 # remove an image
docker pull nginx:latest           # download an image
docker push myrepo/myapp:1.0       # upload an image
docker system prune -a             # reclaim disk (deletes unused images/containers)
```

## Simple usage

Run nginx, mapping host port 8080 → container port 80:

```bash
docker run -d -p 8080:80 --name web nginx
```

- `-d` detached (background), `-p host:container` port map, `--name` friendly name.

A minimal Dockerfile for a Node app:

```dockerfile
FROM node:20-slim
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

Build and run it:

```bash
docker build -t myapp:1.0 .
docker run -p 3000:3000 myapp:1.0
```

## Common flags worth knowing

```bash
docker run \
  -d \                          # detached
  -p 3000:3000 \                # publish port
  -e NODE_ENV=production \      # env var
  -v $(pwd)/data:/app/data \    # bind-mount host dir into container
  --restart unless-stopped \    # auto-restart policy
  --name api \
  myapp:1.0
```

---

## Advanced

### Multi-stage builds (small, secure images)

Build with the full toolchain, ship only the artifact. This is the standard way to keep images small.

```dockerfile
# ---- build stage ----
FROM golang:1.22 AS build
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /app ./cmd/server

# ---- runtime stage ----
FROM gcr.io/distroless/static
COPY --from=build /app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

> **Grounded:** Go's official `golang` Docker image docs recommend exactly this distroless multi-stage pattern; Google's [distroless project](https://github.com/GoogleContainerTools/distroless) was built so production images contain only the app and its runtime deps — no shell, no package manager, smaller attack surface.

### Layer caching: order matters

Docker caches each layer; a changed layer busts all layers below it. Copy dependency manifests **before** source so `npm ci` is only re-run when deps change, not on every code edit.

```dockerfile
COPY package*.json ./    # changes rarely → cached
RUN npm ci
COPY . .                 # changes often → only this layer rebuilds
```

### Build for multiple architectures

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t myrepo/myapp:1.0 --push .
```

### .dockerignore (faster builds, smaller context)

```
node_modules
.git
*.log
.env
```

### Inspect & debug

```bash
docker inspect <container>             # full JSON: mounts, network, env
docker stats                           # live CPU/mem per container
docker run --rm -it myapp:1.0 sh       # --rm auto-deletes on exit
docker diff <container>                # files changed since image
```

### Healthcheck

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:3000/health || exit 1
```

---

## Practical recipes

> ⚠️ Most of these are **destructive** — they delete containers/images/volumes. Read before pasting.

**Stop all running containers:**
```bash
docker stop $(docker ps -q)
```

**Remove all stopped containers:**
```bash
docker container prune -f
```

**Stop AND remove every container (running or not):**
```bash
docker rm -f $(docker ps -aq)
```

**Delete all images:**
```bash
docker rmi -f $(docker images -aq)
```

**Remove only dangling images (untagged `<none>` leftovers):**
```bash
docker image prune -f
```

**Remove all unused images (not referenced by any container):**
```bash
docker image prune -a -f
```

**Remove images older than ~2 days (48h):**
```bash
docker image prune -a -f --filter "until=48h"
```

**Remove containers that exited more than 24h ago:**
```bash
docker container prune -f --filter "until=24h"
```

**Nuke everything unused — containers, networks, images, AND volumes (biggest disk reclaim):**
```bash
docker system prune -a --volumes -f
```

**See what's eating disk before you delete:**
```bash
docker system df            # summary: images / containers / volumes / cache
docker system df -v         # verbose: per-image and per-volume breakdown
```

**Remove all unused volumes (⚠️ deletes data not attached to a container):**
```bash
docker volume prune -f
```

**Remove all unused networks:**
```bash
docker network prune -f
```

**Kill containers by name pattern (e.g. everything starting with `test-`):**
```bash
docker rm -f $(docker ps -aq --filter "name=test-")
```

**Remove images matching a repo (e.g. all `myapp` tags):**
```bash
docker rmi -f $(docker images "myapp" -q)
```

**Free the most disk in one go (build cache is often the hidden hog):**
```bash
docker builder prune -a -f      # wipe all build cache
```

**Follow logs of the most recently started container:**
```bash
docker logs -f $(docker ps -q | head -1)
```

**Shell into the last-started container:**
```bash
docker exec -it $(docker ps -q | head -1) sh
```

**Copy a file out of a container:**
```bash
docker cp <container>:/app/output.log ./output.log
```

**One-off throwaway container (auto-deletes on exit):**
```bash
docker run --rm -it alpine sh
```

---

## Tips

- Pin base image tags (`node:20-slim`, not `node:latest`) for reproducible builds.
- Run as a non-root `USER` in production images.
- One process per container — let the orchestrator handle the rest.
- Treat containers as **ephemeral**; persist state in volumes or external stores, never in the container's writable layer.

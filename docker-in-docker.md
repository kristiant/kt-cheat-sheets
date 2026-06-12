# Docker-in-Docker (DinD)

**What it is:** Running Docker itself *inside* a container — so a container can build images and run other containers.

**Why people use it:** CI/CD jobs run in containers but often need to `docker build` / `docker push`. DinD (or its safer cousin, socket-mounting) gives a containerized job the ability to drive Docker.

**Typically used for:** CI pipelines (GitLab CI, Jenkins) that build and push images; isolated build environments; testing tooling that orchestrates containers.

---

## The two approaches (pick deliberately)

| Approach | How | Trade-off |
|---|---|---|
| **Socket mount** (DooD) | Mount host's `/var/run/docker.sock` into the container | Simple, fast, shares host's cache. Containers it starts are *siblings* on the host. **Security risk: root-equiv access to host.** |
| **True DinD** | Run a nested Docker daemon (`docker:dind`) with `--privileged` | Full isolation — nested containers are hidden from host. Needs `--privileged`, no shared cache, slower. |

Most CI "DinD" setups are actually **Docker-out-of-Docker (DooD)** via the socket — it's lighter. Use true DinD only when you need isolation.

---

## Socket mount (DooD) — simple usage

```bash
docker run -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  docker:cli sh
# inside the container:
docker ps          # lists the HOST's containers — same daemon
docker build -t myapp .
```

The container has the Docker *client*; it talks to the *host's* daemon through the socket. Any container it builds/runs is a sibling on the host.

## True DinD — simple usage

```bash
# start a nested daemon (needs privileged)
docker run --privileged --name dind -d docker:dind

# run a client against it
docker run --rm --link dind:docker docker:cli docker -H tcp://docker:2375 info
```

---

## Advanced

### GitLab CI with DinD (the canonical real-world use)

GitLab's documented pattern uses the `docker:dind` **service** container alongside a `docker` client image:

```yaml
# .gitlab-ci.yml
build:
  image: docker:27
  services:
    - docker:27-dind
  variables:
    DOCKER_HOST: tcp://docker:2376
    DOCKER_TLS_CERTDIR: "/certs"      # enables TLS between client & dind
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```

> **Grounded:** This is straight from [GitLab's official "Use Docker to build Docker images" docs](https://docs.gitlab.com/ci/docker/using_docker_build/). GitLab recommends DinD with TLS (`DOCKER_TLS_CERTDIR`) over socket-binding for shared runners, precisely because the socket approach grants jobs root on the host.

### Why `--privileged` is required for true DinD

The nested daemon needs to manage cgroups, mounts, and networking — capabilities a normal container lacks. This is also *why* it's risky: `--privileged` ≈ root on the host.

### Safer alternatives to DinD (prefer these for building images)

Building images doesn't actually require a Docker daemon. Daemonless builders avoid `--privileged` entirely:

```bash
# Kaniko — builds + pushes from inside a container, no daemon, no privileged
/kaniko/executor \
  --dockerfile=Dockerfile \
  --context=. \
  --destination=myrepo/myapp:1.0
```

```bash
# BuildKit (rootless) — modern, daemonless-friendly builder
buildctl build \
  --frontend dockerfile.v0 \
  --local context=. --local dockerfile=. \
  --output type=image,name=myrepo/myapp:1.0,push=true
```

> **Grounded:** [Kaniko](https://github.com/GoogleContainerTools/kaniko) (Google) and BuildKit/[buildx](https://github.com/docker/buildx) were built specifically to remove the need for privileged DinD in CI. Kubernetes-native CI (Tekton, GitHub Actions on K8s) widely uses Kaniko/BuildKit instead of DinD.

### Decision guide

- **Just building/pushing images in CI?** → Use Kaniko or BuildKit. No daemon, no privilege.
- **Need to actually *run* containers in the job (integration tests)?** → Socket mount (DooD) if you trust the job; true DinD if you need isolation.
- **Shared/untrusted runners?** → Avoid socket mount (host root exposure). Prefer DinD-with-TLS or rootless builders.

---

## Practical recipes

**Quick interactive Docker client against the host daemon (DooD):**
```bash
docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock docker:cli sh
```

**Spin up a throwaway isolated DinD daemon and a client wired to it:**
```bash
docker run --privileged --name dind -d docker:dind
docker run --rm -it --link dind:docker -e DOCKER_HOST=tcp://docker:2375 docker:cli sh
docker rm -f dind          # cleanup when done (⚠️ destroys nested images)
```

**Build + push an image with no daemon and no --privileged (Kaniko):**
```bash
docker run --rm \
  -v "$PWD":/workspace \
  -v ~/.docker/config.json:/kaniko/.docker/config.json:ro \
  gcr.io/kaniko-project/executor:latest \
  --dockerfile=Dockerfile --context=/workspace \
  --destination=myrepo/myapp:1.0
```

**Persist DinD's image cache across runs (avoid re-pulling every job):**
```bash
docker run --privileged -d --name dind \
  -v dind-storage:/var/lib/docker \
  docker:dind
```

**Test whether you're inside a container (handy in CI scripts):**
```bash
[ -f /.dockerenv ] && echo "inside a container"
```

**Check the socket-mounted daemon is reachable:**
```bash
docker -H unix:///var/run/docker.sock info
```

---

## Gotchas

- **Socket mount = host root.** Anyone who controls that container controls the host. Never do it with untrusted code.
- **DinD storage is ephemeral** — the nested daemon's images vanish when the container dies. No layer cache reuse across jobs unless you persist `/var/lib/docker`.
- **Path mismatches:** with DooD, bind-mount paths refer to the *host* filesystem, not the container's — `-v $(pwd):/x` from inside resolves on the host and often surprises people.

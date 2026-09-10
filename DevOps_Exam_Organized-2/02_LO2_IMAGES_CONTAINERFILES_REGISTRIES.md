# LO2 — Images, Containerfiles and Registries

# How to use this file during the exam

Start with Ctrl+F and search the noun from the task, for example `restart`, `port`, `volume`, `Containerfile`, `rollout`, `ImagePullBackOff`, or `OpenShift`.

For practical tasks, always do three things: **perform the task → verify it → take a screenshot showing the command and proof**.

---

## QUICK KNOWLEDGE / COMMAND PATTERNS

## 3. LO2 — Images and Containerfiles

### ARG base-image tag

```Dockerfile
ARG ALPINE_TAG=3.20
FROM alpine:${ALPINE_TAG}
RUN cat /etc/alpine-release
CMD ["cat", "/etc/alpine-release"]
```

```bash
podman build -t arg-demo:3.20 --build-arg ALPINE_TAG=3.20 .
podman build -t arg-demo:3.19 --build-arg ALPINE_TAG=3.19 .
```

### Multi-stage build

```Dockerfile
FROM golang:1.22 AS build
WORKDIR /src
COPY . .
RUN go build -o /out/app .

FROM gcr.io/distroless/base-debian12
COPY --from=build /out/app /app
CMD ["/app"]
```

Point: final image contains only runtime artifact, not compiler/toolchain.

### .containerignore

```bash
cat > .containerignore <<'EOF2'
.git
node_modules
secret.txt
EOF2
```

Then prove ignored files are not copied into the image.

### Tagging

```bash
podman build -t docker.io/YOURUSER/demo:1.0 -t docker.io/YOURUSER/demo:1 -t docker.io/YOURUSER/demo:latest .
```

Convention: `registry/namespace/name:tag`. If registry omitted, tool may use configured defaults; be explicit in exams.

### History and fewer layers

```bash
podman history IMAGE
```

Each `RUN`, `COPY`, `ADD` usually creates a layer. Combine related cleanup with install in the same `RUN`.

### Non-root image

```Dockerfile
FROM alpine:3.20
RUN adduser -D appuser && mkdir /app && chown appuser:appuser /app
WORKDIR /app
USER appuser
CMD ["whoami"]
```

### HEALTHCHECK

```Dockerfile
FROM nginx:latest
HEALTHCHECK --interval=10s --timeout=3s --retries=3 CMD curl -f http://localhost/ || exit 1
```

If curl is absent, install it or use another check. Verify:

```bash
podman ps
podman healthcheck run CONTAINER
podman inspect CONTAINER --format '{{json .State.Health}}'
```

### ENTRYPOINT vs CMD

```Dockerfile
FROM alpine:3.20
ENTRYPOINT ["echo", "Hello"]
CMD ["world"]
```

```bash
podman run --rm entry-demo             # Hello world
podman run --rm entry-demo Ivona       # Hello Ivona
podman run --rm --entrypoint date entry-demo
```

ENTRYPOINT is the fixed executable; CMD is default arguments and easier to override.

### ADD vs COPY

Use COPY for normal local files. Use ADD only when you need its special features, such as extracting a local tar archive or fetching a URL. In most exam answers, COPY is the safer default.

### Save/load image

```bash
podman save -o demo.tar localhost/demo:1.0
podman rmi localhost/demo:1.0
podman load -i demo.tar
```

Use case: air-gapped transfer.

### Pull by digest

```bash
podman pull docker.io/library/nginx@sha256:...
```

Digest pins exact content. Tags can move.

### No cache

```bash
podman build --no-cache -t demo:nocache .
```

Helpful when cached dependency install hides changes. Slower and wastes previous layers.

### Non-default Containerfile

```bash
podman build -f Containerfile.prod -t demo:prod .
```

The final `.` is the build context. COPY paths are relative to that context, not to the Containerfile location.

### COPY --chown

```Dockerfile
FROM alpine:3.20
RUN adduser -D appuser
WORKDIR /app
COPY --chown=appuser:appuser file.txt /app/file.txt
USER appuser
CMD ["ls", "-l", "/app/file.txt"]
```

### ARG vs ENV

```Dockerfile
FROM alpine:3.20
ARG BUILD_VERSION=dev
ENV APP_ENV=prod
RUN echo "build=$BUILD_VERSION" > /build.txt
CMD ["sh", "-c", "cat /build.txt; echo env=$APP_ENV"]
```

ARG exists at build time. ENV is present in the runtime container unless overridden.

### Login and push

```bash
podman login docker.io -u YOURUSER
podman build -t docker.io/YOURUSER/demo:1.0 .
podman push docker.io/YOURUSER/demo:1.0

podman login quay.io -u YOURUSER
podman tag docker.io/YOURUSER/demo:1.0 quay.io/YOURUSER/demo:1.0
podman push quay.io/YOURUSER/demo:1.0
```

Credentials are stored in an auth file, commonly under `${XDG_RUNTIME_DIR}/containers/auth.json` or `~/.config/containers/auth.json` depending on setup.

### Prune and remove

```bash
podman image prune
podman rmi IMAGE
```

Dangling image: untagged layer/image not referenced by a tag.

### Static file server with Python

```Dockerfile
FROM python:3.12-alpine
WORKDIR /site
COPY public/ /site/
EXPOSE 8000
CMD ["python", "-m", "http.server", "8000", "--bind", "0.0.0.0"]
```

```bash
podman build -t py-site .
podman run -d -p 8080:8000 py-site
curl http://localhost:8080
```

### Package pinning

Pinning means installing known versions, not random newest versions. It improves reproducibility but requires updates for security patches.

### Clean package cache in same layer

Debian/Ubuntu:

```Dockerfile
RUN apt-get update \
 && apt-get install -y --no-install-recommends curl ca-certificates \
 && rm -rf /var/lib/apt/lists/*
```

Cleanup must happen in the same `RUN`; deleting in a later layer does not remove bytes from earlier layers.

---

## FULL SOLVED PRACTICE QUESTIONS

# LO2 – Managing and creating container images

## 1. Use an ARG to choose the base-image tag at build time and build twice.

```bash
mkdir -p lo2-arg && cd lo2-arg
cat > Containerfile <<'EOF'
ARG ALPINE_TAG=3.20
FROM docker.io/library/alpine:${ALPINE_TAG}
CMD ["cat", "/etc/alpine-release"]
EOF
podman build --build-arg ALPINE_TAG=3.19 -t argdemo:3.19 .
podman build --build-arg ALPINE_TAG=3.20 -t argdemo:3.20 .
podman run --rm argdemo:3.19
podman run --rm argdemo:3.20
```

`ARG` is available during build and can change the base tag.

## 2. Multi-stage build and compare image size.

```bash
mkdir -p lo2-multistage && cd lo2-multistage
cat > hello.go <<'EOF'
package main
import "fmt"
func main() { fmt.Println("hello") }
EOF
cat > Containerfile <<'EOF'
FROM docker.io/library/golang:1.22 AS build
WORKDIR /src
COPY hello.go .
RUN go build -o hello hello.go

FROM docker.io/library/alpine:3.20
COPY --from=build /src/hello /usr/local/bin/hello
CMD ["hello"]
EOF
cat > Containerfile.single <<'EOF'
FROM docker.io/library/golang:1.22
WORKDIR /src
COPY hello.go .
RUN go build -o hello hello.go
CMD ["/src/hello"]
EOF
podman build -t hello-multi .
podman build -f Containerfile.single -t hello-single .
podman images | grep hello
```

The multi-stage image is smaller because the final image contains only the built binary, not the compiler toolchain.

## 3. Use `.containerignore`.

```bash
mkdir -p lo2-ignore && cd lo2-ignore
echo 'visible' > app.txt
echo 'secret' > secret.txt
cat > .containerignore <<'EOF'
secret.txt
EOF
cat > Containerfile <<'EOF'
FROM docker.io/library/alpine:3.20
WORKDIR /app
COPY . .
CMD ["ls", "-la", "/app"]
EOF
podman build -t ignoredemo .
podman run --rm ignoredemo
```

`secret.txt` is excluded from the build context and is not copied into the image.

## 4. Build an image with three tags and explain image naming.

```bash
mkdir -p lo2-tags && cd lo2-tags
cat > Containerfile <<'EOF'
FROM docker.io/library/alpine:3.20
CMD ["echo", "tag demo"]
EOF
podman build -t localhost/tagdemo:1.0 -t localhost/tagdemo:1 -t localhost/tagdemo:latest .
podman images localhost/tagdemo
```

Image naming format is `registry/namespace/name:tag`. Example: `docker.io/ivona/tagdemo:1.0`.

## 5. Inspect image layer history and reduce layers.

```bash
podman history localhost/tagdemo:latest
```

Each `RUN`, `COPY`, and `ADD` usually creates a layer. Reduce layers by combining related commands into one `RUN`, but do not destroy readability for no reason.

## 6. Set a non-root USER and writable WORKDIR.

```bash
mkdir -p lo2-user && cd lo2-user
cat > Containerfile <<'EOF'
FROM docker.io/library/alpine:3.20
RUN adduser -D appuser && mkdir -p /app && chown appuser:appuser /app
USER appuser
WORKDIR /app
CMD ["sh", "-c", "whoami && touch testfile && ls -l"]
EOF
podman build -t userdemo .
podman run --rm userdemo
```

The container runs as `appuser`, and `/app` is writable by that user.

## 7. Add HEALTHCHECK and show health status.

```bash
mkdir -p lo2-health && cd lo2-health
cat > Containerfile <<'EOF'
FROM docker.io/library/nginx:alpine
HEALTHCHECK --interval=10s --timeout=3s --retries=3 CMD wget -qO- http://localhost/ || exit 1
EOF
podman build -t healthdemo .
podman rm -f healthdemo
podman run -d --name healthdemo -p 8084:80 healthdemo
podman healthcheck run healthdemo
podman ps
podman inspect healthdemo --format '{{json .State.Health}}'
```

The healthcheck verifies that nginx responds on localhost inside the container.

## 8. ENTRYPOINT versus CMD, then override CMD.

```bash
mkdir -p lo2-entrycmd && cd lo2-entrycmd
cat > Containerfile <<'EOF'
FROM docker.io/library/alpine:3.20
ENTRYPOINT ["echo", "Message:"]
CMD ["default text"]
EOF
podman build -t entrycmddemo .
podman run --rm entrycmddemo
podman run --rm entrycmddemo "overridden text"
```

`ENTRYPOINT` is the fixed command. `CMD` provides default arguments that can be overridden at run time.

## 9. Explain ADD versus COPY and justify each.

```bash
mkdir -p lo2-add-copy && cd lo2-add-copy
echo 'plain file' > file.txt
tar -czf files.tar.gz file.txt
cat > Containerfile <<'EOF'
FROM docker.io/library/alpine:3.20
WORKDIR /app
COPY file.txt /app/copied.txt
ADD files.tar.gz /app/extracted/
CMD ["find", "/app", "-type", "f", "-maxdepth", "3"]
EOF
podman build -t addcopydemo .
podman run --rm addcopydemo
```

Use `COPY` for normal file copying. Use `ADD` only when you need automatic archive extraction or remote URL behavior.

## 10. Save, remove, and load an image tarball.

```bash
podman save -o tagdemo.tar localhost/tagdemo:latest
podman rmi localhost/tagdemo:1.0 localhost/tagdemo:1 localhost/tagdemo:latest
podman load -i tagdemo.tar
podman images | grep tagdemo
```

This is useful for air-gapped transfer where the target machine cannot pull from a registry.

## 11. Pull an image by digest and explain reproducibility.

```bash
podman pull docker.io/library/alpine:3.20
DIGEST=$(podman image inspect docker.io/library/alpine:3.20 --format '{{index .RepoDigests 0}}')
echo $DIGEST
podman pull "$DIGEST"
```

A tag can move. A digest points to a specific immutable image content, so it is more reproducible.

## 12. Build with `--no-cache`.

```bash
podman build --no-cache -t userdemo:nocache lo2-user
```

Layer caching reuses unchanged build steps. `--no-cache` forces every step to run again. It helps when cache is stale, but it makes builds slower.

## 13. Build from a non-default Containerfile name and explain context.

```bash
mkdir -p lo2-customfile && cd lo2-customfile
cat > MyContainerfile <<'EOF'
FROM docker.io/library/alpine:3.20
COPY message.txt /message.txt
CMD ["cat", "/message.txt"]
EOF
echo 'custom file build' > message.txt
podman build -f MyContainerfile -t customfiledemo .
podman run --rm customfiledemo
```

`-f` selects the Containerfile. The final `.` is the build context directory sent to the build.

## 14. Use `COPY --chown` and verify ownership.

```bash
mkdir -p lo2-chown && cd lo2-chown
echo 'owned file' > file.txt
cat > Containerfile <<'EOF'
FROM docker.io/library/alpine:3.20
RUN adduser -D appuser
COPY --chown=appuser:appuser file.txt /app/file.txt
CMD ["ls", "-l", "/app/file.txt"]
EOF
podman build -t chowndemo .
podman run --rm chowndemo
```

The copied file is owned by `appuser` inside the image.

## 15. Combine ARG and ENV and demonstrate build-time versus run-time variables.

```bash
mkdir -p lo2-arg-env && cd lo2-arg-env
cat > Containerfile <<'EOF'
ARG BUILD_VERSION=dev
FROM docker.io/library/alpine:3.20
ARG BUILD_VERSION
ENV APP_MODE=production
RUN echo "$BUILD_VERSION" > /build-version.txt
CMD ["sh", "-c", "echo Build=$(cat /build-version.txt); echo Mode=$APP_MODE"]
EOF
podman build --build-arg BUILD_VERSION=1.0 -t argenvdemo .
podman run --rm argenvdemo
podman run --rm -e APP_MODE=test argenvdemo
```

`ARG` is for build time. `ENV` exists at run time and can be overridden with `-e`.

## 16. Login and push an image to a private repository.

```bash
podman login docker.io
podman tag localhost/tagdemo:latest docker.io/<dockerhub_user>/tagdemo:latest
podman push docker.io/<dockerhub_user>/tagdemo:latest
```

Credentials are stored in an auth file, usually `${XDG_RUNTIME_DIR}/containers/auth.json` for rootless Podman or `$HOME/.config/containers/auth.json` depending on setup.

## 17. Remove dangling images and a specific image.

```bash
podman images --filter dangling=true
podman image prune
podman rmi localhost/customfiledemo:latest
```

A dangling image has no tag, often left behind after rebuilding an image with the same tag.

## 18. Static file server with `python -m http.server`.

```bash
mkdir -p lo2-static/site && cd lo2-static
echo '<h1>Hello static server</h1>' > site/index.html
cat > Containerfile <<'EOF'
FROM docker.io/library/python:3.11-alpine
WORKDIR /site
COPY site/ /site/
EXPOSE 8000
CMD ["python", "-m", "http.server", "8000"]
EOF
podman build -t staticdemo .
podman run -d --name staticdemo -p 8085:8000 staticdemo
curl http://localhost:8085
```

The image serves copied static files over HTTP.

## 19. Install a pinned package version and explain why.

```bash
mkdir -p lo2-pin && cd lo2-pin
cat > Containerfile <<'EOF'
FROM docker.io/library/alpine:3.20
RUN apk add --no-cache curl=8.12.1-r0 || apk add --no-cache curl
CMD ["curl", "--version"]
EOF
podman build -t pindemo .
podman run --rm pindemo
```

Pinning package versions improves reproducibility. If the exact version is unavailable in the repository, the build fails unless you intentionally allow a fallback.

## 20. Inspect an unknown image and run it accordingly.

```bash
podman pull docker.io/library/nginx:alpine
podman image inspect docker.io/library/nginx:alpine --format 'Entrypoint={{json .Config.Entrypoint}} Cmd={{json .Config.Cmd}} Ports={{json .Config.ExposedPorts}} Env={{json .Config.Env}}'
podman run -d --name unknownweb -p 8086:80 docker.io/library/nginx:alpine
curl http://localhost:8086
```

Inspect shows default command, exposed ports, and environment so you know how to run the image.

## 21. Multi-line RUN with package install and cache cleanup in the same layer.

```bash
mkdir -p lo2-clean && cd lo2-clean
cat > Containerfile <<'EOF'
FROM docker.io/library/debian:12
RUN apt-get update \
    && apt-get install -y --no-install-recommends curl ca-certificates \
    && rm -rf /var/lib/apt/lists/*
CMD ["curl", "--version"]
EOF
podman build -t cleandemo .
```

Cleanup must happen in the same `RUN` layer. Deleting cache in a later layer does not remove it from the previous layer.

## 22. Tag and push the same image to Docker Hub and Quay.

```bash
podman login docker.io
podman login quay.io
podman tag localhost/tagdemo:latest docker.io/<dockerhub_user>/tagdemo:latest
podman tag localhost/tagdemo:latest quay.io/<quay_user>/tagdemo:latest
podman push docker.io/<dockerhub_user>/tagdemo:latest
podman push quay.io/<quay_user>/tagdemo:latest
```

Mirroring is useful for availability, migration, backup, or reducing dependence on one registry.

## 23. Run an image, list OS packages, and explain minimal base images.

```bash
podman run --rm docker.io/library/debian:12 dpkg -l | head
podman run --rm docker.io/library/alpine:3.20 apk info | head
podman images debian alpine
```

Minimal base images usually have fewer packages, smaller size, and smaller attack surface.

---

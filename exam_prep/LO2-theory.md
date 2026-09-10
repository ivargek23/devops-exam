# LO2 — Managing and Creating Container Images

## 1. What is a container image?

A container image is a read-only template used to create containers.

It contains:
- application files
- runtime and libraries
- configuration
- metadata such as environment variables, exposed ports, default command, and user

A container is a running instance of an image.

```text
Containerfile
    ↓ podman build
Image
    ↓ podman run
Container
```

---

## 2. Image registries

A registry stores and distributes container images.

Examples used in this course:
- Docker Hub
- Quay.io
- Red Hat registries

Typical workflow:

```text
pull → tag → build/use → push
```

Pull:

```bash
podman pull docker.io/library/alpine
```

Login:

```bash
podman login quay.io
```

Push:

```bash
podman push quay.io/USERNAME/app:1.0
```

---

## 3. Image naming

General format:

```text
REGISTRY/NAMESPACE/IMAGE:TAG
```

Example:

```text
quay.io/student/myapp:1.0
```

Meaning:
- `quay.io` = registry
- `student` = namespace/account
- `myapp` = repository/image name
- `1.0` = tag

A tag is a human-readable reference to an image.

Examples:

```text
app:1.0
app:1
app:latest
```

`latest` is only a tag. It does not mean “newest” automatically.

---

## 4. Tags vs digests

A tag can be moved to point to another image.

Example:

```text
nginx:latest
```

may represent different image contents at different times.

A digest identifies exact image content:

```text
nginx@sha256:...
```

Using a digest is more reproducible because it points to one exact image.

General syntax:

```bash
podman pull IMAGE@sha256:DIGEST
```

---

## 5. Image layers

Container images are built from layers.

Many Containerfile instructions create a new image layer.

Example:

```Dockerfile
FROM debian:12
RUN apt-get update
RUN apt-get install -y curl
COPY app.py /app/app.py
```

Each filesystem-changing step contributes a layer.

Inspect history:

```bash
podman history IMAGE
```

Why layers matter:
- layers can be cached
- unchanged layers can be reused
- layers influence image size
- bad ordering can make builds unnecessarily slow

---

## 6. Layer caching

During a build, Podman can reuse unchanged layers from previous builds.

Example:

```Dockerfile
FROM node:20

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .
```

This ordering is useful because changing application source code does not force `npm install` to run again unless the dependency files changed.

Build without cache:

```bash
podman build --no-cache -t app:1.0 .
```

Use `--no-cache` when you deliberately want every build step executed again.

Disadvantage: slower builds and repeated downloads.

---

## 7. Build context

The final argument to `podman build` is the build context.

Example:

```bash
podman build -t app:1.0 .
```

`.` means the current directory is sent as the build context.

`COPY` and `ADD` normally access files from this context.

A different Containerfile can be selected with:

```bash
podman build -f Containerfile.dev -t app:dev .
```

---

## 8. .containerignore

`.containerignore` excludes files from the build context.

Example:

```text
.git
*.log
secrets/
node_modules/
```

Benefits:
- smaller build context
- faster builds
- avoids copying unnecessary files
- reduces risk of accidentally including sensitive files

---

# 9. Important Containerfile instructions

## FROM

Chooses the base image.

```Dockerfile
FROM alpine:3.20
```

---

## RUN

Executes a command while building the image.

```Dockerfile
RUN apk add --no-cache nginx
```

The result becomes part of the image.

---

## COPY

Copies local files from the build context into the image.

```Dockerfile
COPY index.html /usr/share/nginx/html/index.html
```

Use `COPY` for normal file copying.

---

## ADD

Also adds files to the image and has extra behavior, such as automatic extraction of supported local tar archives.

Example:

```Dockerfile
ADD site.tar.gz /srv/site/
```

Prefer `COPY` unless the extra behavior of `ADD` is actually needed.

---

## WORKDIR

Sets the working directory for subsequent instructions and at runtime.

```Dockerfile
WORKDIR /app
```

---

## ENV

Defines an environment variable that persists in the image/container runtime environment.

```Dockerfile
ENV APP_ENV=production
```

Check at runtime:

```bash
podman run --rm IMAGE env
```

---

## ARG

Defines a build-time variable.

```Dockerfile
ARG BASE_TAG=3.20
FROM alpine:${BASE_TAG}
```

Build with:

```bash
podman build --build-arg BASE_TAG=3.19 -t app:3.19 .
```

Key difference:

```text
ARG = build time
ENV = container runtime
```

An `ARG` is not automatically available in the running container.

---

## EXPOSE

Documents the port the application is expected to listen on.

```Dockerfile
EXPOSE 80
```

Important:

`EXPOSE` does NOT publish the port on the host.

You still need:

```bash
podman run -p 8080:80 IMAGE
```

---

## USER

Sets the user used for later build steps and/or when the container runs.

```Dockerfile
USER appuser
```

Running applications as non-root improves security.

---

## CMD

Defines the default command or default arguments.

Exec form:

```Dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

Exec form is generally preferred because signals reach the application directly.

---

## ENTRYPOINT

Defines the executable that should always run.

Example:

```Dockerfile
ENTRYPOINT ["echo"]
CMD ["hello"]
```

Running:

```bash
podman run IMAGE
```

prints:

```text
hello
```

Running:

```bash
podman run IMAGE goodbye
```

replaces `CMD`, so the effective command becomes:

```text
echo goodbye
```

Simple rule:

```text
ENTRYPOINT = fixed executable
CMD = default arguments / default command
```

---

## HEALTHCHECK

Defines a test for whether the application inside the container is healthy.

Example:

```Dockerfile
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -q --spider http://localhost/ || exit 1
```

Run healthcheck manually:

```bash
podman healthcheck run CONTAINER
```

Inspect health:

```bash
podman inspect CONTAINER
```

---

# 10. Shell form vs exec form

Shell form:

```Dockerfile
CMD python app.py
```

Exec form:

```Dockerfile
CMD ["python", "app.py"]
```

Exec form runs the application directly and generally handles Unix signals better.

For long-running application processes, prefer exec form.

---

# 11. Multi-stage builds

A multi-stage build uses one stage to build/compile an application and another smaller stage for the final runtime image.

Example:

```Dockerfile
FROM golang:1.22 AS builder

WORKDIR /src
COPY . .
RUN go build -o app .

FROM alpine:3.20

WORKDIR /app
COPY --from=builder /src/app /app/app

CMD ["/app/app"]
```

The compiler and build tools are not copied into the final image.

Benefits:
- smaller image
- fewer unnecessary packages
- smaller attack surface
- faster distribution

---

# 12. Minimal base images

A minimal runtime image contains fewer packages and tools.

Advantages:
- smaller downloads
- smaller storage use
- fewer packages that may contain vulnerabilities
- reduced attack surface

Trade-off:
- fewer debugging tools are available inside the running container

---

# 13. Cleaning package caches

Bad:

```Dockerfile
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*
```

Better:

```Dockerfile
RUN apt-get update \
    && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*
```

Cleanup should happen in the same layer as installation.

Deleting files in a later layer does not remove them from the size of an earlier layer.

---

# 14. Pinning package versions

Example pattern:

```Dockerfile
RUN apk add --no-cache nginx=<VERSION>
```

Why pin versions?

Without pinning, a future build can install a different package version.

Pinning improves reproducibility.

Trade-off:
- pinned versions must be maintained and updated deliberately

---

# 15. COPY --chown

Files copied into an image can be assigned ownership during the copy.

Example:

```Dockerfile
FROM alpine:3.20

RUN adduser -D appuser
WORKDIR /app

COPY --chown=appuser:appuser app.txt /app/app.txt

USER appuser
CMD ["sh", "-c", "ls -l /app/app.txt"]
```

This avoids later ownership-fixing steps.

---

# 16. Save and load images

Save an image as a tar archive:

```bash
podman save -o image.tar app:1.0
```

Remove local image:

```bash
podman rmi app:1.0
```

Restore:

```bash
podman load -i image.tar
```

Useful for:
- offline transfer
- air-gapped environments
- backups
- moving images without a registry

---

# 17. Image tagging

Add another name/tag to the same image:

```bash
podman tag app:1.0 app:latest
```

One image can have multiple tags.

Example:

```bash
podman tag app:1.0 app:1.0 app:1 app:latest
```

The tags point to the same underlying image until one of them is moved.

---

# 18. Push images

Typical Quay workflow:

```bash
podman login quay.io
```

Tag with full registry path:

```bash
podman tag app:1.0 quay.io/USERNAME/app:1.0
```

Push:

```bash
podman push quay.io/USERNAME/app:1.0
```

The repository must exist and your account must have permission to push.

Authentication information is stored in a Podman auth JSON file. On Linux the default is commonly under:

```text
${XDG_RUNTIME_DIR}/containers/auth.json
```

The location can be changed with `--authfile`.

---

# 19. Image removal and pruning

Remove one image:

```bash
podman rmi IMAGE
```

Remove unused dangling images:

```bash
podman image prune
```

A dangling image is an image/layer that is no longer referenced by a useful repository/tag name.

Use pruning carefully so you do not remove build cache or images you still need.

---

# 20. Image inspection

Inspect an image:

```bash
podman image inspect IMAGE
```

Useful fields include:
- Entrypoint
- Cmd
- environment
- exposed ports
- user

Examples:

```bash
podman image inspect --format '{{.Config.Entrypoint}}' IMAGE
podman image inspect --format '{{.Config.Cmd}}' IMAGE
podman image inspect --format '{{.Config.Env}}' IMAGE
podman image inspect --format '{{.Config.ExposedPorts}}' IMAGE
```

This is useful when you are given an unfamiliar image and need to determine how it expects to run.

---

# 21. Image mirroring

The same image can be tagged for more than one registry.

Example:

```bash
podman tag app:1.0 docker.io/USERNAME/app:1.0
podman tag app:1.0 quay.io/USERNAME/app:1.0
```

Then:

```bash
podman push docker.io/USERNAME/app:1.0
podman push quay.io/USERNAME/app:1.0
```

Mirroring is useful for:
- registry redundancy
- disaster recovery
- moving between registries
- reducing dependency on one registry

---

# 22. Core LO2 model

```text
Containerfile
    ↓
podman build
    ↓
IMAGE
    ↓
tag
    ↓
registry
    ↓
pull
    ↓
podman run
    ↓
CONTAINER
```

LO2 is mainly about making image creation and distribution reproducible, efficient, secure, and manageable.

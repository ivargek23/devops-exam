LO2 — Images & Containerfiles: Exam Cheat Sheet

This file is optimized for fast lookup during the exam.

1. Pull → tag → save → push

TASK

Pull hello-world, tag it firstcontainer:1, save it as filec.tar, and push it to Quay.

COMMAND

podman pull docker.io/library/hello-world

podman tag docker.io/library/hello-world firstcontainer:1

podman save -o filec.tar firstcontainer:1

Create a public repository on Quay, then:

podman login quay.io

podman tag firstcontainer:1 quay.io/USERNAME/firstcontainer:1

podman push quay.io/USERNAME/firstcontainer:1

VERIFY

podman images
ls -lh filec.tar

EXPLAIN

podman save exports the image to a tar archive.

A registry push requires a fully qualified image name containing registry + namespace + repository + tag.

COMMON TRAP

This:

firstcontainer:1

is only a local name.

For Quay you need:

quay.io/USERNAME/firstcontainer:1

2. UBI9 + httpd + custom page

Containerfile

FROM registry.access.redhat.com/ubi9/ubi

RUN dnf -y install httpd \
    && dnf clean all

RUN echo "YOUR NAME SURNAME" > /var/www/html/index.html

EXPOSE 80

CMD ["httpd", "-D", "FOREGROUND"]

BUILD

podman build -t myhttpd:1.0 .

RUN

podman run -d \
  --name myhttpd \
  -p 8080:80 \
  myhttpd:1.0

VERIFY

curl http://localhost:8080
podman ps

EXPLAIN

dnf install httpd installs Apache.

CMD ["httpd","-D","FOREGROUND"] keeps Apache in the foreground so the container remains running.

3. Alpine + nginx + STUDENT + port 80

Containerfile

FROM docker.io/library/alpine:3.20

RUN apk add --no-cache nginx

ENV STUDENT="YOUR USERNAME SURNAME"

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]

BUILD

podman build -t mynginx:1.0 .

RUN

podman run -d \
  --name mynginx \
  -p 8080:80 \
  mynginx:1.0

VERIFY

podman ps
curl http://localhost:8080
podman exec mynginx env | grep STUDENT

EXPLAIN

Alpine uses apk.

Nginx normally daemonizes, so daemon off; keeps it in the foreground as the container's main process.

COMMON TRAP

EXPOSE 80 does not publish host port 80.

You still need -p.

4. ARG chooses base-image tag

Containerfile

ARG BASE_TAG=3.20
FROM docker.io/library/alpine:${BASE_TAG}

CMD ["cat", "/etc/alpine-release"]

BUILD WITH DEFAULT

podman build -t argtest:default .

BUILD WITH ANOTHER TAG

podman build \
  --build-arg BASE_TAG=3.19 \
  -t argtest:3.19 .

VERIFY

podman run --rm argtest:default
podman run --rm argtest:3.19

EXPLAIN

ARG is a build-time variable.

COMMON TRAP

An ARG does not automatically exist as an environment variable when the container runs.

5. Multi-stage build

Containerfile

FROM golang:1.22 AS builder

WORKDIR /src
COPY . .
RUN go build -o app .

FROM alpine:3.20

WORKDIR /app
COPY --from=builder /src/app /app/app

CMD ["/app/app"]

BUILD

podman build -t multistage:1.0 .

VERIFY SIZE

podman images

EXPLAIN

The first stage contains compiler/build tools.

Only the compiled artifact is copied into the final image.

This reduces final image size and attack surface.

6. .containerignore

FILE

.containerignore

.git
*.log
secret.txt
node_modules/

Containerfile

FROM alpine:3.20

WORKDIR /app
COPY . .
CMD ["ls", "-la", "/app"]

BUILD

podman build -t ignoretest .

VERIFY

podman run --rm ignoretest

The ignored files should not appear.

EXPLAIN

.containerignore excludes files from the build context.

7. Multiple tags

Assume:

app:1.0

already exists.

COMMAND

podman tag app:1.0 app:1.0 app:1 app:latest

Or individually:

podman tag app:1.0 app:1
podman tag app:1.0 app:latest

VERIFY

podman images

FORMAT

REGISTRY/NAMESPACE/NAME:TAG

Example:

quay.io/student/app:1.0

8. Inspect image history

podman history IMAGE

EXPLAIN

Shows the image's layers/history.

To reduce unnecessary size:

use minimal bases

combine package install + cleanup in one RUN

avoid copying unnecessary files

use multi-stage builds

9. Non-root USER + writable WORKDIR

Containerfile

FROM alpine:3.20

RUN adduser -D appuser \
    && mkdir -p /app \
    && chown appuser:appuser /app

WORKDIR /app
USER appuser

CMD ["sh", "-c", "whoami && touch /app/test && ls -l /app"]

BUILD + RUN

podman build -t nonroot .
podman run --rm nonroot

VERIFY

Expected user:

appuser

and file creation in /app should succeed.

EXPLAIN

The directory must be writable by the non-root user.

10. HEALTHCHECK

Containerfile

FROM alpine:3.20

RUN apk add --no-cache nginx wget

EXPOSE 80

HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -q --spider http://localhost/ || exit 1

CMD ["nginx", "-g", "daemon off;"]

RUN

podman build -t healthyweb .
podman run -d --name healthyweb healthyweb

VERIFY

podman healthcheck run healthyweb
podman inspect healthyweb

EXPLAIN

A healthcheck tests whether the application is actually functioning, not merely whether the container process exists.

11. ENTRYPOINT vs CMD

Containerfile

FROM alpine:3.20

ENTRYPOINT ["echo"]
CMD ["hello"]

RUN

podman build -t entrytest .
podman run --rm entrytest

Output:

hello

Override CMD:

podman run --rm entrytest goodbye

Output:

goodbye

EXPLAIN

ENTRYPOINT = fixed executable
CMD = default arguments/default command

Arguments after the image name replace CMD.

12. ADD vs COPY

Use COPY for normal files:

COPY config.txt /app/config.txt

Use ADD when you deliberately need its extra behavior, for example extracting a local tar archive:

ADD site.tar.gz /srv/site/

EXPLAIN

Prefer COPY because its behavior is simpler and more explicit.

13. Save → remove → load

podman save -o app.tar app:1.0

podman rmi app:1.0

podman load -i app.tar

VERIFY

podman images

EXPLAIN

Useful for air-gapped/offline image transfer.

14. Pull by digest

Syntax:

podman pull IMAGE@sha256:DIGEST

Example format:

docker.io/library/alpine@sha256:<digest>

EXPLAIN

A tag can move.

A digest identifies exact image content.

Therefore a digest gives a more reproducible deployment.

15. Build with --no-cache

podman build --no-cache -t app:1.0 .

EXPLAIN

Normally Podman reuses unchanged build layers.

--no-cache forces build instructions to execute again.

Useful when:

cached layers may be stale

you deliberately require a complete rebuild

Disadvantage:

slower

repeated downloads/work

16. Non-default Containerfile

podman build \
  -f Containerfile.dev \
  -t app:dev \
  .

EXPLAIN

-f selects the Containerfile.

The final . is the build context.

17. COPY --chown

Containerfile

FROM alpine:3.20

RUN adduser -D appuser

WORKDIR /app

COPY --chown=appuser:appuser app.txt /app/app.txt

USER appuser

CMD ["ls", "-l", "/app/app.txt"]

VERIFY

podman build -t chowntest .
podman run --rm chowntest

The file should belong to appuser.

18. ARG vs ENV

Containerfile

FROM alpine:3.20

ARG BUILD_VALUE=default-build-value
ENV RUNTIME_VALUE=production

RUN echo "Build value: $BUILD_VALUE"

CMD ["env"]

BUILD

podman build \
  --build-arg BUILD_VALUE=exam \
  -t argenv .

RUN

podman run --rm argenv

EXPLAIN

ARG is available during build.

ENV persists into the running container.

19. Login + private repository push

podman login quay.io

podman tag app:1.0 quay.io/USERNAME/privateapp:1.0

podman push quay.io/USERNAME/privateapp:1.0

EXPLAIN

Credentials/auth tokens are stored in Podman's authentication JSON.

Common Linux location:

${XDG_RUNTIME_DIR}/containers/auth.json

The path can be overridden with --authfile.

20. Prune and remove images

Specific image:

podman rmi app:1.0

Dangling/unused images:

podman image prune

EXPLAIN

A dangling image is no longer referenced by a useful repository/tag name.

21. Static Python file server

Containerfile

FROM python:3.12-slim

WORKDIR /site

COPY . /site

EXPOSE 8000

CMD ["python", "-m", "http.server", "8000", "--bind", "0.0.0.0"]

BUILD

podman build -t staticserver .

RUN

podman run -d \
  --name staticserver \
  -p 8080:8000 \
  staticserver

VERIFY

curl http://localhost:8080

22. Pin package version

Example pattern:

FROM alpine:3.20

RUN apk add --no-cache nginx=<VERSION>

EXPLAIN

Pinning prevents future builds from silently installing a different package version.

This improves reproducibility.

COMMON TRAP

Use a version that actually exists in the configured repository.

23. Inspect unknown image

Full:

podman image inspect IMAGE

Entrypoint:

podman image inspect \
  --format '{{.Config.Entrypoint}}' IMAGE

Command:

podman image inspect \
  --format '{{.Config.Cmd}}' IMAGE

Environment:

podman image inspect \
  --format '{{.Config.Env}}' IMAGE

Exposed ports:

podman image inspect \
  --format '{{.Config.ExposedPorts}}' IMAGE

EXPLAIN

Use image metadata to determine how an unfamiliar image expects to run.

24. Install + clean in same layer

GOOD

FROM debian:12

RUN apt-get update \
    && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*

EXPLAIN

Image layers are immutable.

If package-cache files are created in one layer and deleted in another, the earlier layer still contains them.

Therefore install and cleanup should happen in the same RUN.

25. Push same image to Docker Hub + Quay

podman tag app:1.0 docker.io/USERNAME/app:1.0
podman tag app:1.0 quay.io/USERNAME/app:1.0

podman push docker.io/USERNAME/app:1.0
podman push quay.io/USERNAME/app:1.0

EXPLAIN

Mirroring provides another source for the same image and reduces dependency on one registry.

26. Minimal base image

Inspect image:

podman run --rm IMAGE cat /etc/os-release

Package listing depends on the distribution.

Alpine:

podman run --rm IMAGE apk info

Debian/Ubuntu:

podman run --rm IMAGE dpkg -l

RPM-based:

podman run --rm IMAGE rpm -qa

EXPLAIN

Fewer packages usually mean:

smaller image

fewer potential vulnerabilities

smaller attack surface

FAST CONTAINERFILE REFERENCE

FROM IMAGE

ARG BUILD_VAR=value

ENV RUNTIME_VAR=value

WORKDIR /app

COPY source destination

RUN command

USER appuser

EXPOSE 80

ENTRYPOINT ["program"]

CMD ["arg1", "arg2"]

FAST IMAGE COMMANDS

podman pull IMAGE
podman images

podman tag SOURCE TARGET

podman image inspect IMAGE
podman history IMAGE

podman build -t NAME:TAG .
podman build --no-cache -t NAME:TAG .
podman build -f FILE -t NAME:TAG .
podman build --build-arg KEY=VALUE -t NAME:TAG .

podman save -o FILE.tar IMAGE
podman load -i FILE.tar

podman login REGISTRY
podman push IMAGE

podman rmi IMAGE
podman image prune

GOLDEN RULES

Containerfile → build → image → run → container

ARG = build time
ENV = runtime

EXPOSE documents a port
-p publishes a port

ENTRYPOINT = fixed executable
CMD = default command/arguments

COPY = normal copy
ADD = extra behavior such as local tar extraction

tag = movable name
digest = exact image content

same-layer cleanup actually reduces image size

multi-stage build keeps build tools out of final image

minimal images = smaller size + smaller attack surface

final "." in podman build = build context

TROUBLESHOOTING BUILD FAILURES

Package cannot be found

For Debian/Ubuntu:

RUN apt-get update \
    && apt-get install -y PACKAGE

For Alpine:

RUN apk add --no-cache PACKAGE

Container exits immediately

Check:

podman ps -a
podman logs CONTAINER
podman image inspect IMAGE

The default command may have completed or failed.

Port not reachable

Check image metadata:

podman image inspect IMAGE

Check running container:

podman port CONTAINER
podman logs CONTAINER

Remember:

EXPOSE != publish
-p HOST:CONTAINER

Permission denied after USER

Check:

USER placement

directory ownership

copied file ownership

Useful:

RUN chown -R appuser:appuser /app

or:

COPY --chown=appuser:appuser . /app

Build is unexpectedly slow after source changes

Move dependency files before source files.

Node example:

COPY package*.json ./
RUN npm install
COPY . .

This preserves the dependency-install layer cache.
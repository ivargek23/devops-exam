# Intro to DevOps – Solved Practice Questions

Commands assume a Linux VM with Podman and, for LO4/LO5 Kubernetes tasks, Minikube/kubectl. Replace placeholders like `<dockerhub_user>`, `<quay_user>`, and `<tag>` with your own values.

---

# LO1 – Use of containers and container services

## 1. Run an httpd container detached, publish container port 80 to host port 8081, give it a custom name and hostname, and confirm both.

```bash
podman rm -f testweb
podman run -d --name testweb --hostname testhost -p 8081:80 docker.io/library/httpd
podman inspect testweb --format 'Name: {{.Name}} Hostname: {{.Config.Hostname}}'
podman ps
curl http://localhost:8081
```

The container name is `testweb`, hostname is `testhost`, and host port `8081` forwards to container port `80`.

## 2. Run a container with `--restart=always`, kill its main process, and prove Podman restarts it.

```bash
podman rm -f restarttest
podman run -d --name restarttest --restart=always docker.io/library/nginx
podman ps
PID=$(podman inspect restarttest --format '{{.State.Pid}}')
sudo kill -9 $PID
sleep 3
podman ps
podman inspect restarttest --format 'Restart count: {{.RestartCount}}'
```

If the restart policy works, the container appears running again and the restart count increases.

## 3. Run a container with memory and CPU limits and verify them.

```bash
podman rm -f limitstest
podman run -d --name limitstest --memory=256m --cpus=0.5 docker.io/library/nginx
podman stats limitstest --no-stream
podman inspect limitstest --format 'Memory: {{.HostConfig.Memory}} NanoCPUs: {{.HostConfig.NanoCpus}}'
```

`256m` becomes bytes in inspect output. `0.5` CPU is shown as `500000000` NanoCPUs.

## 4. Start one container with `--env` and another with `--env-file`, then compare environment variables.

```bash
podman rm -f env1 env2
printf 'APP_MODE=prod\nAPP_PORT=8080\n' > app.env
podman run -d --name env1 --env KEY=VALUE docker.io/library/alpine sleep 1d
podman run -d --name env2 --env-file app.env docker.io/library/alpine sleep 1d
podman exec env1 env | grep KEY
podman exec env2 env | grep APP
```

`env1` gets one inline variable. `env2` gets all variables from `app.env`.

## 5. Create a user-defined bridge network, attach a container, and inspect its IP.

```bash
podman rm -f nettest
podman network rm ivonasnet
podman network create ivonasnet
podman run -d --name nettest --network ivonasnet docker.io/library/alpine sleep 1d
podman ps
podman network inspect ivonasnet
podman inspect nettest --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
```

The IP is visible in `podman network inspect` and directly through `podman inspect`.

## 6. Pause and unpause a running container and explain the process state.

```bash
podman rm -f pausetest
podman run -d --name pausetest docker.io/library/alpine sleep 1d
podman ps
podman pause pausetest
podman ps -a
podman unpause pausetest
podman ps
```

While paused, the processes are frozen/suspended. They are not killed, but they do not execute until unpaused.

## 7. Rename an existing container and confirm it.

```bash
podman rm -f renametest renamedcontainer
podman run -d --name renametest docker.io/library/alpine sleep 1d
podman ps
podman rename renametest renamedcontainer
podman ps
```

The old name disappears and the new name appears in `podman ps`.

## 8. Use `podman logs` with `--tail`, `--since`, and `-f`.

```bash
podman rm -f logstest
podman run -d --name logstest docker.io/library/alpine sh -c 'while true; do echo "Hello Podman $(date)"; sleep 2; done'
podman logs logstest --tail 5
podman logs logstest --since 10m
podman logs logstest -f
```

`--tail 5` shows the last 5 lines. `--since 10m` shows logs from the last 10 minutes. `-f` follows logs live; stop it with `Ctrl+C`.

## 9. Extract one field using Go-template syntax.

```bash
podman inspect logstest --format '{{.RestartCount}}'
podman inspect nettest --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
```

`--format '{{ ... }}'` prints only the selected field instead of the whole JSON output.

## 10. Copy a file into a running container and back out again.

```bash
podman rm -f copytest
podman run -d --name copytest docker.io/library/alpine sleep 1d
echo 'hello from host' > hostfile.txt
podman cp hostfile.txt copytest:/tmp/inside.txt
podman exec copytest cat /tmp/inside.txt
podman cp copytest:/tmp/inside.txt copied-back.txt
cat copied-back.txt
```

`podman cp` works in both directions: host → container and container → host.

## 11. Exec an interactive shell, install a package, and explain persistence.

```bash
podman rm -f shelltest
podman run -it --name shelltest docker.io/library/alpine sh
apk add --no-cache curl
curl --version
exit
podman start -ai shelltest
curl --version
exit
podman rm -f shelltest
podman run -it --name shelltest docker.io/library/alpine sh
curl --version
```

The installed package survives while the same container exists. It does not survive if the container is removed and recreated from the original image.

## 12. Use `podman top` and `podman stats`.

```bash
podman rm -f proctest
podman run -d --name proctest docker.io/library/nginx
podman top proctest
podman stats proctest --no-stream
```

`podman top` shows running processes. `podman stats` shows live CPU, memory, network, and block I/O usage.

## 13. Run a container with `--rm` and explain auto-removal.

```bash
podman run --rm --name rmtest docker.io/library/alpine echo 'temporary job'
podman ps -a | grep rmtest
```

`--rm` removes the container automatically after it exits. It is useful for temporary jobs. You lose the stopped container, logs, and filesystem changes.

## 14. Bind-mount a host directory read-only and prove writes are rejected.

```bash
mkdir -p readonlydir
echo 'host data' > readonlydir/file.txt
podman rm -f readonlytest
podman run --rm -v "$PWD/readonlydir:/data:ro" docker.io/library/alpine cat /data/file.txt
podman run --rm -v "$PWD/readonlydir:/data:ro" docker.io/library/alpine sh -c 'echo test > /data/new.txt'
```

The write fails because the mount is read-only. On SELinux systems use `:ro,Z` if permission is denied before the read-only test.

## 15. Add custom labels and filter containers by label.

```bash
podman rm -f label1 label2
podman run -d --name label1 --label course=devops --label env=test docker.io/library/alpine sleep 1d
podman run -d --name label2 --label course=other --label env=test docker.io/library/alpine sleep 1d
podman ps --filter label=course=devops
```

Only containers with the matching label are listed.

## 16. Commit a modified container into a new image and run it.

```bash
podman rm -f committest fromcommit
podman rmi localhost/alpine-custom
podman run -d --name committest docker.io/library/alpine sleep 1d
podman exec committest sh -c 'echo "created inside" > /custom.txt'
podman commit committest localhost/alpine-custom:1.0
podman run --rm --name fromcommit localhost/alpine-custom:1.0 cat /custom.txt
```

`podman commit` saves the container filesystem changes as a new image.

## 17. Show filesystem changes with `podman diff` and interpret markers.

```bash
podman rm -f difftest
podman run -d --name difftest docker.io/library/alpine sleep 1d
podman exec difftest sh -c 'echo new > /added.txt && touch /etc/changed.conf && rm /etc/alpine-release'
podman diff difftest
```

`A` means added, `C` means changed, and `D` means deleted.

## 18. Run a container with no network and demonstrate no connectivity.

```bash
podman rm -f nonet
podman run -d --name nonet --network none docker.io/library/alpine sleep 1d
podman exec nonet ip addr
podman exec nonet ping -c 1 8.8.8.8
```

The ping fails because the container has no external network. Use this for isolated batch jobs or malware/suspicious-file analysis labs.

## 19. Publish a port bound only to `127.0.0.1`.

```bash
podman rm -f localweb
podman run -d --name localweb -p 127.0.0.1:8082:80 docker.io/library/nginx
curl http://127.0.0.1:8082
hostname -I
curl http://<VM_IP>:8082
```

It works from localhost, but not through the VM’s other IP address because the port is bound only to loopback.

## 20. Use `podman port` and cross-check with inspect.

```bash
podman port localweb
podman inspect localweb --format '{{json .NetworkSettings.Ports}}'
```

Both commands show how container port `80/tcp` is mapped to the host.

## 21. Run a container as a non-root user and verify UID.

```bash
podman rm -f usertest
podman run -d --name usertest --user 1000:1000 docker.io/library/alpine sleep 1d
podman exec usertest id
podman exec usertest whoami
```

The effective UID is `1000`, not root. `whoami` may show an unknown user if `/etc/passwd` has no matching name.

## 22. Generate a systemd unit or Quadlet file.

Systemd unit:

```bash
podman rm -f systemdweb
podman run -d --name systemdweb -p 8083:80 docker.io/library/nginx
mkdir -p ~/.config/systemd/user
podman generate systemd --name systemdweb --files --new
mv container-systemdweb.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now container-systemdweb.service
systemctl --user status container-systemdweb.service
```

Quadlet alternative:

```bash
mkdir -p ~/.config/containers/systemd
cat > ~/.config/containers/systemd/systemdweb.container <<'EOF'
[Container]
Image=docker.io/library/nginx
ContainerName=systemdweb
PublishPort=8083:80

[Install]
WantedBy=default.target
EOF
systemctl --user daemon-reload
systemctl --user enable --now systemdweb.service
```

This makes systemd start the container automatically when the user service manager starts.

## 23. Run `podman events` while starting/stopping a container.

Terminal 1:

```bash
podman events
```

Terminal 2:

```bash
podman rm -f eventtest
podman run -d --name eventtest docker.io/library/alpine sleep 1d
podman stop eventtest
podman start eventtest
podman rm -f eventtest
```

You should observe lifecycle events such as `create`, `init`, `start`, `stop`, and `remove`.

---

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

# LO3 – Application delivery using containers, networking architecture, and component security

## 1. Create a pod with app and database containers, then reach DB over localhost.

```bash
podman pod rm -f apppod
podman pod create --name apppod -p 8087:80
podman run -d --pod apppod --name poddb -e POSTGRES_PASSWORD=secret docker.io/library/postgres:16
podman run -d --pod apppod --name podapp docker.io/library/nginx
sleep 10
podman exec podapp sh -c 'apt-get update >/dev/null 2>&1 || true'
podman exec poddb pg_isready -h localhost -p 5432
podman pod ps
```

Containers in the same pod share the network namespace, so they can reach each other through `localhost`.

## 2. Create a custom network and prove name resolution by container name.

```bash
podman rm -f appnet dbnet
podman network rm appnet
podman network create appnet
podman run -d --name dbnet --network appnet -e POSTGRES_PASSWORD=secret docker.io/library/postgres:16
podman run -d --name appnet --network appnet docker.io/library/alpine sleep 1d
podman exec appnet ping -c 2 dbnet
```

Podman DNS works on user-defined networks, so `dbnet` resolves by container name.

## 3. Deploy a two-tier stack: Gitea + PostgreSQL.

```bash
podman network rm giteanet
podman network create giteanet
podman volume create gitea-db
podman volume create gitea-data
podman run -d --name gitea-db --network giteanet \
  -e POSTGRES_USER=gitea -e POSTGRES_PASSWORD=gitea -e POSTGRES_DB=gitea \
  -v gitea-db:/var/lib/postgresql/data docker.io/library/postgres:16
podman run -d --name gitea --network giteanet -p 3000:3000 \
  -e GITEA__database__DB_TYPE=postgres \
  -e GITEA__database__HOST=gitea-db:5432 \
  -e GITEA__database__NAME=gitea \
  -e GITEA__database__USER=gitea \
  -e GITEA__database__PASSWD=gitea \
  -v gitea-data:/data docker.io/gitea/gitea:latest
podman ps
```

Open `http://localhost:3000` and finish the first-run setup.

## 4. Persist database data in a named volume.

```bash
podman exec -it gitea-db psql -U gitea -d gitea -c 'CREATE TABLE testdata(id int); INSERT INTO testdata VALUES (1);'
podman rm -f gitea-db
podman run -d --name gitea-db --network giteanet \
  -e POSTGRES_USER=gitea -e POSTGRES_PASSWORD=gitea -e POSTGRES_DB=gitea \
  -v gitea-db:/var/lib/postgresql/data docker.io/library/postgres:16
sleep 5
podman exec -it gitea-db psql -U gitea -d gitea -c 'SELECT * FROM testdata;'
```

The table still exists because the database files are in the named volume.

## 5. Use a Podman secret for the database password.

```bash
printf 'secretpass' | podman secret create db_password -
podman rm -f secretdb
podman run -d --name secretdb --secret db_password \
  -e POSTGRES_PASSWORD_FILE=/run/secrets/db_password \
  docker.io/library/postgres:16
podman exec secretdb ls -l /run/secrets
```

Secrets avoid exposing passwords directly in command history and normal environment listings.

## 6. Compose file for app + database.

```bash
mkdir -p lo3-compose && cd lo3-compose
cat > compose.yaml <<'EOF'
services:
  db:
    image: docker.io/library/postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb
    volumes:
      - dbdata:/var/lib/postgresql/data
  app:
    image: docker.io/library/nginx:alpine
    ports:
      - "8088:80"
    depends_on:
      - db
volumes:
  dbdata:
EOF
podman compose up -d
podman compose ps
```

Both services run from one Compose file.

## 7. Keep DB internal and expose only app.

```yaml
services:
  db:
    image: docker.io/library/postgres:16
    environment:
      POSTGRES_PASSWORD: secret
    networks:
      - backend
  app:
    image: docker.io/library/nginx:alpine
    ports:
      - "8088:80"
    networks:
      - backend
networks:
  backend:
    internal: true
```

```bash
podman compose up -d
podman compose ps
podman port lo3-compose-db-1
```

The DB has no published host port, so the host cannot connect directly to it.

## 8. Add healthcheck and `depends_on: condition: service_healthy`.

```yaml
services:
  db:
    image: docker.io/library/postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d appdb"]
      interval: 5s
      timeout: 3s
      retries: 5
  app:
    image: docker.io/library/nginx:alpine
    ports:
      - "8088:80"
    depends_on:
      db:
        condition: service_healthy
```

```bash
podman compose up -d
podman compose ps
podman compose logs db
```

The app starts only after the database reports healthy.

## 9. Use `env_file` for credentials.

```bash
cat > db.env <<'EOF'
POSTGRES_USER=app
POSTGRES_PASSWORD=secret
POSTGRES_DB=appdb
EOF
cat > compose.yaml <<'EOF'
services:
  db:
    image: docker.io/library/postgres:16
    env_file:
      - db.env
  app:
    image: docker.io/library/nginx:alpine
    ports:
      - "8088:80"
    depends_on:
      - db
EOF
podman compose up -d
podman compose ps
```

`env_file` keeps environment values out of the Compose service block.

## 10. Scale one Compose service to multiple replicas.

```bash
podman compose up -d --scale app=3
podman compose ps
```

Compose creates three app containers. Requests are distributed only if a proxy/load balancer is in front; plain Compose scaling alone does not magically load-balance host ports.

## 11. Bind mount config and named volume data.

```yaml
services:
  app:
    image: docker.io/library/nginx:alpine
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - appdata:/usr/share/nginx/html
volumes:
  appdata:
```

Bind mounts are good for external config files. Named volumes are good for container-managed persistent data.

## 12. Back up and restore a named volume.

```bash
podman volume create oldvol
podman run --rm -v oldvol:/data docker.io/library/alpine sh -c 'echo saved > /data/file.txt'
podman run --rm -v oldvol:/data:ro -v "$PWD:/backup" docker.io/library/alpine tar czf /backup/oldvol.tar.gz -C /data .
podman volume create newvol
podman run --rm -v newvol:/data -v "$PWD:/backup" docker.io/library/alpine tar xzf /backup/oldvol.tar.gz -C /data
podman run --rm -v newvol:/data docker.io/library/alpine cat /data/file.txt
```

The helper container mounts the volume and creates/restores a tarball.

## 13. Nginx reverse proxy in front of an app.

```bash
podman network rm proxynet
podman network create proxynet
podman run -d --name backend --network proxynet docker.io/hashicorp/http-echo -text='hello from backend'
cat > nginx.conf <<'EOF'
events {}
http {
  server {
    listen 80;
    location / {
      proxy_pass http://backend:5678;
    }
  }
}
EOF
podman run -d --name proxy --network proxynet -p 8090:80 -v "$PWD/nginx.conf:/etc/nginx/nginx.conf:ro" docker.io/library/nginx:alpine
curl http://localhost:8090
```

Host traffic goes to nginx, and nginx forwards it to the backend over the shared network.

## 14. Add a database-admin container.

```bash
podman run -d --name adminer --network giteanet -p 8089:8080 docker.io/library/adminer
podman ps
```

Open `http://localhost:8089`, choose PostgreSQL, and use server `gitea-db`, user `gitea`, password `gitea`, database `gitea`.

## 15. Generate Kubernetes YAML from a running pod.

```bash
podman pod rm -f kubepod
podman pod create --name kubepod -p 8091:80
podman run -d --pod kubepod --name kubeweb docker.io/library/nginx
podman generate kube kubepod > kubepod.yaml
cat kubepod.yaml
```

This bridges Podman work to Kubernetes manifests that can be used with `kubectl` or `podman kube play`.

## 16. Run and tear down a pod from Kubernetes YAML.

```bash
podman kube play kubepod.yaml
podman pod ps
podman kube play --down kubepod.yaml
```

`podman kube play` runs Kubernetes-style YAML locally with Podman.

## 17. Set per-service CPU/memory limits in Compose and verify.

```yaml
services:
  app:
    image: docker.io/library/nginx:alpine
    mem_limit: 256m
    cpus: 0.5
    ports:
      - "8092:80"
```

```bash
podman compose up -d
podman stats --no-stream
```

The limits appear in live stats and container inspect output.

## 18. Inspect a pod and show shared namespaces.

```bash
podman pod inspect kubepod
podman inspect kubeweb --format 'Network={{.HostConfig.NetworkMode}} PID={{.HostConfig.PidMode}} IPC={{.HostConfig.IpcMode}}'
```

Containers in a pod share the network namespace and usually IPC. PID namespace is not shared by default unless configured.

## 19. Show name discovery failing on different networks, then fix it.

```bash
podman network create neta
podman network create netb
podman run -d --name ca --network neta docker.io/library/alpine sleep 1d
podman run -d --name cb --network netb docker.io/library/alpine sleep 1d
podman exec ca ping -c 1 cb
podman network connect neta cb
podman exec ca ping -c 1 cb
```

Name resolution fails because the containers are on different networks. It works after both share `neta`.

## 20. Attach one container to frontend and backend networks.

```bash
podman network create frontend
podman network create backend
podman run -d --name dualnet --network frontend docker.io/library/alpine sleep 1d
podman network connect backend dualnet
podman inspect dualnet --format '{{json .NetworkSettings.Networks}}'
```

This allows one container to communicate with public-facing services on `frontend` and internal services on `backend`.

## 21. Use Compose logs and ps.

```bash
podman compose ps
podman compose logs
podman compose logs app
podman compose logs -f db
```

`ps` shows service status. `logs` helps troubleshoot startup and runtime errors.

## 22. Share configuration across replicas through a named volume.

```yaml
services:
  app:
    image: docker.io/library/nginx:alpine
    volumes:
      - sharedconfig:/config
volumes:
  sharedconfig:
```

```bash
podman compose up -d --scale app=2
podman compose ps
```

Shared read-write storage can cause race conditions or corrupted files if multiple replicas write at the same time.

## 23. Tear down a Compose stack and explain volumes.

```bash
podman compose down
podman volume ls
podman compose down --volumes
```

`podman compose down` removes containers and networks but keeps named volumes. `--volumes` also removes named volumes created by the stack.

---

# LO4 – Accelerated delivery of multilayer applications using containers

## 1. Create Deployment `web` with nginx:1.25 and 3 replicas, then export YAML.

```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3
kubectl get deployment web -o yaml > web-deployment.yaml
kubectl get pods
```

This creates a Deployment, ReplicaSet, and 3 Pods.

## 2. Enable authenticated Docker Hub pulls.

```bash
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<dockerhub_user> \
  --docker-password=<dockerhub_token> \
  --docker-email=<email>
```

You need a `docker-registry` Secret, then reference it with `imagePullSecrets`.

## 3. Scale from 3 to 5 replicas two ways.

```bash
kubectl scale deployment web --replicas=5
kubectl get rs
kubectl get deployment web -o yaml > web-deployment.yaml
# edit replicas: 5 in the YAML if needed
kubectl apply -f web-deployment.yaml
kubectl get deployment web
kubectl get rs
```

Scaling changes the replica count. It does not necessarily create a new ReplicaSet unless the pod template changes.

## 4. Rolling update from nginx:1.25 to nginx:1.27.

```bash
kubectl set image deployment/web nginx=nginx:1.27
kubectl rollout status deployment/web
kubectl get pods -l app=web
```

Kubernetes gradually replaces old Pods with new Pods.

## 5. Rollout history, rollback, and CHANGE-CAUSE.

```bash
kubectl annotate deployment web kubernetes.io/change-cause='Update nginx to 1.27'
kubectl rollout history deployment/web
kubectl rollout undo deployment/web
kubectl rollout status deployment/web
```

`CHANGE-CAUSE` shows the recorded reason for a rollout. Populate it with the `kubernetes.io/change-cause` annotation.

## 6. RollingUpdate with `maxSurge: 1` and `maxUnavailable: 0`.

```bash
kubectl patch deployment web -p '{"spec":{"strategy":{"type":"RollingUpdate","rollingUpdate":{"maxSurge":1,"maxUnavailable":0}}}}'
kubectl describe deployment web | grep -A5 Strategy
```

Kubernetes may create 1 extra Pod during update and keeps all desired Pods available. Availability is protected, but the update may use extra capacity.

## 7. Switch strategy to Recreate.

```bash
kubectl patch deployment web -p '{"spec":{"strategy":{"type":"Recreate"}}}'
kubectl describe deployment web | grep -A3 Strategy
```

`Recreate` stops old Pods before starting new ones. Use it when two versions cannot run at the same time, for example with an app that needs exclusive database migration access.

## 8. Add CPU/memory requests and limits.

```bash
kubectl set resources deployment web \
  --requests=cpu=100m,memory=128Mi \
  --limits=cpu=500m,memory=256Mi
kubectl get pod -l app=web -o jsonpath='{.items[0].spec.containers[0].resources}'
```

Requests affect scheduling. Limits cap container usage.

## 9. Set `revisionHistoryLimit: 3`.

```bash
kubectl patch deployment web -p '{"spec":{"revisionHistoryLimit":3}}'
kubectl get deployment web -o jsonpath='{.spec.revisionHistoryLimit}'
```

Kubernetes keeps up to 3 old ReplicaSets for rollback. Older revisions are cleaned up.

## 10. Set image to a non-existent tag and observe stuck rollout.

```bash
kubectl set image deployment/web nginx=nginx:no-such-tag
kubectl rollout status deployment/web --timeout=30s
kubectl get pods
kubectl describe deployment web
```

New Pods fail with image pull errors. Old ready Pods keep serving traffic because RollingUpdate does not remove all old Pods before new ones become available.

## 11. List Pods by selector and explain Deployment → ReplicaSet → Pod.

```bash
kubectl get pods -l app=web
kubectl get rs -l app=web
kubectl get deployment web -o jsonpath='{.spec.selector.matchLabels}'
```

The Deployment selector matches ReplicaSets, and ReplicaSets manage Pods whose template labels match that selector.

## 12. Expose a Deployment.

```bash
kubectl expose deployment web --port=80 --target-port=80 --type=ClusterIP
kubectl get service web
kubectl describe service web
```

A Service is created. Its selector is derived from the Deployment’s Pod labels.

## 13. Add a sidecar container.

```bash
kubectl patch deployment web --type='json' -p='[
{"op":"add","path":"/spec/template/spec/containers/-","value":{"name":"sidecar","image":"busybox","command":["sh","-c","while true; do date; sleep 10; done"]}}
]'
kubectl get pods -l app=web
```

Containers in the same Pod share network namespace and can share volumes, but they have separate filesystems and processes.

## 14. Deployment manifest for httpd:2.4 with nodeSelector.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: httpd
  template:
    metadata:
      labels:
        app: httpd
    spec:
      nodeSelector:
        disk: fast
      containers:
        - name: httpd
          image: httpd:2.4
          ports:
            - name: http
              containerPort: 80
```

```bash
kubectl apply -f httpd-deploy.yaml
kubectl get pods
kubectl describe pod <pod-name>
```

If no node has label `disk=fast`, Pods stay Pending.

## 15. Bare Pod versus Deployment-managed Pod.

```bash
kubectl run barebox --image=busybox --command -- sleep 1d
kubectl delete pod barebox
kubectl get pod barebox
kubectl delete pod -l app=web
kubectl get pods -l app=web
```

A bare Pod is gone when deleted. A Deployment-managed Pod is recreated by the ReplicaSet.

## 16. StatefulSet for redis with headless Service.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  clusterIP: None
  selector:
    app: redis
  ports:
    - port: 6379
      name: redis
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7
          ports:
            - containerPort: 6379
              name: redis
```

```bash
kubectl apply -f redis-sts.yaml
kubectl get pods -l app=redis
kubectl run dns --rm -it --image=busybox -- nslookup redis-0.redis.default.svc.cluster.local
```

Stable names are `redis-0`, `redis-1`, and `redis-2`.

## 17. DaemonSet running a busybox agent.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: agent
spec:
  selector:
    matchLabels:
      app: agent
  template:
    metadata:
      labels:
        app: agent
    spec:
      containers:
        - name: agent
          image: busybox
          command: ["sh", "-c", "while true; do echo agent; sleep 60; done"]
```

```bash
kubectl apply -f daemonset.yaml
kubectl get daemonset
kubectl get pods -l app=agent -o wide
```

A DaemonSet runs one Pod per matching node.

## 18. Job that computes once and completes.

```bash
kubectl create job calc --image=perl -- perl -Mbignum=bpi -wle 'print bpi(20)'
kubectl get jobs
kubectl logs job/calc
```

The Job shows `1/1` completions after the command finishes.

## 19. CronJob every minute, suspend it, and list spawned Jobs.

```bash
kubectl create cronjob datejob --image=busybox --schedule='* * * * *' -- date
kubectl get cronjob
kubectl get jobs --watch
kubectl patch cronjob datejob -p '{"spec":{"suspend":true}}'
kubectl get jobs
```

Suspending stops future scheduled Jobs, not already created Jobs.

## 20. Manually run a Job from a CronJob.

```bash
kubectl create job manual-datejob --from=cronjob/datejob
kubectl get job manual-datejob
kubectl logs job/manual-datejob
```

This tests the CronJob template on demand.

## 21. StatefulSet ordered creation/termination versus Deployment.

```bash
kubectl scale statefulset redis --replicas=0
kubectl get pods -w
kubectl scale statefulset redis --replicas=3
kubectl get pods -w
```

StatefulSets create and delete Pods in order by ordinal. Deployments create/delete interchangeable Pods without stable identity.

## 22. Multi-container Pod sharing `emptyDir`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-volume-pod
spec:
  volumes:
    - name: shared
      emptyDir: {}
  containers:
    - name: writer
      image: busybox
      command: ["sh", "-c", "echo hello > /shared/file.txt; sleep 1d"]
      volumeMounts:
        - name: shared
          mountPath: /shared
    - name: reader
      image: busybox
      command: ["sh", "-c", "while true; do cat /shared/file.txt; sleep 30; done"]
      volumeMounts:
        - name: shared
          mountPath: /shared
```

```bash
kubectl apply -f shared-volume-pod.yaml
kubectl logs shared-volume-pod -c reader
```

Both containers mount the same `emptyDir` volume.

## 23. List StorageClasses, PVs, and PVCs.

```bash
kubectl get storageclass
kubectl get pv
kubectl get pvc
```

StorageClasses define dynamic provisioning. PVs are cluster storage resources. PVCs are user claims for storage.

## 24. Mount ConfigMap as volume, each key as a file.

```bash
kubectl create configmap app-config --from-literal=app.properties='mode=dev' --from-literal=message='hello'
cat > cm-volume-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: cm-volume-pod
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "ls -l /config && cat /config/message && sleep 1d"]
      volumeMounts:
        - name: config
          mountPath: /config
  volumes:
    - name: config
      configMap:
        name: app-config
EOF
kubectl apply -f cm-volume-pod.yaml
kubectl logs cm-volume-pod
```

Each ConfigMap key becomes a file.

## 25. Mount one ConfigMap key with `subPath`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cm-subpath-pod
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "cat /app/message.txt; sleep 1d"]
      volumeMounts:
        - name: config
          mountPath: /app/message.txt
          subPath: message
  volumes:
    - name: config
      configMap:
        name: app-config
```

`subPath` is needed when one key must be mounted at one exact file path. Caveat: subPath mounts do not receive live ConfigMap updates.

## 26. Mount Secret as volume and verify decoded files/permissions.

```bash
kubectl create secret generic db-secret --from-literal=username=admin --from-literal=password=secret
cat > secret-vol-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: secret-vol-pod
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "ls -l /secret && cat /secret/username && sleep 1d"]
      volumeMounts:
        - name: secret
          mountPath: /secret
          readOnly: true
  volumes:
    - name: secret
      secret:
        secretName: db-secret
EOF
kubectl apply -f secret-vol-pod.yaml
kubectl logs secret-vol-pod
```

Secret files contain decoded values and are mounted with restrictive permissions.

## 27. StatefulSet PVC per replica and deletion behavior.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-pvc
spec:
  serviceName: redis
  replicas: 3
  selector:
    matchLabels:
      app: redis-pvc
  template:
    metadata:
      labels:
        app: redis-pvc
    spec:
      containers:
        - name: redis
          image: redis:7
          volumeMounts:
            - name: data
              mountPath: /data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

```bash
kubectl apply -f redis-pvc.yaml
kubectl get pvc
kubectl delete statefulset redis-pvc
kubectl get pvc
```

PVCs remain after StatefulSet deletion to protect data.

## 28. Use initContainer to pre-populate a shared volume.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  volumes:
    - name: data
      emptyDir: {}
  initContainers:
    - name: init
      image: busybox
      command: ["sh", "-c", "echo initialized > /data/file.txt"]
      volumeMounts:
        - name: data
          mountPath: /data
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "cat /data/file.txt; sleep 1d"]
      volumeMounts:
        - name: data
          mountPath: /data
```

The initContainer runs first, writes data, and the main container reads it.

## 29. Set `readOnly: true` and prove writes fail.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readonly-volume-pod
spec:
  volumes:
    - name: config
      configMap:
        name: app-config
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "echo test > /config/newfile"]
      volumeMounts:
        - name: config
          mountPath: /config
          readOnly: true
```

```bash
kubectl apply -f readonly-volume-pod.yaml
kubectl logs readonly-volume-pod
```

The write fails because the volume mount is read-only.

## 30. Set `sizeLimit` on `emptyDir`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-limit
spec:
  volumes:
    - name: cache
      emptyDir:
        sizeLimit: 10Mi
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "df -h /cache; sleep 1d"]
      volumeMounts:
        - name: cache
          mountPath: /cache
```

If the container exceeds the limit, writes fail or the Pod may be evicted depending on node storage pressure.

## 31. Create generic Secret and show base64 encoding.

```bash
kubectl create secret generic db-auth --from-literal=username=admin --from-literal=password=secret
kubectl get secret db-auth -o yaml
kubectl get secret db-auth -o jsonpath='{.data.password}' | base64 -d
```

Kubernetes Secret data is base64-encoded, not encrypted by default.

## 32. Create Secret from certificate/key files and identify keys.

```bash
echo 'fake cert' > tls.crt
echo 'fake key' > tls.key
kubectl create secret generic cert-secret --from-file=tls.crt --from-file=tls.key
kubectl get secret cert-secret -o jsonpath='{.data}'
```

The resulting Secret keys are `tls.crt` and `tls.key`.

## 33. Create `kubernetes.io/tls` Secret.

```bash
openssl req -x509 -nodes -newkey rsa:2048 -keyout tls.key -out tls.crt -subj '/CN=example.local' -days 1
kubectl create secret tls web-tls --cert=tls.crt --key=tls.key
kubectl get secret web-tls -o yaml
```

TLS Secrets are consumed by Ingress controllers, Routes, or applications that need certificate/key files.

## 34. Secret volume trade-offs versus env vars.

```bash
kubectl apply -f secret-vol-pod.yaml
kubectl exec secret-vol-pod -- ls -l /secret
```

Secret volumes can be read as files and can update without restarting the Pod. Env vars are simpler but are visible in process environments and require restart to change.

## 35. Use `envFrom` with `secretRef`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-envfrom-pod
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "env | grep -E 'username|password'; sleep 1d"]
      envFrom:
        - secretRef:
            name: db-auth
```

All Secret keys become environment variables.

## 36. Create image-pull Secret and reference it.

```bash
kubectl create secret docker-registry regcred \
  --docker-server=quay.io \
  --docker-username=<quay_user> \
  --docker-password=<quay_password> \
  --docker-email=<email>
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-image-pod
spec:
  imagePullSecrets:
    - name: regcred
  containers:
    - name: app
      image: quay.io/<quay_user>/<private_repo>:latest
```

It is required when pulling from a private registry or when authenticated pulls are needed.

## 37. Selectively mount one Secret key.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: one-secret-key
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "ls -l /secret && cat /secret/password; sleep 1d"]
      volumeMounts:
        - name: secret
          mountPath: /secret
  volumes:
    - name: secret
      secret:
        secretName: db-auth
        items:
          - key: password
            path: password
```

Only the selected key is mounted.

## 38. ClusterIP Service and DNS resolution.

```bash
kubectl expose deployment web --name=web-clusterip --port=80 --target-port=80 --type=ClusterIP
kubectl run dns-test --rm -it --image=busybox -- nslookup web-clusterip.default.svc.cluster.local
```

ClusterIP Services are reachable inside the cluster through DNS.

## 39. NodePort Service and access through Minikube.

```bash
kubectl expose deployment web --name=web-nodeport --port=80 --target-port=80 --type=NodePort
kubectl get svc web-nodeport
minikube service web-nodeport --url
curl $(minikube service web-nodeport --url)
```

NodePort exposes the Service on a port on each node.

## 40. LoadBalancer Service on Minikube.

```bash
kubectl expose deployment web --name=web-lb --port=80 --target-port=80 --type=LoadBalancer
kubectl get svc web-lb
minikube tunnel
```

On Minikube, `EXTERNAL-IP` is often `<pending>` because there is no real cloud load balancer. `minikube tunnel` provides one locally.

## 41. Headless Service returns per-pod records.

```bash
kubectl get svc redis
kubectl run dns-redis --rm -it --image=busybox -- nslookup redis.default.svc.cluster.local
kubectl run dns-redis0 --rm -it --image=busybox -- nslookup redis-0.redis.default.svc.cluster.local
```

A headless Service has no virtual IP. DNS points directly to Pod records.

## 42. Inspect Endpoints/EndpointSlice.

```bash
kubectl get endpoints web-clusterip
kubectl get endpointslice -l kubernetes.io/service-name=web-clusterip
kubectl describe service web-clusterip
```

Endpoints are populated from Pods matching the Service selector and readiness state.

## 43. Cross-namespace access using FQDN.

```bash
kubectl create namespace testns
kubectl run -n testns tmp --rm -it --image=busybox -- nslookup web-clusterip.default.svc.cluster.local
kubectl run -n testns tmp2 --rm -it --image=busybox -- nslookup web-clusterip
```

The FQDN works across namespaces. The short name fails because it searches the current namespace.

## 44. Port-forward Service versus Pod.

```bash
kubectl port-forward service/web-clusterip 8080:80
kubectl port-forward pod/<pod-name> 8081:80
```

Forwarding to a Service targets one backing Pod through the Service. Forwarding to a Pod targets that exact Pod only.

## 45. Service `port`, `targetPort`, and `nodePort`.

```yaml
ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
```

`port` is the Service port inside the cluster. `targetPort` is the container port. `nodePort` is the external node port for NodePort/LoadBalancer Services.

## 46. Verify ClusterIP connectivity from a debug Pod.

```bash
kubectl run tmp --rm -it --image=busybox -- sh
wget -qO- http://web-clusterip.default.svc.cluster.local
nc -vz web-clusterip.default.svc.cluster.local 80
exit
```

This tests connectivity from inside the cluster.

## 47. One Service load-balances across two Deployments sharing a label.

```bash
kubectl create deployment app-a --image=hashicorp/http-echo -- -text='A'
kubectl create deployment app-b --image=hashicorp/http-echo -- -text='B'
kubectl label deployment app-a group=shared
kubectl label deployment app-b group=shared
kubectl patch deployment app-a -p '{"spec":{"template":{"metadata":{"labels":{"group":"shared","app":"app-a"}}}}}'
kubectl patch deployment app-b -p '{"spec":{"template":{"metadata":{"labels":{"group":"shared","app":"app-b"}}}}}'
kubectl expose deployment app-a --name=shared-svc --port=5678 --target-port=5678
kubectl patch service shared-svc -p '{"spec":{"selector":{"group":"shared"}}}'
kubectl get endpoints shared-svc
```

The Service sends traffic to all ready Pods with label `group=shared`.

## 48. Diagnose Pod cannot reach Service.

```bash
kubectl run debug --rm -it --image=busybox -- sh
nslookup web-clusterip.default.svc.cluster.local
wget -S -O- http://web-clusterip.default.svc.cluster.local
exit
kubectl get svc web-clusterip -o yaml
kubectl get endpoints web-clusterip
kubectl get pods -l app=web --show-labels
kubectl describe pod -l app=web
```

Check in order: DNS, endpoints, selector labels, readiness, and port mapping.

## 49. Expose a Service with NodePort and verify.

```bash
kubectl expose deployment web --name=web-np2 --type=NodePort --port=80 --target-port=80
kubectl get svc web-np2
curl $(minikube service web-np2 --url)
```

The NodePort Service is reachable from outside the cluster through the Minikube node.

## 50. Add HTTP livenessProbe to nginx.

```bash
kubectl patch deployment web --type='json' -p='[
{"op":"add","path":"/spec/template/spec/containers/0/livenessProbe","value":{"httpGet":{"path":"/","port":80},"initialDelaySeconds":5,"periodSeconds":10}}
]'
kubectl describe pod -l app=web | grep -A5 Liveness
```

The kubelet restarts the container if the HTTP liveness probe fails repeatedly.

## 51. Add tcpSocket readiness probe to redis.

```bash
kubectl patch statefulset redis --type='json' -p='[
{"op":"add","path":"/spec/template/spec/containers/0/readinessProbe","value":{"tcpSocket":{"port":6379},"initialDelaySeconds":5,"periodSeconds":10}}
]'
kubectl describe pod redis-0 | grep -A5 Readiness
```

The Pod becomes Ready only when TCP port `6379` accepts connections.

## 52. Configure startup probe for around 2-minute boot.

```yaml
startupProbe:
  httpGet:
    path: /
    port: 80
  periodSeconds: 10
  failureThreshold: 12
```

Budget is `10 × 12 = 120 seconds`. During startup probe failure, liveness/readiness are delayed, which protects slow-starting apps.

## 53. Default behavior with no probes.

If no probes are defined, Kubernetes assumes the container is alive after it starts and ready unless it crashes. It will not detect broken application logic, only process exit.

---

# LO5 – Solve problems with application shipping by using containers

## 1. Wrong port mapping for nginx.

Wrong:

```bash
podman run -d --name web -p 80:8080 docker.io/library/nginx
```

Fixed:

```bash
podman rm -f web
podman run -d --name web -p 8080:80 docker.io/library/nginx
curl http://localhost:8080
```

The order is `hostPort:containerPort`. Nginx listens on container port `80`, not `8080`.

## 2. MySQL root password variable missing value.

Wrong:

```bash
podman run -d --name db -e MYSQL_ROOT_PASSWORD docker.io/library/mysql
```

Fixed:

```bash
podman rm -f db
podman run -d --name db -e MYSQL_ROOT_PASSWORD=secret docker.io/library/mysql
podman logs db
```

`MYSQL_ROOT_PASSWORD` must have a value. Without it, MySQL initialization exits.

## 3. `--network host` ignores `-p`.

Wrong:

```bash
podman run -d --name app --network host -p 8080:80 docker.io/library/nginx
```

Fixed:

```bash
podman rm -f app
podman run -d --name app -p 8080:80 docker.io/library/nginx
```

With host networking, the container uses the host network namespace directly, so port publishing is unnecessary and effectively ignored.

## 4. Busybox exits immediately.

Wrong:

```bash
podman run -d --name c1 docker.io/library/busybox
```

Fixed:

```bash
podman rm -f c1
podman run -d --name c1 docker.io/library/busybox sleep 1d
podman ps
```

Busybox has no long-running default process here. Add a command like `sleep 1d`.

## 5. `--rm` plus detached job removes logs.

Wrong:

```bash
podman run --rm -d --name job docker.io/library/alpine echo hello
podman logs job
```

Fixed:

```bash
podman run --name job docker.io/library/alpine echo hello
podman logs job
podman rm job
```

With `--rm`, the container is removed when it exits, so `podman logs job` has nothing to read.

## 6. MySQL with 8 MB memory limit.

Wrong:

```bash
podman run -d -p 8080:80 --memory 8m docker.io/library/mysql
```

Fixed:

```bash
podman run -d --name mysqlok --memory 512m -e MYSQL_ROOT_PASSWORD=secret docker.io/library/mysql
podman ps
```

`8m` is far too low for MySQL. The DB is killed or fails to initialize.

## 7. Container name resolution fails on default network.

Wrong:

```bash
podman run -d --name a docker.io/library/alpine sleep 1d
podman run -d --name b docker.io/library/alpine sleep 1d
podman exec a ping b
```

Fixed:

```bash
podman network create fixnet
podman run -d --name a --network fixnet docker.io/library/alpine sleep 1d
podman run -d --name b --network fixnet docker.io/library/alpine sleep 1d
podman exec a ping -c 1 b
```

Podman DNS name resolution works on user-defined networks, not reliably on the default network.

## 8. Bind mount files invisible or SELinux denied.

Wrong:

```bash
podman run -d --name web -v ./html:/usr/share/nginx/html docker.io/library/nginx
```

Fixed:

```bash
mkdir -p html
echo hello > html/index.html
podman run -d --name web -p 8080:80 -v "$PWD/html:/usr/share/nginx/html:Z" docker.io/library/nginx
```

On SELinux systems, `:Z` relabels the bind mount so the container can access it.

## 9. Containerfile mistakes: general method.

```bash
podman build -t broken-test .
podman build --no-cache -t broken-test .
podman run --rm broken-test
podman logs <container>
```

Read the first real error. Most Containerfile problems are missing package index updates, wrong command form, wrong user/permissions, broken PATH, or bad layer ordering.

## 10. Debian package install without `apt-get update`.

Wrong:

```Dockerfile
FROM debian:12
RUN apt-get install -y nginx
```

Fixed:

```Dockerfile
FROM debian:12
RUN apt-get update && apt-get install -y --no-install-recommends nginx && rm -rf /var/lib/apt/lists/*
```

`apt-get update` downloads package indexes. Without it, packages may not be found.

## 11. Shell form CMD handles signals poorly.

Wrong:

```Dockerfile
CMD python app.py
```

Fixed:

```Dockerfile
CMD ["python", "app.py"]
```

Exec form runs Python as PID 1 directly, so it handles signals more cleanly.

## 12. Bad layer order for npm install.

Wrong:

```Dockerfile
FROM node:20
COPY . /app
WORKDIR /app
RUN npm install
```

Fixed:

```Dockerfile
FROM node:20
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
```

Copy dependency files first so `npm install` is cached unless dependencies change.

## 13. PATH overwritten.

Wrong:

```Dockerfile
ENV PATH=/app/bin
```

Fixed:

```Dockerfile
ENV PATH=/app/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

The bad PATH removes system binary directories, so commands like `sh` or `curl` cannot be found.

## 14–15. EXPOSE mismatch with actual server port.

Wrong:

```Dockerfile
FROM ubuntu:24.04
EXPOSE 8080
CMD ["python3", "-m", "http.server", "3000"]
```

Fixed:

```Dockerfile
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y --no-install-recommends python3 && rm -rf /var/lib/apt/lists/*
EXPOSE 3000
CMD ["python3", "-m", "http.server", "3000"]
```

Run:

```bash
podman build -t pyserver .
podman run -d --name pyserver -p 8080:3000 pyserver
curl http://localhost:8080
```

`EXPOSE` is documentation. It does not publish ports. The app listens on `3000`, so map host `8080` to container `3000`.

## 16. Huge Debian image due to build packages/cache.

Wrong:

```Dockerfile
RUN apt-get update && apt-get install -y build-essential
```

Fixed:

```Dockerfile
RUN apt-get update \
 && apt-get install -y --no-install-recommends build-essential \
 && rm -rf /var/lib/apt/lists/*
```

Install and cleanup in the same layer.

## 17. USER/COPY ordering causes permission errors.

Wrong:

```Dockerfile
FROM python:3.11
USER appuser
COPY requirements.txt /app/requirements.txt
RUN pip install -r /app/requirements.txt
```

Fixed:

```Dockerfile
FROM python:3.11
RUN useradd -m appuser && mkdir -p /app && chown appuser:appuser /app
WORKDIR /app
COPY --chown=appuser:appuser requirements.txt /app/requirements.txt
USER appuser
RUN pip install --user -r /app/requirements.txt
```

Create the user first, set ownership, then switch user.

## 18. Go final image too large.

Wrong:

```Dockerfile
FROM golang:1.22
COPY . /src
WORKDIR /src
RUN go build -o app .
CMD ["/src/app"]
```

Fixed:

```Dockerfile
FROM golang:1.22 AS build
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o app .

FROM scratch
COPY --from=build /src/app /app
CMD ["/app"]
```

The final image contains only the binary, not the Go compiler and source tree.

## 19. Pod stuck Pending diagnosis.

```bash
kubectl get pod <pod>
kubectl describe pod <pod>
kubectl get events --sort-by=.lastTimestamp
```

Look at Events. Common causes: insufficient CPU/memory, missing PVC, nodeSelector mismatch, taints/tolerations, or no available nodes.

## 20. Broken YAML due to removed spaces.

```bash
kubectl apply -f broken.yaml --dry-run=server --validate=true
kubectl apply -f fixed.yaml
```

YAML is indentation-sensitive. The error usually says where mapping/sequence indentation broke.

## 21. ImagePullBackOff.

```bash
kubectl describe pod <pod>
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].state}'
```

Fix image typo/tag:

```bash
kubectl set image deployment/<deploy> <container>=nginx:1.27
```

For private registry, create and reference `imagePullSecrets`.

## 22. CrashLoopBackOff.

```bash
kubectl logs <pod> --previous
kubectl describe pod <pod>
```

`--previous` shows logs from the last crashed container instance. Fix the command, config, missing env, or app error causing the crash.

## 23. OOMKilled.

```bash
kubectl describe pod <pod> | grep -A5 -i oom
kubectl get pod <pod> -o yaml | grep -A10 lastState
```

Fix by raising memory limits or reducing app memory usage.

## 24. Pods never become Ready because readiness probe fails.

```bash
kubectl describe pod <pod> | grep -A10 Readiness
kubectl logs <pod>
```

Fix the probe path, port, initial delay, or app endpoint.

## 25. Service returns nothing due to label mismatch.

```bash
kubectl get svc <svc> -o yaml
kubectl get endpoints <svc>
kubectl get pods --show-labels
```

If endpoints are empty, fix the Service selector so it matches Pod labels.

## 26. Deployment selector does not match template labels.

Bad manifests where `spec.selector.matchLabels` differs from `spec.template.metadata.labels` fail validation.

Fixed pattern:

```yaml
selector:
  matchLabels:
    app: demo
template:
  metadata:
    labels:
      app: demo
```

The selector must match the Pod template labels.

## 27. Requests exceed node capacity.

```bash
kubectl describe pod <pod>
kubectl describe node
```

Events show `Insufficient cpu` or `Insufficient memory`. Fix by lowering requests or adding a larger node.

## 28. PVC does not exist.

```bash
kubectl describe pod <pod>
kubectl get pvc
```

Create the missing PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 1Gi
```

## 29. Ubuntu container completes because no long-running command.

```yaml
containers:
  - name: ubuntu
    image: ubuntu:24.04
    command: ["sleep", "1d"]
```

Containers need a foreground process. If the process exits, the container ends.

## 30. Use events to triage timeline.

```bash
kubectl get events --sort-by=.lastTimestamp
kubectl events --for pod/<pod>
```

Events show scheduling, pulling, mounting, starting, killing, and probe failures over time.

## 31. Exec into a Pod and debug DNS.

```bash
kubectl exec -it <pod> -- sh
cat /etc/resolv.conf
nslookup kubernetes.default.svc.cluster.local
exit
```

This checks cluster DNS configuration from inside the Pod.

## 32. Ephemeral debug container for distroless/no-shell container.

```bash
kubectl debug -it <pod> --image=busybox --target=<container> -- sh
```

Use this when the original image has no shell or debugging tools.

## 33. Throwaway Pod for Service connectivity.

```bash
kubectl run tmp --rm -it --image=busybox -- sh
wget -qO- http://<service-name>:<port>
nslookup <service-name>
exit
```

This tests connectivity from inside the same namespace.

## 34. Node NotReady.

```bash
kubectl get nodes
kubectl describe node <node>
minikube ssh -- sudo journalctl -u kubelet -n 100
minikube logs
```

Check kubelet, disk pressure, memory pressure, CNI/network plugin, container runtime, and node conditions.

## 35. Pod stuck Terminating.

```bash
kubectl describe pod <pod>
kubectl get pod <pod> -o yaml | grep -A5 finalizers
kubectl delete pod <pod> --grace-period=0 --force
```

Finalizers and grace periods can delay deletion. Force deletion risks leaving processes, volumes, or external resources in an inconsistent state.

## 36. Wrong apiVersion/kind pairing.

Wrong example:

```yaml
apiVersion: apps/v1
kind: Pod
```

Fixed:

```yaml
apiVersion: v1
kind: Pod
```

Pods belong to core API group `v1`; Deployments belong to `apps/v1`.

## 37. `CreateContainerConfigError` from missing ConfigMap key.

```bash
kubectl describe pod <pod>
kubectl get configmap <cm> -o yaml
```

Fix by adding the missing key or changing `configMapKeyRef.key` to an existing key.

## 38. Secret in different namespace cannot be mounted.

```bash
kubectl get secret -A | grep <secret>
kubectl get pod <pod> -n <namespace>
```

Secrets are namespace-scoped. Create the Secret in the same namespace as the Pod.

## 39. Rollout stuck with `ProgressDeadlineExceeded`.

```bash
kubectl rollout status deployment/<deploy>
kubectl describe deployment <deploy>
kubectl get rs,pods
kubectl describe pod <bad-pod>
```

Find the cause: image pull failure, probe failure, crash, resource shortage, or scheduling issue. Fix the root cause and re-apply/update.

## 40. Catch indentation/field typos before applying.

```bash
kubectl apply -f manifest.yaml --dry-run=server --validate=true
```

Examples: `imagePullpolicy` should be `imagePullPolicy`; misnested `ports` must be under the container.

## 41. App cannot reach database Service.

```bash
kubectl get pod <app-pod>
kubectl get svc <db-svc>
kubectl get endpoints <db-svc>
kubectl exec -it <app-pod> -- nslookup <db-svc>
kubectl exec -it <app-pod> -- nc -vz <db-svc> <port>
```

Verify: app Pod running → DB Service exists → endpoints populated → DNS resolves → correct port.

## 42. Read container state, lastState, and restartCount with jsonpath.

```bash
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].state}'
echo
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState}'
echo
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].restartCount}'
echo
```

This identifies repeated restarts and their previous failure reason.

## 43. Compare OpenShift Route to Ingress.

OpenShift Route is OpenShift’s built-in way to expose HTTP/HTTPS services externally. It is integrated with OpenShift routers and TLS options. Kubernetes Ingress is the standard Kubernetes object for HTTP routing, but it requires an Ingress controller. Route is more opinionated and integrated; Ingress is more portable across Kubernetes distributions.

---

# LO6 – Evaluate selected container orchestration systems – theoretical

## 1. Kubernetes networking versus Docker networking.

Kubernetes gives every Pod its own IP and expects flat Pod-to-Pod connectivity across nodes. Services provide stable virtual access to changing Pods. Docker/Podman networking is usually host-local, bridge-based, and simpler. Kubernetes networking is better for multi-node orchestration; Docker networking is easier for single-host development.

## 2. Kubernetes storage abstraction versus Podman storage plugins.

Kubernetes is more flexible for stateful workloads because it has CSI, StorageClasses, PersistentVolumes, PersistentVolumeClaims, and dynamic provisioning. Podman volumes are useful locally, but Kubernetes integrates storage with scheduling, cloud providers, and automatic provisioning.

## 3. Operator pattern versus plain manifests.

Operators combine CRDs and controllers to manage complex applications automatically. They handle backups, upgrades, failover, scaling, and recovery logic. Plain manifests are simpler but require humans or scripts to perform operational tasks.

## 4. Plain Kubernetes versus OpenShift S2I/developer console.

Plain Kubernetes optimizes for portability and direct control through manifests and `kubectl`. OpenShift optimizes for enterprise developer productivity with Source-to-Image, templates, integrated registry, web console, routes, and stricter defaults.

## 5. Migrating from OpenShift to Kubernetes.

Migration is possible, but OpenShift-specific objects such as Routes, BuildConfigs, ImageStreams, SecurityContextConstraints, and S2I workflows must be replaced. Standard Kubernetes manifests move easily; OpenShift-integrated pipelines and security policies require rework.

## 6. CI/CD agents inside Kubernetes versus dedicated VMs.

Kubernetes agents scale dynamically and isolate builds in Pods. They are efficient for bursty workloads. Dedicated VMs are simpler and more predictable but waste capacity and scale manually. Kubernetes adds noisy-neighbor risk unless resource limits and node isolation are configured.

## 7. Kubernetes cloud integration and dynamic capacity.

Kubernetes integrates with cloud load balancers, storage, autoscaling, IAM, node provisioning, and container registries. Cluster autoscalers and managed Kubernetes services can add/remove nodes based on workload demand.

## 8. OpenShift security versus vanilla Kubernetes.

OpenShift ships stricter defaults: restricted security contexts, integrated OAuth/RBAC, image policies, routes, registry, monitoring, and vendor-supported hardening. Regulated enterprises often choose it because it reduces integration work and provides supported compliance-oriented defaults.

## 9. OpenShift versus Kubernetes learning curve.

OpenShift adds concepts, but it also gives guardrails and built-in tools. For a team new to containers, it can help by reducing platform assembly work. It can hinder if the team first needs to understand raw Kubernetes fundamentals.

## 10. Full Kubernetes versus k3s footprint.

Full Kubernetes is heavier and better for large, highly available, enterprise clusters. k3s is lightweight and good for small on-premise, edge, or lab deployments. Full Kubernetes is overkill when the workload is small, the team is tiny, and high availability is not required.

## 11. Docker Compose on one host versus Kubernetes for a small team.

A small team should start with Compose or Podman Compose if one host is enough. It is cheaper and simpler. Move to Kubernetes when the app needs self-healing, scaling, rolling updates, service discovery, multi-node resilience, and stronger operational control.

## 12. Vendor lock-in: Kubernetes versus OpenShift.

Kubernetes is open source and widely supported across clouds and vendors. OpenShift is Kubernetes-based but adds Red Hat-specific APIs and tooling. OpenShift improves enterprise support but increases migration work if moving away later.

## 13. Recommendation: OpenShift over vanilla Kubernetes.

Choose OpenShift when the company wants an integrated, vendor-supported platform with developer console, builds, registry, routing, monitoring, RBAC, security policies, and enterprise support. It costs more but saves platform engineering effort.

## 14. Kubernetes storage versus regular VM storage.

VM storage is usually attached manually to a server. Kubernetes storage is declared through PVCs and provisioned dynamically through StorageClasses. Kubernetes is better for automated scheduling and rescheduling, but stateful apps still need careful backup and performance planning.

## 15. Startup with two engineers shipping quickly and cheaply.

Use Docker Compose or Podman Compose on a managed VM. It has the lowest operational overhead and cost. Kubernetes is too much unless they already need multiple nodes, autoscaling, or strict high availability.

## 16. Docker versus Podman architecture and rootless security.

Docker traditionally uses a central daemon. Podman is daemonless and can run rootless more naturally. A security-conscious team may prefer Podman because containers do not require a root-owned daemon as the control point.

## 17. Self-managed Kubernetes versus managed Kubernetes day-2 overhead.

Self-managed Kubernetes requires cluster upgrades, etcd backups, node patching, networking, storage, monitoring, and incident response. Managed Kubernetes offloads much of the control-plane maintenance, but the team still owns workloads, security, cost, and observability.

## 18. Media-streaming company with spiky global traffic.

Use managed Kubernetes plus CDN and cloud autoscaling. Kubernetes handles container orchestration and horizontal scaling, while the CDN absorbs global traffic close to users. Self-hosted single-node solutions would collapse under unpredictable spikes.

## 19. Migrating from Docker Swarm or Compose to Kubernetes.

Migrate when scaling, resilience, rolling updates, service discovery, and ecosystem integrations justify the complexity. Do not migrate just because Kubernetes is popular. The migration cost is real: manifests, CI/CD, monitoring, secrets, storage, and team training.

## 20. Kubernetes for microservices.

Yes, if the system has multiple services that need orchestration, service discovery, scaling, rolling updates, and resilience. Kubernetes is strong for microservices because it standardizes deployment and recovery. For two tiny services, it is probably overengineering.

## 21. Kubernetes for enterprise-grade applications.

Yes, when the enterprise can operate it properly. Kubernetes has scalability, broad community support, portability, and a rich ecosystem. The downside is operational complexity; without platform discipline, it becomes YAML-powered chaos.

## 22. OpenShift for microservices.

Yes, especially in companies that want Kubernetes plus integrated developer and operations tooling. OpenShift supports orchestration, resilience, service routing, builds, CI/CD integrations, and stricter security defaults. It is heavier than vanilla Kubernetes but more complete.

## 23. OpenShift for enterprise-grade applications.

Yes. OpenShift is a strong fit for enterprise workloads because it combines Kubernetes scalability with vendor support, security hardening, monitoring, registry, routing, developer tools, and lifecycle management. The trade-off is cost and stronger Red Hat platform dependency.

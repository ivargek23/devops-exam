# LO3 — Multi-container Apps, Networking, Storage and Security

# How to use this file during the exam

Start with Ctrl+F and search the noun from the task, for example `restart`, `port`, `volume`, `Containerfile`, `rollout`, `ImagePullBackOff`, or `OpenShift`.

For practical tasks, always do three things: **perform the task → verify it → take a screenshot showing the command and proof**.

---

## QUICK KNOWLEDGE / COMMAND PATTERNS

## 4. LO3 — Multi-container apps with Podman

### Pod with app + DB sharing localhost

```bash
podman pod create --name apppod -p 8080:80
podman run -d --pod apppod --name web docker.io/library/nginx:latest
podman run -d --pod apppod --name sidecar docker.io/library/alpine:3.20 sleep 1d
podman exec sidecar wget -qO- http://localhost:80
podman pod inspect apppod
```

Containers in a pod share network namespace, so they reach each other via localhost.

### App + Postgres on user network

```bash
podman network create backend
podman volume create pgdata
podman run -d --name db --network backend -e POSTGRES_PASSWORD=pass -e POSTGRES_DB=appdb -v pgdata:/var/lib/postgresql/data docker.io/library/postgres:16
podman run --rm --network backend docker.io/library/alpine:3.20 sh -c 'apk add --no-cache bind-tools >/dev/null && nslookup db'
```

### Named volume persistence proof

```bash
podman volume create demo-data
podman run --rm -v demo-data:/data alpine sh -c 'echo saved > /data/file.txt'
podman run --rm -v demo-data:/data alpine cat /data/file.txt
```

### Podman secrets

```bash
echo 'superpass' | podman secret create db_password -
podman run -d --name secret-test --secret db_password docker.io/library/alpine:3.20 sleep 1d
podman exec secret-test ls /run/secrets
podman exec secret-test cat /run/secrets/db_password
```

Benefit: secret is not directly exposed in `podman inspect` like plain env values.

### Compose template: app + DB

```yaml
services:
  db:
    image: docker.io/library/postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: appdb
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d appdb"]
      interval: 5s
      timeout: 3s
      retries: 10

  app:
    image: docker.io/library/adminer:latest
    ports:
      - "8080:8080"
    depends_on:
      db:
        condition: service_healthy
    networks:
      - backend
      - frontend

networks:
  frontend: {}
  backend:
    internal: true

volumes:
  pgdata: {}
```

```bash
podman compose up -d
podman compose ps
podman compose logs
podman compose down
podman compose down --volumes   # also removes named volumes created by the stack
```

Only publish the app port. Do not publish DB ports unless needed.

### env_file

```yaml
env_file:
  - db.env
```

### Scale service

```bash
podman compose up -d --scale app=3
```

Requests are distributed only if a fronting service/proxy/load balancer exists. Plain port mapping may conflict if every replica tries to bind the same host port.

### Bind mount vs named volume

Bind mount: exact host path, good for config/dev files. Named volume: managed by container engine, good for persistent app/db data.

### Volume backup/restore

```bash
podman volume create myvol
podman run --rm -v myvol:/data -v "$PWD":/backup alpine tar czf /backup/myvol.tar.gz -C /data .
podman volume create restored
podman run --rm -v restored:/data -v "$PWD":/backup alpine tar xzf /backup/myvol.tar.gz -C /data
```

### Reverse proxy idea

Nginx shares network with app. Host traffic goes to nginx; nginx proxies to app by container name.

### Generate/play kube YAML

```bash
podman generate kube apppod > apppod.yaml
podman kube play apppod.yaml
podman kube play --down apppod.yaml
```

Bridge from Podman pods to Kubernetes-style manifests.

---

## FULL SOLVED PRACTICE QUESTIONS

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

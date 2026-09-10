# LO3 — Exam Cheat Sheet

> Tasks mirror the official LO3 practice-question set. Commands are concise study examples.

# 1. POD: APP + DATABASE THROUGH LOCALHOST

## CREATE POD

```bash
podman pod create --name apppod -p 8080:80
```

## ADD CONTAINERS

```bash
podman run -d --pod apppod --name app APP_IMAGE
podman run -d --pod apppod --name db DB_IMAGE
```

## VERIFY

```bash
podman pod ps
podman pod inspect apppod
```

From `app`, contact the DB using:

```text
localhost:DB_PORT
```

## EXPLAIN

Containers in a pod share the network namespace, so they use the same IP and can communicate over `localhost`.

---

# 2. CUSTOM NETWORK + POSTGRES NAME RESOLUTION

```bash
podman network create backend
```

```bash
podman run -d \
  --name db \
  --network backend \
  -e POSTGRES_USER=app \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=appdb \
  postgres
```

```bash
podman run -d \
  --name app \
  --network backend \
  APP_IMAGE
```

## VERIFY

```bash
podman network inspect backend
podman exec app ping -c 1 db
```

## EXPLAIN

On a user-defined network, containers can normally resolve each other by container name.

---

# 3. TWO-TIER STACK

Pattern:

```text
app container
    ↓ name-based connection
database container
```

Checklist:

```bash
podman network create appnet
podman run ... --name db --network appnet ...
podman run ... --name app --network appnet -p HOST:APP_PORT ...
podman ps
podman logs app
podman logs db
```

Do not publish the DB unless the task explicitly requires host access.

---

# 4. DATABASE NAMED VOLUME PERSISTENCE

```bash
podman volume create dbdata
```

Run DB:

```bash
podman run -d \
  --name db \
  -v dbdata:/var/lib/postgresql/data \
  ...
```

Create test data.

Then:

```bash
podman rm -f db
```

Recreate with the same volume:

```bash
podman run -d \
  --name db \
  -v dbdata:/var/lib/postgresql/data \
  ...
```

## PROVE

Query the data again.

## EXPLAIN

The container changed; the volume did not.

---

# 5. PODMAN SECRET

Create:

```bash
printf '%s' 'secret-password' | podman secret create db_password -
```

List:

```bash
podman secret ls
```

Use:

```bash
podman run --secret db_password IMAGE
```

## EXPLAIN

Secrets avoid placing sensitive values directly in the image or command line. The application/image must know how to read the mounted secret.

---

# 6. COMPOSE APP + DB

`compose.yaml`:

```yaml
services:
  db:
    image: postgres
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb
    networks:
      - backend

  app:
    image: APP_IMAGE
    ports:
      - "8080:80"
    networks:
      - backend

networks:
  backend:
```

Run:

```bash
podman compose up -d
```

Verify:

```bash
podman compose ps
```

---

# 7. DB INTERNAL / ONLY APP EXPOSED

```yaml
services:
  db:
    image: postgres
    networks:
      - backend

  app:
    image: APP_IMAGE
    ports:
      - "8080:80"
    networks:
      - frontend
      - backend

networks:
  frontend:
  backend:
    internal: true
```

## VERIFY

```bash
podman compose ps
```

There should be no host port published for `db`.

## EXPLAIN

Only the app needs external access. Keeping the DB internal reduces attack surface.

---

# 8. HEALTHCHECK + DEPENDS_ON

```yaml
services:
  db:
    image: postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 3s
      retries: 10

  app:
    image: APP_IMAGE
    depends_on:
      db:
        condition: service_healthy
```

## EXPLAIN

Process started ≠ service ready.

The app waits until the DB healthcheck succeeds.

---

# 9. env_file

`db.env`:

```text
POSTGRES_USER=app
POSTGRES_PASSWORD=secret
POSTGRES_DB=appdb
```

Compose:

```yaml
services:
  db:
    image: postgres
    env_file:
      - db.env
```

---

# 10. SCALE COMPOSE SERVICE

```bash
podman compose up -d --scale app=3
```

Verify:

```bash
podman compose ps
```

## EXPLAIN

This creates multiple service replicas.

Request distribution requires a proxy/load-balancing mechanism; do not assume fixed host-port publishing can be duplicated across all replicas.

---

# 11. BIND MOUNT + NAMED VOLUME IN SAME CONTAINER

```yaml
services:
  app:
    image: APP_IMAGE
    volumes:
      - ./config:/app/config:ro
      - appdata:/app/data

volumes:
  appdata:
```

## EXPLAIN

Bind mount → host-managed config.

Named volume → application data managed by Podman.

---

# 12. BACK UP NAMED VOLUME

Assume volume:

```text
appdata
```

Backup:

```bash
podman run --rm \
  -v appdata:/data:ro \
  -v "$PWD":/backup \
  alpine \
  tar czf /backup/appdata.tar.gz -C /data .
```

Create new volume:

```bash
podman volume create appdata-restored
```

Restore:

```bash
podman run --rm \
  -v appdata-restored:/data \
  -v "$PWD":/backup \
  alpine \
  tar xzf /backup/appdata.tar.gz -C /data
```

---

# 13. NGINX REVERSE PROXY

Architecture:

```text
HOST:8080 → nginx → app:APP_PORT
```

Shared network:

```bash
podman network create frontend
```

Run app on it, then nginx on the same network.

Nginx upstream uses the app container/service name:

```nginx
location / {
    proxy_pass http://app:8080;
}
```

## EXPLAIN

Only nginx needs a published host port.

---

# 14. DATABASE ADMIN CONTAINER

Attach Adminer/pgAdmin to the DB network.

Pattern:

```bash
podman run -d \
  --name admin \
  --network backend \
  -p HOST_PORT:CONTAINER_PORT \
  ADMIN_IMAGE
```

Use `db` as the database hostname if the DB container/service is named `db`.

---

# 15. GENERATE KUBERNETES YAML

```bash
podman generate kube apppod > apppod.yaml
```

## EXPLAIN

Exports a Podman pod into Kubernetes-style YAML, bridging LO3 container/pod concepts into LO4 Kubernetes resources.

---

# 16. PLAY KUBERNETES YAML WITH PODMAN

```bash
podman kube play apppod.yaml
```

Tear down:

```bash
podman kube play --down apppod.yaml
```

---

# 17. CPU/MEMORY LIMITS IN COMPOSE

Example:

```yaml
services:
  app:
    image: APP_IMAGE
    mem_limit: 256m
    cpus: 0.5
```

Verify:

```bash
podman stats
```

If your Compose implementation rejects a field, inspect the supported Compose syntax on the exam VM before assuming it is applied.

---

# 18. INSPECT POD NAMESPACES

```bash
podman pod inspect apppod
```

## EXPLAIN

Pod containers share network and normally IPC namespaces. PID is not shared by default unless configured.

---

# 19. NAME RESOLUTION FAILS ON DIFFERENT NETWORKS

Diagnosis:

```bash
podman inspect app
podman inspect db
podman network ls
```

Fix:

```bash
podman network connect backend app
```

Test:

```bash
podman exec app ping -c 1 db
```

---

# 20. ONE CONTAINER ON TWO NETWORKS

```bash
podman network create frontend
podman network create backend
```

```bash
podman run -d \
  --name app \
  --network frontend \
  APP_IMAGE
```

Attach second network:

```bash
podman network connect backend app
```

DB only on backend.

## EXPLAIN

The app can talk to both sides; the DB is not directly present on the frontend network.

---

# 21. COMPOSE OBSERVABILITY

Status:

```bash
podman compose ps
```

Logs:

```bash
podman compose logs
```

Follow:

```bash
podman compose logs -f
```

Specific service:

```bash
podman compose logs app
```

---

# 22. SHARED NAMED VOLUME ACROSS REPLICAS

Example concept:

```yaml
services:
  app:
    volumes:
      - shared:/app/shared

volumes:
  shared:
```

## RISK

Multiple replicas writing concurrently can cause:
- race conditions
- file corruption
- application-level consistency problems

Shared read-write storage must be safe for concurrent access.

---

# 23. COMPOSE DOWN VS --volumes

Remove stack:

```bash
podman compose down
```

Named volumes normally remain.

Remove stack and volumes:

```bash
podman compose down --volumes
```

## WARNING

Removing volumes can delete persistent application data.

---

# OFFICIAL DRUPAL + POSTGRES PATTERN

Create network:

```bash
podman network create drupal-net
```

Database:

```bash
podman run -d \
  --name postgres \
  --network drupal-net \
  -e POSTGRES_PASSWORD=my-secret-pw \
  -e POSTGRES_DB=drupal \
  -e POSTGRES_USER=drupal_user \
  -e PGPASSWORD=drupal_password \
  docker.io/library/postgres
```

Drupal:

```bash
podman run -d \
  --name drupal \
  --network drupal-net \
  -p 8080:80 \
  docker.io/library/drupal
```

During Drupal setup:

```text
DB host: postgres
DB name: drupal
DB user: drupal_user
DB password: my-secret-pw
```

Important:

```text
POSTGRES_PASSWORD = database password created by the image
PGPASSWORD = client-side PostgreSQL password variable
```

---

# DRUPAL BIND MOUNT

Create host directory:

```bash
sudo mkdir -p /drupal_data
```

For Drupal upload/files persistence, a practical mount target is:

```text
/var/www/html/sites/default/files
```

Example:

```bash
podman run -d \
  --name drupal \
  --network drupal-net \
  -p 8080:80 \
  -v /drupal_data:/var/www/html/sites/default/files:Z \
  docker.io/library/drupal
```

If permissions block writes, adjust ownership/permissions appropriately for the container user.

Verify:

```bash
podman exec drupal \
  sh -c 'touch /var/www/html/sites/default/files/write-test'
```

Host:

```bash
ls -l /drupal_data
```

## EXPLAIN

The file should appear on the host because the container path is bind-mounted to `/drupal_data`.

---

# FAST TROUBLESHOOTING

## APP CANNOT REACH DB

```bash
podman ps
podman network inspect NETWORK
podman logs db
podman logs app
podman exec app ping -c 1 db
```

Check:
1. both containers running
2. same network
3. correct DB hostname
4. correct port
5. correct credentials
6. DB ready

---

## DATA DISAPPEARS AFTER DB RECREATE

Check whether the DB uses:

```text
named volume or bind mount
```

Container writable layer is not persistent storage.

---

## DB EXPOSED TO HOST BY MISTAKE

Remove its `ports:` / `-p`.

Internal containers communicate over container networks; the DB usually does not need a host port.

---

# GOLDEN RULES

```text
pod → localhost communication

custom network → container-name DNS

publish only services that need external access

database data → volume

host-managed configuration → bind mount

container removed ≠ volume removed

healthcheck → service is actually ready

depends_on alone is not the same as application readiness unless health is used

frontend/backend networks → segmentation

podman compose down ≠ delete named volumes

podman compose down --volumes → volumes removed
```

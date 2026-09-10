# LO3 — Application Delivery with Containers, Networking, and Component Security

> Coverage follows the LO3 practice-question set. Commands and explanations below are study guidance for executing those tasks.

# 1. Core idea

LO3 is about delivering a multi-container application safely and predictably.

The main pieces are:

```text
Application container
        ↓
container network / pod
        ↓
Database container
        ↓
persistent storage
```

You should understand:
- how containers find and reach each other
- when to use a pod versus a user-defined network
- how to persist data
- how to keep databases internal
- how Compose describes multi-container applications
- how secrets/configuration are passed
- how to troubleshoot communication between services

---

# 2. User-defined networks

Create:

```bash
podman network create backend
```

Run two containers on it:

```bash
podman run -d --name app --network backend IMAGE
podman run -d --name db --network backend IMAGE
```

Containers on the same user-defined network can normally resolve each other by container name.

Example:

```bash
podman exec app ping -c 1 db
```

Why use names instead of IP addresses?

Container IP addresses can change when containers are recreated. Names are more stable.

---

# 3. Pods

A Podman pod groups containers that share selected Linux namespaces.

Create a pod with a published port:

```bash
podman pod create --name apppod -p 8080:80
```

Add containers:

```bash
podman run -d --pod apppod --name app IMAGE
podman run -d --pod apppod --name db IMAGE
```

Containers in the same pod share the network namespace, so they can communicate through `localhost`.

Example:

```text
app → localhost:5432 → database
```

This differs from two containers on a user-defined network, where communication normally uses container names.

---

# 4. Pod vs custom network

## Pod

Use when containers are tightly coupled and should share network identity.

```text
same IP
shared ports
localhost communication
```

## User-defined network

Use when services should remain separate containers with separate network identities.

```text
app → db:5432
```

This model is closer to typical multi-service deployment.

---

# 5. PostgreSQL container basics

Typical PostgreSQL image initialization variables:

```text
POSTGRES_USER
POSTGRES_PASSWORD
POSTGRES_DB
```

Example:

```bash
podman run -d \
  --name postgres \
  --network backend \
  -e POSTGRES_USER=drupal_user \
  -e POSTGRES_PASSWORD=my-secret-pw \
  -e POSTGRES_DB=drupal \
  docker.io/library/postgres
```

The practice sheet also supplies:

```text
PGPASSWORD=drupal_password
```

`PGPASSWORD` is a PostgreSQL client environment variable. It does not replace `POSTGRES_PASSWORD` as the server initialization password.

---

# 6. Drupal + PostgreSQL

Basic architecture:

```text
Browser
   ↓
published Drupal port
   ↓
Drupal container
   ↓
custom Podman network
   ↓
PostgreSQL container
```

Only Drupal needs to be exposed to the host.

The database normally does not need a published host port.

Example network:

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

During Drupal setup, use:

```text
Database host: postgres
Database name: drupal
Database user: drupal_user
Database password: my-secret-pw
```

because `postgres` is the database container name on the shared network.

---

# 7. Bind mounts

A bind mount maps a host path directly into a container.

General syntax:

```bash
-v HOST_PATH:CONTAINER_PATH
```

Example:

```bash
-v /drupal_data:/var/www/html/sites/default/files
```

Useful for:
- host-managed configuration
- application files
- persistence
- inspecting data directly from the host

On SELinux systems, `:Z` or `:z` may be required:

```bash
-v /drupal_data:/var/www/html/sites/default/files:Z
```

---

# 8. Named volumes

Create:

```bash
podman volume create pgdata
```

Use:

```bash
podman run -d \
  --name db \
  -v pgdata:/var/lib/postgresql/data \
  postgres
```

Named volumes are managed by Podman.

Use them when you want persistent application data without caring about an exact host directory.

---

# 9. Bind mount vs named volume

## Bind mount

```text
Host path ↔ container path
```

Best for:
- configuration files
- development files
- direct host access

## Named volume

```text
Podman-managed storage ↔ container path
```

Best for:
- database data
- application-managed persistent data

Named volumes are usually cleaner and more portable for container-managed data.

---

# 10. Persistence

A container's writable layer is disposable.

If you remove and recreate a database container without external storage, its data can be lost.

Persistent storage separates data from container lifecycle:

```text
container removed
      ↓
volume remains
      ↓
new container mounts volume
      ↓
data is still available
```

---

# 11. Compose

Compose describes a multi-container application in YAML.

Typical structure:

```yaml
services:
  app:
    image: ...
  db:
    image: ...

networks:
  backend:

volumes:
  data:
```

Start:

```bash
podman compose up -d
```

Show status:

```bash
podman compose ps
```

Logs:

```bash
podman compose logs
```

Stop/remove stack:

```bash
podman compose down
```

---

# 12. Compose networks

Example:

```yaml
services:
  app:
    image: myapp
    networks:
      - frontend
      - backend
    ports:
      - "8080:80"

  db:
    image: postgres
    networks:
      - backend

networks:
  frontend:
  backend:
    internal: true
```

The database is not published to the host.

Network segmentation reduces unnecessary exposure.

---

# 13. env_file

Instead of placing credentials inline:

```yaml
environment:
  POSTGRES_PASSWORD: secret
```

you can use:

```yaml
env_file:
  - db.env
```

Example `db.env`:

```text
POSTGRES_USER=app
POSTGRES_PASSWORD=secret
POSTGRES_DB=appdb
```

This separates configuration from the Compose file.

It is not the same as a dedicated secret mechanism.

---

# 14. Podman secrets

Create a secret:

```bash
printf '%s' 'my-secret-pw' | podman secret create db_password -
```

Use:

```bash
podman run --secret db_password IMAGE
```

Secrets are preferable to writing sensitive values directly into command lines or image metadata.

Exact consumption depends on the image/application. The secret is normally mounted into the container rather than baked into the image.

---

# 15. Healthchecks and startup ordering

A running database process is not necessarily ready to accept connections.

A healthcheck tests readiness.

Compose can make an application wait for a healthy database:

```yaml
depends_on:
  db:
    condition: service_healthy
```

The database also needs a healthcheck.

Example concept:

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U app"]
  interval: 5s
  timeout: 3s
  retries: 10
```

---

# 16. Scaling Compose services

Example:

```bash
podman compose up -d --scale app=3
```

This creates multiple replicas of a service.

Important:
- a service using a fixed host port cannot normally map the same host port for every replica
- load balancing requires an appropriate frontend/reverse proxy or networking layer

---

# 17. Reverse proxy

A reverse proxy such as nginx can sit in front of an application.

```text
Host
 ↓
nginx
 ↓
app
```

Benefits:
- one public entry point
- routing
- TLS termination
- hiding internal services
- load balancing when configured

Both proxy and application need a common network.

---

# 18. Database administration container

An administration tool such as Adminer or pgAdmin can be attached to the same database network.

```text
Admin UI
   ↓
db container name
   ↓
database
```

It should normally not be exposed more broadly than necessary.

---

# 19. Backup and restore of a named volume

A helper container can mount a volume and create a tar archive.

Concept:

```text
named volume
    ↓ mounted into helper container
tar archive
```

Restore reverses the process into a new volume.

This demonstrates that container storage can be backed up independently from the container itself.

---

# 20. Multiple networks

A container can join more than one network.

Example:

```text
frontend network
      ↓
     app
      ↓
backend network
      ↓
     db
```

The database joins only the backend network.

This creates network segmentation.

---

# 21. Name resolution failure

Two containers on different networks do not automatically discover each other.

Diagnosis:

```bash
podman inspect app
podman inspect db
podman network ls
podman network inspect NETWORK
```

Fix:

```bash
podman network connect backend app
```

Then test:

```bash
podman exec app ping -c 1 db
```

---

# 22. Pod inspection

Inspect a pod:

```bash
podman pod inspect POD
```

The practice material expects awareness of shared namespaces.

In a Podman pod, containers share the pod's network namespace and normally IPC. PID is not shared by default unless configured.

---

# 23. Kubernetes YAML bridge

Generate Kubernetes-style YAML from a Podman pod:

```bash
podman generate kube POD > pod.yaml
```

Run Kubernetes YAML with Podman:

```bash
podman kube play pod.yaml
```

Tear it down:

```bash
podman kube play --down pod.yaml
```

This bridges local container/pod work toward Kubernetes concepts used in LO4.

---

# 24. Compose teardown

```bash
podman compose down
```

Removes the Compose-created containers and networks.

Named volumes normally remain unless explicitly removed.

```bash
podman compose down --volumes
```

also removes Compose-managed volumes.

This distinction matters because deleting volumes means deleting persistent application data.

---

# 25. LO3 core model

```text
users
  ↓
published app
  ↓
frontend network
  ↓
application
  ↓
backend network
  ↓
database
  ↓
persistent volume
```

LO3 is about choosing the right network, storage, configuration and security boundaries for a multi-container application.

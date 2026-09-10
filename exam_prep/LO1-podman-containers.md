# Basic Podman Commands

1. Pulling images
`podman pull <image_name>`
2. List images
`podman images`
3. Run a container
`podman run <image_name>`
4. List running containers
`podman ps`
5. List all containers (running and stopped)
`podman ps -a`
6. Automatically remove a container after it exits
`podman run --rm <image_name>`
7. Assign a name to a container
`podman run --name <container_name> <image_name>`
8. Retrieve containers list as JSON
`podman ps --format json`
9. Create a new container and map port
`podman run -d -p <host_port>:<container_port> <image_name>`
10. Use environment variables in a container
`podman run -e NAME='Name' <image_name>`

# Container networking
1. Create a new network
`podman network create <network_name>`
2. List networks
`podman network ls`
3. Inspect a network
`podman network inspect <network_name>`
4. Remove a network
`podman network rm <network_name>`
5. Remove unused networks
`podman network prune`
6. Connect a container to a network
`podman network connect <network_name> <container_name>`
7. Forward a port from the host to a container
`podman run -d -p <host_port>:<container_port> <image_name>`
8. Find container IP address
`podman inspect -f '{{ .NetworkSettings.Networks.network_name.IPAddress }}' <container_name>`

## Run a container
```bash
podman run -d \
  --name web \
  --hostname myweb \
  -p 8081:80 \
  docker.io/library/httpd
```

# LO1 – Exam Cheat Sheet

## CONTAINERS VS VMs

Containers share the host kernel and are lightweight, fast and portable.

VMs run a complete guest OS with their own kernel and provide stronger OS-level isolation but require more resources.

Use containers for portable applications and microservices.

Use VMs when full OS isolation, another kernel or legacy OS requirements are needed.

---

# RUN CONTAINER

## TASK

Run `httpd` detached with custom name, hostname and port.

```bash
podman run -d \
  --name web \
  --hostname webserver \
  -p 8081:80 \
  docker.io/library/httpd
```

## VERIFY

```bash
podman ps
podman port web
podman inspect --format '{{.Name}}' web
podman inspect --format '{{.Config.Hostname}}' web
```

## REMEMBER

```text
-p HOST_PORT:CONTAINER_PORT
```

---

# LIST CONTAINERS

Running:

```bash
podman ps
```

All:

```bash
podman ps -a
```

---

# LIFECYCLE

```bash
podman start web
podman stop web
podman restart web
podman pause web
podman unpause web
podman rm web
podman rm -f web
podman rename web newweb
```

Pause freezes processes.

Stop terminates them.

---

# RESTART POLICY

## TASK

```bash
podman run -d \
  --name web \
  --restart=always \
  docker.io/library/httpd
```

## TEST

```bash
podman exec web kill 1
```

## VERIFY

```bash
podman ps
podman inspect --format '{{.RestartCount}}' web
```

## EXPLAIN

`--restart=always` causes Podman to restart the container after it exits.

---

# CPU + MEMORY LIMIT

## TASK

```bash
podman run -d \
  --name limited \
  --memory=256m \
  --cpus=0.5 \
  docker.io/library/httpd
```

## VERIFY

```bash
podman stats limited
podman inspect limited
```

---

# ENVIRONMENT VARIABLE

Direct:

```bash
podman run -d \
  --name app \
  --env KEY=VALUE \
  alpine sleep 1d
```

Verify:

```bash
podman exec app env
```

File:

```bash
podman run -d \
  --name app \
  --env-file env.txt \
  alpine sleep 1d
```

---

# CUSTOM NETWORK

## TASK

```bash
podman network create backend
```

```bash
podman run -d \
  --name app \
  --network backend \
  alpine sleep 1d
```

```bash
podman run -d \
  --name db \
  --network backend \
  alpine sleep 1d
```

## VERIFY MEMBERSHIP

```bash
podman network inspect backend
```

## VERIFY DNS / COMMUNICATION

```bash
podman exec app ping -c 1 db
```

## EXPLAIN

Containers on the same user-defined network can resolve each other using container names.

---

# CONNECT EXISTING CONTAINER

```bash
podman network connect backend app
```

Disconnect:

```bash
podman network disconnect backend app
```

---

# NETWORK NONE

```bash
podman run -d \
  --name isolated \
  --network none \
  alpine sleep 1d
```

No normal external network connectivity.

Use when network access is unnecessary and should be restricted.

---

# EXEC

Command:

```bash
podman exec web id
```

Shell:

```bash
podman exec -it web /bin/bash
```

Alpine:

```bash
podman exec -it web sh
```

---

# RESTART VS RECREATE

`podman restart` → same container, writable-layer changes remain.

Remove + recreate → changes inside old container disappear.

External mounted data persists separately.

---

# LOGS

```bash
podman logs web
```

Last 20:

```bash
podman logs --tail 20 web
```

Recent:

```bash
podman logs --since 10m web
```

Follow:

```bash
podman logs -f web
```

---

# INSPECT

Everything:

```bash
podman inspect web
```

Status:

```bash
podman inspect --format '{{.State.Status}}' web
```

Name:

```bash
podman inspect --format '{{.Name}}' web
```

Hostname:

```bash
podman inspect --format '{{.Config.Hostname}}' web
```

Restart count:

```bash
podman inspect --format '{{.RestartCount}}' web
```

---

# TOP VS STATS

Processes:

```bash
podman top web
```

Resources:

```bash
podman stats web
```

Remember:

```text
top = processes
stats = CPU/memory/resource usage
```

---

# COPY FILES

Host → container:

```bash
podman cp file.txt web:/tmp/file.txt
```

Container → host:

```bash
podman cp web:/tmp/file.txt ./file.txt
```

---

# BIND MOUNT READ-ONLY

```bash
podman run -it \
  -v ./data:/data:ro \
  alpine sh
```

Test:

```bash
touch /data/test
```

Should fail.

`:ro` = read-only.

---

# --rm

```bash
podman run --rm alpine echo hello
```

Container is automatically removed when it exits.

Disadvantage: cannot inspect stopped container afterward.

---

# LABELS

```bash
podman run -d \
  --name web1 \
  --label environment=production \
  httpd
```

Filter:

```bash
podman ps --filter label=environment=production
```

---

# FILESYSTEM CHANGES

```bash
podman diff web
```

```text
A = Added
C = Changed
D = Deleted
```

---

# COMMIT

```bash
podman commit web myimage:v1
```

Run:

```bash
podman run myimage:v1
```

Creates an image from the modified container.

Containerfile is normally more reproducible.

---

# PORTS

```bash
podman port web
```

Cross-check:

```bash
podman inspect web
```

Specific host IP only:

```bash
podman run -d \
  -p 127.0.0.1:8082:80 \
  httpd
```

Only localhost can reach it.

---

# NON-ROOT USER

```bash
podman run --rm \
  --user 1000 \
  alpine id
```

Verify:

```bash
id
```

Non-root reduces security risk.

---

# EVENTS

Terminal 1:

```bash
podman events
```

Terminal 2:

```bash
podman start web
podman stop web
```

Shows lifecycle events.

---

# SYSTEMD / QUADLET

Purpose:

```text
systemd manages container
→ start at boot
→ restart management
→ service lifecycle
```

Modern Podman supports Quadlet `.container` files.

---

# FAST TROUBLESHOOTING

## Container stopped

```bash
podman ps -a
podman logs CONTAINER
podman inspect CONTAINER
```

Likely reason: main process exited.

---

## Website not reachable

```bash
podman ps
podman port CONTAINER
podman logs CONTAINER
```

Check:

```text
-p HOST:CONTAINER
```

---

## Containers cannot communicate

```bash
podman network ls
podman network inspect NETWORK
podman exec app ping -c 1 db
```

Both containers must be on a common network.

---

# COMMAND INDEX

```bash
podman run
podman ps
podman ps -a

podman start
podman stop
podman restart
podman pause
podman unpause
podman rename
podman rm

podman exec
podman logs
podman inspect
podman top
podman stats

podman network ls
podman network create
podman network inspect
podman network connect

podman port
podman cp
podman diff
podman commit
podman events
```

---

# MOST IMPORTANT FLAGS

```text
-d                         detached

--name NAME                container name

--hostname NAME            hostname inside container

-p HOST:CONTAINER          port mapping

-e KEY=VALUE               environment variable

--env-file FILE            environment file

--network NETWORK          choose network

--network none             disable normal networking

--memory=256m              memory limit

--cpus=0.5                 CPU limit

-v HOST:CONTAINER          bind mount

-v HOST:CONTAINER:ro       read-only mount

--restart=always           restart policy

--rm                       remove after exit

--user UID                 run as specified user

--label KEY=VALUE          metadata label
```

# GOLDEN RULES

```text
-p is HOST:CONTAINER

podman ps = running
podman ps -a = everything

top = processes
stats = resources

restart = same container
recreate = new container

network inspect = network membership
ping/name lookup = proves communication

container stops when its main process stops
```

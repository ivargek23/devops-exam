# LO1 – Use of Containers and Container Services

## 1. What is a container?

A container is an isolated environment used to run an application together with its dependencies.

A container is created from an **image**.

* **Image** = read-only template.
* **Container** = running instance of that image.

Example:

```bash
podman run docker.io/library/httpd
```

Podman uses the `httpd` image to create a new container.

A container normally exists only while its main process is running. If that process exits, the container stops.

---

# 2. Containers vs Virtual Machines

Containers share the host operating-system kernel.

Virtual machines run a complete guest operating system with their own kernel.

| Containers                              | Virtual Machines           |
| --------------------------------------- | -------------------------- |
| Share host kernel                       | Own kernel                 |
| Lightweight                             | More resource-intensive    |
| Start quickly                           | Slower startup             |
| Smaller images                          | Large virtual disks        |
| High workload density                   | More overhead              |
| Easy to recreate                        | Heavier to recreate        |
| Good for applications and microservices | Good for full OS isolation |

## Why use containers?

Containers provide:

* portability
* reproducibility
* fast startup
* low resource consumption
* isolation
* easy deployment
* easy scaling

The same container image can be used in development, testing and production.

## When use a VM instead?

A VM can be preferable when:

* a different OS/kernel is required
* stronger isolation is needed
* the application requires full OS control
* legacy software is difficult to containerize

---

# 3. Running Containers

Basic syntax:

```bash
podman run [OPTIONS] IMAGE
```

Example:

```bash
podman run -d --name web docker.io/library/httpd
```

`-d` means detached mode.

The container runs in the background.

`--name web` assigns a custom name.

---

# 4. Container Name vs Hostname

Container name:

```bash
--name web
```

Used by Podman to identify the container.

Hostname:

```bash
--hostname webserver
```

Used as the hostname inside the container.

Example:

```bash
podman run -d \
  --name web \
  --hostname webserver \
  docker.io/library/httpd
```

Verify:

```bash
podman inspect --format '{{.Name}}' web
```

```bash
podman inspect --format '{{.Config.Hostname}}' web
```

---

# 5. Port Mapping

Containers have their own network namespace.

A service running on a container port is not automatically available through the host.

Publish it with:

```bash
-p HOST_PORT:CONTAINER_PORT
```

Example:

```bash
podman run -d \
  --name web \
  -p 8081:80 \
  docker.io/library/httpd
```

Meaning:

```text
Host 8081 → Container 80
```

Access:

```text
http://HOST_IP:8081
```

Verify:

```bash
podman port web
```

or:

```bash
podman ps
```

## Bind to only one host interface

```bash
podman run -d \
  --name web \
  -p 127.0.0.1:8082:80 \
  docker.io/library/httpd
```

Now the application is available only through host localhost.

It cannot normally be accessed through the VM's external IP.

---

# 6. Listing Containers

Running containers:

```bash
podman ps
```

All containers:

```bash
podman ps -a
```

`podman ps -a` is useful when a container has stopped unexpectedly.

---

# 7. Container Lifecycle

Start:

```bash
podman start web
```

Stop:

```bash
podman stop web
```

Restart:

```bash
podman restart web
```

Remove:

```bash
podman rm web
```

Force remove:

```bash
podman rm -f web
```

Rename:

```bash
podman rename web newweb
```

Verify:

```bash
podman ps
```

---

# 8. Container Restart vs Recreate

Restart:

```bash
podman restart web
```

This stops and starts the **same container**.

Changes made inside its writable filesystem remain.

If the container is removed:

```bash
podman rm web
```

and recreated from the original image, filesystem changes made manually inside the old container disappear.

Persistent data should therefore normally be stored using volumes or bind mounts.

---

# 9. Pause and Unpause

Pause:

```bash
podman pause web
```

Unpause:

```bash
podman unpause web
```

While paused, the container processes remain in memory but are frozen and do not execute.

This differs from `stop`, which terminates the processes.

---

# 10. Restart Policies

Example:

```bash
podman run -d \
  --name web \
  --restart=always \
  docker.io/library/httpd
```

`--restart=always` tells Podman to restart the container if it stops.

Useful for long-running services.

Verify configuration:

```bash
podman inspect web
```

Test the actual behavior:

```bash
podman exec web kill 1
```

Then:

```bash
podman ps
```

and:

```bash
podman inspect --format '{{.RestartCount}}' web
```

If the container is running again and the count increased, the restart policy worked.

---

# 11. Executing Commands Inside Containers

Run one command:

```bash
podman exec web id
```

Example:

```bash
podman exec web hostname
```

Interactive shell:

```bash
podman exec -it web /bin/bash
```

For Alpine:

```bash
podman exec -it container sh
```

`-i` = interactive input.

`-t` = terminal.

---

# 12. Installing Something Inside a Running Container

Example:

```bash
podman exec -it web bash
```

Then install software inside the container.

The change:

* survives `podman restart`
* does not survive removing and recreating the container from the original image

Permanent application configuration should normally be defined in the image using a Containerfile.

---

# 13. Environment Variables

Directly:

```bash
podman run -d \
  --name app \
  --env KEY=VALUE \
  docker.io/library/alpine sleep 1d
```

Check:

```bash
podman exec app env
```

Using a file:

```text
KEY=VALUE
ENVIRONMENT=production
```

Run:

```bash
podman run -d \
  --name app \
  --env-file env.txt \
  docker.io/library/alpine sleep 1d
```

Check:

```bash
podman exec app env
```

Environment files are useful when many variables must be supplied.

---

# 14. Resource Limits

Example:

```bash
podman run -d \
  --name limited \
  --memory=256m \
  --cpus=0.5 \
  docker.io/library/httpd
```

`--memory=256m` limits memory usage.

`--cpus=0.5` limits CPU usage to approximately half one CPU.

Verify:

```bash
podman stats limited
```

and:

```bash
podman inspect limited
```

Resource limits prevent one container from consuming excessive host resources.

---

# 15. podman top vs podman stats

Processes:

```bash
podman top web
```

Shows processes running inside the container.

Resources:

```bash
podman stats web
```

Shows live CPU, memory and other resource usage.

Remember:

```text
top = WHAT is running
stats = HOW MUCH it uses
```

---

# 16. Logs

All logs:

```bash
podman logs web
```

Last 20 lines:

```bash
podman logs --tail 20 web
```

Logs from recent period:

```bash
podman logs --since 10m web
```

Follow new logs:

```bash
podman logs -f web
```

Meaning:

```text
--tail → last N lines
--since → logs after specified time
-f → follow live output
```

Logs are one of the first things to inspect when a container fails.

---

# 17. podman inspect

Full information:

```bash
podman inspect web
```

It produces detailed JSON.

Specific values can be extracted with Go templates.

Status:

```bash
podman inspect --format '{{.State.Status}}' web
```

Hostname:

```bash
podman inspect --format '{{.Config.Hostname}}' web
```

Name:

```bash
podman inspect --format '{{.Name}}' web
```

Restart count:

```bash
podman inspect --format '{{.RestartCount}}' web
```

The exact field path depends on the value you want from the inspection JSON.

---

# 18. Podman Networks

List:

```bash
podman network ls
```

Create:

```bash
podman network create backend
```

Run containers on it:

```bash
podman run -d \
  --name app \
  --network backend \
  docker.io/library/alpine sleep 1d
```

```bash
podman run -d \
  --name db \
  --network backend \
  docker.io/library/alpine sleep 1d
```

Inspect:

```bash
podman network inspect backend
```

Containers on the same user-defined network can communicate and resolve each other by container name.

Test:

```bash
podman exec app ping -c 1 db
```

This is better than relying on changing container IP addresses.

---

# 19. Attach Existing Container to Network

If a container already exists:

```bash
podman network connect backend app
```

Disconnect:

```bash
podman network disconnect backend app
```

---

# 20. No Network

Run:

```bash
podman run -d \
  --name isolated \
  --network none \
  docker.io/library/alpine sleep 1d
```

The container has no normal external network connectivity.

Use case:

A batch-processing workload that does not require network access.

Security benefit:

Less network access means a smaller attack surface.

---

# 21. Copy Files

Host to container:

```bash
podman cp file.txt web:/tmp/file.txt
```

Container to host:

```bash
podman cp web:/tmp/file.txt ./file.txt
```

Useful for debugging, configuration and retrieving generated output.

---

# 22. Bind Mounts

A bind mount maps an existing host path into a container.

Example:

```bash
mkdir data
```

```bash
podman run -it \
  -v ./data:/data \
  docker.io/library/alpine sh
```

Host:

```text
./data
```

Container:

```text
/data
```

---

# 23. Read-only Bind Mount

```bash
podman run -it \
  -v ./data:/data:ro \
  docker.io/library/alpine sh
```

`:ro` means read-only.

Inside:

```bash
touch /data/test
```

This should fail.

The container can read the directory but cannot modify it.

---

# 24. --rm

Example:

```bash
podman run --rm docker.io/library/alpine echo hello
```

When the main process exits, Podman automatically removes the container.

Useful for:

* temporary commands
* tests
* one-off jobs

Disadvantage:

The stopped container and its writable filesystem are unavailable for later debugging.

External volume data is not automatically removed.

---

# 25. Labels

Run:

```bash
podman run -d \
  --name web1 \
  --label environment=production \
  docker.io/library/httpd
```

Another:

```bash
podman run -d \
  --name web2 \
  --label environment=test \
  docker.io/library/httpd
```

Filter:

```bash
podman ps --filter label=environment=production
```

Labels provide metadata for organization and filtering.

---

# 26. Filesystem Changes

```bash
podman diff web
```

Markers:

```text
A = Added
C = Changed
D = Deleted
```

Example:

```text
A /tmp/file.txt
```

means a file was added after the container was created.

Useful for investigating changes inside containers.

---

# 27. podman commit

A modified container can be captured as an image.

```bash
podman commit web myimage:v1
```

Run:

```bash
podman run myimage:v1
```

The new image contains the current container filesystem changes.

A Containerfile is normally preferable because it is reproducible and documents how the image was built.

---

# 28. Published Ports

Show:

```bash
podman port web
```

Cross-check:

```bash
podman inspect web
```

Remember:

```text
-p HOST:CONTAINER
```

This order is extremely important.

---

# 29. Running as Non-root

Example:

```bash
podman run --rm \
  --user 1000 \
  docker.io/library/alpine id
```

Verify:

```bash
id
```

or:

```bash
whoami
```

Running applications as non-root reduces the impact of a security compromise.

---

# 30. Podman Events

Terminal 1:

```bash
podman events
```

Terminal 2:

```bash
podman start web
podman stop web
```

Events show lifecycle activity such as:

```text
create
start
stop
remove
```

Useful for observing container behavior in real time.

---

# 31. systemd / Quadlet

Podman containers can be managed by systemd.

Modern Podman can use Quadlet `.container` definitions.

The purpose is:

* automatic startup at boot
* restart management
* dependency management
* management through systemd

Conceptually:

```text
Container definition
        ↓
systemd
        ↓
container managed as a service
```

---

# 32. Troubleshooting Logic

## Container is missing from podman ps

Check:

```bash
podman ps -a
```

Then:

```bash
podman logs CONTAINER
```

and:

```bash
podman inspect CONTAINER
```

A container stops when its main process exits.

---

## Web application cannot be reached

Check:

```bash
podman ps
```

Then:

```bash
podman port CONTAINER
```

Then:

```bash
podman logs CONTAINER
```

Remember:

```text
-p HOST:CONTAINER
```

---

## Containers cannot communicate

Check:

```bash
podman network ls
```

```bash
podman network inspect NETWORK
```

Make sure both containers are on the same network.

Test:

```bash
podman exec app ping -c 1 db
```

---

# 33. Core LO1 Model

```text
IMAGE
  ↓
podman run
  ↓
CONTAINER
  ↓
main process
  ↓
ports
network
environment
storage
resource limits
  ↓
Podman manages lifecycle
```

The most important LO1 concept is that containers provide lightweight, reproducible application environments, while Podman manages their creation, configuration, networking, resource use and lifecycle.

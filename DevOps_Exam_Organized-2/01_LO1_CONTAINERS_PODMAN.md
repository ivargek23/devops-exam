# LO1 — Containers and Podman

# How to use this file during the exam

Start with Ctrl+F and search the noun from the task, for example `restart`, `port`, `volume`, `Containerfile`, `rollout`, `ImagePullBackOff`, or `OpenShift`.

For practical tasks, always do three things: **perform the task → verify it → take a screenshot showing the command and proof**.

---

## QUICK KNOWLEDGE / COMMAND PATTERNS

## 2. LO1 — Podman containers: command patterns

### Run httpd detached with port, name, hostname

```bash
podman run -d --name web1 --hostname custom-web -p 8081:80 docker.io/library/httpd:2.4
podman ps
podman inspect web1 --format 'Name={{.Name}} Hostname={{.Config.Hostname}}'
curl http://localhost:8081
```

Port rule: `HOST_PORT:CONTAINER_PORT`. If nginx/httpd listens on 80 and you want host 8081, use `-p 8081:80`.

### Restart policy

```bash
podman run -d --name restart-demo --restart=always docker.io/library/httpd:2.4
podman inspect restart-demo --format '{{.HostConfig.RestartPolicy.Name}}'
PID=$(podman inspect restart-demo --format '{{.State.Pid}}')
kill -9 $PID
sleep 3
podman ps
podman inspect restart-demo --format 'RestartCount={{.RestartCount}} State={{.State.Status}}'
# Kill the main process externally; do not use `podman stop` to test restart policy.
```

### Resource limits

```bash
podman run -d --name limited --memory=256m --cpus=0.5 docker.io/library/httpd:2.4
podman stats --no-stream limited
podman inspect limited --format 'Memory={{.HostConfig.Memory}} NanoCPUs={{.HostConfig.NanoCpus}}'
```

### Env vars and env-file

```bash
podman run -d --name env1 --env KEY=VALUE docker.io/library/alpine:3.20 sleep 1d
cat > app.env <<'EOT'
A=one
B=two
EOT
podman run -d --name env2 --env-file app.env docker.io/library/alpine:3.20 sleep 1d
podman exec env1 env | grep KEY
podman exec env2 env | grep -E 'A|B'
```

### User-defined bridge network + DNS

```bash
podman network create devnet
podman run -d --name a --network devnet docker.io/library/alpine:3.20 sleep 1d
podman run -d --name b --network devnet docker.io/library/alpine:3.20 sleep 1d
podman exec a ping -c 2 b
podman network inspect devnet
```

Default Podman network often does not provide name DNS between standalone containers. User-defined networks do.

### Pause/unpause

```bash
podman run -d --name pauseme docker.io/library/httpd:2.4
podman pause pauseme
podman ps -a
podman unpause pauseme
podman ps
```

Paused containers have their processes frozen/suspended by cgroups; they are not killed.

### Rename

```bash
podman rename oldname newname
podman ps -a --format '{{.Names}}'
```

### Logs flags

```bash
podman logs web --tail 10       # last 10 lines
podman logs web --since 5m      # logs newer than 5 minutes
podman logs -f web              # follow live logs
```

### Inspect with Go template

```bash
podman inspect web --format '{{.NetworkSettings.IPAddress}}'
podman inspect web --format '{{.RestartCount}}'
podman inspect web --format '{{json .Config.Env}}'
```

### Copy files in/out

```bash
echo hello > host.txt
podman cp host.txt web:/tmp/inside.txt
podman exec web cat /tmp/inside.txt
podman cp web:/tmp/inside.txt copied-back.txt
cat copied-back.txt
```

### Exec shell and package install

```bash
podman exec -it web sh
# inside: install package if package manager exists
```

Changes inside a container survive stop/start, but do not survive deleting and recreating from the original image. To preserve them as an image, use `podman commit`, but Containerfile is cleaner.

### top and stats

```bash
podman top web
podman stats --no-stream web
```

### --rm

```bash
podman run --rm docker.io/library/alpine:3.20 echo hello
```

Useful for throwaway jobs. You lose the stopped container and its logs/metadata after it exits.

### Read-only bind mount

```bash
mkdir -p html
podman run -d --name rotest -v ./html:/data:ro docker.io/library/alpine:3.20 sleep 1d
podman exec rotest sh -c 'echo x > /data/x.txt'
# expect read-only filesystem / permission denied
```

On SELinux systems, add `:Z` or `:z` when needed:

```bash
-v ./html:/usr/share/nginx/html:Z
```

### Labels and filters

```bash
podman run -d --name app1 --label course=devops docker.io/library/alpine:3.20 sleep 1d
podman run -d --name app2 --label course=other docker.io/library/alpine:3.20 sleep 1d
podman ps --filter label=course=devops
```

### Commit and run image

```bash
podman exec app1 sh -c 'echo created > /created.txt'
podman commit app1 localhost/app1-snapshot:1.0
podman run --rm localhost/app1-snapshot:1.0 cat /created.txt
```

### diff markers

```bash
podman diff app1
```

A = added, C = changed, D = deleted.

### No network

```bash
podman run --rm --network none docker.io/library/alpine:3.20 sh -c 'ip a; wget -T 3 http://example.com'
```

Use case: batch jobs or untrusted processing that must not call outside systems.

### Bind to localhost only

```bash
podman run -d --name localweb -p 127.0.0.1:8082:80 docker.io/library/nginx:latest
curl http://127.0.0.1:8082
ip a
# It should not be reachable via the VM's non-loopback IP.
```

### Published ports

```bash
podman port localweb
podman inspect localweb --format '{{json .NetworkSettings.Ports}}'
```

### Non-root user

```bash
podman run --rm --user 1001 docker.io/library/alpine:3.20 id
```

### Systemd / Quadlet idea

Modern Podman prefers Quadlet files. Example:

```ini
# ~/.config/containers/systemd/web.container
[Container]
Image=docker.io/library/nginx:latest
ContainerName=web
PublishPort=8080:80

[Service]
Restart=always

[Install]
WantedBy=default.target
```

Then:

```bash
systemctl --user daemon-reload
systemctl --user enable --now web.service
```

Purpose: systemd starts/restarts the container on boot/login.

### Events

Terminal 1:

```bash
podman events
```

Terminal 2:

```bash
podman run -d --name eventweb docker.io/library/nginx
podman stop eventweb
podman rm eventweb
```

You should see create, start, stop, remove events.

---

## FULL SOLVED PRACTICE QUESTIONS

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

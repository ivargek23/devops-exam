# Intro to DevOps Exam Prep Pack

This is a GitHub-ready prep note. During the exam, do not use AI. Use your own notes, official documentation allowed by the professor, and the VM environment only.

# EXAM ACCESS — READY TO COPY

```text
VPN host: vpn.vua.cloud
VPN port: 15443
VPN username: vuastudent
VPN password: St00dent123

Exam VM username: student
Exam VM password: student

Docker Hub: YOUR OWN USERNAME + PASSWORD
Quay.io: YOUR OWN USERNAME + PASSWORD
Red Hat Academy: YOUR OWN USERNAME + PASSWORD
Algebra cloud/VCSA: YOUR INDIVIDUAL CREDENTIALS FROM ENVIRONMENT ACCESS INSTRUCTIONS
```

Important: work inside the VM as `student`, not `root`. `kubectl` is preconfigured. From the student's home directory, `bash dashboard.sh` opens the Minikube dashboard if needed.

---

## 0. Exam reality

The exam is written inside an Algebra cloud VM. Everything must be done from inside that VM: browsing allowed sites, reading tasks, writing answers, taking screenshots, and submitting. Screenshots are taken inside the VM via **Applications > Utilities > Screenshot**. Required accounts: Docker Hub, Quay.io, Red Hat Academy, Infoeduka.

Allowed documentation is limited. Expect only official Docker, Podman, Kubernetes, Minikube, Red Hat Academy, image registries, GitHub/artifact sites, and package registries to work.

## 1. Exam-day setup checklist

### 1.1 VPN and cloud access

Use the values from the cloud-access PDF. Do not guess them.

FortiClient VPN setup:

1. Install FortiClient VPN.
2. Configure new VPN.
3. Type: IPsec VPN.
4. Remote Gateway: value from the PDF.
5. Authentication method: pre-shared key; use the PSK from the PDF.
6. Advanced Settings > Phase 1 > Local ID: value from the PDF.
7. Save.
8. Connect using the VPN username/password from the PDF.
9. Open the cloud web UI from the PDF.
10. Find your VM folder, open VM console, and test login.

Inside the VM, test:

```bash
hostname
ip a
podman --version
kubectl version --client
minikube version
podman compose version || podman-compose --version
```

If one of these fails, do not panic. Screenshot the error and use the allowed docs.

### 1.2 Account checks: Docker Hub, Quay.io, Red Hat Academy

You need usernames and passwords memorized or stored somewhere allowed before the exam. Do not rely on browser autofill inside the exam VM.

Docker Hub:

1. Log in at hub.docker.com.
2. Avatar/profile menu > Account settings > 2FA.
3. If 2FA is enabled, disable it for the exam according to Docker's own 2FA flow.
4. Confirm you can log out and log back in with only username/email + password.
5. Confirm CLI login works:

```bash
podman logout docker.io
podman login docker.io -u YOUR_DOCKER_USERNAME
podman pull docker.io/library/alpine:3.20
podman run --rm docker.io/library/alpine:3.20 echo ok
```

Quay.io:

1. Log in at quay.io.
2. Open username/profile > Account Settings.
3. Check security / external login / applications sections. If any second-factor prompt is required, fix it before the exam.
4. Confirm CLI login works:

```bash
podman logout quay.io
podman login quay.io -u YOUR_QUAY_USERNAME
podman pull quay.io/libpod/alpine:latest || podman pull quay.io/prometheus/busybox:latest
```

Red Hat Academy:

1. Log in before the exam.
2. Confirm password works without password reset.
3. Keep username/password accessible in a way allowed by exam rules.

### 1.3 Screenshot habit

For every practical answer, capture three things in one screenshot when possible:

```bash
# command
# output proving it worked
# identity/context
hostname; date
```

Good proof commands:

```bash
podman ps -a
podman inspect NAME | less
podman inspect --format '{{.NetworkSettings.IPAddress}}' NAME
podman logs NAME --tail 20
kubectl get all -o wide
kubectl describe pod POD
kubectl get events --sort-by=.lastTimestamp
```

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
podman kill restart-demo
sleep 3
podman ps
podman inspect restart-demo --format 'RestartCount={{.RestartCount}} State={{.State.Status}}'
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

## 5. LO4 — Kubernetes / Minikube command patterns

Start with:

```bash
minikube start
kubectl get nodes -o wide
kubectl config current-context
```

### Deployment imperative + export YAML

```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3 --dry-run=client -o yaml > web.yaml
kubectl apply -f web.yaml
kubectl get deploy,rs,pods -l app=web -o wide
```

### Authenticated Docker Hub pulls

Create an image-pull Secret:

```bash
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=YOURUSER \
  --docker-password=YOURPASSWORD \
  --docker-email=YOURMAIL
```

Reference it:

```yaml
imagePullSecrets:
  - name: dockerhub-secret
```

### Scaling

```bash
kubectl scale deployment web --replicas=5
kubectl get rs,pods -l app=web
kubectl edit deployment web   # change spec.replicas
```

### Rolling update and rollback

```bash
kubectl set image deployment/web nginx=nginx:1.27
kubectl rollout status deployment/web
kubectl rollout history deployment/web
kubectl rollout undo deployment/web
```

Populate change cause:

```bash
kubectl annotate deployment web kubernetes.io/change-cause='update nginx to 1.27' --overwrite
```

### RollingUpdate strategy

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

Effect: Kubernetes may create one extra pod above desired count and allows zero unavailable pods, prioritizing availability.

Recreate strategy stops old pods before new pods. Use when two versions cannot run together, for example schema-incompatible single-writer apps.

### Requests and limits

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

Verify:

```bash
kubectl get pod POD -o yaml | grep -A10 resources:
```

### revisionHistoryLimit

```yaml
revisionHistoryLimit: 3
```

Keeps up to 3 old ReplicaSets for rollback history.

### Bad image rollout

```bash
kubectl set image deployment/web nginx=nginx:does-not-exist
kubectl rollout status deployment/web
kubectl get pods
kubectl describe pod BADPOD
```

Old pods keep serving because rolling update does not remove all old pods until new ones become available.

### Selector relationship

Deployment selector matches ReplicaSet selector, which matches Pod template labels. If selector and pod labels do not match, the manifest is invalid or controller cannot manage pods.

### Expose deployment

```bash
kubectl expose deployment web --port=80 --target-port=80 --type=ClusterIP
kubectl get svc web -o yaml
```

Service selector is derived from deployment pod labels, usually `app=web`.

### Bare pod vs Deployment-managed pod

Bare pod deleted = gone. Deployment-managed pod deleted = ReplicaSet recreates it.

### Deployment manifest skeleton

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd
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

If no node has `disk=fast`, pods stay Pending.

### StatefulSet + headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-headless
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
  serviceName: redis-headless
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

Stable names: `redis-0`, `redis-1`, `redis-2`. DNS: `redis-0.redis-headless.NAMESPACE.svc.cluster.local`.

### DaemonSet idea

One pod per node, often for agents/log collectors/network plugins.

### Job

```bash
kubectl create job calc --image=perl:5 -- perl -Mbignum=bpi -wle 'print bpi(20)'
kubectl get jobs
kubectl logs job/calc
```

### CronJob

```bash
kubectl create cronjob datejob --image=busybox --schedule='* * * * *' -- date
kubectl get cronjob,jobs
kubectl patch cronjob datejob -p '{"spec":{"suspend":true}}'
kubectl create job manual-date --from=cronjob/datejob
```

### emptyDir shared volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared
spec:
  volumes:
    - name: cache
      emptyDir: {}
  containers:
    - name: writer
      image: busybox
      command: ["sh", "-c", "echo hi > /data/msg; sleep 1d"]
      volumeMounts:
        - name: cache
          mountPath: /data
    - name: reader
      image: busybox
      command: ["sh", "-c", "cat /data/msg; sleep 1d"]
      volumeMounts:
        - name: cache
          mountPath: /data
```

### Storage listing

```bash
kubectl get storageclass
kubectl get pv
kubectl get pvc
```

### ConfigMap from literals and mount as files

```bash
kubectl create configmap app-config --from-literal=mode=prod --from-literal=color=blue
```

```yaml
volumes:
  - name: config
    configMap:
      name: app-config
volumeMounts:
  - name: config
    mountPath: /etc/config
```

Each key becomes a file.

### subPath ConfigMap key

```yaml
volumeMounts:
  - name: config
    mountPath: /app/config.txt
    subPath: mode
```

Needed when mounting one key to one file path without replacing the whole directory. Caveat: subPath mounts do not get live updates like normal ConfigMap volume projections.

### Secret basics

```bash
kubectl create secret generic db-secret --from-literal=username=app --from-literal=password=pass
kubectl get secret db-secret -o yaml
```

Values are base64-encoded, not encrypted by default.

Decode:

```bash
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d; echo
```

Secret from files:

```bash
kubectl create secret generic cert-secret --from-file=tls.crt --from-file=tls.key
```

TLS typed Secret:

```bash
kubectl create secret tls mytls --cert=tls.crt --key=tls.key
```

Consumed by Ingress/controllers needing TLS certs.

Secret as envFrom:

```yaml
envFrom:
  - secretRef:
      name: db-secret
```

Secret volume is better than env vars when you want file permissions and easier rotation. Env vars are easy but can leak through process/env inspection and do not update without restart.

### ClusterIP Service DNS

```bash
kubectl expose deployment web --name=web-svc --port=80 --target-port=80
kubectl run tmp --rm -it --image=busybox:1.36 -- sh
# inside:
nslookup web-svc.default.svc.cluster.local
wget -qO- http://web-svc
```

### NodePort

```bash
kubectl expose deployment web --name=web-node --port=80 --target-port=80 --type=NodePort
kubectl get svc web-node
minikube service web-node --url
curl $(minikube service web-node --url)
```

Service ports:

- `port`: service's internal port.
- `targetPort`: container/pod port receiving traffic.
- `nodePort`: port opened on each node for external access.

### LoadBalancer on minikube

`EXTERNAL-IP` is usually `<pending>` because minikube has no real cloud load balancer. Use:

```bash
minikube tunnel
```

### Endpoints

```bash
kubectl get endpoints web-svc
kubectl get endpointslice
```

Endpoints are populated from pods matching the Service selector and readiness state.

### Cross-namespace DNS

Short name works in same namespace. Across namespace use:

```text
service-name.namespace.svc.cluster.local
```

### Port-forward

```bash
kubectl port-forward pod/POD 8080:80
kubectl port-forward svc/web-svc 8080:80
```

To pod: direct to that pod. To service: traffic goes through service to selected backend.

### Probes

HTTP liveness:

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 10
```

TCP readiness for Redis:

```yaml
readinessProbe:
  tcpSocket:
    port: 6379
  initialDelaySeconds: 5
  periodSeconds: 10
```

Startup probe for ~2 minutes:

```yaml
startupProbe:
  httpGet:
    path: /
    port: 80
  periodSeconds: 10
  failureThreshold: 12
```

Budget = 10s × 12 = 120s.

No probes: Kubernetes only knows whether the process is running. It does not know whether the app is actually healthy or ready to serve traffic.

## 6. LO5 — Troubleshooting: fastest diagnosis path

### Podman mistakes

Wrong port order:

```bash
# wrong for nginx host 80 -> container 8080
-p 80:8080
# correct if nginx listens on 80 and you want host 8080
-p 8080:80
```

Missing env value:

```bash
# wrong
-e MYSQL_ROOT_PASSWORD
# correct
-e MYSQL_ROOT_PASSWORD=secret
```

Host network ignores `-p`:

```bash
--network host
```

In host networking, container uses host network namespace directly; port mapping is meaningless.

Busybox exits:

```bash
podman run -d --name c1 busybox sleep 1d
```

A container runs only as long as PID 1 runs.

`--rm -d` gotcha:

A detached throwaway job can finish and auto-delete before you inspect logs.

Memory too low:

`--memory 8m` is unrealistic for MySQL. Raise limit.

Name DNS fails:

Use a user-defined network, not default:

```bash
podman network create n1
podman run -d --network n1 --name a alpine sleep 1d
podman run -d --network n1 --name b alpine sleep 1d
podman exec a ping b
```

SELinux mount issue:

```bash
-v ./html:/usr/share/nginx/html:Z
```

### Containerfile mistakes

Debian install without update:

```Dockerfile
RUN apt-get update && apt-get install -y nginx
```

Shell vs exec form:

```Dockerfile
CMD ["python", "app.py"]
```

Exec form handles signals better than `CMD python app.py` shell form.

Node caching:

```Dockerfile
COPY package*.json /app/
WORKDIR /app
RUN npm install
COPY . /app
```

PATH mistake:

```Dockerfile
ENV PATH=/app/bin:$PATH
```

Do not replace system PATH entirely.

EXPOSE mismatch:

```Dockerfile
EXPOSE 8080
CMD ["python3", "-m", "http.server", "3000"]
```

App listens on 3000, so use `-p 8080:3000`. EXPOSE is metadata; it does not publish ports.

Huge apt image:

```Dockerfile
RUN apt-get update \
 && apt-get install -y --no-install-recommends build-essential \
 && rm -rf /var/lib/apt/lists/*
```

USER before installing/copying:

Create user, copy with ownership, install as appropriate. If installing system packages, do it before `USER appuser`.

Go huge final image:

Use multi-stage build: build in `golang`, copy binary into minimal runtime image.

### Kubernetes diagnosis flow

Always run these first:

```bash
kubectl get pods -o wide
kubectl describe pod POD
kubectl logs POD
kubectl logs POD --previous
kubectl get events --sort-by=.lastTimestamp
kubectl get deploy,rs,pods,svc,endpoints -o wide
```

Pending:

```bash
kubectl describe pod POD
```

Look at Events: Insufficient cpu/memory, nodeSelector mismatch, PVC missing, image pull pending, taints/tolerations.

ImagePullBackOff:

Check image name/tag/registry, private registry needs `imagePullSecrets`.

CrashLoopBackOff:

```bash
kubectl logs POD --previous
kubectl describe pod POD
```

Fix app command/config/env/dependency.

OOMKilled:

```bash
kubectl get pod POD -o yaml | grep -A10 lastState
kubectl describe pod POD | grep -i oom -A5
```

Raise memory limit or fix app memory use.

Not Ready:

Check readiness probe. If wrong path/port/initial delay, correct it.

Service returns nothing:

```bash
kubectl get svc SVC -o yaml
kubectl get endpoints SVC
kubectl get pods --show-labels
```

If endpoints are empty, selector labels do not match ready pods.

Selector mismatch in Deployment:

`spec.selector.matchLabels` must match `spec.template.metadata.labels` and is immutable after creation.

PVC missing:

Events show FailedMount or persistentvolumeclaim not found. Create PVC in same namespace.

Wrong apiVersion/kind:

Pod uses `apiVersion: v1`, not `apps/v1`. Deployment/StatefulSet/DaemonSet use `apps/v1`.

ConfigMap key missing:

Create the missing key or fix `configMapKeyRef.key`.

Secret in different namespace:

Secrets are namespaced. Create/copy Secret into the pod's namespace.

Rollout stuck:

```bash
kubectl rollout status deployment/NAME
kubectl describe deployment NAME
kubectl get rs,pods
```

ProgressDeadlineExceeded usually means new pods never became available.

Dry-run validation:

```bash
kubectl apply --dry-run=server --validate=true -f file.yaml
```

JSONPath restart count:

```bash
kubectl get pod POD -o jsonpath='{.status.containerStatuses[*].restartCount}'
kubectl get pod POD -o jsonpath='{.status.containerStatuses[*].lastState}'
```

Ephemeral debug:

```bash
kubectl debug -it POD --image=busybox --target=CONTAINER -- sh
```

Throwaway debug pod:

```bash
kubectl run tmp --rm -it --image=busybox:1.36 -- sh
```

Node NotReady:

```bash
kubectl describe node NODE
minikube logs
```

Check kubelet, disk pressure, network/CNI, container runtime.

Pod stuck Terminating:

Finalizers or long grace period can block deletion. Force deletion:

```bash
kubectl delete pod POD --grace-period=0 --force
```

Risk: app may not shut down cleanly; storage/state may be inconsistent.

OpenShift Route vs Ingress:

Ingress is Kubernetes API for HTTP routing, normally implemented by an ingress controller. OpenShift Route is OpenShift's built-in external routing object, integrated with OpenShift router/HAProxy, TLS options, and platform conventions.

## 7. LO6 — Theory answer bank

Use this structure for every theory answer: recommendation, reason, trade-off, concrete example.

### 1. Kubernetes networking vs Docker/Podman networking

Docker/Podman on a single host commonly use bridge networks, NAT, published ports, and optional user-defined DNS. Kubernetes networking is cluster-level: every pod gets an IP, pods can talk across nodes, Services provide stable virtual IP/DNS/load balancing, and CNI plugins implement the network. Kubernetes is more complex but designed for multi-node scheduling and service discovery. Docker/Podman networking is simpler and enough for one host.

### 2. Kubernetes storage vs Podman storage plugins

Kubernetes is more flexible for stateful workloads because it abstracts storage through PVs, PVCs, StorageClasses, CSI drivers, dynamic provisioning, reclaim policies, and per-workload claims. Podman volumes are good for single-host persistence, but Kubernetes better handles multi-node scheduling, cloud disks, and stateful controllers.

### 3. Operators vs plain manifests

Plain manifests declare desired resources, but operators add domain-specific automation through CRDs and controllers. For a database, an operator can handle backups, failover, upgrades, replication, and recovery. The trade-off is higher complexity and dependency on the operator's quality. Use manifests for simple stateless apps; use operators for complex stateful systems.

### 4. Plain Kubernetes vs OpenShift S2I/developer console

Plain Kubernetes optimizes for portability and low-level control through kubectl and YAML. OpenShift S2I and the developer console optimize for developer productivity: build source into images, deploy with guided UI, integrate routes, image streams, templates, and security defaults. Kubernetes is leaner; OpenShift is more batteries-included.

### 5. Migrating OpenShift to Kubernetes

Migration is possible but not automatic. Standard Kubernetes resources move easily; OpenShift-specific objects like Routes, BuildConfigs, ImageStreams, SCCs, and S2I workflows need replacements. Benefit: less vendor lock-in and more platform choice. Cost: rebuilding CI/CD, security policy, routing, registry integration, and developer workflows.

### 6. CI/CD agents in Kubernetes vs VMs

Kubernetes agents are good for autoscaling, ephemeral clean environments, and resource isolation by namespace/limits. Dedicated VMs are simpler and can be more predictable for heavy builds or privileged tooling. Kubernetes has noisy-neighbor risks if limits are bad; VMs have lower orchestration complexity but waste idle capacity.

### 7. Kubernetes and cloud dynamic capacity

Kubernetes integrates with cloud through cloud-controller managers, CSI storage, load balancers, node autoscaling, cluster autoscaling, and managed services. Workloads request resources; autoscalers can add pods or nodes. This makes Kubernetes strong for elastic demand.

### 8. OpenShift security vs vanilla Kubernetes

OpenShift is more secure by default: stricter security constraints, integrated RBAC, image registry/scanning options, routes/TLS integration, opinionated defaults, and Red Hat support. Vanilla Kubernetes can be hardened, but more work is left to the team. Regulated enterprises often choose OpenShift for support, compliance posture, and integrated platform governance.

### 9. Learning curve OpenShift vs Kubernetes

OpenShift adds concepts, but also gives safer defaults and UI workflows. For a team new to containers, it can help if they accept the platform's opinionated path. It can hinder if they need to understand raw Kubernetes first or want maximum portability. Net: OpenShift reduces operational assembly work but increases platform-specific knowledge.

### 10. Full Kubernetes vs k3s footprint

Full Kubernetes can be overkill for small on-prem deployments because it needs more resources and operational care. k3s is lightweight, easier to run on edge/small servers, and removes/streamlines components. Use full Kubernetes for larger clusters, strict HA, broad integrations, and enterprise operations; use k3s for small teams, labs, edge, or constrained hardware.

### 11. Compose single host vs Kubernetes for small team

A small team should start with Docker/Podman Compose if the app fits one host and needs fast delivery. Compose has lower cost and mental overhead. Move to Kubernetes when they need self-healing across nodes, autoscaling, rolling updates, service discovery, and stronger platform standards. Do not buy a forklift to move a sandwich.

### 12. Vendor lock-in: Kubernetes vs OpenShift

Kubernetes is open source/CNCF with many providers, so it offers broad portability. OpenShift is Kubernetes-based but adds Red Hat-specific APIs, workflows, support model, and tooling. OpenShift improves enterprise experience but increases dependency on Red Hat conventions. Long-term flexibility depends on avoiding platform-specific resources unless their value justifies it.

### 13. Recommend OpenShift over Kubernetes

Choose OpenShift when a company wants an integrated, vendor-supported platform with developer console, S2I/builds, routes, registry integration, monitoring/logging options, stricter defaults, and enterprise support. It costs more and is heavier, but reduces platform assembly work and risk for regulated or large organizations.

### 14. Kubernetes storage vs VM storage

VM storage is usually attached to a machine and managed by infrastructure admins. Kubernetes storage is requested declaratively by applications through PVCs and dynamically provisioned by StorageClasses/CSI. Kubernetes makes storage part of the app deployment model, but stateful workloads still require careful backup, performance, and failover design.

### 15. Startup with two engineers

Recommend Compose or a managed PaaS/container service first, not self-managed Kubernetes. The priority is shipping cheaply with low operational overhead. Use simple containers, managed database, and CI/CD. Move to Kubernetes later when scaling/resilience needs justify the complexity.

### 16. Docker vs Podman

Docker traditionally uses a daemon architecture; clients talk to the Docker daemon. Podman is daemonless and can run rootless more naturally. A security-conscious team may prefer Podman because rootless containers reduce the impact of container escape/misconfiguration and avoid a long-running root daemon.

### 17. Self-managed Kubernetes vs managed service

Self-managed Kubernetes means you own upgrades, etcd backups, node scaling, control-plane health, networking, storage integration, and security patches. Managed Kubernetes offloads much of that to the provider but costs money and can introduce cloud-specific behavior. For most production teams, managed is saner unless they have strong platform engineering capability.

### 18. Media-streaming with spiky global traffic

Recommend managed Kubernetes or a cloud-native container platform plus CDN. Kubernetes handles horizontal scaling, rolling updates, resilience, and service orchestration; CDN handles global media delivery. Self-managed single-host/Swarm is too limited for unpredictable global traffic.

### 19. Outgrowing Docker Swarm/Compose

Migration to Kubernetes is justified when the company needs stronger ecosystem, autoscaling, multi-node resilience, standardized deployments, service discovery, secrets/config management, and broad tooling. Cost: rewrite manifests, retrain team, change CI/CD, observe production risk. If growth is real, long-term benefits usually beat staying on a platform that no longer fits.

### 20. Kubernetes for microservices

Yes, if the microservices need orchestration, resilience, service discovery, rolling deployments, scaling, and observability ecosystem. Kubernetes fits many independent services better than hand-managed hosts. It is not worth it for a tiny app or a team without operational capacity.

### 21. Kubernetes for enterprise apps

Yes, often. Kubernetes has scalability, huge community support, mature tooling, portability, and feature richness. But enterprise-grade use requires hardening, governance, monitoring, backup, network policy, secrets management, and platform skills. Kubernetes is the engine, not the whole car.

### 22. OpenShift for microservices

Yes when the organization wants Kubernetes orchestration plus an integrated enterprise developer/operations platform. OpenShift adds routes, builds/S2I, stricter defaults, console, RBAC patterns, and support. It is heavier than plain Kubernetes, but useful when teams need a governed microservices platform.

### 23. OpenShift for enterprise-grade apps

Yes for enterprises that value support, security defaults, integrated tooling, scalability, compliance posture, and consistent operations across hybrid environments. It is less minimal and more vendor-aligned than vanilla Kubernetes, but that trade-off is often exactly what enterprise IT wants.

## 8. Last-night drill plan

### 2-hour emergency plan

1. 20 min: verify VPN, VM console, screenshot tool, all account logins.
2. 35 min: practice LO1 commands: run, inspect, logs, exec, cp, volume, network.
3. 30 min: practice LO2: write two Containerfiles, build, run, tag, push one image.
4. 35 min: practice LO4/LO5: create deployment, expose service, break image tag, diagnose ImagePullBackOff, rollback.

### 4-hour safer plan

1. 30 min setup/account checks.
2. 60 min LO1.
3. 60 min LO2.
4. 45 min LO3 compose/pod/volume.
5. 60 min LO4 Kubernetes.
6. 45 min LO5 troubleshooting.
7. 20 min LO6 theory read-through.

### What to memorize cold

- Podman port order: `host:container`.
- Kubernetes selector chain: Service selector -> Pod labels; Deployment selector -> Pod template labels.
- Pending = scheduler/storage/node problem.
- ImagePullBackOff = image/registry/auth problem.
- CrashLoopBackOff = app starts then crashes.
- OOMKilled = memory limit too low or app leak.
- Empty endpoints = Service selector mismatch or pods not ready.
- Secret values are base64-encoded, not encrypted by default.
- Compose is for simple/single-host; Kubernetes/OpenShift for orchestration/resilience at scale.


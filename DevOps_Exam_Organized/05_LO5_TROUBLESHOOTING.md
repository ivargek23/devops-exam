# LO5 — Troubleshooting

# How to use this file during the exam

Start with Ctrl+F and search the noun from the task, for example `restart`, `port`, `volume`, `Containerfile`, `rollout`, `ImagePullBackOff`, or `OpenShift`.

For practical tasks, always do three things: **perform the task → verify it → take a screenshot showing the command and proof**.

---

## QUICK KNOWLEDGE / COMMAND PATTERNS

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

---

## FULL SOLVED PRACTICE QUESTIONS

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

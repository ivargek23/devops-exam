# LO5 — Troubleshooting Exam Cheat Sheet

> Mirrors the official LO5 practice-question set.

# PODMAN / CONTAINERFILE FAILURES

## 1. WRONG NGINX PORT MAPPING

Wrong:

```bash
podman run -d --name web -p 80:8080 nginx
```

Nginx listens on container port 80.

Correct if you want host 8080:

```bash
podman run -d --name web -p 8080:80 nginx
```

Rule:

```text
-p HOST:CONTAINER
```

---

## 2. MYSQL PASSWORD VARIABLE HAS NO VALUE

Wrong:

```bash
podman run -d \
  --name db \
  -e MYSQL_ROOT_PASSWORD \
  mysql
```

Fix:

```bash
podman run -d \
  --name db \
  -e MYSQL_ROOT_PASSWORD=secret \
  mysql
```

Diagnose:

```bash
podman ps -a
podman logs db
```

---

## 3. --network host + -p

Wrong/redundant:

```bash
podman run -d \
  --name app \
  --network host \
  -p 8080:80 \
  nginx
```

With host networking, the container uses the host network namespace, so normal port publishing is not meaningful.

Fix either:

```bash
podman run -d --name app --network host nginx
```

or use normal container networking:

```bash
podman run -d --name app -p 8080:80 nginx
```

---

## 4. BUSYBOX EXITS IMMEDIATELY

```bash
podman run -d --name c1 busybox
```

Problem:

No long-running main process.

Check:

```bash
podman ps -a
```

Fix:

```bash
podman run -d --name c1 busybox sleep 1d
```

---

## 5. --rm + DETACHED SHORT JOB

```bash
podman run --rm -d \
  --name job \
  alpine echo hello
```

The command finishes immediately and `--rm` deletes the container.

Therefore:

```bash
podman logs job
```

can fail because `job` no longer exists.

Fix for inspection:

```bash
podman run -d --name job alpine sh -c 'echo hello; sleep 60'
```

or omit `--rm`.

---

## 6. MYSQL MEMORY 8 MB

```bash
--memory 8m
```

is far too small for MySQL.

Check:

```bash
podman ps -a
podman logs CONTAINER
podman inspect CONTAINER
```

Fix: provide realistic memory or remove the artificial limit.

---

## 7. CONTAINER NAME DNS FAILS

Containers:

```bash
podman run -d --name a alpine sleep 1d
podman run -d --name b alpine sleep 1d
```

Then:

```bash
podman exec a ping b
```

may fail due to networking/name-resolution setup.

Fix:

```bash
podman network create appnet
```

```bash
podman network connect appnet a
podman network connect appnet b
```

Test:

```bash
podman exec a ping -c 1 b
```

---

## 8. SELINUX BIND MOUNT

Problem:

```bash
-v ./html:/usr/share/nginx/html
```

may be denied by SELinux.

Fix for private relabel:

```bash
-v ./html:/usr/share/nginx/html:Z
```

---

## 10. DEBIAN: INSTALL WITHOUT UPDATE

Wrong:

```Dockerfile
FROM debian:12
RUN apt-get install -y nginx
```

Fix:

```Dockerfile
FROM debian:12

RUN apt-get update \
    && apt-get install -y nginx \
    && rm -rf /var/lib/apt/lists/*
```

---

## 11. CMD SHELL FORM SIGNAL HANDLING

Less suitable:

```Dockerfile
CMD python app.py
```

Better:

```Dockerfile
CMD ["python", "app.py"]
```

Exec form lets the application run directly as PID 1 and receive signals correctly.

---

## 12. NODE LAYER CACHE

Bad:

```Dockerfile
COPY . /app
WORKDIR /app
RUN npm install
```

Better:

```Dockerfile
WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .
```

Source changes no longer invalidate the dependency-install layer unless package metadata changed.

---

## 13. PATH OVERWRITTEN

Wrong:

```Dockerfile
ENV PATH=/app/bin
```

Fix:

```Dockerfile
ENV PATH="/app/bin:${PATH}"
```

Otherwise standard commands may disappear from command lookup.

---

## 14–15. EXPOSE 8080 BUT APP LISTENS ON 3000

```Dockerfile
EXPOSE 8080
CMD ["python3", "-m", "http.server", "3000"]
```

Problem:

App listens on 3000, not 8080.

`EXPOSE` does not change the application port and does not publish it.

Fix mapping:

```bash
podman run -p 8080:3000 IMAGE
```

or change app command to listen on 8080.

---

## 16. HUGE DEBIAN IMAGE

Current:

```Dockerfile
RUN apt-get update && apt-get install -y build-essential
```

Better:

```Dockerfile
RUN apt-get update \
    && apt-get install -y --no-install-recommends build-essential \
    && rm -rf /var/lib/apt/lists/*
```

If build tools are only needed to compile, use a multi-stage build and exclude them from final image.

---

## 17. USER TOO EARLY / OWNERSHIP

Problem pattern:

```Dockerfile
USER appuser
COPY requirements.txt /app/requirements.txt
RUN pip install -r /app/requirements.txt
```

Possible fix:

```Dockerfile
FROM python:3.11

RUN useradd -m appuser

WORKDIR /app
COPY --chown=appuser:appuser requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY --chown=appuser:appuser . .

USER appuser
```

The exact best placement depends on whether package installation needs root/system paths.

---

## 18. GO IMAGE ~1 GB

Bad: compiler remains in final image.

Use multi-stage:

```Dockerfile
FROM golang:1.22 AS builder
WORKDIR /src
COPY . .
RUN go build -o app .

FROM alpine:3.20
WORKDIR /app
COPY --from=builder /src/app /app/app
CMD ["/app/app"]
```

---

# KUBERNETES FAILURES

## 19. POD PENDING

First:

```bash
kubectl describe pod POD
```

Read Events.

Possible causes:

```text
FailedScheduling
Insufficient cpu
Insufficient memory
nodeSelector mismatch
PVC unbound/missing
```

Fix the actual event cause.

---

## 20. BROKEN YAML INDENTATION

Try:

```bash
kubectl apply -f file.yaml
```

Safer before applying:

```bash
kubectl apply --dry-run=client -f file.yaml
```

or:

```bash
kubectl apply --dry-run=server -f file.yaml
```

Fix YAML indentation/schema error reported.

---

## 21. ImagePullBackOff

```bash
kubectl describe pod POD
```

Check Events for:
- typo in image
- missing tag
- private registry auth failure

Fix image:

```bash
kubectl set image deployment/APP \
  CONTAINER=CORRECT_IMAGE
```

Private registry → configure `imagePullSecrets`.

---

## 22. CrashLoopBackOff

```bash
kubectl logs POD --previous
```

Also:

```bash
kubectl describe pod POD
```

Fix the application/configuration error shown in the previous crash logs.

---

## 23. OOMKilled

```bash
kubectl describe pod POD
```

or:

```bash
kubectl get pod POD -o yaml
```

Look for:

```text
reason: OOMKilled
```

Fix:
- raise memory limit if appropriate
- reduce app memory consumption

---

## 24. POD NEVER READY

```bash
kubectl describe pod POD
```

Look for readiness probe failures.

Check:
- path
- port
- protocol
- timing
- actual app behavior

Readiness failure means pod does not receive Service traffic.

---

## 25. SERVICE RETURNS NOTHING

```bash
kubectl get svc
kubectl get endpoints
kubectl get pods --show-labels
```

If endpoints empty:

```text
Service selector does not match pod labels
or pods are not Ready
```

Fix labels/selectors.

---

## 26. DEPLOYMENT SELECTOR DOES NOT MATCH TEMPLATE

Wrong pattern:

```yaml
selector:
  matchLabels:
    app: web

template:
  metadata:
    labels:
      app: api
```

Fix so they match:

```yaml
selector:
  matchLabels:
    app: web

template:
  metadata:
    labels:
      app: web
```

---

## 27. RESOURCE REQUEST TOO LARGE

If:

```yaml
requests:
  cpu: "100"
```

and no node can provide that, pod stays Pending.

Check:

```bash
kubectl describe pod POD
```

Event:

```text
Insufficient cpu
```

Fix requests or add capacity.

---

## 28. MISSING PVC

Check:

```bash
kubectl get pvc
kubectl describe pod POD
```

If pod references `data-pvc` but it does not exist, create it or correct the claim name.

---

## 29. UBUNTU CONTAINER HAS NO LONG-RUNNING COMMAND

A base Ubuntu image does not automatically run a service forever.

Add:

```yaml
command: ["sh", "-c", "sleep 1d"]
```

for testing, or configure the real application command.

---

## 30. EVENTS TIMELINE

```bash
kubectl get events --sort-by=.lastTimestamp
```

or:

```bash
kubectl events
```

Use to reconstruct what happened over time.

---

## 31. DNS DEBUG FROM EXISTING POD

```bash
kubectl exec -it POD -- sh
```

Inside:

```bash
nslookup SERVICE
cat /etc/resolv.conf
```

---

## 32. EPHEMERAL DEBUG CONTAINER

```bash
kubectl debug -it POD \
  --image=busybox \
  --target=CONTAINER
```

Use when the original image has no shell/debug tools.

---

## 33. THROWAWAY DEBUG POD

```bash
kubectl run tmp \
  --rm -it \
  --image=busybox \
  -- sh
```

Then:

```bash
nslookup SERVICE
wget -O- http://SERVICE
nc -vz SERVICE PORT
```

---

## 34. NODE NotReady

```bash
kubectl get nodes
kubectl describe node NODE
```

Minikube:

```bash
minikube logs
```

Check:
- kubelet
- disk pressure
- memory pressure
- CNI/networking

---

## 35. POD STUCK Terminating

Inspect:

```bash
kubectl get pod POD -o yaml
```

Look for finalizers.

Force only if necessary:

```bash
kubectl delete pod POD \
  --grace-period=0 \
  --force
```

Risk: skips graceful cleanup and can leave application/storage state inconsistent.

---

## 36. WRONG apiVersion/kind

Wrong:

```yaml
apiVersion: apps/v1
kind: Pod
```

Correct:

```yaml
apiVersion: v1
kind: Pod
```

Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
```

---

## 37. CreateContainerConfigError: MISSING ConfigMap KEY

Check:

```bash
kubectl describe pod POD
kubectl get configmap NAME -o yaml
```

Fix:
- `configMapKeyRef.key`
- or add the missing key to ConfigMap

---

## 38. SECRET IN DIFFERENT NAMESPACE

A pod cannot mount a Secret from another namespace.

Check:

```bash
kubectl get secret -n POD_NAMESPACE
```

Fix by creating/copying the Secret into the pod's namespace.

---

## 39. ProgressDeadlineExceeded

```bash
kubectl rollout status deployment/APP
kubectl describe deployment APP
kubectl get pods
kubectl describe pod POD
```

Typical root causes:
- image pull
- crashes
- readiness failure
- scheduling

Fix root cause; then rollout can progress.

---

## 40. FIELD TYPO / BAD NESTING

Examples:

```text
imagePullpolicy
misnested ports
```

Validate:

```bash
kubectl apply --dry-run=server -f file.yaml
```

Correct field:

```text
imagePullPolicy
```

---

## 41. APP CANNOT REACH DB SERVICE

Check in this order:

```text
1. DB pod running?
2. Service exists?
3. Endpoints populated?
4. DNS resolves?
5. correct Service port?
6. DB application listening?
```

Commands:

```bash
kubectl get pods
kubectl get svc
kubectl get endpoints
kubectl get endpointslice
kubectl exec -it APP_POD -- nslookup DB_SERVICE
```

---

## 42. state / lastState / restartCount

Restart count:

```bash
kubectl get pod POD \
  -o jsonpath='{.status.containerStatuses[*].restartCount}'
```

Current state:

```bash
kubectl get pod POD \
  -o jsonpath='{.status.containerStatuses[*].state}'
```

Last state:

```bash
kubectl get pod POD \
  -o jsonpath='{.status.containerStatuses[*].lastState}'
```

Use these to identify repeated restart cause.

---

## 43. OPENSHIFT ROUTE VS INGRESS

```text
Ingress:
Kubernetes-standard HTTP/HTTPS routing resource.
Needs an Ingress controller.

Route:
OpenShift-specific HTTP/HTTPS exposure resource.
Integrated with OpenShift router/platform.
```

---

# FAST PODMAN TRIAGE

```bash
podman ps -a
podman logs CONTAINER
podman inspect CONTAINER
podman port CONTAINER
podman network inspect NETWORK
podman stats CONTAINER
```

# FAST KUBERNETES TRIAGE

```bash
kubectl get pods
kubectl describe pod POD
kubectl logs POD
kubectl logs POD --previous
kubectl get events --sort-by=.lastTimestamp

kubectl get svc
kubectl get endpoints
kubectl get endpointslice
kubectl get pods --show-labels

kubectl get pvc
kubectl describe node NODE
```

# GOLDEN RULES

```text
Podman:
ps -a → logs → inspect

Kubernetes:
get → describe → events → logs → related resource

Pending → scheduler/storage
ImagePullBackOff → image/auth
CrashLoopBackOff → application crash
OOMKilled → memory
NotReady → readiness probe/app readiness
CreateContainerConfigError → missing/bad config
no Service endpoints → selector/readiness
ProgressDeadlineExceeded → rollout cannot make progress
```

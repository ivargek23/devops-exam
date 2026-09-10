# LO5 — Troubleshooting Application Shipping with Containers

> Coverage follows the LO5 practice-question set. This file focuses on troubleshooting method; the companion cheat sheet maps specific practice failures to fixes.

# 1. Core troubleshooting rule

Do not guess.

Use evidence in this order:

```text
state
 ↓
events/status
 ↓
logs
 ↓
configuration
 ↓
network/storage/resources
```

For Podman:

```bash
podman ps -a
podman logs CONTAINER
podman inspect CONTAINER
```

For Kubernetes:

```bash
kubectl get pods
kubectl describe pod POD
kubectl logs POD
kubectl get events --sort-by=.lastTimestamp
```

---

# 2. Containers stop when the main process stops

A container is not a VM.

It normally runs while its main process (PID 1) runs.

Example:

```bash
podman run -d alpine
```

may exit immediately because there is no long-running workload.

Keep it running:

```bash
podman run -d alpine sleep 1d
```

The same principle applies to Kubernetes containers.

---

# 3. Port mapping

Podman syntax:

```text
-p HOST_PORT:CONTAINER_PORT
```

If nginx listens on container port 80:

```bash
-p 8080:80
```

means host port 8080 forwards to nginx port 80.

Common mistake: reversing host and container ports.

---

# 4. host networking

With:

```bash
--network host
```

the container uses the host network namespace.

Port publishing with `-p` is therefore effectively unnecessary/ignored because there is no separate container network namespace to translate.

---

# 5. Required environment variables

Database images often require initialization variables.

Example MySQL:

```text
MYSQL_ROOT_PASSWORD
```

This is wrong:

```bash
-e MYSQL_ROOT_PASSWORD
```

unless the variable already exists in the host environment and is intentionally passed through.

Safer explicit form:

```bash
-e MYSQL_ROOT_PASSWORD=secret
```

Always check:

```bash
podman logs db
```

---

# 6. Memory limits

Too-low memory limits can cause:
- startup failure
- OOM kill
- repeated crashes
- failed healthchecks

Podman:

```bash
podman stats
podman inspect CONTAINER
```

Kubernetes:

```bash
kubectl describe pod POD
kubectl get pod POD -o yaml
```

Look for:

```text
OOMKilled
```

---

# 7. Network name resolution

On a default/basic network, container-name DNS behavior may not provide what you expect.

For reliable name-based communication, create a user-defined network:

```bash
podman network create appnet
```

Run both containers there.

Then:

```bash
podman exec a ping -c 1 b
```

---

# 8. SELinux bind mounts

On SELinux systems, a host bind mount can be denied even if Unix permissions look correct.

Use appropriate relabeling:

```bash
-v ./html:/usr/share/nginx/html:Z
```

or shared relabeling when appropriate:

```bash
:z
```

`Z` gives a private label for one container use.

---

# 9. Package installation in images

Debian/Ubuntu normally require package metadata refresh before install:

```Dockerfile
RUN apt-get update \
    && apt-get install -y nginx
```

Without `apt-get update`, package resolution can fail or use stale metadata.

---

# 10. Shell vs exec form

Shell form:

```Dockerfile
CMD python app.py
```

Exec form:

```Dockerfile
CMD ["python", "app.py"]
```

Exec form is preferable for long-running processes because the application receives Unix signals directly and behaves better as PID 1.

---

# 11. Layer cache ordering

Bad Node build:

```Dockerfile
COPY . /app
RUN npm install
```

Any source-code change invalidates the COPY layer and therefore reruns `npm install`.

Better:

```Dockerfile
COPY package*.json /app/
RUN npm install
COPY . /app
```

Dependency installation is reused when package files have not changed.

---

# 12. PATH mistakes

This is dangerous:

```Dockerfile
ENV PATH=/app/bin
```

because it replaces the normal system PATH.

Better:

```Dockerfile
ENV PATH="/app/bin:${PATH}"
```

Otherwise commands such as `sh` and installed binaries may no longer be found.

---

# 13. EXPOSE mismatch

`EXPOSE` is image metadata/documentation.

It does not force an application to listen on that port and it does not publish the port.

If:

```Dockerfile
EXPOSE 8080
CMD ["python3", "-m", "http.server", "3000"]
```

the app actually listens on 3000.

You would need:

```bash
-p 8080:3000
```

or change the application to listen on 8080.

---

# 14. Image-size cleanup

Package caches created in one layer remain part of that layer even if removed later.

Use one RUN:

```Dockerfile
RUN apt-get update \
    && apt-get install -y --no-install-recommends build-essential \
    && rm -rf /var/lib/apt/lists/*
```

For compiled apps, use multi-stage builds so build tools are absent from the final image.

---

# 15. USER and permissions

Switching to a non-root user before creating/copying writable files can cause permission errors.

Typical solution:

```Dockerfile
RUN useradd ...
COPY --chown=appuser:appuser ...
USER appuser
```

or create/chown directories before `USER`.

---

# 16. Kubernetes states

Important problem states:

```text
Pending
ImagePullBackOff
CrashLoopBackOff
OOMKilled
CreateContainerConfigError
Terminating
ProgressDeadlineExceeded
```

Each points toward a different category of failure.

---

# 17. Pending

A Pending pod has not successfully started.

Common causes:
- insufficient CPU/memory
- nodeSelector/affinity cannot match
- unbound/missing PVC
- scheduling constraints

Best command:

```bash
kubectl describe pod POD
```

Read the Events section.

---

# 18. ImagePullBackOff

Means Kubernetes cannot pull the image.

Check:
- image/repository spelling
- tag exists
- registry access
- imagePullSecret for private images
- network connectivity

```bash
kubectl describe pod POD
```

Events usually state the pull error.

---

# 19. CrashLoopBackOff

The container starts and repeatedly crashes.

Check current logs:

```bash
kubectl logs POD
```

Previous crashed instance:

```bash
kubectl logs POD --previous
```

This is essential when the current container restarted and the useful error is in the previous run.

---

# 20. OOMKilled

Means the container exceeded available/allowed memory and was killed.

Check:

```bash
kubectl describe pod POD
kubectl get pod POD -o yaml
```

Fix:
- raise memory limit if justified
- reduce application memory use
- correct memory leak

---

# 21. Readiness failure

A pod can be Running but not Ready.

If readiness fails:
- the container can continue running
- the pod is removed from Service traffic

Check:

```bash
kubectl describe pod POD
```

Look at readiness probe failures.

---

# 22. Service has no traffic

Start with:

```bash
kubectl get svc
kubectl get endpoints
kubectl get endpointslice
```

If no endpoints:
- Service selector may not match pod labels
- pods may not be Ready

Then verify:
- Service `port`
- `targetPort`
- application listening port

---

# 23. Deployment selector mismatch

Deployment:

```text
spec.selector.matchLabels
```

must match:

```text
spec.template.metadata.labels
```

If not, the manifest is invalid.

---

# 24. Missing PVC

If a pod references a nonexistent PVC, it cannot mount the storage.

Check:

```bash
kubectl get pvc
kubectl describe pod POD
```

Create/fix the claim.

---

# 25. Events

Events are one of the highest-value Kubernetes troubleshooting tools.

```bash
kubectl get events --sort-by=.lastTimestamp
```

or:

```bash
kubectl events
```

Events show:
- scheduling
- image pulling
- mounts
- probe failures
- restarts
- node issues

---

# 26. DNS debugging

Shell into a pod:

```bash
kubectl exec -it POD -- sh
```

Then:

```bash
nslookup SERVICE
cat /etc/resolv.conf
```

For minimal/distroless images without a shell, use an ephemeral debug container or a throwaway pod.

---

# 27. Ephemeral debug container

Example:

```bash
kubectl debug -it POD \
  --image=busybox \
  --target=CONTAINER
```

Useful when the application image has no shell/debug tools.

---

# 28. Throwaway debug pod

```bash
kubectl run tmp \
  --rm -it \
  --image=busybox \
  -- sh
```

Use it to test:
- DNS
- Service connectivity
- ports

---

# 29. Node NotReady

Possible causes:
- kubelet problems
- disk pressure
- memory pressure
- networking/CNI problems
- node-level failures

Check:

```bash
kubectl describe node NODE
```

On Minikube:

```bash
minikube logs
```

---

# 30. Stuck Terminating

Possible reasons:
- finalizers
- unavailable node/kubelet
- volume detach/unmount
- long termination grace period

Force delete:

```bash
kubectl delete pod POD \
  --grace-period=0 \
  --force
```

Risk: resources/application state may not be cleaned up gracefully.

---

# 31. API version and kind

Kubernetes resource kind must belong to the correct API group/version.

Example:

```yaml
apiVersion: v1
kind: Pod
```

not:

```yaml
apiVersion: apps/v1
kind: Pod
```

Validation catches incorrect pairings.

---

# 32. ConfigMap/Secret reference errors

If a referenced key does not exist:

```text
CreateContainerConfigError
```

Check:

```bash
kubectl describe pod POD
kubectl get configmap NAME -o yaml
kubectl get secret NAME -o yaml
```

Fix the key name or resource.

---

# 33. Namespace scoping

ConfigMaps, Secrets, Services and many other resources are namespace-scoped.

A pod cannot directly mount a Secret from another namespace.

Fix:
- create/copy the Secret into the pod's namespace
- or redesign the configuration flow

---

# 34. ProgressDeadlineExceeded

A Deployment rollout exceeded its progress deadline.

Check:

```bash
kubectl rollout status deployment/NAME
kubectl describe deployment NAME
kubectl get pods
kubectl describe pod POD
```

Common root causes:
- bad image
- readiness failure
- scheduling failure
- crash loop

---

# 35. Manifest validation

Before applying:

```bash
kubectl apply --dry-run=client -f file.yaml
```

Server-side validation:

```bash
kubectl apply --dry-run=server -f file.yaml
```

Then:

```bash
kubectl apply -f file.yaml
```

This catches:
- syntax
- indentation
- field names
- API/schema errors

---

# 36. Pod state / lastState / restartCount

Useful:

```bash
kubectl get pod POD -o jsonpath='{.status.containerStatuses[*].restartCount}'
```

State:

```bash
kubectl get pod POD -o jsonpath='{.status.containerStatuses[*].state}'
```

Last state:

```bash
kubectl get pod POD -o jsonpath='{.status.containerStatuses[*].lastState}'
```

This helps identify repeated crash causes.

---

# 37. OpenShift Route vs Kubernetes Ingress

## Ingress

Kubernetes API resource describing HTTP/HTTPS routing.

Requires an Ingress controller.

## OpenShift Route

OpenShift-native routing resource.

Integrated with OpenShift router and platform conventions.

Both can expose HTTP(S) applications, but Route is OpenShift-specific while Ingress is Kubernetes-standard.

---

# 38. LO5 golden method

For Podman:

```text
ps -a → logs → inspect → network/storage/resources
```

For Kubernetes:

```text
get → describe → events → logs/previous → related resource
```

Troubleshooting is evidence-driven. The visible status tells you which subsystem to inspect next.

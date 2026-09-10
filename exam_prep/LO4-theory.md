# LO4 — Accelerated Delivery of Multilayer Applications with Kubernetes

> Coverage follows the LO4 practice-question set. Commands and YAML examples below are study guidance.

# 1. Kubernetes resource model

Core hierarchy:

```text
Deployment
    ↓ manages
ReplicaSet
    ↓ manages
Pods
    ↓ contain
Containers
```

A Deployment is normally used for stateless, long-running applications.

You declare the desired state and Kubernetes continuously tries to make actual state match it.

---

# 2. Pod

A Pod is Kubernetes' smallest deployable unit.

A pod can contain one or more containers.

Containers in the same pod:
- share the pod IP/network namespace
- can communicate through `localhost`
- can share volumes

A bare pod is not recreated by a higher-level controller if deleted.

---

# 3. Deployment

A Deployment manages stateless replicas.

Example:

```bash
kubectl create deployment web \
  --image=nginx:1.25 \
  --replicas=3
```

Benefits:
- desired replica count
- rolling updates
- rollback
- self-healing through ReplicaSets

---

# 4. ReplicaSet

A ReplicaSet maintains the requested number of pod replicas.

Deployments normally create and manage ReplicaSets automatically.

During a rolling update:

```text
Deployment
   ↓
old ReplicaSet + new ReplicaSet
   ↓
pods transition gradually
```

---

# 5. Labels and selectors

Labels identify resources:

```yaml
metadata:
  labels:
    app: web
```

Selectors choose resources:

```yaml
selector:
  matchLabels:
    app: web
```

For a Deployment:

```text
spec.selector.matchLabels
must match
spec.template.metadata.labels
```

A Service also uses selectors to choose backend pods.

---

# 6. Declarative vs imperative

Imperative:

```bash
kubectl create deployment web --image=nginx
```

Declarative:

```bash
kubectl apply -f deployment.yaml
```

Declarative YAML is better for:
- version control
- reproducibility
- review
- automation

Imperative commands are useful for fast creation and testing.

---

# 7. Scaling

Imperative:

```bash
kubectl scale deployment web --replicas=5
```

Declarative:

```yaml
spec:
  replicas: 5
```

Then:

```bash
kubectl apply -f deployment.yaml
```

Kubernetes adjusts the ReplicaSet to match desired replicas.

---

# 8. Rolling updates

Update image:

```bash
kubectl set image deployment/web nginx=nginx:1.27
```

Watch:

```bash
kubectl rollout status deployment/web
```

A rolling update gradually replaces old pods with new pods.

This allows application availability during upgrades when configured correctly.

---

# 9. Rollout history and rollback

History:

```bash
kubectl rollout history deployment/web
```

Rollback:

```bash
kubectl rollout undo deployment/web
```

Specific revision:

```bash
kubectl rollout undo deployment/web --to-revision=2
```

`CHANGE-CAUSE` is only populated when a change cause is recorded, for example through an annotation.

Modern workflows commonly use:

```bash
kubectl annotate deployment/web \
  kubernetes.io/change-cause="Upgrade nginx to 1.27"
```

---

# 10. RollingUpdate strategy

Example:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

Meaning:
- `maxSurge: 1` → one extra pod may temporarily exist
- `maxUnavailable: 0` → no desired replica may be unavailable during the rollout

This prioritizes availability.

---

# 11. Recreate strategy

```yaml
strategy:
  type: Recreate
```

Old pods are stopped before new pods are created.

Use when old and new versions cannot run at the same time, for example:
- incompatible schema/data access
- exclusive resource ownership
- application cannot tolerate mixed versions

Trade-off: downtime.

---

# 12. Resources: requests and limits

Example:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

Requests:
- used by the scheduler
- represent resources the pod needs

Limits:
- cap resource consumption

A pod can remain Pending if no node has enough capacity for its requests.

---

# 13. revisionHistoryLimit

Example:

```yaml
spec:
  revisionHistoryLimit: 3
```

Kubernetes keeps a limited number of old ReplicaSets.

Fewer old revisions:
- less cluster clutter
- fewer rollback points

---

# 14. Failed rollout

If a new image cannot be pulled, new pods may enter:

```text
ImagePullBackOff
```

During a RollingUpdate, old healthy pods can continue serving while the new ReplicaSet fails to become ready.

Diagnosis:

```bash
kubectl rollout status deployment/web
kubectl describe deployment web
kubectl describe pod POD
```

---

# 15. Sidecars

A sidecar is an additional container in the same pod.

Example uses:
- log shipping
- proxy
- helper/adapter

Sidecar and main container share:
- pod IP/network
- localhost
- mounted shared volumes when configured

---

# 16. nodeSelector

Example:

```yaml
spec:
  template:
    spec:
      nodeSelector:
        disktype: ssd
```

If no node has matching labels, the pod remains Pending.

Check:

```bash
kubectl describe pod POD
```

---

# 17. StatefulSet

StatefulSets are designed for stateful applications that need stable identity.

With 3 replicas:

```text
redis-0
redis-1
redis-2
```

Properties:
- stable pod names
- ordered creation/termination by default
- stable storage through volumeClaimTemplates
- often paired with a headless Service

---

# 18. Headless Service

```yaml
spec:
  clusterIP: None
```

A headless Service does not provide one virtual ClusterIP.

Instead DNS can return individual pod addresses, which is useful for StatefulSets.

---

# 19. DaemonSet

A DaemonSet ensures a pod runs on each eligible node.

Typical uses:
- log agents
- monitoring agents
- node-level networking/storage components

---

# 20. Job

A Job runs work until completion.

Example use:
- batch task
- migration
- calculation

Unlike a Deployment, a completed Job is expected to stop.

---

# 21. CronJob

A CronJob creates Jobs on a schedule.

Example:

```yaml
schedule: "* * * * *"
```

A CronJob itself does not run the workload directly. It creates Jobs.

---

# 22. Volumes

Volumes provide storage to containers in a pod.

Important types in the practice sheet:
- `emptyDir`
- ConfigMap volume
- Secret volume
- PersistentVolumeClaim

---

# 23. emptyDir

Created when the pod starts and shared by containers in that pod.

Deleted when the pod itself is removed.

Example use:
- shared temporary files between sidecar and main container

Can have a size limit:

```yaml
emptyDir:
  sizeLimit: 100Mi
```

---

# 24. Persistent storage

Core model:

```text
StorageClass
    ↓ dynamic provisioning
PersistentVolume
    ↑ bound to
PersistentVolumeClaim
    ↑ mounted by
Pod
```

## PV

Cluster storage resource.

## PVC

Application request for storage.

## StorageClass

Defines classes/provisioners for storage and can support dynamic provisioning.

---

# 25. StatefulSet storage

A StatefulSet can use `volumeClaimTemplates`.

Each replica receives its own PVC.

Example:

```text
data-redis-0
data-redis-1
data-redis-2
```

Deleting a StatefulSet does not necessarily delete its PVCs automatically. Persistent data is deliberately treated separately from workload lifecycle.

---

# 26. ConfigMap

Stores non-sensitive configuration.

Create:

```bash
kubectl create configmap app-config \
  --from-literal=MODE=production
```

Consume as:
- environment variables
- files mounted in a volume

---

# 27. ConfigMap as volume

Each key can appear as a file.

Concept:

```text
ConfigMap key MODE
       ↓
/config/MODE
```

Useful when applications expect configuration files rather than environment variables.

---

# 28. subPath

`subPath` mounts one file/key at a specific path without replacing the whole target directory.

Caveat:

Updates to the ConfigMap/Secret are not automatically reflected through a `subPath` mount in the same way as a normal projected volume.

---

# 29. Secrets

Kubernetes Secrets store sensitive configuration values.

Important:

Kubernetes Secret data is base64-encoded by default, not automatically encrypted just because it is a Secret.

Security still depends on:
- API/RBAC access
- etcd encryption configuration
- namespace boundaries
- workload permissions

---

# 30. Secret consumption

Can be:
- mounted as files
- loaded as environment variables
- referenced selectively
- used for image registry authentication

Mounted secret files usually get restrictive file permissions.

---

# 31. TLS Secret

Create:

```bash
kubectl create secret tls mytls \
  --cert=tls.crt \
  --key=tls.key
```

Type:

```text
kubernetes.io/tls
```

Commonly consumed by ingress controllers or applications terminating TLS.

---

# 32. imagePullSecrets

A private registry may require authentication.

Create:

```bash
kubectl create secret docker-registry regcred \
  --docker-server=REGISTRY \
  --docker-username=USER \
  --docker-password=PASSWORD
```

Pod:

```yaml
imagePullSecrets:
  - name: regcred
```

---

# 33. Services

A Service provides stable network access to a set of pods.

Main practice types:
- ClusterIP
- NodePort
- LoadBalancer
- headless Service

The Service selects pods through labels.

---

# 34. ClusterIP

Default Service type.

Accessible from inside the cluster.

DNS:

```text
service.namespace.svc.cluster.local
```

Example:

```text
web.default.svc.cluster.local
```

---

# 35. NodePort

Exposes a Service on a port of every node.

Concept:

```text
NODE_IP:NODE_PORT
        ↓
Service
        ↓
Pod targetPort
```

---

# 36. LoadBalancer

Requests an external load balancer from the infrastructure provider.

On Minikube there is no cloud load balancer by default, so `EXTERNAL-IP` may remain pending.

`minikube tunnel` can emulate/provide access for LoadBalancer services.

---

# 37. Service ports

Example:

```yaml
ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
```

Meaning:

```text
Service port: 80
     ↓
Pod targetPort: 8080

NodePort external access:
NODE_IP:30080
```

---

# 38. Endpoints / EndpointSlice

Service selectors determine which pods back the Service.

Kubernetes creates EndpointSlices containing the matching pod addresses.

If a Service has no endpoints:
- selector may be wrong
- pods may not match
- readiness may prevent endpoints from being usable

---

# 39. Namespace DNS

Inside the same namespace:

```text
service-name
```

can usually resolve.

Across namespaces, use:

```text
service.namespace
```

or full:

```text
service.namespace.svc.cluster.local
```

---

# 40. Port-forward

Pod:

```bash
kubectl port-forward pod/POD 8080:80
```

Service:

```bash
kubectl port-forward service/SVC 8080:80
```

Service port-forward uses the Service abstraction; Pod port-forward targets one specific pod.

Useful for temporary local access and debugging, not production exposure.

---

# 41. Probes

## Liveness

Asks:

```text
Should Kubernetes restart this container?
```

## Readiness

Asks:

```text
Should this pod receive Service traffic?
```

## Startup

Protects slow-starting applications from liveness checks until startup succeeds.

---

# 42. No probes

Without probes:
- Kubernetes treats the container as alive while its process is running
- a pod can become Ready without application-specific health verification
- Kubernetes cannot detect many application-level failures

---

# 43. Init containers

Init containers run before regular application containers.

They must complete successfully before the main containers start.

Uses:
- pre-populate files
- wait for dependencies
- initialization/migration steps

---

# 44. Image identity and authenticated pulls

Private registries and Docker Hub authenticated pulls use an image-pull secret.

This is a Kubernetes Secret of type:

```text
kubernetes.io/dockerconfigjson
```

referenced through:

```yaml
imagePullSecrets:
```

---

# 45. LO4 core model

```text
Deployment / StatefulSet / DaemonSet / Job / CronJob
                 ↓
               Pods
                 ↓
             Containers
                 ↓
   ConfigMaps / Secrets / Volumes
                 ↓
              Services
                 ↓
      internal/external access
```

LO4 is about choosing Kubernetes resources that make deployment reproducible, scalable, resilient and maintainable.

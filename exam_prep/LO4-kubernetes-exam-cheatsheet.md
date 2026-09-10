# LO4 — Kubernetes Exam Cheat Sheet

> Mirrors the official LO4 practice-question set.

# A. DEPLOYMENTS / ROLLOUTS

## 1. CREATE DEPLOYMENT + EXPORT YAML

```bash
kubectl create deployment web \
  --image=nginx:1.25 \
  --replicas=3
```

Export desired object:

```bash
kubectl get deployment web -o yaml > web.yaml
```

If you want generated YAML before creating:

```bash
kubectl create deployment web \
  --image=nginx:1.25 \
  --replicas=3 \
  --dry-run=client -o yaml > web.yaml
```

---

## 2. AUTHENTICATED DOCKER HUB PULL

Create a registry Secret:

```bash
kubectl create secret docker-registry regcred \
  --docker-server=docker.io \
  --docker-username=USERNAME \
  --docker-password=PASSWORD
```

Reference:

```yaml
spec:
  imagePullSecrets:
    - name: regcred
```

Resource type:

```text
Secret
```

specifically a docker-registry/image-pull Secret.

---

## 3. SCALE 3 → 5 TWO WAYS

Imperative:

```bash
kubectl scale deployment web --replicas=5
```

Declarative:

```yaml
spec:
  replicas: 5
```

then:

```bash
kubectl apply -f web.yaml
```

Show ReplicaSet:

```bash
kubectl get rs
```

---

## 4. ROLLING UPDATE IMAGE

```bash
kubectl set image deployment/web \
  nginx=nginx:1.27
```

Watch:

```bash
kubectl rollout status deployment/web
```

---

## 5. HISTORY + ROLLBACK

```bash
kubectl rollout history deployment/web
```

Rollback:

```bash
kubectl rollout undo deployment/web
```

Specific:

```bash
kubectl rollout undo deployment/web --to-revision=2
```

Populate change cause:

```bash
kubectl annotate deployment/web \
  kubernetes.io/change-cause="Upgrade nginx to 1.27" \
  --overwrite
```

---

## 6. RollingUpdate maxSurge / maxUnavailable

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

Meaning:

```text
maxSurge 1 → one extra pod allowed
maxUnavailable 0 → no desired replica may be unavailable
```

---

## 7. RECREATE

```yaml
strategy:
  type: Recreate
```

Use when old and new versions cannot run simultaneously.

Trade-off: downtime.

---

## 8. CPU/MEMORY REQUESTS + LIMITS

```yaml
containers:
  - name: nginx
    image: nginx:1.25
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
kubectl get pod POD -o yaml
```

or:

```bash
kubectl describe pod POD
```

---

## 9. revisionHistoryLimit

```yaml
spec:
  revisionHistoryLimit: 3
```

Keeps up to 3 old ReplicaSets for rollback history, subject to Kubernetes cleanup behavior.

---

## 10. BAD IMAGE TAG

```bash
kubectl set image deployment/web \
  nginx=nginx:DOES-NOT-EXIST
```

Watch:

```bash
kubectl rollout status deployment/web
kubectl get pods
kubectl describe pod NEW_POD
```

Likely:

```text
ImagePullBackOff
```

Old healthy pods can remain serving because RollingUpdate does not need to remove them all before replacements become ready.

---

## 11. LABEL SELECTOR + HIERARCHY

```bash
kubectl get pods -l app=web
```

Relationship:

```text
Deployment selector
    ↓
ReplicaSet selector
    ↓
Pod template labels
```

Deployment selector must match pod-template labels.

---

## 12. EXPOSE DEPLOYMENT

```bash
kubectl expose deployment web \
  --port=80 \
  --target-port=80
```

Creates:

```text
Service
```

The Service selector is derived from the Deployment's pod labels/selectors.

---

## 13. SIDECAR

Pod template:

```yaml
spec:
  containers:
    - name: app
      image: nginx
    - name: sidecar
      image: busybox
      command: ["sh", "-c", "sleep 1d"]
```

Explain:

```text
same pod IP
localhost communication
can share volumes
```

---

## 14. HTTPD DEPLOYMENT + nodeSelector

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      nodeSelector:
        disktype: ssd
      containers:
        - name: httpd
          image: httpd:2.4
          ports:
            - name: http
              containerPort: 80
```

If no node has:

```text
disktype=ssd
```

pods remain Pending.

Check:

```bash
kubectl describe pod POD
```

---

## 15. BARE POD

```bash
kubectl run sleeper \
  --image=busybox \
  --command -- sleep 1d
```

Delete:

```bash
kubectl delete pod sleeper
```

Bare pod → stays deleted.

Deployment-managed pod → ReplicaSet creates replacement.

---

# B. STATEFUL / NODE / BATCH WORKLOADS

## 16. STATEFULSET REDIS + HEADLESS SERVICE

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  clusterIP: None
  selector:
    app: redis
  ports:
    - port: 6379
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis
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
```

Stable names:

```text
redis-0
redis-1
redis-2
```

DNS pattern:

```text
redis-0.redis.NAMESPACE.svc.cluster.local
```

---

## 17. DAEMONSET

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: agent
spec:
  selector:
    matchLabels:
      app: agent
  template:
    metadata:
      labels:
        app: agent
    spec:
      containers:
        - name: agent
          image: busybox
          command: ["sh", "-c", "sleep 1d"]
```

One pod per eligible node.

---

## 18. JOB

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: calculate
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: calc
          image: busybox
          command: ["sh", "-c", "echo $((6*7))"]
```

Apply:

```bash
kubectl apply -f job.yaml
kubectl get jobs
kubectl logs job/calculate
```

---

## 19. CRONJOB EVERY MINUTE

```bash
kubectl create cronjob show-date \
  --image=busybox \
  --schedule="* * * * *" \
  -- date
```

Suspend:

```bash
kubectl patch cronjob show-date \
  -p '{"spec":{"suspend":true}}'
```

List spawned Jobs:

```bash
kubectl get jobs
```

---

## 20. MANUAL JOB FROM CRONJOB

```bash
kubectl create job test-date \
  --from=cronjob/show-date
```

---

## 21. STATEFULSET ORDER

```bash
kubectl get pods -w
```

StatefulSet default ordering:

```text
redis-0 → redis-1 → redis-2
```

Termination normally reverses order.

Deployment pods do not provide stable ordered identities.

---

# C. VOLUMES / CONFIGMAPS / SECRETS

## 22. TWO CONTAINERS SHARE emptyDir

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared
spec:
  volumes:
    - name: shared-data
      emptyDir: {}
  containers:
    - name: writer
      image: busybox
      command: ["sh", "-c", "echo hello > /data/file; sleep 1d"]
      volumeMounts:
        - name: shared-data
          mountPath: /data
    - name: reader
      image: busybox
      command: ["sh", "-c", "sleep 1d"]
      volumeMounts:
        - name: shared-data
          mountPath: /data
```

Verify:

```bash
kubectl exec shared -c reader -- cat /data/file
```

---

## 23. LIST STORAGE

```bash
kubectl get storageclass
kubectl get pv
kubectl get pvc
```

Short:

```bash
kubectl get sc,pv,pvc
```

---

## 24. CONFIGMAP AS FILES

Create:

```bash
kubectl create configmap app-config \
  --from-literal=MODE=production \
  --from-literal=PORT=8080
```

Mount:

```yaml
volumes:
  - name: config
    configMap:
      name: app-config

containers:
  - name: app
    volumeMounts:
      - name: config
        mountPath: /config
```

Verify:

```bash
kubectl exec POD -- ls /config
kubectl exec POD -- cat /config/MODE
```

---

## 25. CONFIGMAP ONE KEY WITH subPath

```yaml
volumeMounts:
  - name: config
    mountPath: /etc/app/config.txt
    subPath: MODE
```

Use when you need one key at one exact path.

Caveat:

```text
subPath-mounted content does not automatically receive projected ConfigMap updates.
```

---

## 26. SECRET AS VOLUME

Create:

```bash
kubectl create secret generic db-secret \
  --from-literal=username=app \
  --from-literal=password=secret
```

Mount:

```yaml
volumes:
  - name: secret
    secret:
      secretName: db-secret
```

```yaml
volumeMounts:
  - name: secret
    mountPath: /secrets
    readOnly: true
```

Verify:

```bash
kubectl exec POD -- ls -l /secrets
kubectl exec POD -- cat /secrets/username
```

---

## 27. STATEFULSET ONE PVC PER REPLICA

Use `volumeClaimTemplates`:

```yaml
volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

Verify:

```bash
kubectl get pvc
```

Each replica receives a distinct PVC.

Deleting StatefulSet does not normally mean “delete all persistent data”; check PVCs explicitly:

```bash
kubectl get pvc
```

---

## 28. INIT CONTAINER PRE-POPULATES VOLUME

```yaml
initContainers:
  - name: init
    image: busybox
    command: ["sh", "-c", "echo hello > /data/file"]
    volumeMounts:
      - name: data
        mountPath: /data
```

Main container mounts same volume and reads `/data/file`.

Init container must finish before main container starts.

---

## 29. READ-ONLY VOLUME MOUNT

```yaml
volumeMounts:
  - name: data
    mountPath: /data
    readOnly: true
```

Test:

```bash
kubectl exec POD -- touch /data/test
```

Should fail.

---

## 30. emptyDir sizeLimit

```yaml
volumes:
  - name: cache
    emptyDir:
      sizeLimit: 100Mi
```

If use exceeds the limit, writes can fail and the pod can be affected by ephemeral-storage enforcement/eviction depending on the environment.

---

## 31. GENERIC SECRET + BASE64

```bash
kubectl create secret generic db-secret \
  --from-literal=username=app \
  --from-literal=password=secret
```

Show YAML:

```bash
kubectl get secret db-secret -o yaml
```

Decode:

```bash
echo BASE64_VALUE | base64 -d
```

Explain:

```text
base64 encoding ≠ encryption
```

---

## 32. SECRET FROM FILES

```bash
kubectl create secret generic cert-files \
  --from-file=tls.crt \
  --from-file=tls.key
```

Keys are normally file basenames:

```text
tls.crt
tls.key
```

Show:

```bash
kubectl get secret cert-files -o yaml
```

---

## 33. TLS SECRET

```bash
kubectl create secret tls mytls \
  --cert=tls.crt \
  --key=tls.key
```

Type:

```text
kubernetes.io/tls
```

Used by Ingress/controllers/apps for TLS.

---

## 34. SECRET VOLUME VS ENV

Volume advantages:
- can expose only required files
- can use restrictive file permissions
- secret is not necessarily copied into process environment

Env disadvantages:
- inherited by child processes
- may be exposed by process/debug tooling
- changing secret normally requires process/pod restart to update env

---

## 35. envFrom secretRef

```yaml
envFrom:
  - secretRef:
      name: db-secret
```

Every secret key becomes an environment variable.

---

## 36. IMAGE PULL SECRET

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

Required for private/authenticated registry pulls.

---

## 37. MOUNT ONLY ONE SECRET KEY

```yaml
volumes:
  - name: secret
    secret:
      secretName: db-secret
      items:
        - key: password
          path: db-password
```

Only selected key becomes a file.

---

# D. SERVICES / NETWORKING

## 38. CLUSTERIP + DNS

Create:

```bash
kubectl expose deployment web \
  --name=web-svc \
  --port=80 \
  --target-port=80 \
  --type=ClusterIP
```

From another pod:

```bash
nslookup web-svc.default.svc.cluster.local
```

---

## 39. NODEPORT

```bash
kubectl expose deployment web \
  --name=web-node \
  --port=80 \
  --target-port=80 \
  --type=NodePort
```

Show:

```bash
kubectl get svc web-node
```

Minikube:

```bash
minikube service web-node --url
```

or:

```text
$(minikube ip):NODEPORT
```

---

## 40. LOADBALANCER ON MINIKUBE

```bash
kubectl expose deployment web \
  --name=web-lb \
  --port=80 \
  --target-port=80 \
  --type=LoadBalancer
```

Likely:

```text
EXTERNAL-IP <pending>
```

because Minikube does not have a cloud load balancer by default.

Run:

```bash
minikube tunnel
```

---

## 41. HEADLESS SERVICE

```yaml
spec:
  clusterIP: None
```

Instead of one Service VIP, DNS returns addresses associated with individual backing pods/endpoints.

Common with StatefulSets.

---

## 42. ENDPOINTS / ENDPOINTSLICE

```bash
kubectl get endpoints
kubectl get endpointslice
```

Detailed:

```bash
kubectl describe service SVC
```

If no endpoints:

```text
check Service selector ↔ pod labels
check readiness
```

---

## 43. CROSS-NAMESPACE DNS

Same namespace:

```text
web-svc
```

Another namespace:

```text
web-svc.default
```

Full:

```text
web-svc.default.svc.cluster.local
```

Short name resolves relative to the caller's namespace, so it can fail across namespaces.

---

## 44. PORT-FORWARD SERVICE VS POD

Service:

```bash
kubectl port-forward service/web-svc 8080:80
```

Pod:

```bash
kubectl port-forward pod/POD 8080:80
```

Difference:

```text
Service → Service abstraction/backend selection
Pod → one exact pod
```

Debug/local access only, not production exposure.

---

## 45. port / targetPort / nodePort

```yaml
port: 80
targetPort: 8080
nodePort: 30080
```

Meaning:

```text
client inside cluster → Service:80
Service → Pod:8080
external NodePort → NODE_IP:30080
```

---

## 46. DEBUG CLUSTERIP CONNECTIVITY

```bash
kubectl run tmp \
  --rm -it \
  --image=busybox \
  -- sh
```

Inside:

```bash
wget -O- http://web-svc
```

or:

```bash
nc -vz web-svc 80
```

---

## 47. ONE SERVICE ACROSS TWO DEPLOYMENTS

Give pods from both Deployments the same label:

```text
app=shared-web
```

Service selector:

```yaml
selector:
  app: shared-web
```

Verify endpoints:

```bash
kubectl get endpoints
kubectl get endpointslice
```

The Service can send traffic to pods from both Deployments because selection is label-based, not Deployment-name-based.

---

## 48. SERVICE DIAGNOSIS ORDER

Use this order:

```text
1. DNS
2. Service exists
3. Endpoints / EndpointSlice
4. selector ↔ pod labels
5. pod readiness
6. Service port / targetPort
7. application listening
```

Commands:

```bash
kubectl get svc
kubectl get endpoints
kubectl get pods --show-labels
kubectl describe svc SVC
kubectl describe pod POD
```

---

## 49. NODEPORT VERIFY

```bash
kubectl get svc
minikube service SVC --url
```

or:

```text
http://$(minikube ip):NODEPORT
```

---

# E. HEALTH PROBES

## 50. HTTP LIVENESS PROBE

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 10
```

Verify:

```bash
kubectl describe pod POD
```

Liveness failure → container restart.

---

## 51. TCP READINESS FOR REDIS

```yaml
readinessProbe:
  tcpSocket:
    port: 6379
  initialDelaySeconds: 5
  periodSeconds: 5
```

Readiness failure → pod removed from Service traffic, not necessarily restarted.

---

## 52. STARTUP PROBE ~2 MINUTES

Example:

```yaml
startupProbe:
  tcpSocket:
    port: 8080
  periodSeconds: 10
  failureThreshold: 12
```

Budget:

```text
10 seconds × 12 failures = about 120 seconds
```

Startup probe protects slow startup from liveness restarts.

---

## 53. NO PROBES

Without probes:

```text
process running → Kubernetes considers container alive
pod may become Ready without application-level verification
```

Kubernetes cannot detect many “process alive but app broken” states.

---

# FAST MANIFEST CHECKS

```bash
kubectl apply --dry-run=client -f file.yaml
kubectl apply --dry-run=server -f file.yaml
```

Inspect:

```bash
kubectl get RESOURCE NAME -o yaml
kubectl describe RESOURCE NAME
```

---

# GOLDEN RULES

```text
Deployment → ReplicaSet → Pod

labels identify
selectors choose

requests → scheduling
limits → maximum usage

liveness → restart?
readiness → receive traffic?
startup → finished starting?

ClusterIP → inside cluster
NodePort → node IP + node port
LoadBalancer → external LB
headless → no virtual ClusterIP

Service selects pods by labels

ConfigMap → non-sensitive config
Secret → sensitive config
base64 ≠ encryption

StatefulSet → stable identity + ordered behavior
DaemonSet → one pod per node
Job → run to completion
CronJob → scheduled Jobs

emptyDir → pod-lifetime temporary storage
PVC → persistent storage

imagePullSecrets → registry authentication
```

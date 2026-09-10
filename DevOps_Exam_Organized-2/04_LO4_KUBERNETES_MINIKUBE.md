# LO4 — Kubernetes / Minikube

# How to use this file during the exam

Start with Ctrl+F and search the noun from the task, for example `restart`, `port`, `volume`, `Containerfile`, `rollout`, `ImagePullBackOff`, or `OpenShift`.

For practical tasks, always do three things: **perform the task → verify it → take a screenshot showing the command and proof**.

---

## QUICK KNOWLEDGE / COMMAND PATTERNS

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

---

## FULL SOLVED PRACTICE QUESTIONS

# LO4 – Accelerated delivery of multilayer applications using containers

## 1. Create Deployment `web` with nginx:1.25 and 3 replicas, then export YAML.

```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3
kubectl get deployment web -o yaml > web-deployment.yaml
kubectl get pods
```

This creates a Deployment, ReplicaSet, and 3 Pods.

## 2. Enable authenticated Docker Hub pulls.

```bash
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<dockerhub_user> \
  --docker-password=<dockerhub_token> \
  --docker-email=<email>
```

You need a `docker-registry` Secret, then reference it with `imagePullSecrets`.

## 3. Scale from 3 to 5 replicas two ways.

```bash
kubectl scale deployment web --replicas=5
kubectl get rs
kubectl get deployment web -o yaml > web-deployment.yaml
# edit replicas: 5 in the YAML if needed
kubectl apply -f web-deployment.yaml
kubectl get deployment web
kubectl get rs
```

Scaling changes the replica count. It does not necessarily create a new ReplicaSet unless the pod template changes.

## 4. Rolling update from nginx:1.25 to nginx:1.27.

```bash
kubectl set image deployment/web nginx=nginx:1.27
kubectl rollout status deployment/web
kubectl get pods -l app=web
```

Kubernetes gradually replaces old Pods with new Pods.

## 5. Rollout history, rollback, and CHANGE-CAUSE.

```bash
kubectl annotate deployment web kubernetes.io/change-cause='Update nginx to 1.27'
kubectl rollout history deployment/web
kubectl rollout undo deployment/web
kubectl rollout status deployment/web
```

`CHANGE-CAUSE` shows the recorded reason for a rollout. Populate it with the `kubernetes.io/change-cause` annotation.

## 6. RollingUpdate with `maxSurge: 1` and `maxUnavailable: 0`.

```bash
kubectl patch deployment web -p '{"spec":{"strategy":{"type":"RollingUpdate","rollingUpdate":{"maxSurge":1,"maxUnavailable":0}}}}'
kubectl describe deployment web | grep -A5 Strategy
```

Kubernetes may create 1 extra Pod during update and keeps all desired Pods available. Availability is protected, but the update may use extra capacity.

## 7. Switch strategy to Recreate.

```bash
kubectl patch deployment web -p '{"spec":{"strategy":{"type":"Recreate"}}}'
kubectl describe deployment web | grep -A3 Strategy
```

`Recreate` stops old Pods before starting new ones. Use it when two versions cannot run at the same time, for example with an app that needs exclusive database migration access.

## 8. Add CPU/memory requests and limits.

```bash
kubectl set resources deployment web \
  --requests=cpu=100m,memory=128Mi \
  --limits=cpu=500m,memory=256Mi
kubectl get pod -l app=web -o jsonpath='{.items[0].spec.containers[0].resources}'
```

Requests affect scheduling. Limits cap container usage.

## 9. Set `revisionHistoryLimit: 3`.

```bash
kubectl patch deployment web -p '{"spec":{"revisionHistoryLimit":3}}'
kubectl get deployment web -o jsonpath='{.spec.revisionHistoryLimit}'
```

Kubernetes keeps up to 3 old ReplicaSets for rollback. Older revisions are cleaned up.

## 10. Set image to a non-existent tag and observe stuck rollout.

```bash
kubectl set image deployment/web nginx=nginx:no-such-tag
kubectl rollout status deployment/web --timeout=30s
kubectl get pods
kubectl describe deployment web
```

New Pods fail with image pull errors. Old ready Pods keep serving traffic because RollingUpdate does not remove all old Pods before new ones become available.

## 11. List Pods by selector and explain Deployment → ReplicaSet → Pod.

```bash
kubectl get pods -l app=web
kubectl get rs -l app=web
kubectl get deployment web -o jsonpath='{.spec.selector.matchLabels}'
```

The Deployment selector matches ReplicaSets, and ReplicaSets manage Pods whose template labels match that selector.

## 12. Expose a Deployment.

```bash
kubectl expose deployment web --port=80 --target-port=80 --type=ClusterIP
kubectl get service web
kubectl describe service web
```

A Service is created. Its selector is derived from the Deployment’s Pod labels.

## 13. Add a sidecar container.

```bash
kubectl patch deployment web --type='json' -p='[
{"op":"add","path":"/spec/template/spec/containers/-","value":{"name":"sidecar","image":"busybox","command":["sh","-c","while true; do date; sleep 10; done"]}}
]'
kubectl get pods -l app=web
```

Containers in the same Pod share network namespace and can share volumes, but they have separate filesystems and processes.

## 14. Deployment manifest for httpd:2.4 with nodeSelector.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd-deploy
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

```bash
kubectl apply -f httpd-deploy.yaml
kubectl get pods
kubectl describe pod <pod-name>
```

If no node has label `disk=fast`, Pods stay Pending.

## 15. Bare Pod versus Deployment-managed Pod.

```bash
kubectl run barebox --image=busybox --command -- sleep 1d
kubectl delete pod barebox
kubectl get pod barebox
kubectl delete pod -l app=web
kubectl get pods -l app=web
```

A bare Pod is gone when deleted. A Deployment-managed Pod is recreated by the ReplicaSet.

## 16. StatefulSet for redis with headless Service.

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
      name: redis
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
              name: redis
```

```bash
kubectl apply -f redis-sts.yaml
kubectl get pods -l app=redis
kubectl run dns --rm -it --image=busybox -- nslookup redis-0.redis.default.svc.cluster.local
```

Stable names are `redis-0`, `redis-1`, and `redis-2`.

## 17. DaemonSet running a busybox agent.

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
          command: ["sh", "-c", "while true; do echo agent; sleep 60; done"]
```

```bash
kubectl apply -f daemonset.yaml
kubectl get daemonset
kubectl get pods -l app=agent -o wide
```

A DaemonSet runs one Pod per matching node.

## 18. Job that computes once and completes.

```bash
kubectl create job calc --image=perl -- perl -Mbignum=bpi -wle 'print bpi(20)'
kubectl get jobs
kubectl logs job/calc
```

The Job shows `1/1` completions after the command finishes.

## 19. CronJob every minute, suspend it, and list spawned Jobs.

```bash
kubectl create cronjob datejob --image=busybox --schedule='* * * * *' -- date
kubectl get cronjob
kubectl get jobs --watch
kubectl patch cronjob datejob -p '{"spec":{"suspend":true}}'
kubectl get jobs
```

Suspending stops future scheduled Jobs, not already created Jobs.

## 20. Manually run a Job from a CronJob.

```bash
kubectl create job manual-datejob --from=cronjob/datejob
kubectl get job manual-datejob
kubectl logs job/manual-datejob
```

This tests the CronJob template on demand.

## 21. StatefulSet ordered creation/termination versus Deployment.

```bash
kubectl scale statefulset redis --replicas=0
kubectl get pods -w
kubectl scale statefulset redis --replicas=3
kubectl get pods -w
```

StatefulSets create and delete Pods in order by ordinal. Deployments create/delete interchangeable Pods without stable identity.

## 22. Multi-container Pod sharing `emptyDir`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-volume-pod
spec:
  volumes:
    - name: shared
      emptyDir: {}
  containers:
    - name: writer
      image: busybox
      command: ["sh", "-c", "echo hello > /shared/file.txt; sleep 1d"]
      volumeMounts:
        - name: shared
          mountPath: /shared
    - name: reader
      image: busybox
      command: ["sh", "-c", "while true; do cat /shared/file.txt; sleep 30; done"]
      volumeMounts:
        - name: shared
          mountPath: /shared
```

```bash
kubectl apply -f shared-volume-pod.yaml
kubectl logs shared-volume-pod -c reader
```

Both containers mount the same `emptyDir` volume.

## 23. List StorageClasses, PVs, and PVCs.

```bash
kubectl get storageclass
kubectl get pv
kubectl get pvc
```

StorageClasses define dynamic provisioning. PVs are cluster storage resources. PVCs are user claims for storage.

## 24. Mount ConfigMap as volume, each key as a file.

```bash
kubectl create configmap app-config --from-literal=app.properties='mode=dev' --from-literal=message='hello'
cat > cm-volume-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: cm-volume-pod
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "ls -l /config && cat /config/message && sleep 1d"]
      volumeMounts:
        - name: config
          mountPath: /config
  volumes:
    - name: config
      configMap:
        name: app-config
EOF
kubectl apply -f cm-volume-pod.yaml
kubectl logs cm-volume-pod
```

Each ConfigMap key becomes a file.

## 25. Mount one ConfigMap key with `subPath`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cm-subpath-pod
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "cat /app/message.txt; sleep 1d"]
      volumeMounts:
        - name: config
          mountPath: /app/message.txt
          subPath: message
  volumes:
    - name: config
      configMap:
        name: app-config
```

`subPath` is needed when one key must be mounted at one exact file path. Caveat: subPath mounts do not receive live ConfigMap updates.

## 26. Mount Secret as volume and verify decoded files/permissions.

```bash
kubectl create secret generic db-secret --from-literal=username=admin --from-literal=password=secret
cat > secret-vol-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: secret-vol-pod
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "ls -l /secret && cat /secret/username && sleep 1d"]
      volumeMounts:
        - name: secret
          mountPath: /secret
          readOnly: true
  volumes:
    - name: secret
      secret:
        secretName: db-secret
EOF
kubectl apply -f secret-vol-pod.yaml
kubectl logs secret-vol-pod
```

Secret files contain decoded values and are mounted with restrictive permissions.

## 27. StatefulSet PVC per replica and deletion behavior.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-pvc
spec:
  serviceName: redis
  replicas: 3
  selector:
    matchLabels:
      app: redis-pvc
  template:
    metadata:
      labels:
        app: redis-pvc
    spec:
      containers:
        - name: redis
          image: redis:7
          volumeMounts:
            - name: data
              mountPath: /data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

```bash
kubectl apply -f redis-pvc.yaml
kubectl get pvc
kubectl delete statefulset redis-pvc
kubectl get pvc
```

PVCs remain after StatefulSet deletion to protect data.

## 28. Use initContainer to pre-populate a shared volume.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  volumes:
    - name: data
      emptyDir: {}
  initContainers:
    - name: init
      image: busybox
      command: ["sh", "-c", "echo initialized > /data/file.txt"]
      volumeMounts:
        - name: data
          mountPath: /data
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "cat /data/file.txt; sleep 1d"]
      volumeMounts:
        - name: data
          mountPath: /data
```

The initContainer runs first, writes data, and the main container reads it.

## 29. Set `readOnly: true` and prove writes fail.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readonly-volume-pod
spec:
  volumes:
    - name: config
      configMap:
        name: app-config
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "echo test > /config/newfile"]
      volumeMounts:
        - name: config
          mountPath: /config
          readOnly: true
```

```bash
kubectl apply -f readonly-volume-pod.yaml
kubectl logs readonly-volume-pod
```

The write fails because the volume mount is read-only.

## 30. Set `sizeLimit` on `emptyDir`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-limit
spec:
  volumes:
    - name: cache
      emptyDir:
        sizeLimit: 10Mi
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "df -h /cache; sleep 1d"]
      volumeMounts:
        - name: cache
          mountPath: /cache
```

If the container exceeds the limit, writes fail or the Pod may be evicted depending on node storage pressure.

## 31. Create generic Secret and show base64 encoding.

```bash
kubectl create secret generic db-auth --from-literal=username=admin --from-literal=password=secret
kubectl get secret db-auth -o yaml
kubectl get secret db-auth -o jsonpath='{.data.password}' | base64 -d
```

Kubernetes Secret data is base64-encoded, not encrypted by default.

## 32. Create Secret from certificate/key files and identify keys.

```bash
echo 'fake cert' > tls.crt
echo 'fake key' > tls.key
kubectl create secret generic cert-secret --from-file=tls.crt --from-file=tls.key
kubectl get secret cert-secret -o jsonpath='{.data}'
```

The resulting Secret keys are `tls.crt` and `tls.key`.

## 33. Create `kubernetes.io/tls` Secret.

```bash
openssl req -x509 -nodes -newkey rsa:2048 -keyout tls.key -out tls.crt -subj '/CN=example.local' -days 1
kubectl create secret tls web-tls --cert=tls.crt --key=tls.key
kubectl get secret web-tls -o yaml
```

TLS Secrets are consumed by Ingress controllers, Routes, or applications that need certificate/key files.

## 34. Secret volume trade-offs versus env vars.

```bash
kubectl apply -f secret-vol-pod.yaml
kubectl exec secret-vol-pod -- ls -l /secret
```

Secret volumes can be read as files and can update without restarting the Pod. Env vars are simpler but are visible in process environments and require restart to change.

## 35. Use `envFrom` with `secretRef`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-envfrom-pod
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "env | grep -E 'username|password'; sleep 1d"]
      envFrom:
        - secretRef:
            name: db-auth
```

All Secret keys become environment variables.

## 36. Create image-pull Secret and reference it.

```bash
kubectl create secret docker-registry regcred \
  --docker-server=quay.io \
  --docker-username=<quay_user> \
  --docker-password=<quay_password> \
  --docker-email=<email>
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-image-pod
spec:
  imagePullSecrets:
    - name: regcred
  containers:
    - name: app
      image: quay.io/<quay_user>/<private_repo>:latest
```

It is required when pulling from a private registry or when authenticated pulls are needed.

## 37. Selectively mount one Secret key.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: one-secret-key
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "ls -l /secret && cat /secret/password; sleep 1d"]
      volumeMounts:
        - name: secret
          mountPath: /secret
  volumes:
    - name: secret
      secret:
        secretName: db-auth
        items:
          - key: password
            path: password
```

Only the selected key is mounted.

## 38. ClusterIP Service and DNS resolution.

```bash
kubectl expose deployment web --name=web-clusterip --port=80 --target-port=80 --type=ClusterIP
kubectl run dns-test --rm -it --image=busybox -- nslookup web-clusterip.default.svc.cluster.local
```

ClusterIP Services are reachable inside the cluster through DNS.

## 39. NodePort Service and access through Minikube.

```bash
kubectl expose deployment web --name=web-nodeport --port=80 --target-port=80 --type=NodePort
kubectl get svc web-nodeport
minikube service web-nodeport --url
curl $(minikube service web-nodeport --url)
```

NodePort exposes the Service on a port on each node.

## 40. LoadBalancer Service on Minikube.

```bash
kubectl expose deployment web --name=web-lb --port=80 --target-port=80 --type=LoadBalancer
kubectl get svc web-lb
minikube tunnel
```

On Minikube, `EXTERNAL-IP` is often `<pending>` because there is no real cloud load balancer. `minikube tunnel` provides one locally.

## 41. Headless Service returns per-pod records.

```bash
kubectl get svc redis
kubectl run dns-redis --rm -it --image=busybox -- nslookup redis.default.svc.cluster.local
kubectl run dns-redis0 --rm -it --image=busybox -- nslookup redis-0.redis.default.svc.cluster.local
```

A headless Service has no virtual IP. DNS points directly to Pod records.

## 42. Inspect Endpoints/EndpointSlice.

```bash
kubectl get endpoints web-clusterip
kubectl get endpointslice -l kubernetes.io/service-name=web-clusterip
kubectl describe service web-clusterip
```

Endpoints are populated from Pods matching the Service selector and readiness state.

## 43. Cross-namespace access using FQDN.

```bash
kubectl create namespace testns
kubectl run -n testns tmp --rm -it --image=busybox -- nslookup web-clusterip.default.svc.cluster.local
kubectl run -n testns tmp2 --rm -it --image=busybox -- nslookup web-clusterip
```

The FQDN works across namespaces. The short name fails because it searches the current namespace.

## 44. Port-forward Service versus Pod.

```bash
kubectl port-forward service/web-clusterip 8080:80
kubectl port-forward pod/<pod-name> 8081:80
```

Forwarding to a Service targets one backing Pod through the Service. Forwarding to a Pod targets that exact Pod only.

## 45. Service `port`, `targetPort`, and `nodePort`.

```yaml
ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
```

`port` is the Service port inside the cluster. `targetPort` is the container port. `nodePort` is the external node port for NodePort/LoadBalancer Services.

## 46. Verify ClusterIP connectivity from a debug Pod.

```bash
kubectl run tmp --rm -it --image=busybox -- sh
wget -qO- http://web-clusterip.default.svc.cluster.local
nc -vz web-clusterip.default.svc.cluster.local 80
exit
```

This tests connectivity from inside the cluster.

## 47. One Service load-balances across two Deployments sharing a label.

```bash
kubectl create deployment app-a --image=hashicorp/http-echo -- -text='A'
kubectl create deployment app-b --image=hashicorp/http-echo -- -text='B'
kubectl label deployment app-a group=shared
kubectl label deployment app-b group=shared
kubectl patch deployment app-a -p '{"spec":{"template":{"metadata":{"labels":{"group":"shared","app":"app-a"}}}}}'
kubectl patch deployment app-b -p '{"spec":{"template":{"metadata":{"labels":{"group":"shared","app":"app-b"}}}}}'
kubectl expose deployment app-a --name=shared-svc --port=5678 --target-port=5678
kubectl patch service shared-svc -p '{"spec":{"selector":{"group":"shared"}}}'
kubectl get endpoints shared-svc
```

The Service sends traffic to all ready Pods with label `group=shared`.

## 48. Diagnose Pod cannot reach Service.

```bash
kubectl run debug --rm -it --image=busybox -- sh
nslookup web-clusterip.default.svc.cluster.local
wget -S -O- http://web-clusterip.default.svc.cluster.local
exit
kubectl get svc web-clusterip -o yaml
kubectl get endpoints web-clusterip
kubectl get pods -l app=web --show-labels
kubectl describe pod -l app=web
```

Check in order: DNS, endpoints, selector labels, readiness, and port mapping.

## 49. Expose a Service with NodePort and verify.

```bash
kubectl expose deployment web --name=web-np2 --type=NodePort --port=80 --target-port=80
kubectl get svc web-np2
curl $(minikube service web-np2 --url)
```

The NodePort Service is reachable from outside the cluster through the Minikube node.

## 50. Add HTTP livenessProbe to nginx.

```bash
kubectl patch deployment web --type='json' -p='[
{"op":"add","path":"/spec/template/spec/containers/0/livenessProbe","value":{"httpGet":{"path":"/","port":80},"initialDelaySeconds":5,"periodSeconds":10}}
]'
kubectl describe pod -l app=web | grep -A5 Liveness
```

The kubelet restarts the container if the HTTP liveness probe fails repeatedly.

## 51. Add tcpSocket readiness probe to redis.

```bash
kubectl patch statefulset redis --type='json' -p='[
{"op":"add","path":"/spec/template/spec/containers/0/readinessProbe","value":{"tcpSocket":{"port":6379},"initialDelaySeconds":5,"periodSeconds":10}}
]'
kubectl describe pod redis-0 | grep -A5 Readiness
```

The Pod becomes Ready only when TCP port `6379` accepts connections.

## 52. Configure startup probe for around 2-minute boot.

```yaml
startupProbe:
  httpGet:
    path: /
    port: 80
  periodSeconds: 10
  failureThreshold: 12
```

Budget is `10 × 12 = 120 seconds`. During startup probe failure, liveness/readiness are delayed, which protects slow-starting apps.

## 53. Default behavior with no probes.

If no probes are defined, Kubernetes assumes the container is alive after it starts and ready unless it crashes. It will not detect broken application logic, only process exit.

---

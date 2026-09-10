# Ready-to-Copy Templates

Replace the uppercase placeholders. These are starting points; adjust ports, image names, labels, paths, and environment variables to match the question exactly.

## Podman run

```bash
podman run -d \
  --name NAME \
  --network NETWORK \
  -p HOST_PORT:CONTAINER_PORT \
  -e KEY=value \
  -v HOST_OR_VOLUME:CONTAINER_PATH:Z \
  IMAGE
```

## Containerfile — UBI9 + Apache

```Dockerfile
FROM registry.access.redhat.com/ubi9/ubi:latest
RUN dnf -y install httpd && dnf clean all
RUN echo "YOUR TEXT" > /var/www/html/index.html
EXPOSE 80
CMD ["/usr/sbin/httpd", "-DFOREGROUND"]
```

```bash
podman build -t my-httpd:1 .
podman run -d --name my-httpd -p 8080:80 my-httpd:1
curl http://localhost:8080
```

## Containerfile — Alpine + nginx

```Dockerfile
FROM alpine:latest
RUN apk add --no-cache nginx
ENV STUDENT="YOUR VALUE"
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

## Compose — application + PostgreSQL

```yaml
services:
  db:
    image: docker.io/library/postgres:16
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: password
    volumes:
      - dbdata:/var/lib/postgresql/data

  app:
    image: YOUR_IMAGE
    depends_on:
      - db
    ports:
      - "8080:8080"
    environment:
      DB_HOST: db
      DB_NAME: appdb
      DB_USER: appuser
      DB_PASSWORD: password

volumes:
  dbdata:
```

## Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx:1.27
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "64Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
```

## Kubernetes ClusterIP Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

## Kubernetes NodePort Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
  type: NodePort
```

## ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_MODE: production
  config.txt: |
    example=true
```

## Secret

Fastest imperative form:

```bash
kubectl create secret generic app-secret \
  --from-literal=username=user \
  --from-literal=password=pass
```

Use as env vars:

```yaml
envFrom:
  - secretRef:
      name: app-secret
```

## PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

## Probes

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 2
  periodSeconds: 5
```

## Fast Kubernetes debug sequence

```bash
kubectl get pods -o wide
kubectl describe pod POD
kubectl logs POD
kubectl logs POD --previous
kubectl get events --sort-by=.lastTimestamp
kubectl get svc,endpoints
```

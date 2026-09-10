# DevOps Exam — Full Solved Version

Use this as an exam reference. Replace placeholders such as `YOUR_QUAY_USERNAME` where needed.

---

# Learning Outcome 1 — Containers

## 1. Start InfluxDB in background, port 8086, reachable only from host

Use loopback binding so the service is NOT exposed to the rest of the network:

```bash
podman run -d \
  --name influxdb \
  -p 127.0.0.1:8086:8086 \
  docker.io/library/influxdb:latest
```

Verify:

```bash
podman ps
curl http://127.0.0.1:8086/health
```

Important:

```text
127.0.0.1:8086:8086
HOST_IP : HOST_PORT : CONTAINER_PORT
```

Binding to `127.0.0.1` means only the host can reach it.

---

## 2a. Display logs

```bash
podman logs influxdb
```

Live logs:

```bash
podman logs -f influxdb
```

---

## 2b. Inspect container and show environment variables

Full inspect:

```bash
podman inspect influxdb
```

Only environment variables:

```bash
podman inspect --format '{{range .Config.Env}}{{println .}}{{end}}' influxdb
```

Alternative:

```bash
podman inspect --format '{{.Config.Env}}' influxdb
```

---

## 2c. Stop and remove

```bash
podman stop influxdb
podman rm influxdb
```

Verify:

```bash
podman ps -a
```

---

## 3. Container vs VM — data storage

A container normally stores changes in its writable container layer. That data is tied to that container and can disappear when the container is removed, so persistent application data should be stored in a named volume or bind mount outside the container.

A VM normally stores data on its virtual disk. The virtual disk persists when applications stop or the VM reboots and usually remains until the VM/disk itself is deleted.

Containers are designed to be disposable and recreated from images, so persistent data should be externalized. VMs behave more like complete persistent machines and include their own virtual disks and operating system.

---

# Learning Outcome 2 — Container Images / Dockerfiles

## 1. Pull hello-world, tag it, push to public Quay repository

Pull:

```bash
podman pull docker.io/library/hello-world:latest
```

Tag locally:

```bash
podman tag docker.io/library/hello-world:latest firstcontainer:1
```

Check:

```bash
podman images
```

Log in to Quay:

```bash
podman login quay.io
```

In the Quay web interface create a PUBLIC repository named:

```text
firstcontainer
```

Tag with the full registry name:

```bash
podman tag firstcontainer:1 quay.io/YOUR_QUAY_USERNAME/firstcontainer:1
```

Push:

```bash
podman push quay.io/YOUR_QUAY_USERNAME/firstcontainer:1
```

Verify:

```bash
podman images
```

---

## 2. Dockerfile image that runs kubectl

Create a file named `Dockerfile`:

```dockerfile
FROM alpine:3.20

RUN apk add --no-cache curl ca-certificates

RUN curl -L "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" \
    -o /usr/local/bin/kubectl \
    && chmod +x /usr/local/bin/kubectl

ENTRYPOINT ["kubectl"]
```

Build:

```bash
podman build -t kubectl-image .
```

Test the image:

```bash
podman run --rm kubectl-image version --client
```

Because `kubectl` is the entry point, arguments written after the image name are passed directly to kubectl:

```bash
podman run --rm kubectl-image get all
```

Concept:

```text
podman run kubectl-image get all
                    ↓
             kubectl get all
```

If actual access to the host's Kubernetes cluster is required, the container also needs access to the kubeconfig and cluster network.

---

## 3. Advanced Ruby Dockerfile

Expected Dockerfile:

```dockerfile
FROM ruby:3.3-slim

WORKDIR /app

COPY Gemfile Gemfile.lock ./

ENV BUNDLE_FROZEN=true

RUN gem install bundler && bundle install

COPY . .

EXPOSE 3000

ENTRYPOINT ["ruby", "./app.rb"]
```

Build:

```bash
podman build -t rails-app .
```

Run:

```bash
podman run -d \
  --name rails-app \
  -p 3000:3000 \
  rails-app
```

Check:

```bash
podman ps
podman logs rails-app
curl http://localhost:3000
```

What each required line does:

```text
FROM       -> base image
WORKDIR    -> working directory inside image
COPY       -> copy files into image
ENV        -> environment variable
RUN        -> execute command while BUILDING image
EXPOSE     -> document application port
ENTRYPOINT -> command executed when container starts
```

---

# Learning Outcome 3 — Multi-container App / Networking / Persistence

## 1. Drupal + MySQL using Podman

### Step 1 — Create network

```bash
podman network create drupal-net
```

Check:

```bash
podman network ls
```

### Step 2 — Run MySQL

```bash
podman run -d \
  --name mysql \
  --network drupal-net \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  -e MYSQL_DATABASE=drupal \
  -e MYSQL_USER=drupal \
  -e MYSQL_PASSWORD=drupalpass \
  docker.io/library/mysql:8.0
```

Check:

```bash
podman ps
podman logs mysql
```

Wait until MySQL is ready.

Optional check:

```bash
podman exec mysql mysqladmin ping -uroot -prootpass
```

### Step 3 — Run Drupal

```bash
podman run -d \
  --name drupal \
  --network drupal-net \
  -p 8080:80 \
  docker.io/library/drupal:latest
```

Check:

```bash
podman ps
```

Open:

```text
http://localhost:8080
```

During Drupal installation use:

```text
Database type: MySQL / MariaDB
Database name: drupal
Database user: drupal
Database password: drupalpass
Database host: mysql
```

The important part is:

```text
Database host = mysql
```

Because both containers are on `drupal-net`, the Drupal container can resolve the MySQL container by its container name.

Verify network membership:

```bash
podman network inspect drupal-net
```

---

## 2. Persist Drupal data in volume `drupal_data`

Create the volume:

```bash
podman volume create drupal_data
```

If Drupal was already created, remove only the Drupal container:

```bash
podman stop drupal
podman rm drupal
```

Recreate Drupal with the volume:

```bash
podman run -d \
  --name drupal \
  --network drupal-net \
  -p 8080:80 \
  -v drupal_data:/var/www/html/sites \
  docker.io/library/drupal:latest
```

Verify:

```bash
podman volume ls
podman volume inspect drupal_data
podman inspect drupal
```

Demonstrate the container can write to the volume:

```bash
podman exec drupal sh -c 'echo "volume works" > /var/www/html/sites/volume-test.txt'
podman exec drupal cat /var/www/html/sites/volume-test.txt
```

The named volume survives removal/recreation of the Drupal container.

---

# Learning Outcome 4 — Kubernetes

## 1. Deployment `mysql-deployment`, one replica, `mysql:latest`

Create:

```bash
kubectl create deployment mysql-deployment \
  --image=mysql:latest \
  --replicas=1
```

MySQL needs initialization configuration, otherwise the container will terminate. Set a root password:

```bash
kubectl set env deployment/mysql-deployment \
  MYSQL_ROOT_PASSWORD=rootpass
```

Check:

```bash
kubectl get deployments
kubectl get pods
kubectl rollout status deployment/mysql-deployment
```

Useful troubleshooting:

```bash
kubectl describe deployment mysql-deployment
kubectl describe pod POD_NAME
kubectl logs POD_NAME
```

---

## 2. Enable internal connectivity

Create a ClusterIP Service:

```bash
kubectl expose deployment mysql-deployment \
  --name=mysql-service \
  --port=3306 \
  --target-port=3306 \
  --type=ClusterIP
```

Check:

```bash
kubectl get services
```

Expected idea:

```text
mysql-service  -> ClusterIP -> mysql-deployment Pod :3306
```

`ClusterIP` provides connectivity inside the Kubernetes cluster.

---

## 3. Connect to the MySQL pod on its port

Find the pod:

```bash
kubectl get pods
```

Save its name automatically:

```bash
POD=$(kubectl get pods -l app=mysql-deployment -o jsonpath='{.items[0].metadata.name}')
echo $POD
```

Port-forward MySQL port 3306:

```bash
kubectl port-forward pod/$POD 3306:3306
```

Keep that terminal running.

From another terminal, if the MySQL client exists:

```bash
mysql -h 127.0.0.1 -P 3306 -u root -p
```

Password:

```text
rootpass
```

If no MySQL client is installed, test the port with:

```bash
nc -vz 127.0.0.1 3306
```

The main command to note down for the task is:

```bash
kubectl port-forward pod/$POD 3306:3306
```

Alternative direct connection from inside the pod:

```bash
kubectl exec -it $POD -- mysql -uroot -prootpass
```

---

# Learning Outcome 5 — Troubleshooting

## 1. Broken Podman command

Broken:

```bash
podman run -name mycontainer -p 8080:80 arm32v5/nginx
```

Main mistake:

```text
-name
```

must be:

```text
--name
```

Corrected:

```bash
podman run --name mycontainer -p 8080:80 nginx
```

Why:

`--name` is a long command-line option and therefore requires two hyphens.

Also note: `arm32v5/nginx` is an ARM image. On a normal x86_64 exam VM this may be incompatible and can cause an architecture/exec-format problem. The normal `nginx` image is safer because it provides the appropriate architecture.

If background mode is desired:

```bash
podman run -d --name mycontainer -p 8080:80 nginx
```

---

## 2. Broken Dockerfile

Broken:

```dockerfile
FROM python:3.8
WRKDIR /app
EXPOSE 8000
ENTRYPOINT ["pythn", "-m", "http.server"]
```

Mistakes:

```text
WRKDIR -> WORKDIR
pythn  -> python
```

Correct:

```dockerfile
FROM python:3.8

WORKDIR /app

EXPOSE 8000

ENTRYPOINT ["python", "-m", "http.server", "8000"]
```

Build and test:

```bash
podman build -t python-web .
podman run -d --name python-web -p 8000:8000 python-web
curl localhost:8000
```

Note: Python's `http.server` already defaults to port 8000, but specifying `"8000"` makes the intended port explicit.

---

## 3. Broken Kubernetes Deployment manifest

Correct manifest:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: myapp

spec:
  replicas: 1

  selector:
    matchLabels:
      app: myapp

  template:
    metadata:
      labels:
        app: myapp

    spec:
      containers:
        - name: mycontainer
          image: httpd
          ports:
            - name: http
              containerPort: 80
```

Important mistakes to identify:

1. YAML indentation/structure must be correct.
2. `containers` is a LIST, so the container needs `-`.
3. `ports` is also a LIST.
4. `containerPort` should be an integer:

```yaml
containerPort: 80
```

not:

```yaml
containerPort: "80"
```

5. Deployment selector and Pod labels must match:

```yaml
selector:
  matchLabels:
    app: myapp
```

and:

```yaml
labels:
  app: myapp
```

Validate/apply:

```bash
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl get pods
```

If it fails:

```bash
kubectl apply -f deployment.yaml --dry-run=client
```

Then inspect:

```bash
kubectl describe deployment myapp
kubectl describe pod POD_NAME
kubectl logs POD_NAME
```

---

# Learning Outcome 6 — Theory Answers

## 1. Kubernetes vs Docker Swarm — learning curve and complexity

Docker Swarm has a lower learning curve. A new DevOps team that already understands Docker can learn its basic concepts quickly because the configuration and commands are relatively simple. It requires fewer concepts and less initial operational knowledge.

Kubernetes has a significantly steeper learning curve. Teams must understand Pods, Deployments, Services, ConfigMaps, Secrets, namespaces, networking, storage, health probes, scheduling and YAML manifests. Operating production Kubernetes also requires knowledge of monitoring, security, upgrades and cluster administration.

However, Kubernetes has much stronger industry adoption, documentation, training material, certifications and community support. Swarm is easier to begin with, but Kubernetes provides substantially more functionality and better long-term career and ecosystem support.

Conclusion: Swarm is simpler for a small or inexperienced team that needs basic orchestration quickly. Kubernetes requires more initial investment but is generally the stronger choice for complex and growing production systems.

---

## 2. Kubernetes vs OpenShift — multi-cloud, hybrid deployment, lock-in

Kubernetes is highly portable and can run on public clouds, private infrastructure, virtual machines and bare metal. Major providers such as AWS, Azure and Google Cloud provide managed Kubernetes services. Because Kubernetes is an open standard with a large ecosystem, workloads can generally be moved between providers with relatively little vendor lock-in if applications avoid provider-specific services.

OpenShift is based on Kubernetes and can also support hybrid and multi-cloud deployments. It adds an integrated platform around Kubernetes, including security defaults, developer tooling, operators, routing, monitoring and enterprise management.

OpenShift can improve consistency across environments, but using OpenShift-specific features creates greater dependence on the Red Hat ecosystem. Kubernetes by itself generally offers greater flexibility and the lowest platform-specific lock-in, while OpenShift trades some of that flexibility for a more integrated enterprise platform and vendor support.

Conclusion: both support hybrid and multi-cloud strategies, but plain Kubernetes gives maximum portability while OpenShift provides stronger integrated enterprise functionality.

---

## 3. Why Kubernetes is preferred for microservices

Kubernetes is well suited to microservices because it automates deployment, scaling, networking and recovery for many independently deployed services.

Deployments maintain the desired number of Pod replicas and support rolling updates and rollbacks. Services provide stable discovery and networking even when individual Pods are recreated. Readiness and liveness probes allow Kubernetes to detect unhealthy application instances, while the scheduler and controllers automatically replace failed Pods.

Horizontal scaling makes it possible to run additional replicas when demand increases. ConfigMaps and Secrets separate configuration from container images. Namespaces and policies help organize and isolate workloads.

Kubernetes also has a very large ecosystem including Helm, Operators, monitoring systems, service meshes, CI/CD integrations and managed Kubernetes offerings from major cloud vendors.

For microservices, the main advantage is therefore not simply running containers. Kubernetes continuously manages a large distributed application and maintains its desired state despite failures, updates and scaling requirements.

Conclusion: Kubernetes is preferred for microservices because it combines orchestration, self-healing, service discovery, scaling, rolling deployment and a mature ecosystem in one widely supported platform.

---

# FAST EXAM INDEX

If the question says...

```text
start container        -> podman run
background             -> -d
host only              -> -p 127.0.0.1:HOST:CONTAINER
network accessible     -> -p HOST:CONTAINER
logs                   -> podman logs
inspect                -> podman inspect
environment variables  -> podman inspect --format ...
stop                   -> podman stop
remove                 -> podman rm

pull image             -> podman pull
tag image              -> podman tag
push image             -> podman push
build image            -> podman build -t NAME .
Dockerfile base        -> FROM
install during build   -> RUN
working directory      -> WORKDIR
environment variable   -> ENV
copy files             -> COPY
port                   -> EXPOSE
startup command        -> CMD / ENTRYPOINT

container network      -> podman network create
persistent volume      -> podman volume create
mount volume           -> -v VOLUME:CONTAINER_PATH

Kubernetes deployment  -> kubectl create deployment
internal connectivity  -> ClusterIP Service
replicas               -> kubectl scale
see Pods               -> kubectl get pods
see everything         -> kubectl get all
details/problem        -> kubectl describe
logs                   -> kubectl logs
connect local port     -> kubectl port-forward
run inside Pod         -> kubectl exec
apply YAML             -> kubectl apply -f FILE
```

---

# 30-SECOND ORIENTATION

For every practical question:

```text
1. WHAT am I creating?
   container / image / network / volume / deployment / service

2. WHAT requirements are stated?
   name / image / port / environment variables / replicas / persistence

3. Execute ONE requirement at a time.

4. VERIFY:
   podman ps
   podman images
   podman inspect ...
   kubectl get all

5. Only then continue.
```

# Previous Example Exam — Fully Solved

This is the fastest file to open if the actual exam resembles the previous one. Search for `Tomcat`, `Redis`, `hello-world`, `Quay`, `UBI9`, `nginx`, `Drupal`, `PostgreSQL`, or `bind mount`.

---

> Replace placeholders such as `YOUR_QUAY_USERNAME`, `yourusername`, `Name Surname`, and `VM_IP` with your actual values.

---

# Learning Outcome 1 — 12 points / 30 min

## 1. Start Tomcat in the background and map container port 8080 to host port 9080

```bash
podman pull docker.io/library/tomcat:latest

podman run -d \
  --name tomcat \
  -p 9080:8080 \
  docker.io/library/tomcat:latest
```

Check that it is running:

```bash
podman ps
```

Test from the VM:

```bash
curl -I http://localhost:9080
```

Because no host IP was specified in `-p 9080:8080`, Podman publishes the port on the host interfaces, so the service can be reached through:

```text
http://VM_IP:9080
```

If `firewalld` is active and blocks incoming traffic:

```bash
sudo firewall-cmd --permanent --add-port=9080/tcp
sudo firewall-cmd --reload
```

> The official Tomcat image can return HTTP `404` on `/`. That still proves Tomcat is reachable; the task only asks you to start and expose the container.

---

## 2a. List all running containers

```bash
podman ps
```

To list running **and stopped** containers:

```bash
podman ps -a
```

---

## 2b. Execute `id` inside the Tomcat container

```bash
podman exec tomcat id
```

Alternative using the container ID:

```bash
podman ps
podman exec <CONTAINER_ID> id
```

---

## 2c. Restart the Tomcat container

```bash
podman restart tomcat
```

Verify:

```bash
podman ps
```

---

## 3. Start Redis in the background and map its exposed port to the same host port

Pull Redis:

```bash
podman pull docker.io/library/redis:latest
```

Find the port exposed by the image:

```bash
podman image inspect docker.io/library/redis:latest \
  --format '{{json .Config.ExposedPorts}}'
```

Expected result:

```text
{"6379/tcp":{}}
```

Therefore Redis exposes TCP port `6379`.

Run it:

```bash
podman run -d \
  --name redis \
  -p 6379:6379 \
  docker.io/library/redis:latest
```

Verify:

```bash
podman ps
```

Test Redis:

```bash
podman exec redis redis-cli ping
```

Expected:

```text
PONG
```

If `firewalld` is active:

```bash
sudo firewall-cmd --permanent --add-port=6379/tcp
sudo firewall-cmd --reload
```

### Explanation for the exam

I inspected the image metadata with:

```bash
podman image inspect redis --format '{{json .Config.ExposedPorts}}'
```

It showed `6379/tcp`, so I mapped container port `6379` to host port `6379` using:

```bash
-p 6379:6379
```

---

# Learning Outcome 2 — 12 points / 30 min

## 1. Pull hello-world, tag it, save it to `filec.tar`, and push it to Quay.io

Pull the image:

```bash
podman pull docker.io/library/hello-world:latest
```

Tag it as required:

```bash
podman tag docker.io/library/hello-world:latest firstcontainer:1
```

Check:

```bash
podman images
```

Save it:

```bash
podman save -o filec.tar firstcontainer:1
```

Verify:

```bash
ls -lh filec.tar
```

### Push to Quay.io

Create a **public repository** called `firstcontainer` in Quay.io.

Login:

```bash
podman login quay.io
```

Tag the local image with the full Quay.io name:

```bash
podman tag firstcontainer:1 \
  quay.io/YOUR_QUAY_USERNAME/firstcontainer:1
```

Push:

```bash
podman push quay.io/YOUR_QUAY_USERNAME/firstcontainer:1
```

Verify locally:

```bash
podman images
```

Final image name:

```text
quay.io/YOUR_QUAY_USERNAME/firstcontainer:1
```

---

## 2. UBI9 + Apache HTTP Server + custom default page

Create a working directory:

```bash
mkdir ubi-httpd
cd ubi-httpd
```

Create `Containerfile`:

```bash
nano Containerfile
```

Contents:

```Dockerfile
FROM registry.access.redhat.com/ubi9/ubi:latest

RUN dnf -y install httpd && \
    dnf clean all

RUN echo "Name Surname" > /var/www/html/index.html

EXPOSE 80

CMD ["/usr/sbin/httpd", "-DFOREGROUND"]
```

Build:

```bash
podman build -t my-httpd:1 .
```

Run:

```bash
podman run -d \
  --name my-httpd \
  -p 8080:80 \
  my-httpd:1
```

Test:

```bash
curl http://localhost:8080
```

Expected:

```text
Name Surname
```

Useful checks:

```bash
podman ps
podman images
podman logs my-httpd
```

If external access is required and `firewalld` is active:

```bash
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```

---

## 3. Alpine + nginx + environment variable + foreground command

Create a directory:

```bash
mkdir alpine-nginx
cd alpine-nginx
```

Create `Containerfile`:

```bash
nano Containerfile
```

Contents:

```Dockerfile
FROM alpine:latest

RUN apk add --no-cache nginx

ENV STUDENT="yourusername Name Surname"

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

Build:

```bash
podman build -t alpine-nginx:1 .
```

Run:

```bash
podman run -d \
  --name alpine-nginx \
  -p 8081:80 \
  alpine-nginx:1
```

Check container:

```bash
podman ps
```

Test nginx:

```bash
curl http://localhost:8081
```

Check the environment variable:

```bash
podman exec alpine-nginx printenv STUDENT
```

Expected:

```text
yourusername Name Surname
```

Check the configured command:

```bash
podman inspect alpine-nginx \
  --format '{{.Config.Cmd}}'
```

### Why `daemon off;`?

Containers should keep their main process in the foreground. If nginx daemonizes itself, the container's main process can exit and the container stops. Therefore:

```Dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

keeps nginx running in the foreground.

---

# Learning Outcome 3 — 12 points / 30 min

## 1. Deploy Drupal + PostgreSQL and allow communication between them

Create a dedicated network:

```bash
podman network create drupal-net
```

Check it:

```bash
podman network ls
```

### PostgreSQL container

Run PostgreSQL with exactly the requested environment variables:

```bash
podman run -d \
  --name postgres \
  --network drupal-net \
  -e POSTGRES_PASSWORD=my-secret-pw \
  -e POSTGRES_DB=drupal \
  -e POSTGRES_USER=drupal_user \
  -e PGPASSWORD=drupal_password \
  docker.io/library/postgres:latest
```

Check:

```bash
podman ps
podman logs postgres
```

### Important password detail

The password created for `drupal_user` is:

```text
my-secret-pw
```

because `POSTGRES_PASSWORD` initializes the PostgreSQL user's password.

The requested:

```text
PGPASSWORD=drupal_password
```

is only a client-side PostgreSQL environment variable. It does **not** change the database user's password.

### Drupal container

Run Drupal on the same network:

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
http://VM_IP:8080
```

During Drupal setup use:

```text
Database type: PostgreSQL
Database name: drupal
Database username: drupal_user
Database password: my-secret-pw
Database host: postgres
Database port: 5432
```

The hostname is `postgres` because both containers are attached to `drupal-net`, and Podman DNS resolves container names on that network.

### Demonstrate that Drupal can reach PostgreSQL

Check DNS resolution from Drupal:

```bash
podman exec drupal getent hosts postgres
```

Then make a real PostgreSQL connection from the Drupal container:

```bash
podman exec drupal php -r \
'$pdo=new PDO("pgsql:host=postgres;port=5432;dbname=drupal","drupal_user","my-secret-pw"); echo "DB CONNECTION OK\n";'
```

Expected:

```text
DB CONNECTION OK
```

You can also inspect the network:

```bash
podman network inspect drupal-net
```

---

## 2. Bind mount `/drupal_data` and demonstrate that Drupal can write to it

The official Drupal image stores site-specific data under:

```text
/var/www/html/sites
```

First create the host directory:

```bash
sudo mkdir -p /drupal_data
sudo chown $USER:$USER /drupal_data
```

Because mounting an empty directory over `/var/www/html/sites` would hide the files already provided by the Drupal image, copy them to the host first:

```bash
podman cp drupal:/var/www/html/sites/. /drupal_data/
```

Remove the old Drupal container:

```bash
podman rm -f drupal
```

Recreate it with the bind mount:

```bash
podman run -d \
  --name drupal \
  --network drupal-net \
  -p 8080:80 \
  -v /drupal_data:/var/www/html/sites:Z,U \
  docker.io/library/drupal:latest
```

### Meaning of the mount

```text
/drupal_data:/var/www/html/sites
```

means:

```text
HOST PATH                  CONTAINER PATH
/drupal_data      --->     /var/www/html/sites
```

`Z` handles SELinux labeling.

`U` adjusts ownership for the container's user so the container can write to the bind-mounted directory.

### Demonstrate that Drupal can write there

Create a file **inside the container**:

```bash
podman exec drupal \
  sh -c 'echo "Drupal write test" > /var/www/html/sites/write-test.txt'
```

Show it inside the container:

```bash
podman exec drupal \
  cat /var/www/html/sites/write-test.txt
```

Now show the same file on the host:

```bash
cat /drupal_data/write-test.txt
```

Expected:

```text
Drupal write test
```

This proves:

1. `/drupal_data` is bind-mounted into the Drupal container.
2. Drupal/container processes can write to the mounted location.
3. The written data exists on the host and therefore persists independently of the container.

### Persistence demonstration

Remove and recreate Drupal:

```bash
podman rm -f drupal
```

```bash
podman run -d \
  --name drupal \
  --network drupal-net \
  -p 8080:80 \
  -v /drupal_data:/var/www/html/sites:Z,U \
  docker.io/library/drupal:latest
```

Check the old file:

```bash
podman exec drupal \
  cat /var/www/html/sites/write-test.txt
```

If it still prints:

```text
Drupal write test
```

the persistence requirement is demonstrated.

---

# Fast Command Reference

```bash
# Containers
podman ps
podman ps -a
podman exec CONTAINER COMMAND
podman restart CONTAINER
podman logs CONTAINER
podman rm -f CONTAINER

# Images
podman pull IMAGE
podman images
podman tag SOURCE TARGET
podman save -o FILE.tar IMAGE
podman build -t NAME:TAG .
podman image inspect IMAGE

# Registry
podman login quay.io
podman push quay.io/USER/REPOSITORY:TAG

# Networks
podman network create NAME
podman network ls
podman network inspect NAME

# Ports
podman run -p HOST_PORT:CONTAINER_PORT IMAGE

# Bind mount
podman run -v HOST_PATH:CONTAINER_PATH:Z,U IMAGE
```

# 5-Minute Exam Quick Reference

Use this when you know roughly what you need and only want the command pattern.

## PODMAN — CORE

```bash
podman ps                         # running containers
podman ps -a                      # all containers
podman images                     # images
podman run -d --name NAME IMAGE  # detached container
podman exec NAME COMMAND          # command inside container
podman exec -it NAME sh           # interactive shell
podman logs NAME                  # logs
podman inspect NAME               # full metadata
podman restart NAME               # restart
podman stop NAME                  # stop
podman rm -f NAME                 # force remove container
podman rmi IMAGE                  # remove image
```

## PORTS

```bash
podman run -d -p HOST_PORT:CONTAINER_PORT IMAGE
podman port NAME
curl http://localhost:HOST_PORT
```

Remember: **HOST first, CONTAINER second**. Example: `-p 9080:8080` means host `9080` → container `8080`.

## NETWORKS / DNS

```bash
podman network create mynet
podman run -d --name app --network mynet IMAGE
podman network inspect mynet
podman inspect app --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
```

Containers on the same user-defined network can normally resolve each other by container name.

## ENVIRONMENT VARIABLES

```bash
podman run -d -e KEY=value IMAGE
podman run -d --env-file app.env IMAGE
podman exec NAME printenv KEY
```

## VOLUMES / BIND MOUNTS

```bash
podman volume create data
podman run -d -v data:/container/path IMAGE
podman run -d -v /host/path:/container/path:Z IMAGE
```

`Z` is useful on SELinux systems. `:ro` makes a mount read-only.

## IMAGES / REGISTRY

```bash
podman pull docker.io/library/alpine:latest
podman tag SOURCE_IMAGE newname:1
podman save -o image.tar newname:1
podman load -i image.tar
podman login quay.io
podman tag newname:1 quay.io/USER/REPO:1
podman push quay.io/USER/REPO:1
```

## BUILD

```bash
podman build -t app:1 .
podman build --no-cache -t app:1 .
podman build -f CustomContainerfile -t app:1 .
```

Minimal Containerfile pattern:

```Dockerfile
FROM alpine:latest
RUN apk add --no-cache nginx
ENV STUDENT="value"
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

## PODMAN TROUBLESHOOTING

```bash
podman ps -a
podman logs NAME
podman inspect NAME
podman port NAME
podman network inspect NETWORK
podman exec -it NAME sh
```

Typical mistakes: wrong `HOST:CONTAINER` port order, missing env var, container process exits immediately, wrong network, bind-mount permissions/SELinux, image/tag typo.

## KUBERNETES — CORE

```bash
kubectl get pods -o wide
kubectl get deployments
kubectl get services
kubectl get all
kubectl describe pod POD
kubectl logs POD
kubectl get events --sort-by=.lastTimestamp
```

Create/scale/update:

```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3
kubectl scale deployment web --replicas=5
kubectl set image deployment/web nginx=nginx:1.27
kubectl rollout status deployment/web
kubectl rollout history deployment/web
kubectl rollout undo deployment/web
```

Expose:

```bash
kubectl expose deployment web --port=80 --target-port=80 --type=ClusterIP
kubectl expose deployment web --port=80 --target-port=80 --type=NodePort
minikube service web --url
```

## KUBERNETES STATUS → FIRST THING TO CHECK

- `Pending` → `kubectl describe pod` + events; usually scheduling/resources/PVC/node.
- `ImagePullBackOff` / `ErrImagePull` → image name/tag/registry credentials.
- `CrashLoopBackOff` → `kubectl logs POD` and `kubectl logs POD --previous`.
- `OOMKilled` → memory limit/application memory.
- Running but Service gives nothing → compare Service selector with Pod labels and check endpoints.
- Not Ready → readiness probe and application port/path.

## SEARCH THE WHOLE FOLDER

If the notes are downloaded locally:

```bash
grep -Rni "ImagePullBackOff" .
grep -Rni "restart" .
grep -Rni "bind mount" .
```

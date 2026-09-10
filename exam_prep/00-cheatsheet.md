# CONTAINERS

podman ps
podman ps -a
podman logs NAME
podman inspect NAME
podman exec -it NAME sh
podman port NAME
podman stats

# IMAGES
podman images
podman pull IMAGE
podman build -t NAME:TAG .
podman tag OLD NEW
podman save -o image.tar IMAGE
podman load -i image.tar

# KUBERNETES
kubectl get pods -o wide
kubectl describe pod NAME
kubectl logs NAME
kubects logs NAME --previous
kubectl get events --sort-by=.lastTimestamp
kubectl get svc
kubectl get endpoints
kubectl get deployments
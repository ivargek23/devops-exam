# 1: Containers

- Container = a running instance of a container image
- Container image = a single file that contains the application and its dependencies (a .tar archive), metadata which describes the image
- Image registry = a web service which hosts container images and allows users to download them

VMs
- started from VM image
- image is larger (GBs)
- has boot loader, kernel, systemd
- uses a hypervisor to run the OS
- portability based on hypervisor supporting vm image
- uses more resources (CPU, memory, disk) than containers
- has low-level access to hardware

Containers
- started from container image
- image is smaller (MBs)
- no boot loader, no kernel
- may have systemd to start several processes
- no hypervisor, uses container engine
- OCI container images run on any OCI-compliant container engine
- uses fewer resources (CPU, memory, disk) than VMs
- no low-level access to hardware

# 2: Podman basics

`podman ps` - list running containers
`podman ps -a` - list all containers (running and stopped)
`podman rm <container_id>` - remove a container
`podman images` - list all images
`podman rmi <image_id>` - remove an image
`podman run -d --name ivonin quay.io/rdacosta/my_https:latest` - run a container in detached mode with the name "ivonin" from the specified image
`podman stop ivonin` - stop the container named "ivonin"
`podman run -d --name ivonin -p 8081:8080 quay.io/rdacosta/my_https:latest` - run a container in detached mode with the name "ivonin" and map port 8081 on the host to port 8080 in the container
`podman run -d --rm --name ivonin -p 8081:8080 quay.io/rdacosta/my_https:latest` - run a container in detached mode with the name "ivonin", map port 8081 on the host to port 8080 in the container, and automatically remove the container when it stops
`curl http://localhost:8081` - test the container by sending a request to the mapped port on the host
`podman logs ivonin` - view the logs of the container named "ivonin"

## Creating containers with Podman

`podman pull registry.ocp4.example.com:8443/ubi8/ubi-minal:8.5` - download the container image
`podman run --rm registry.ocp4.example.com:8443/ubi8/ubi-minal:8.5 echo 'Holla` - run a container from the downloaded image and remove it after it stops, executing the command `echo 'Holla'` inside the container
`podman run --rm -e GREET=Holla -e NAME='Ivona' registry.ocp4.example.com:8443/ubi8/ubi-minal:8.5 printenv GREET NAME` - run a container from the downloaded image, set environment variables `GREET` and `NAME`, and execute the command `printenv GREET NAME` inside the container to print the values of the environment variables

## Container networking basics
- Podman bridge = a virtual network that allows containers to communicate with each other and the host system, all containers are connected to the same bridge network by default; containers cannot communicate directly, we have to set up port forwarding to allow communication between the host and the container; DNS name resolution is not available by default

`podman network ls` - list all networks on the container host
`podman network inspect podman` - view details of the default Podman bridge network
`podman network create mynetwork` - create a new Podman network named "mynetwork"
`podman run -d --name ivonin --net mynetwork -p 8082:8080 quay.io/rdacosta/my_https:latest` - run a container in detached mode with the name "ivonin"
`podman exec web1 curl http://ivonin:8080` - execute a command inside the container named "web1" to send a request to the container named "ivonin" on port 8080, using the container's name as the hostname => could not resolve hostname ivonin because DNS name resolution is not available by default, we have to set up a user-defined network to enable DNS name resolution between containers
`podman inspect ivonin | jq .[].NetworkSettings.Networks` - view the network settings of the container named "ivonin" and its IP address on the default Podman bridge network

## Accessing containerized network services
`podman port -a` - list all port mappings for all containers
`podman port ivonin` - list the port mappings for the container named "ivonin"
`podman inspect ivonin` - view detailed information about the container named "ivonin", including its network settings and port mappings
`podman inspect ivonin -f '{{.NetworkSettings.Ports}}'` - view the port mappings for the container named "ivonin" in a more readable format

`grep -i listen podman-info-times/app/main.go` - search for the string "listen" in the file `main.go` to find the port on which the application is listening

`podman network create cities` - create a new Podman network named "cities"
`podman network inspect cities` - view details of the Podman network named "cities"

## Accessing containers

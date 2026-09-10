# DevOps Exam — START HERE

This folder is organized for **fast Ctrl+F use**, not for reading front-to-back during the exam.

## Which file do I open?

| Task contains... | Open |
|---|---|
| `run`, `exec`, `restart`, `logs`, `port`, `CPU`, `memory`, `pause`, `network` | `01_LO1_CONTAINERS_PODMAN.md` |
| `image`, `Containerfile`, `Dockerfile`, `build`, `tag`, `push`, `Quay`, `save`, `load`, `ENTRYPOINT`, `CMD` | `02_LO2_IMAGES_CONTAINERFILES_REGISTRIES.md` |
| `two containers`, `database`, `Postgres`, `pod`, `Compose`, `volume`, `secret`, `reverse proxy` | `03_LO3_MULTI_CONTAINER_NETWORKING_STORAGE_SECURITY.md` |
| `kubectl`, `Deployment`, `Service`, `StatefulSet`, `ConfigMap`, `Secret`, `PVC`, `probe`, `rollout` | `04_LO4_KUBERNETES_MINIKUBE.md` |
| something is broken / `Pending` / `CrashLoopBackOff` / `ImagePullBackOff` / `OOMKilled` | `05_LO5_TROUBLESHOOTING.md` |
| compare / recommend / explain Kubernetes, OpenShift, k3s, Docker/Podman, Compose | `06_LO6_THEORY_ORCHESTRATION.md` |
| task looks like the old exam | `07_PREVIOUS_EXAM_SOLVED.md` |
| you only need a command quickly | `00_QUICK_REFERENCE.md` |
| you do not know which wording to search | `08_KEYWORD_INDEX.md` |
| you need YAML / Containerfile templates | `09_READY_TO_COPY_TEMPLATES.md` |

## Exam workflow

1. Read the task and identify the noun: container, image, network, volume, Deployment, Service, etc.
2. Open the matching LO file and Ctrl+F that noun.
3. Run the minimum commands required.
4. **Verify the result** with `podman ps/inspect/logs` or `kubectl get/describe/logs`.
5. Take the required screenshot **inside the exam VM** using `Applications → Utilities → Screenshot`.

## Absolute basics to remember

```text
Podman ports: HOST:CONTAINER
Service selector must match Pod labels.
Deployment selector must match Pod template labels.
Pending            = scheduling/resources/storage/node problem
ImagePullBackOff   = image/tag/registry/auth problem
CrashLoopBackOff   = application starts and crashes repeatedly
OOMKilled          = memory problem
No Service traffic = selector/endpoints/readiness/targetPort
```

## Search all notes from a terminal

```bash
grep -Rni "SEARCH_TERM" .
```

Example:

```bash
grep -Rni "NodePort" .
grep -Rni "restart policy" .
grep -Rni "PostgreSQL" .
```

## Security note

The uploaded source folder contained account credentials. They are **not copied into this organized pack**, because this pack is suitable for a GitHub repository and passwords should not be committed there.

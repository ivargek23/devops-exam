# LO6 — Orchestration Theory

# How to use this file during the exam

Start with Ctrl+F and search the noun from the task, for example `restart`, `port`, `volume`, `Containerfile`, `rollout`, `ImagePullBackOff`, or `OpenShift`.

For practical tasks, always do three things: **perform the task → verify it → take a screenshot showing the command and proof**.

---

## QUICK KNOWLEDGE / COMMAND PATTERNS

## 7. LO6 — Theory answer bank

Use this structure for every theory answer: recommendation, reason, trade-off, concrete example.

### 1. Kubernetes networking vs Docker/Podman networking

Docker/Podman on a single host commonly use bridge networks, NAT, published ports, and optional user-defined DNS. Kubernetes networking is cluster-level: every pod gets an IP, pods can talk across nodes, Services provide stable virtual IP/DNS/load balancing, and CNI plugins implement the network. Kubernetes is more complex but designed for multi-node scheduling and service discovery. Docker/Podman networking is simpler and enough for one host.

### 2. Kubernetes storage vs Podman storage plugins

Kubernetes is more flexible for stateful workloads because it abstracts storage through PVs, PVCs, StorageClasses, CSI drivers, dynamic provisioning, reclaim policies, and per-workload claims. Podman volumes are good for single-host persistence, but Kubernetes better handles multi-node scheduling, cloud disks, and stateful controllers.

### 3. Operators vs plain manifests

Plain manifests declare desired resources, but operators add domain-specific automation through CRDs and controllers. For a database, an operator can handle backups, failover, upgrades, replication, and recovery. The trade-off is higher complexity and dependency on the operator's quality. Use manifests for simple stateless apps; use operators for complex stateful systems.

### 4. Plain Kubernetes vs OpenShift S2I/developer console

Plain Kubernetes optimizes for portability and low-level control through kubectl and YAML. OpenShift S2I and the developer console optimize for developer productivity: build source into images, deploy with guided UI, integrate routes, image streams, templates, and security defaults. Kubernetes is leaner; OpenShift is more batteries-included.

### 5. Migrating OpenShift to Kubernetes

Migration is possible but not automatic. Standard Kubernetes resources move easily; OpenShift-specific objects like Routes, BuildConfigs, ImageStreams, SCCs, and S2I workflows need replacements. Benefit: less vendor lock-in and more platform choice. Cost: rebuilding CI/CD, security policy, routing, registry integration, and developer workflows.

### 6. CI/CD agents in Kubernetes vs VMs

Kubernetes agents are good for autoscaling, ephemeral clean environments, and resource isolation by namespace/limits. Dedicated VMs are simpler and can be more predictable for heavy builds or privileged tooling. Kubernetes has noisy-neighbor risks if limits are bad; VMs have lower orchestration complexity but waste idle capacity.

### 7. Kubernetes and cloud dynamic capacity

Kubernetes integrates with cloud through cloud-controller managers, CSI storage, load balancers, node autoscaling, cluster autoscaling, and managed services. Workloads request resources; autoscalers can add pods or nodes. This makes Kubernetes strong for elastic demand.

### 8. OpenShift security vs vanilla Kubernetes

OpenShift is more secure by default: stricter security constraints, integrated RBAC, image registry/scanning options, routes/TLS integration, opinionated defaults, and Red Hat support. Vanilla Kubernetes can be hardened, but more work is left to the team. Regulated enterprises often choose OpenShift for support, compliance posture, and integrated platform governance.

### 9. Learning curve OpenShift vs Kubernetes

OpenShift adds concepts, but also gives safer defaults and UI workflows. For a team new to containers, it can help if they accept the platform's opinionated path. It can hinder if they need to understand raw Kubernetes first or want maximum portability. Net: OpenShift reduces operational assembly work but increases platform-specific knowledge.

### 10. Full Kubernetes vs k3s footprint

Full Kubernetes can be overkill for small on-prem deployments because it needs more resources and operational care. k3s is lightweight, easier to run on edge/small servers, and removes/streamlines components. Use full Kubernetes for larger clusters, strict HA, broad integrations, and enterprise operations; use k3s for small teams, labs, edge, or constrained hardware.

### 11. Compose single host vs Kubernetes for small team

A small team should start with Docker/Podman Compose if the app fits one host and needs fast delivery. Compose has lower cost and mental overhead. Move to Kubernetes when they need self-healing across nodes, autoscaling, rolling updates, service discovery, and stronger platform standards. Do not buy a forklift to move a sandwich.

### 12. Vendor lock-in: Kubernetes vs OpenShift

Kubernetes is open source/CNCF with many providers, so it offers broad portability. OpenShift is Kubernetes-based but adds Red Hat-specific APIs, workflows, support model, and tooling. OpenShift improves enterprise experience but increases dependency on Red Hat conventions. Long-term flexibility depends on avoiding platform-specific resources unless their value justifies it.

### 13. Recommend OpenShift over Kubernetes

Choose OpenShift when a company wants an integrated, vendor-supported platform with developer console, S2I/builds, routes, registry integration, monitoring/logging options, stricter defaults, and enterprise support. It costs more and is heavier, but reduces platform assembly work and risk for regulated or large organizations.

### 14. Kubernetes storage vs VM storage

VM storage is usually attached to a machine and managed by infrastructure admins. Kubernetes storage is requested declaratively by applications through PVCs and dynamically provisioned by StorageClasses/CSI. Kubernetes makes storage part of the app deployment model, but stateful workloads still require careful backup, performance, and failover design.

### 15. Startup with two engineers

Recommend Compose or a managed PaaS/container service first, not self-managed Kubernetes. The priority is shipping cheaply with low operational overhead. Use simple containers, managed database, and CI/CD. Move to Kubernetes later when scaling/resilience needs justify the complexity.

### 16. Docker vs Podman

Docker traditionally uses a daemon architecture; clients talk to the Docker daemon. Podman is daemonless and can run rootless more naturally. A security-conscious team may prefer Podman because rootless containers reduce the impact of container escape/misconfiguration and avoid a long-running root daemon.

### 17. Self-managed Kubernetes vs managed service

Self-managed Kubernetes means you own upgrades, etcd backups, node scaling, control-plane health, networking, storage integration, and security patches. Managed Kubernetes offloads much of that to the provider but costs money and can introduce cloud-specific behavior. For most production teams, managed is saner unless they have strong platform engineering capability.

### 18. Media-streaming with spiky global traffic

Recommend managed Kubernetes or a cloud-native container platform plus CDN. Kubernetes handles horizontal scaling, rolling updates, resilience, and service orchestration; CDN handles global media delivery. Self-managed single-host/Swarm is too limited for unpredictable global traffic.

### 19. Outgrowing Docker Swarm/Compose

Migration to Kubernetes is justified when the company needs stronger ecosystem, autoscaling, multi-node resilience, standardized deployments, service discovery, secrets/config management, and broad tooling. Cost: rewrite manifests, retrain team, change CI/CD, observe production risk. If growth is real, long-term benefits usually beat staying on a platform that no longer fits.

### 20. Kubernetes for microservices

Yes, if the microservices need orchestration, resilience, service discovery, rolling deployments, scaling, and observability ecosystem. Kubernetes fits many independent services better than hand-managed hosts. It is not worth it for a tiny app or a team without operational capacity.

### 21. Kubernetes for enterprise apps

Yes, often. Kubernetes has scalability, huge community support, mature tooling, portability, and feature richness. But enterprise-grade use requires hardening, governance, monitoring, backup, network policy, secrets management, and platform skills. Kubernetes is the engine, not the whole car.

### 22. OpenShift for microservices

Yes when the organization wants Kubernetes orchestration plus an integrated enterprise developer/operations platform. OpenShift adds routes, builds/S2I, stricter defaults, console, RBAC patterns, and support. It is heavier than plain Kubernetes, but useful when teams need a governed microservices platform.

### 23. OpenShift for enterprise-grade apps

Yes for enterprises that value support, security defaults, integrated tooling, scalability, compliance posture, and consistent operations across hybrid environments. It is less minimal and more vendor-aligned than vanilla Kubernetes, but that trade-off is often exactly what enterprise IT wants.

---

## FULL SOLVED PRACTICE QUESTIONS

# LO6 – Evaluate selected container orchestration systems – theoretical

## 1. Kubernetes networking versus Docker networking.

Kubernetes gives every Pod its own IP and expects flat Pod-to-Pod connectivity across nodes. Services provide stable virtual access to changing Pods. Docker/Podman networking is usually host-local, bridge-based, and simpler. Kubernetes networking is better for multi-node orchestration; Docker networking is easier for single-host development.

## 2. Kubernetes storage abstraction versus Podman storage plugins.

Kubernetes is more flexible for stateful workloads because it has CSI, StorageClasses, PersistentVolumes, PersistentVolumeClaims, and dynamic provisioning. Podman volumes are useful locally, but Kubernetes integrates storage with scheduling, cloud providers, and automatic provisioning.

## 3. Operator pattern versus plain manifests.

Operators combine CRDs and controllers to manage complex applications automatically. They handle backups, upgrades, failover, scaling, and recovery logic. Plain manifests are simpler but require humans or scripts to perform operational tasks.

## 4. Plain Kubernetes versus OpenShift S2I/developer console.

Plain Kubernetes optimizes for portability and direct control through manifests and `kubectl`. OpenShift optimizes for enterprise developer productivity with Source-to-Image, templates, integrated registry, web console, routes, and stricter defaults.

## 5. Migrating from OpenShift to Kubernetes.

Migration is possible, but OpenShift-specific objects such as Routes, BuildConfigs, ImageStreams, SecurityContextConstraints, and S2I workflows must be replaced. Standard Kubernetes manifests move easily; OpenShift-integrated pipelines and security policies require rework.

## 6. CI/CD agents inside Kubernetes versus dedicated VMs.

Kubernetes agents scale dynamically and isolate builds in Pods. They are efficient for bursty workloads. Dedicated VMs are simpler and more predictable but waste capacity and scale manually. Kubernetes adds noisy-neighbor risk unless resource limits and node isolation are configured.

## 7. Kubernetes cloud integration and dynamic capacity.

Kubernetes integrates with cloud load balancers, storage, autoscaling, IAM, node provisioning, and container registries. Cluster autoscalers and managed Kubernetes services can add/remove nodes based on workload demand.

## 8. OpenShift security versus vanilla Kubernetes.

OpenShift ships stricter defaults: restricted security contexts, integrated OAuth/RBAC, image policies, routes, registry, monitoring, and vendor-supported hardening. Regulated enterprises often choose it because it reduces integration work and provides supported compliance-oriented defaults.

## 9. OpenShift versus Kubernetes learning curve.

OpenShift adds concepts, but it also gives guardrails and built-in tools. For a team new to containers, it can help by reducing platform assembly work. It can hinder if the team first needs to understand raw Kubernetes fundamentals.

## 10. Full Kubernetes versus k3s footprint.

Full Kubernetes is heavier and better for large, highly available, enterprise clusters. k3s is lightweight and good for small on-premise, edge, or lab deployments. Full Kubernetes is overkill when the workload is small, the team is tiny, and high availability is not required.

## 11. Docker Compose on one host versus Kubernetes for a small team.

A small team should start with Compose or Podman Compose if one host is enough. It is cheaper and simpler. Move to Kubernetes when the app needs self-healing, scaling, rolling updates, service discovery, multi-node resilience, and stronger operational control.

## 12. Vendor lock-in: Kubernetes versus OpenShift.

Kubernetes is open source and widely supported across clouds and vendors. OpenShift is Kubernetes-based but adds Red Hat-specific APIs and tooling. OpenShift improves enterprise support but increases migration work if moving away later.

## 13. Recommendation: OpenShift over vanilla Kubernetes.

Choose OpenShift when the company wants an integrated, vendor-supported platform with developer console, builds, registry, routing, monitoring, RBAC, security policies, and enterprise support. It costs more but saves platform engineering effort.

## 14. Kubernetes storage versus regular VM storage.

VM storage is usually attached manually to a server. Kubernetes storage is declared through PVCs and provisioned dynamically through StorageClasses. Kubernetes is better for automated scheduling and rescheduling, but stateful apps still need careful backup and performance planning.

## 15. Startup with two engineers shipping quickly and cheaply.

Use Docker Compose or Podman Compose on a managed VM. It has the lowest operational overhead and cost. Kubernetes is too much unless they already need multiple nodes, autoscaling, or strict high availability.

## 16. Docker versus Podman architecture and rootless security.

Docker traditionally uses a central daemon. Podman is daemonless and can run rootless more naturally. A security-conscious team may prefer Podman because containers do not require a root-owned daemon as the control point.

## 17. Self-managed Kubernetes versus managed Kubernetes day-2 overhead.

Self-managed Kubernetes requires cluster upgrades, etcd backups, node patching, networking, storage, monitoring, and incident response. Managed Kubernetes offloads much of the control-plane maintenance, but the team still owns workloads, security, cost, and observability.

## 18. Media-streaming company with spiky global traffic.

Use managed Kubernetes plus CDN and cloud autoscaling. Kubernetes handles container orchestration and horizontal scaling, while the CDN absorbs global traffic close to users. Self-hosted single-node solutions would collapse under unpredictable spikes.

## 19. Migrating from Docker Swarm or Compose to Kubernetes.

Migrate when scaling, resilience, rolling updates, service discovery, and ecosystem integrations justify the complexity. Do not migrate just because Kubernetes is popular. The migration cost is real: manifests, CI/CD, monitoring, secrets, storage, and team training.

## 20. Kubernetes for microservices.

Yes, if the system has multiple services that need orchestration, service discovery, scaling, rolling updates, and resilience. Kubernetes is strong for microservices because it standardizes deployment and recovery. For two tiny services, it is probably overengineering.

## 21. Kubernetes for enterprise-grade applications.

Yes, when the enterprise can operate it properly. Kubernetes has scalability, broad community support, portability, and a rich ecosystem. The downside is operational complexity; without platform discipline, it becomes YAML-powered chaos.

## 22. OpenShift for microservices.

Yes, especially in companies that want Kubernetes plus integrated developer and operations tooling. OpenShift supports orchestration, resilience, service routing, builds, CI/CD integrations, and stricter security defaults. It is heavier than vanilla Kubernetes but more complete.

## 23. OpenShift for enterprise-grade applications.

Yes. OpenShift is a strong fit for enterprise workloads because it combines Kubernetes scalability with vendor support, security hardening, monitoring, registry, routing, developer tools, and lifecycle management. The trade-off is cost and stronger Red Hat platform dependency.

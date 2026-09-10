# LO6 — Exam Answer Bank

> Short answer templates for the official LO6 practice questions. Expand each with scenario-specific detail if needed.

# 1. Kubernetes networking vs Docker networking

Kubernetes uses a cluster-wide networking model where pods receive network identities and Services provide stable discovery and load balancing. Docker/Podman networking is simpler and usually host-oriented, using bridge networks, published ports and container-name DNS. Docker/Podman is easier for a single host, while Kubernetes is better suited to multi-node applications but has significantly more networking complexity.

---

# 2. Kubernetes storage vs Podman storage

Kubernetes is more flexible for stateful workloads because CSI, StorageClasses and dynamic provisioning separate the application's storage request from the underlying storage implementation. Podman volumes and bind mounts are much simpler and work well on one host, but they do not provide the same cluster-level scheduling and provisioning abstraction. The trade-off is that Kubernetes storage is considerably more complex to operate.

---

# 3. Operators vs plain manifests

I would use an Operator when the application has complex day-2 operations such as backup, failover, upgrades or cluster membership. An Operator encodes this application-specific knowledge in CRDs and controllers and can continuously reconcile the service. Plain manifests are easier and more transparent for simple workloads, so an Operator is only justified when its automation saves more effort than it adds.

---

# 4. Kubernetes vs OpenShift/S2I developer experience

Plain Kubernetes optimizes for flexibility and portability but requires developers/platform teams to assemble many parts themselves. OpenShift adds an integrated console, developer workflows and features such as S2I that can simplify going from source code to a deployed application. OpenShift improves the integrated developer experience but adds platform-specific concepts and cost.

---

# 5. Migrating OpenShift → Kubernetes

The migration is feasible because OpenShift is Kubernetes-based and standard Deployments, Services, ConfigMaps and many other resources are portable. The difficulty depends on how much the application uses OpenShift-specific features such as Routes, ImageStreams, BuildConfigs/S2I or SecurityContextConstraints. A workload based mainly on standard Kubernetes APIs is much easier to migrate.

---

# 6. CI/CD agents in Kubernetes vs VMs

Kubernetes is attractive for CI/CD agents because agents can be ephemeral, isolated and automatically scaled according to demand. This reduces idle capacity, but builds can create noisy-neighbor problems and privileged build requirements can complicate security. Dedicated VMs are more predictable and can suit unusual toolchains, but they usually scale more slowly and require more manual capacity management.

---

# 7. Kubernetes + cloud dynamic capacity

Kubernetes integrates well with cloud environments through load balancers, CSI storage, managed clusters and node autoscaling. When workload demand increases, pod autoscaling can increase replicas and infrastructure autoscaling can add nodes when capacity is insufficient. This makes Kubernetes suitable for elastic environments, although the actual scaling capability depends on the cloud and cluster configuration.

---

# 8. OpenShift security vs vanilla Kubernetes

I would generally rate OpenShift as stronger out of the box for enterprises that want a standardized hardened platform. It provides integrated security controls, stricter defaults, authentication/RBAC integration and vendor-supported platform lifecycle. Vanilla Kubernetes can be hardened to a similar level, but more integration and policy decisions are left to the organization. This is one reason regulated enterprises may prefer OpenShift despite its cost.

---

# 9. OpenShift vs Kubernetes learning curve

OpenShift adds additional concepts to Kubernetes, so the total platform is not inherently simpler. However, its console and integrated workflows can make common developer and operational tasks easier. For a new team, OpenShift can reduce the need to select and integrate many separate tools, but the team still needs Kubernetes knowledge plus OpenShift-specific knowledge.

---

# 10. Full Kubernetes vs k3s

For a small on-premises deployment, I would choose k3s when hardware and operational resources are limited. It provides Kubernetes APIs with a much smaller footprint and simpler installation. Full Kubernetes is justified when the organization requires larger-scale enterprise integration or more control over components; otherwise it can be unnecessary overhead.

---

# 11. Compose vs Kubernetes for a small team

I would start with Compose for a small single-host application unless there is a clear need for multi-node resilience or large-scale orchestration. Compose is easier to operate, cheaper and faster to understand. Kubernetes provides stronger scaling, self-healing and deployment automation, but those benefits do not justify its operational complexity for every small application.

---

# 12. Vendor lock-in

Kubernetes reduces platform lock-in because it is open source and supported by many clouds and vendors, although workloads can still depend on provider-specific storage, load balancers and managed features. OpenShift remains Kubernetes-based but adds Red Hat-specific APIs and workflows, which increases platform dependency. The trade-off is that OpenShift provides a more integrated and supported environment.

---

# 13. Defend OpenShift over vanilla Kubernetes

I would recommend OpenShift when the company explicitly wants vendor support, integrated developer and operations tooling, and standardized security. It reduces the number of separate platform components the company must select and integrate itself. The cost and OpenShift-specific dependency are the main disadvantages, so vanilla Kubernetes may be better for a highly experienced team that wants maximum flexibility.

---

# 14. Kubernetes storage vs VM storage

VM storage is commonly attached directly to a machine and managed as infrastructure for that VM. Kubernetes introduces PVs, PVCs, StorageClasses and dynamic provisioning so applications request storage abstractly and the platform can provision it. This improves automation and portability for orchestrated workloads but adds another abstraction layer to troubleshoot and operate.

---

# 15. Two-engineer startup

I would not start with self-managed Kubernetes. A two-engineer startup should prioritize shipping the product and minimizing operational overhead, so Compose or a managed container service is more appropriate. Kubernetes becomes worthwhile later if the company needs multi-node high availability, autoscaling, many services or a mature platform engineering capability.

---

# 16. Docker vs Podman

Docker traditionally uses a central daemon, while Podman is daemonless and has strong rootless operation. Podman's architecture can reduce reliance on a privileged long-running daemon and fits security-conscious Linux environments well. Docker has an extremely mature ecosystem and also supports rootless workflows, so the decision should consider tooling and operational requirements rather than claiming one is universally secure.

---

# 17. Self-managed vs managed Kubernetes

I would choose managed Kubernetes for most teams that do not need direct control of the control plane. The provider handles much of the upgrade and control-plane lifecycle, reducing day-2 operational work. Self-managed Kubernetes gives more control and can fit special on-premises requirements, but the organization must own upgrades, backups, reliability and much more cluster maintenance.

---

# 18. Media streaming with spiky global traffic

I would use a managed Kubernetes platform with horizontal autoscaling and cloud capacity autoscaling, combined with global load balancing and a CDN. Kubernetes can scale stateless services and recover failed replicas, while managed infrastructure reduces operational burden. Kubernetes alone is not sufficient for global delivery, so CDN, caching and global traffic management remain important parts of the architecture.

---

# 19. Swarm/Compose → Kubernetes

I would migrate when the current platform has become a bottleneck for resilience, scaling, scheduling or operational automation. Kubernetes provides a much larger ecosystem and stronger multi-node orchestration, but migration requires training, manifest changes and ongoing cluster operations. If the current system still meets requirements reliably, migrating only for technology fashion would not be justified.

---

# 20. Kubernetes for microservices

Yes, when the microservices estate is large enough to justify orchestration. Kubernetes provides service discovery, independent scaling, rolling updates, self-healing and a strong ecosystem, which align well with microservice requirements. For a very small system, the platform overhead can outweigh those advantages.

---

# 21. Kubernetes for enterprise applications

Yes, provided the organization has the operational expertise to run or consume it as a managed service. Kubernetes offers strong scalability, a large community, broad vendor support and a rich ecosystem of networking, storage, observability and security tools. Its main weakness is complexity, so governance and platform engineering are important for enterprise use.

---

# 22. OpenShift for microservices

I would choose OpenShift for microservices when the organization wants Kubernetes orchestration together with integrated enterprise tooling and support. It provides resilience, scaling and the Kubernetes ecosystem while adding developer workflows, security controls and Red Hat support. The main trade-offs are licensing cost and additional OpenShift-specific concepts.

---

# 23. OpenShift for enterprise applications

OpenShift is a strong enterprise choice when the company values vendor support, integrated security, lifecycle management and a standardized Kubernetes platform. It can reduce the effort of assembling and supporting many separate platform components. It is less attractive for small organizations where cost and platform complexity outweigh those enterprise benefits.

---

# FAST COMPARISON TABLE

| Scenario | Best starting choice | Why |
|---|---|---|
| One host / tiny team | Compose | lowest complexity |
| Small edge/on-prem cluster | k3s | Kubernetes with lower footprint |
| General multi-node orchestration | Kubernetes | portable ecosystem |
| Enterprise wanting integrated supported platform | OpenShift | tooling + security + support |
| Team wants less cluster maintenance | Managed Kubernetes | lower day-2 overhead |
| Complex stateful app lifecycle | Kubernetes Operator | automates domain operations |

---

# ANSWER FORMULA

For almost every LO6 question:

```text
I would choose/recommend X because:
1. reason tied to scenario
2. second reason
3. third reason

Main disadvantage:
- cost / complexity / lock-in / resource overhead

Alternative Y is better when:
- condition

Therefore for this scenario, X is the better fit.
```

Avoid feature dumping. The outcome asks you to evaluate and defend a position.

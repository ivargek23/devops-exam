# LO6 — Evaluating Container Orchestration Systems

> LO6 is theoretical. Coverage follows the official LO6 practice-question set, including Kubernetes, OpenShift, Docker/Podman, Compose, Docker Swarm, k3s, managed Kubernetes and enterprise trade-offs.

# 1. What is orchestration?

Container orchestration manages applications made of many containers across one or more machines.

Typical responsibilities:
- scheduling
- scaling
- networking
- service discovery
- self-healing
- rolling updates
- configuration and secrets
- storage integration
- workload placement

Kubernetes is the dominant general-purpose orchestration platform.

---

# 2. Kubernetes vs Docker/Podman networking

## Docker/Podman

Typical single-host model:
- bridge networks
- port publishing
- container-name DNS on custom networks
- simpler topology

Good for:
- local development
- small single-host deployments
- simple application stacks

## Kubernetes

Cluster-wide model:
- every pod gets a cluster-network identity
- Services provide stable virtual endpoints
- DNS provides service discovery
- CNI plugins implement pod networking
- NetworkPolicies can restrict traffic

Kubernetes is more complex but designed for multi-node orchestration.

---

# 3. Kubernetes storage vs Podman storage

## Podman

Good local/single-host primitives:
- bind mounts
- named volumes
- storage drivers/plugins

Simple and effective on one host.

## Kubernetes

Storage abstractions:
- PersistentVolumes
- PersistentVolumeClaims
- StorageClasses
- CSI drivers
- dynamic provisioning

Kubernetes is more flexible for distributed/stateful workloads because applications request storage abstractly instead of depending on one host path.

Trade-off: much more complexity.

---

# 4. Operators

An Operator combines:
- Custom Resource Definitions (CRDs)
- controllers
- application-specific operational logic

Instead of only describing objects, an Operator can automate domain knowledge such as:
- backups
- upgrades
- failover
- cluster membership
- scaling
- recovery

Advantages:
- powerful automation
- repeatable day-2 operations

Disadvantages:
- more code/components
- operational complexity
- dependency on Operator quality

For simple applications, plain manifests may be easier.

---

# 5. Kubernetes vs OpenShift developer experience

## Plain Kubernetes

Usually:
- `kubectl`
- YAML manifests
- separate choices for registry, ingress, CI/CD, monitoring, developer workflows

Optimizes for:
- flexibility
- portability
- broad ecosystem
- infrastructure-neutral design

## OpenShift

Adds an integrated platform around Kubernetes:
- web console
- Routes
- integrated security defaults
- developer workflows
- operators
- enterprise support
- platform tooling

Source-to-Image (S2I) can build runnable images from source without requiring developers to manually create every image build workflow.

Optimizes for:
- integrated enterprise developer/operations experience

Trade-off:
- more platform abstraction
- Red Hat-specific components and conventions
- licensing/support cost

---

# 6. Migrating OpenShift → Kubernetes

Possible because OpenShift is Kubernetes-based, but not always frictionless.

Portable:
- standard Deployments
- Services
- ConfigMaps
- Secrets
- many standard workloads

Potential migration work:
- OpenShift Routes → Ingress/Gateway
- OpenShift-specific APIs
- SecurityContextConstraints
- ImageStreams
- BuildConfigs/S2I
- Operators/platform integrations

Evaluation:

```text
more use of standard Kubernetes APIs → easier migration
more OpenShift-specific features → more migration effort
```

---

# 7. CI/CD agents inside Kubernetes vs VMs

## Kubernetes agents

Advantages:
- elastic scaling
- ephemeral clean environments
- good isolation between jobs
- resources can be requested/limited

Disadvantages:
- cluster complexity
- noisy-neighbor risk
- privileged builds can be difficult/risky
- dependency on cluster availability

## Dedicated VMs

Advantages:
- predictable isolation
- easier support for unusual/privileged tools
- simpler troubleshooting in some environments

Disadvantages:
- slower scaling
- idle capacity
- more manual lifecycle management

---

# 8. Kubernetes and cloud capacity

Kubernetes integrates with cloud providers through:
- cloud load balancers
- CSI storage
- autoscaling
- node groups
- Cluster Autoscaler / provider-specific scaling
- managed Kubernetes services

The cluster can add/remove compute capacity as workload demand changes when infrastructure supports it.

---

# 9. OpenShift security vs vanilla Kubernetes

OpenShift typically provides stricter integrated defaults and enterprise security controls.

Examples:
- stronger default restrictions on container privileges
- SecurityContextConstraints
- integrated authentication/RBAC
- supported platform lifecycle
- image/build/security tooling depending on edition/configuration

Vanilla Kubernetes can also be hardened strongly, but more design/integration decisions are left to the operator.

Why regulated enterprises may choose OpenShift:
- vendor support
- standardized configuration
- integrated security controls
- certified ecosystem
- predictable lifecycle

Trade-off:
- cost
- platform complexity
- less freedom in some defaults

---

# 10. Learning curve

## Kubernetes

Requires understanding:
- pods
- Deployments
- Services
- storage
- networking
- RBAC
- observability
- cluster operations

## OpenShift

Adds more concepts but also provides integrated tooling and UI.

For beginners:
- abstraction/UI can make common tasks easier
- platform-specific concepts can increase total learning surface

So it helps operationally while adding platform-specific knowledge.

---

# 11. Full Kubernetes vs k3s

k3s is a lightweight Kubernetes distribution designed for smaller environments.

k3s:
- lower resource footprint
- simpler installation
- useful for edge, labs, small on-prem clusters

Full/standard Kubernetes distributions:
- broader enterprise integration
- more control over components
- suited to larger production environments

Full Kubernetes can be overkill when:
- only a few services
- small team
- limited hardware
- no high-availability/scaling requirement

---

# 12. Compose vs Kubernetes

## Docker/Podman Compose

Good for:
- one host
- small team
- development
- simple production stacks
- low operational overhead

## Kubernetes

Good for:
- many services
- multiple nodes
- scaling
- self-healing
- rolling updates
- strong platform automation

A small team should not adopt Kubernetes only because it is popular.

Use it when requirements justify its operational cost.

---

# 13. Vendor lock-in

## Kubernetes

Open source, CNCF ecosystem and supported by many vendors/clouds.

This gives relatively high portability at the Kubernetes API level.

But real clusters still depend on:
- cloud load balancers
- storage classes
- managed-service features
- proprietary integrations

## OpenShift

Kubernetes-based, so much workload YAML remains portable.

However, OpenShift-specific APIs and workflows increase dependency on Red Hat's platform.

Trade-off:

```text
OpenShift → more integration/support, more platform-specific dependency
Kubernetes → more choice, more integration work
```

---

# 14. Why recommend OpenShift?

OpenShift is strong when a company prioritizes:
- enterprise support
- integrated tooling
- standardized security
- developer self-service
- Kubernetes foundation
- predictable platform lifecycle

It is less attractive when:
- cost is critical
- team wants minimal platform abstraction
- small/simple deployment
- existing team already operates Kubernetes well

---

# 15. Kubernetes storage vs VM storage

VM workload:
- storage attached directly to a machine/VM
- application often depends on host-level disk configuration

Kubernetes:
- application asks for storage using PVC
- StorageClass/provisioner can dynamically create backing storage
- pod can be rescheduled independently from the original node when storage backend supports it

Kubernetes adds abstraction and automation at the cost of complexity.

---

# 16. Startup with two engineers

Likely recommendation:

```text
Compose / managed simple container platform
```

rather than self-managed Kubernetes.

Reason:
- lower operational overhead
- faster shipping
- lower cost
- less cluster administration

Move to Kubernetes when scaling/resilience/team requirements justify it.

---

# 17. Docker vs Podman

## Docker

Traditional architecture:
- Docker daemon (`dockerd`)
- client talks to daemon

## Podman

Daemonless container engine:
- containers run as regular processes
- strong rootless workflow
- Docker-compatible command style

Why security-conscious teams may prefer Podman:
- daemonless architecture
- strong rootless support
- reduced need for a privileged central daemon

Docker also supports rootless modes, so the comparison should not be exaggerated.

---

# 18. Self-managed vs managed Kubernetes

## Self-managed

You operate:
- control plane
- upgrades
- etcd/backups
- node lifecycle
- networking integrations
- monitoring/security integration

Advantages:
- maximum control
- potentially useful for special on-prem requirements

## Managed service

Cloud/provider handles much of the control-plane lifecycle.

Advantages:
- lower day-2 overhead
- easier upgrades
- integrated cloud services
- support/SLA depending on provider

Trade-off:
- provider cost
- cloud dependency
- some loss of control

---

# 19. Spiky global traffic

Strong candidate:

```text
managed Kubernetes + autoscaling + global load balancing/CDN
```

Why:
- horizontal scaling
- resilient multi-replica services
- cloud integration
- automated node capacity
- rolling updates
- global infrastructure integration

But Kubernetes alone is not a global traffic solution; CDN/global load balancing and cloud architecture are still needed.

---

# 20. Docker Swarm / Compose → Kubernetes

Reasons to migrate:
- multi-node scheduling
- stronger ecosystem
- self-healing
- scaling
- richer networking/storage
- widespread tooling

Costs:
- migration effort
- operational complexity
- training
- YAML/platform overhead

Migration is justified when long-term requirements exceed what the simpler platform handles comfortably.

---

# 21. Kubernetes for microservices

Strong fit when:
- many independently deployed services
- scaling differs per service
- resilience/self-healing needed
- service discovery required
- frequent rolling updates

Potential downside:
- operational complexity can exceed the value for small systems

---

# 22. Kubernetes for enterprise applications

Strengths:
- scale
- mature ecosystem
- strong community
- declarative deployment
- automation
- extensibility
- broad vendor support

Challenges:
- expertise
- governance
- security design
- platform operations

Enterprise suitability is high when an organization can support the platform operationally.

---

# 23. OpenShift for microservices

Strengths:
- Kubernetes orchestration
- integrated developer tooling
- Routes/networking
- enterprise security defaults
- operators
- vendor support

Trade-offs:
- cost
- platform-specific learning
- more opinionated environment

Good fit for organizations that want an integrated enterprise platform rather than assembling Kubernetes components themselves.

---

# 24. OpenShift for enterprise applications

Strong when an enterprise values:
- vendor support
- Kubernetes compatibility
- integrated security
- lifecycle management
- developer/operations tooling
- certified ecosystem

Weak when:
- budget is limited
- small/simple environment
- team wants a minimal/open Kubernetes stack without Red Hat-specific layers

---

# 25. How to answer LO6

Use this structure:

```text
1. State your recommendation.
2. Give 2–4 reasons.
3. Give the main trade-off.
4. State when the alternative would be better.
5. Conclude for the scenario.
```

Example:

```text
I would choose Compose for a two-person startup because it has much lower
operational overhead and can deploy a small multi-container application on one
host quickly. Kubernetes provides stronger scaling and resilience, but those
benefits do not justify the cluster-management cost at this stage. I would move
to Kubernetes later if the product requires multi-node availability,
autoscaling, or a much larger service estate.
```

That is a stronger LO6 answer than simply listing features.

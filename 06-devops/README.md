# DevOps — Senior Interview Guide

This chapter covers the containers, Kubernetes, CI/CD, cloud infrastructure, and operations questions expected from a senior engineer. Strong answers do not stop at naming a tool; they explain the underlying mechanism, the failure mode it changes, and the operational trade-off it introduces.

> **How to answer:** name the mechanism, explain what it protects against or costs, and finish with a concrete failure mode or production consequence.

## Contents

1. [Containers and images](#1-containers-and-images)
2. [Kubernetes core objects](#2-kubernetes-core-objects)
3. [Scheduling and resource management](#3-scheduling-and-resource-management)
4. [Configuration, secrets, and workload identity](#4-configuration-secrets-and-workload-identity)
5. [CI/CD and supply-chain security](#5-cicd-and-supply-chain-security)
6. [Deployment strategies](#6-deployment-strategies)
7. [Infrastructure as code](#7-infrastructure-as-code)
8. [Cloud networking and managed services](#8-cloud-networking-and-managed-services)
9. [Observability and incident response](#9-observability-and-incident-response)
10. [Production and reliability scenarios](#10-production-and-reliability-scenarios)
11. [Rapid revision](#11-rapid-revision)

---

## 1. Containers and images

### 1. What is a container, and how does it differ from a VM?

A container is an isolated process (or process group) sharing the host kernel, using namespaces for isolation (PID, network, mount, UTS, IPC, user) and cgroups for resource limits. A VM virtualizes hardware and runs its own kernel under a hypervisor.

Containers start in milliseconds and pack more densely per host because there is no second kernel, but they isolate less than a VM: a kernel exploit or misconfigured namespace can cross container boundaries in a way a hypervisor boundary normally does not.

### 2. What are image layers, and why do they matter for build speed and size?

An image is a stack of read-only layers, each the filesystem diff produced by one Dockerfile instruction; a thin writable layer is added per running container. Layers are content-addressed and cached, so an unchanged instruction reuses its existing layer on rebuild.

Ordering matters: put instructions that change rarely (base image, dependency installation) before instructions that change often (application code) so cache hits cover the expensive steps. A layer that adds a file and a later layer that deletes it both still exist in the image—deleting in a later layer does not shrink it.

### 3. How do you write a secure, minimal Dockerfile?

```dockerfile
FROM eclipse-temurin:21-jre-jammy AS run
RUN groupadd -r app && useradd -r -g app app
COPY --from=build /workspace/target/app.jar /app/app.jar
USER app
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

Use multi-stage builds so build tools and source never reach the runtime image. Pin base image versions, run as a non-root user, avoid baking secrets or `.git` history into layers, and keep the final image to what the application needs to run—not a full build toolchain.

### 4. Image versus container—what is the actual relationship?

An image is an immutable, versioned template: layered filesystem plus metadata (entrypoint, exposed ports, environment defaults). A container is a running (or stopped) instance of an image with its own writable layer, namespaces, and cgroup limits. Many containers can run from the same image concurrently without affecting each other.

### 5. How are container resource limits enforced by the kernel?

CPU and memory limits are cgroup settings, not application configuration. A CPU limit is enforced through a scheduling quota (throttling the process once it exceeds its share of CPU time in a period); a memory limit is enforced by the kernel's out-of-memory killer terminating the cgroup when it exceeds its byte ceiling.

CPU throttling shows up as added latency, not an error—a container can be “fine” by every health check while being throttled into unacceptable tail latency. This is invisible unless the specific throttling metric is monitored.

### 6. What happens when a container exceeds its memory limit?

The kernel OOM-killer terminates a process in the cgroup, and the container exits with code 137 (128 + SIGKILL). There is no graceful shutdown hook for this path—it is not `SIGTERM`, so `preStop` hooks and in-process cleanup do not run.

Set the JVM's heap and other memory pools comfortably under the container's memory limit (leaving room for metaspace, thread stacks, and native/off-heap memory), and prefer a container-aware JVM that reads the cgroup limit rather than the host's total memory.

### 7. How do you reduce container supply-chain risk?

- Use minimal or distroless base images to shrink the attack surface and the set of scannable vulnerabilities.
- Pin base images and dependencies to specific digests, not floating tags like `latest`.
- Scan images for known vulnerabilities in CI, and fail the build above an agreed severity.
- Sign images and verify signatures before deployment; generate an SBOM (software bill of materials) so a newly disclosed CVE can be matched against what is actually running.
- Avoid embedding credentials or build secrets in any layer, including ones later removed.

### 8. What is a container runtime, and how does Kubernetes use it?

The container runtime (containerd, CRI-O) is the component that actually pulls images and creates namespaces, cgroups, and processes; `runc` (or a similar low-level runtime) performs the final OCI-spec container creation. Kubernetes talks to the runtime through the Container Runtime Interface (CRI) rather than depending on one specific implementation.

The kubelet on each node is the client of this interface: it tells the runtime which containers should exist for the pods scheduled to that node and reports their status back to the control plane.

---

## 2. Kubernetes core objects

### 9. What is a Pod, and why is it the smallest deployable unit rather than a container?

A Pod is one or more containers that share a network namespace (one IP, one port space) and can share volumes, scheduled together onto the same node as a single unit. Containers in a pod are meant to be tightly coupled—for example, an application container and a sidecar that ships its logs.

You do not scale containers independently within a pod; you scale pod replicas. Treating “pod” and “container” as interchangeable causes confusion about networking (localhost between containers in a pod) and lifecycle (the whole pod is scheduled, evicted, and restarted together).

### 10. What does a Deployment manage, and how does a rolling update work under the hood?

A Deployment manages ReplicaSets, and each ReplicaSet manages a set of identical pod replicas. Updating a Deployment's pod template creates a new ReplicaSet and gradually scales it up while scaling the old ReplicaSet down, bounded by `maxSurge` and `maxUnavailable`.

The old ReplicaSet is kept (scaled to zero) for rollback, which is why `kubectl rollout undo` is fast—it just re-activates the previous ReplicaSet rather than rebuilding anything. A Deployment alone does not guarantee zero-downtime; that also depends on readiness probes gating traffic and the application handling in-flight requests during shutdown.

### 11. StatefulSet versus Deployment?

A Deployment's pods are interchangeable: any replica can be replaced with an identical, arbitrarily named one. A StatefulSet gives each replica a stable, ordinal identity (`app-0`, `app-1`, …), a stable network name, and its own persistent volume that follows that identity across rescheduling.

Use a StatefulSet for anything where identity or attached storage matters—databases, brokers, or clustered systems that track peer membership by name. Ordered, one-at-a-time startup and shutdown is also a StatefulSet default, which matters for systems with leader election or replication bootstrapping.

### 12. What is a Service, and how does it provide stable networking to ephemeral pods?

A Service defines a stable virtual IP and DNS name in front of a dynamic set of pod endpoints selected by label. `kube-proxy` (or an eBPF-based equivalent) programs the node's networking so traffic to the Service IP is load-balanced across the current healthy endpoints, updated automatically as pods come and go.

Without this indirection, clients would need to track individual pod IPs, which are reassigned on every pod restart or rescheduling.

### 13. `ClusterIP` versus `NodePort` versus `LoadBalancer` versus `Ingress`?

| Type | Reachable from | Typical use |
|---|---|---|
| `ClusterIP` | Inside the cluster only | Default; internal service-to-service traffic |
| `NodePort` | Any node's IP, on a fixed port | Simple external access, mostly for development |
| `LoadBalancer` | Internet, via a cloud load balancer | One Service directly exposed externally |
| `Ingress` | Internet, via a shared entry point | HTTP(S) routing, TLS termination, host/path rules across many Services |

An `Ingress` is not a Service type; it is a separate object interpreted by an Ingress controller, and it normally routes to `ClusterIP` Services rather than exposing each one with its own cloud load balancer.

### 14. What does Ingress add over a plain `LoadBalancer` Service?

Layer 7 routing: host- and path-based rules that fan a single external IP and load balancer out to many backend Services, plus centralized TLS termination and often authentication, rewriting, and rate limiting at the controller. A `LoadBalancer` Service, by contrast, is one IP dedicated to one Service.

Running one cloud load balancer per Service is more expensive and gives up L7 features that a shared Ingress controller (or gateway) provides.

### 15. Liveness versus readiness versus startup probes?

- **Liveness** answers “is this process stuck and should it be restarted?” Failing it kills and restarts the container.
- **Readiness** answers “can this pod currently serve traffic?” Failing it removes the pod from Service endpoints without restarting it.
- **Startup** protects slow-starting applications from being killed by liveness before they have finished initializing; liveness and readiness only begin once startup succeeds.

A common production incident is a liveness probe that depends on a downstream dependency: the dependency degrades, liveness starts failing, and Kubernetes restarts otherwise-healthy pods into a crash loop instead of just marking them not-ready. Liveness should check the process's own health, not its dependencies.

### 16. What happens during a Pod's graceful termination?

On delete, the pod is marked `Terminating`, immediately removed from Service endpoints, and sent `SIGTERM`; any `preStop` hook runs first. The process has `terminationGracePeriodSeconds` (default 30s) to exit before Kubernetes sends `SIGKILL`.

Because endpoint removal and `SIGTERM` delivery are not perfectly synchronized, a brief window can exist where a request still arrives after the process starts shutting down. Handle `SIGTERM` by finishing in-flight requests and stopping acceptance of new ones (or use a `preStop` sleep) rather than exiting immediately, and make sure the grace period is long enough for the slowest in-flight request.

### 17. What is a namespace, and what does it isolate—and not isolate?

A namespace partitions names, RBAC scope, resource quotas, and network policy targeting within one cluster. It does not provide node-level or kernel-level isolation—pods from different namespaces can still land on the same node and, without a `NetworkPolicy`, can reach each other over the network by default.

Namespaces are an organizational and policy boundary for a shared cluster, not a hard multi-tenancy boundary; strong tenant isolation additionally needs network policy, resource quotas, and sometimes dedicated node pools.

---

## 3. Scheduling and resource management

### 18. Requests versus limits?

A **request** is what the scheduler reserves for a pod—it will only place the pod on a node with that much allocatable capacity, and it is what the pod is guaranteed. A **limit** is the ceiling the kernel enforces at runtime—CPU is throttled, memory beyond the limit triggers an OOM-kill.

Setting limits without requests, or requests far below real usage, causes over-commitment: the node advertises more scheduled capacity than it can actually deliver simultaneously, and pods compete or get throttled under load.

### 19. What are Kubernetes QoS classes, and how do they affect eviction order?

- **Guaranteed**: requests equal limits for every container, for both CPU and memory.
- **Burstable**: requests are set but below limits (or only some resources/containers set them).
- **BestEffort**: no requests or limits at all.

Under node memory pressure, the kubelet evicts BestEffort pods first, then Burstable pods (ranked by how far usage exceeds requests), and Guaranteed pods last. A pod with no requests set is not “unlimited” in a good way—it is first in line for eviction.

### 20. How does the Horizontal Pod Autoscaler work?

The HPA periodically compares an observed metric (commonly CPU or memory utilization relative to requests, or a custom/external metric) against a target, and adjusts replica count proportionally: `desiredReplicas = currentReplicas × (currentMetric / targetMetric)`, subject to min/max bounds and stabilization windows that prevent flapping.

It scales replica count, not resource requests per pod—if a single pod cannot handle its share of traffic even at low replica count, horizontal scaling alone will not fix a per-pod bottleneck.

### 21. HPA versus VPA versus Cluster Autoscaler?

| Autoscaler | Scales | Typical trigger |
|---|---|---|
| Horizontal Pod Autoscaler (HPA) | Number of pod replicas | CPU/memory/custom metric utilization |
| Vertical Pod Autoscaler (VPA) | A pod's requests/limits | Historical usage recommendations |
| Cluster Autoscaler | Number of nodes | Pending pods that don't fit existing capacity |

They address different bottlenecks: HPA adds capacity horizontally, VPA right-sizes a single pod's resource footprint, and Cluster Autoscaler makes sure the cluster has enough nodes for whatever HPA/VPA decide to schedule. Running HPA and VPA on CPU for the same workload simultaneously can fight each other.

### 22. What is a PodDisruptionBudget, and when does it matter?

A PodDisruptionBudget (PDB) declares the minimum available (or maximum unavailable) replicas of a workload during *voluntary* disruptions—node drains for upgrades, cluster-autoscaler scale-down, or manual maintenance. The eviction API respects it; it does not prevent involuntary disruption like a node crashing.

Without a PDB, a node drain can legally evict every replica of a service at once if they happen to sit on the same node. A PDB is what makes “drain this node” and “still have quorum” compatible for clustered or low-replica-count workloads.

### 23. Node affinity/anti-affinity versus taints and tolerations?

Affinity and anti-affinity are expressed on the pod: “prefer/require scheduling near/away from nodes or pods matching these labels” (for example, spreading replicas across zones). Taints are expressed on the node: “do not schedule here unless you tolerate this taint,” used to reserve nodes (GPU pools, dedicated tenants) or repel workloads from degraded nodes.

They are complementary, not alternatives: taints/tolerations gate *which* pods are allowed on a node at all, while affinity/anti-affinity expresses a *preference or requirement* among the nodes that are otherwise eligible.

### 24. What causes a Pod to be stuck `Pending`, `CrashLoopBackOff`, or `Evicted`?

- **`Pending`**: the scheduler cannot place it—insufficient node resources for the requests, an unsatisfiable affinity/taint constraint, or an unbound `PersistentVolumeClaim`.
- **`CrashLoopBackOff`**: the container starts and exits repeatedly—application startup failure, missing configuration/dependency, or a failing liveness/startup probe restarting it before it stabilizes. Backoff delay grows between restarts.
- **`Evicted`**: the kubelet reclaimed resources under node pressure (usually memory or disk), removing the pod outside the normal scheduler/controller flow.

`kubectl describe pod` and events are the first stop for `Pending`/`Evicted`; `kubectl logs --previous` is usually more useful than the current log for `CrashLoopBackOff`, since the current attempt may not have logged anything yet.

### 25. At a high level, how does the scheduler decide where a pod runs?

It filters nodes that satisfy hard constraints (resource requests, taints/tolerations, required affinity, volume topology), then scores the remaining candidates (spreading, resource balance, affinity preferences) and picks the highest-scoring node. This runs once per pod at scheduling time—it is not continuously rebalancing already-running pods.

Because placement is a point-in-time decision, a cluster can accumulate uneven bin-packing over time as pods churn; some environments run a separate descheduler to periodically rebalance.

---

## 4. Configuration, secrets, and workload identity

### 26. `ConfigMap` versus `Secret`—and why is a Secret not automatically secure?

Both hold key-value configuration injected as environment variables or mounted files; a `Secret` is intended for sensitive values. By default, `Secret` data is only base64-encoded, not encrypted—readable by anyone who can read the object via the API, and only as secure as RBAC and (if enabled) etcd encryption-at-rest make it.

Treat a Secret as “access-controlled,” not “encrypted,” unless the cluster has encryption at rest configured and RBAC actually restricts who can `get`/`list` Secrets.

### 27. How should secrets be managed in production Kubernetes?

Prefer an external secrets manager (cloud KMS-backed secret store, Vault, or similar) as the source of truth, synced into the cluster or fetched at runtime, rather than raw manifests committed anywhere. Mount secrets as files rather than environment variables where possible—env vars are more likely to leak into logs, crash dumps, or child-process environments and are visible via `/proc/<pid>/environ`.

Rotate credentials on a schedule and support rotation without a full redeploy (the application should re-read a mounted secret, or handle a controlled restart), and scope each secret narrowly rather than sharing one broad credential across services.

### 28. What is workload identity, and why is it preferred over long-lived credentials?

Workload identity lets a pod or service account assume a cloud IAM identity directly (through short-lived, automatically rotated tokens federated from the Kubernetes service account) instead of the application holding a long-lived cloud access key as a Secret. The credential never needs to be stored, copied, or manually rotated.

This removes an entire class of incidents: a leaked long-lived key valid indefinitely versus a leaked token that expires in minutes and was scoped to exactly one workload's permissions.

### 29. What does a `NetworkPolicy` do, and what is the default without one?

By default, pods can reach any other pod in the cluster with no network-level restriction—namespaces and Services do not implicitly isolate traffic. A `NetworkPolicy` allow-lists ingress and/or egress traffic for pods matching a label selector; once any policy selects a pod, all traffic not explicitly allowed is denied for the direction(s) covered.

A common gap is applying policies to some services but not enforcing “default deny” at the namespace level, leaving unlabeled or new pods implicitly open.

### 30. How do you manage configuration across environments without drift?

Externalize configuration from the image (twelve-factor style): the same built artifact is promoted across environments, with environment-specific values supplied through ConfigMaps/Secrets or a config service, not by rebuilding the image per environment. Keep configuration in version control and apply it declaratively, so environment differences are reviewable diffs rather than manual edits.

Rebuilding per environment (even from the “same” source) reintroduces exactly the risk artifact promotion is meant to remove: what you tested is not bit-for-bit what you ship.

### 31. What does a service mesh and mTLS add beyond application-level code?

A service mesh (sidecar proxies alongside each pod) can provide mutual TLS, retries, timeouts, load balancing, and traffic shifting uniformly across services without each application implementing them, plus consistent telemetry regardless of language. mTLS specifically gives every service a cryptographic identity and encrypts and authenticates service-to-service traffic, so a compromised or misconfigured pod cannot silently impersonate another service or read plaintext traffic.

The cost is added latency per hop, operational complexity of the mesh control plane itself, and another moving part that can fail independently of the applications it fronts.

### 32. How do you rotate a secret without downtime?

Support two valid values during the transition: issue the new secret, deploy consumers that accept both old and new (or fetch dynamically rather than caching indefinitely), confirm all instances have picked up the new value, then revoke the old one. Coordinate rotation order for interdependent systems—rotating a database password before the applications that use it can pick up the new value causes an outage, not a rotation.

Tie rotation to detection, too: a credential that can only be rotated through a manual, risky process will not actually get rotated promptly during an incident.

---

## 5. CI/CD and supply-chain security

### 33. What are the typical stages of a CI/CD pipeline, and what should gate promotion between them?

Build → unit/integration tests → security and dependency scanning → package/publish a single immutable artifact → deploy to a staging-like environment → automated verification (smoke tests, contract tests) → promote the *same* artifact to production, often behind a progressive rollout. Each gate should be a pass/fail check the pipeline enforces, not a step a human remembers to run.

The critical property is that the artifact promoted to production is byte-identical to the one that passed every earlier stage—rebuilding at each stage reopens the gap between “what we tested” and “what we ship.”

### 34. What is artifact promotion, and why build once and promote rather than rebuilding per environment?

Build once, produce one versioned, immutable artifact (container image, jar), and move that same artifact through environments by changing only its configuration and target. Rebuilding per environment risks picking up a different dependency version, base image patch, or non-deterministic build step, so “it passed staging” no longer guarantees anything about the production build.

Promotion should be an explicit, auditable action (tag, registry copy, or gated pipeline stage) rather than an implicit side effect of merging code.

### 35. How do you keep CI/CD secrets and credentials safe?

Prefer short-lived, narrowly scoped credentials issued per run over long-lived static secrets stored in the CI system—for example, OIDC federation from the CI provider to the cloud account, so a token is minted just-in-time and expires with the job. Mask secret values in logs, restrict which branches/environments can access production credentials, and avoid printing environment variables or full command lines that might contain them.

A compromised long-lived CI secret is a standing risk until someone notices and rotates it; a compromised short-lived federated token is only valid for the remainder of that job.

### 36. What does supply-chain security mean for a build pipeline?

Protecting the pipeline that produces the artifact, not just the artifact's own code: pinned and verified dependencies, a locked-down build environment, signed commits and artifacts, and generated provenance (what source, what pipeline, what inputs produced this exact artifact) so a consumer can verify it wasn't tampered with after the fact. Frameworks like SLSA describe graduated levels of this rigor.

Real incidents in this category include compromised build dependencies, poisoned base images, and pipeline credentials that let an attacker inject code without ever touching the source repository.

### 37. Trunk-based development versus long-lived feature branches, from a CI/CD perspective?

Trunk-based development keeps branches short-lived and integrates to a shared branch frequently, which keeps CI fast, integration conflicts small, and continuous delivery realistic. Long-lived feature branches defer integration, which tends to produce large, risky merges and stale CI feedback about how the branch behaves with everyone else's changes.

Trunk-based development depends on techniques that make incomplete work safe to merge—feature flags, backward-compatible changes—rather than using the branch itself to hide unfinished work.

### 38. What should a deployment pipeline verify before promoting to production?

Automated tests at the appropriate levels, security/dependency scan results within policy, a successful deploy and health check in a pre-production environment, and (for a progressive rollout) live signal from the canary stage—error rate, latency, and business metrics compared against a baseline—before continuing the rollout. A pipeline that only checks “did the deploy command succeed” has verified availability of the process, not correctness of the release.

### 39. What is GitOps, and how does it differ from push-based CD?

GitOps declares the desired cluster/infrastructure state in a Git repository, and an in-cluster controller continuously reconciles actual state to match it—rather than a CI pipeline pushing changes directly to the cluster with its own credentials. Git becomes the auditable single source of truth and the rollback mechanism (revert the commit), and drift is detected and corrected automatically instead of silently accumulating.

The trade-off is a layer of indirection: a deploy is “done” when the controller reconciles it, not the instant the pipeline finishes, and troubleshooting requires understanding both the desired-state repository and the reconciler's current view of the cluster.

---

## 6. Deployment strategies

### 40. Rolling versus blue-green versus canary deployment?

| Strategy | Mechanism | Rollback speed | Blast radius during rollout |
|---|---|---|---|
| Rolling | Gradually replace old replicas with new ones in place | Moderate—must roll back through replicas | Partial, grows with progress |
| Blue-green | Deploy the new version fully alongside the old, then switch traffic | Fast—switch routing back | All-or-nothing at cutover |
| Canary | Route a small percentage of traffic to the new version, then increase | Fast—shift traffic back | Small, controlled by traffic percentage |

Rolling is the Kubernetes Deployment default and is resource-efficient but runs both versions simultaneously during the transition, which requires backward-compatible contracts. Blue-green needs double the capacity briefly but gives instant rollback. Canary trades slower full rollout for the smallest blast radius and the chance to catch a bad release on a fraction of traffic.

### 41. What is expand-and-contract (parallel change), and why does it matter for schema or API changes?

Split a breaking change into safe steps: **expand** by adding the new field/column/endpoint alongside the old one (both work), migrate all producers and consumers to the new shape, then **contract** by removing the old one only once nothing depends on it. At every intermediate step, both old and new code can run against the same schema or contract.

This is what makes rolling deployments (old and new versions running simultaneously) and independent service deployment safe for changes that would otherwise be breaking if done as one atomic cutover.

### 42. How do you evaluate a canary automatically rather than by watching a dashboard?

Compare the canary's key metrics (error rate, latency percentiles, saturation, relevant business metrics) against the baseline (previous version or a control group) using predefined thresholds, and automatically halt or roll back the rollout if the canary breaches them—rather than relying on a human noticing in time. Give the canary enough traffic and time to be statistically meaningful; too small or too short a canary will not surface a regression that only appears under load or after some warm-up period.

### 43. What role do feature flags play in decoupling deploy from release?

A flag lets code ship to production dark (deployed but inactive), and be enabled separately—for a percentage of users, a specific cohort, or instantly toggled off if something goes wrong—without a new deployment. This separates “is the code running in production” from “is the behavior visible to users,” which shrinks the blast radius of a bad feature to a flag flip instead of a rollback.

Flags accumulate cost too: stale flags left in code after full rollout become dead branches and technical debt if not cleaned up.

### 44. How do database migrations complicate a deployment strategy?

During a rolling or canary deployment, old and new application versions run against the same database simultaneously, so a migration must be compatible with both: add columns as nullable or with defaults, don't drop a column the old version still reads, and generally follow expand-and-contract rather than an atomic destructive migration. A migration that is only safe for the new code breaks the old code still running mid-rollout.

Long-running or locking migrations also need to be decoupled from deploy timing—running a lock-heavy migration synchronously with a deploy risks the migration and the rollout blocking each other.

### 45. What causes mixed-version failures during a rolling deploy, and how do you prevent them?

Old and new versions coexist for the duration of the rollout; a mixed-version failure happens when they disagree about a contract—a changed API shape, an incompatible message schema, or a migration only one version understands. Prevention is making every change backward- and forward-compatible for at least one deploy cycle (expand-and-contract, additive schema/API changes, versioned messages) rather than assuming the rollout completes instantly.

If a mixed-version failure is discovered mid-rollout, the fastest mitigation is usually to pause or roll back the rollout, not to “push through” faster.

---

## 7. Infrastructure as code

### 46. What is Infrastructure as Code, and what problem does it solve versus manual changes?

Infrastructure is defined in version-controlled, declarative (or programmatic) files and applied through automation, rather than created or edited by hand through a console. It gives infrastructure the same review, diff, history, and repeatability guarantees as application code, and it removes the “what exactly did someone click” gap that makes manual changes (ClickOps) hard to reproduce, audit, or roll back.

### 47. Declarative versus imperative IaC?

Declarative IaC (Terraform, Kubernetes manifests, CloudFormation) states the desired end state, and the tool computes the diff and the steps to reach it. Imperative IaC (a sequence of provisioning scripts or CLI calls) states the steps to perform directly, and the resulting state is whatever running those steps produced.

Declarative tools can detect and reconcile drift because they track desired versus actual state; imperative scripts generally cannot, since they have no model of current state to compare against—rerunning them is only as safe as each individual step being idempotent.

### 48. What is “state” in a tool like Terraform, and why is it dangerous to lose or corrupt?

State is the tool's record of which real-world resources correspond to which declared resources, including attributes not visible from the configuration alone. Without accurate state, the tool cannot compute a correct diff—it may try to recreate resources that already exist, or fail to detect that a resource was deleted out-of-band.

Store state remotely with locking (not a local file) so concurrent runs cannot corrupt it, and treat it as sensitive—it often contains resource identifiers and sometimes secrets.

### 49. What is environment drift, and how do you detect and prevent it?

Drift is a gap between the declared infrastructure configuration and the actual running state, usually caused by manual out-of-band changes (an emergency console fix that was never ported back into code). Detect it by periodically running a plan/diff against real infrastructure and alerting on unexpected changes; prevent it by restricting direct console/API write access and requiring all changes to go through the IaC pipeline, even during incidents.

Drift is dangerous specifically because the next automated apply may “correct” a manual emergency fix back to the old, broken configuration, reintroducing the very incident it fixed.

### 50. How do you safely roll out an infrastructure change?

Generate and review a plan (the diff between desired and current state) before applying, especially checking for unexpected destroys or replacements of stateful resources. Apply to a lower environment first, limit blast radius by scoping changes narrowly (module/target) rather than applying broadly, and require approval for changes affecting production or anything stateful.

Some changes cannot be trivially rolled back (a destroyed database, a changed CIDR block that breaks routing)—for those, plan review and staged rollout matter more than the ability to revert afterward.

### 51. Why must applying IaC twice be safe (idempotency)?

Pipelines retry, humans re-run commands, and reconciliation loops apply the same configuration repeatedly by design (as in GitOps)—if applying the same configuration twice produced a different or broken result, every one of those normal operations would be a hazard. Idempotency means “ensure this state exists,” computed as a diff against current state, rather than “perform this action,” which is why declarative tools are naturally idempotent and hand-rolled imperative scripts often are not unless deliberately written to check before acting.

---

## 8. Cloud networking and managed services

### 52. How does DNS-based routing/failover work, and what are its limits?

A DNS record can point to multiple targets or be updated to redirect traffic (for failover, geo-routing, or weighted splitting), and clients resolve it through recursive resolvers that cache the answer for the record's TTL. This makes DNS changes slow and imprecise as a failover mechanism: caching resolvers, clients that ignore TTL, and stale application-level DNS caches (notably the JVM's default infinite DNS cache) mean some fraction of clients keep hitting the old target well past the TTL.

For fast, precise failover, prefer a load balancer or routing layer under your control over relying on DNS propagation alone, and set conservative TTLs in advance of any planned failover.

### 53. What does a load balancer do at Layer 4 versus Layer 7?

An L4 load balancer distributes connections based on IP/port (TCP/UDP) without inspecting application content—fast and protocol-agnostic, but it cannot route on URL path, header, or hostname. An L7 load balancer understands the application protocol (typically HTTP), so it can route by path or host, terminate TLS, inspect and modify headers, and make routing decisions based on request content—at the cost of more processing per request.

Ingress controllers and API gateways operate at L7 for exactly this reason: they need host/path-based routing, which an L4 balancer cannot express.

### 54. What is a CDN, and when does it help—and not help?

A CDN caches content at edge locations closer to users, reducing latency and origin load for cacheable content (static assets, and increasingly cacheable API responses). It helps most for content that is the same for many users and safe to serve stale for some interval.

It does not help—and can actively hurt—for highly personalized or rapidly changing responses, where cache misses add a hop with no benefit, or where an overly aggressive cache policy serves stale/incorrect data. Cache invalidation strategy (TTL, purge-on-write, cache keys that vary correctly by relevant request attributes) is the actual hard part.

### 55. Public versus private subnets—why put databases in a private subnet?

A public subnet has a route to an internet gateway, so resources there can be reached from (and reach) the internet directly. A private subnet has no direct inbound route from the internet; outbound internet access, if needed, goes through a NAT gateway.

Putting a database in a private subnet removes an entire class of exposure—it simply cannot be reached from the public internet regardless of security group misconfiguration—leaving network access control as a second layer of defense rather than the only one.

### 56. When should you use a managed database or service instead of self-hosting?

Default to managed unless there is a specific reason not to: the provider handles patching, backups, failover, and much of the operational burden, which is usually cheaper than the engineering time to replicate it well. Reasons to self-host include a requirement the managed offering doesn't support, cost at very large predictable scale, or regulatory/data-residency constraints the managed option can't satisfy.

The trade-off is control and portability versus operational burden—self-hosting means your team owns every failure mode the managed service would otherwise absorb.

### 57. What is a NAT gateway, and why do private subnets need one?

A NAT gateway lets resources in a private subnet initiate outbound connections to the internet (to pull a package, call an external API) while presenting a shared public IP, without exposing those resources to *inbound* connections from the internet. It preserves the private subnet's core property—not directly reachable from outside—while still allowing necessary outbound traffic.

Without it, a private-subnet resource that needs any outbound internet access would have to move to a public subnet, giving up that protection for a need that is actually only outbound.

### 58. How do you design for zero-downtime DNS or load balancer cutover?

Bring the new target up and verified healthy before changing any routing, lower the DNS TTL well in advance if DNS is involved, and prefer changing a load balancer's target group or an internal routing rule (immediate) over changing a DNS record (slow, cache-dependent) when the option exists. Keep the old target running and healthy for a rollback window after cutover rather than tearing it down immediately.

Validate with real traffic at low weight (a canary-style shift) before moving all traffic, so a bad cutover affects a fraction of requests instead of everyone at once.

---

## 9. Observability and incident response

### 59. What makes an alert actionable, and what is alert fatigue?

An actionable alert states a symptom that requires a human response now (user-facing error rate above threshold, saturation nearing capacity), links to context (dashboard, runbook), and has a clear owner. Alerting on causes rather than symptoms (a single retry, a brief GC pause) produces alerts nobody needs to act on individually.

Alert fatigue is the result of too many low-value or non-actionable alerts: on-call engineers start reflexively acknowledging and dismissing pages, and a real signal gets buried in noise exactly when it matters most.

### 60. What is a blameless postmortem for, and what should it capture?

Its purpose is to extract system-level lessons and preventive action, not to assign individual fault—assigning blame makes people hide information in the next incident instead of surfacing it. It should capture a timeline, the actual impact, the immediate mitigation, root and contributing causes, and concrete follow-up actions with owners and due dates.

A postmortem with no resulting action items has not actually improved anything; “retrain the team to be more careful” is rarely a sufficient action item on its own.

### 61. What is chaos engineering or a game day, and why run one deliberately?

Deliberately injecting failure (killing an instance, adding latency, cutting a dependency) in a controlled way, to verify that resilience mechanisms—failover, retries, alerting, runbooks—actually work, rather than discovering they don't during a real incident. A game day is the scheduled, rehearsed version involving the on-call team, testing both the system's behavior and the humans' response process.

The value is finding gaps (a failover that was never actually tested end-to-end, an alert that never fires, a runbook that's out of date) while everyone is prepared for it, instead of at 3 a.m. during a genuine outage.

### 62. How do dashboards differ for triage versus long-term capacity planning?

A triage dashboard needs to answer “what is wrong right now and where” fast: current error rates, latency, saturation, and recent deploys, at a time resolution fine enough to correlate with an incident. A capacity dashboard needs longer time ranges and trend lines—growth rate, headroom against limits, seasonality—to answer “when do we run out, and of what.”

Using one dashboard for both usually serves neither well: the noise-reduction and long time ranges useful for capacity planning obscure the sharp, recent signal needed for triage.

### 63. What is a runbook, and why automate it into remediation where possible?

A runbook is a documented, step-by-step response to a known failure mode, so an on-call engineer unfamiliar with a specific subsystem can respond correctly under pressure. Where the diagnosis and fix are well-understood and low-risk, automating the runbook into a script or auto-remediation removes both human reaction time and the chance of a tired engineer executing a manual step incorrectly.

A runbook that only exists in someone's memory does not survive that person being unavailable during the incident; a runbook that is never updated as the system changes becomes actively misleading.

### 64. How do you manage log and metric cost and cardinality at platform scale?

High-cardinality labels (raw user ID, full URL with query string, request ID as a metric label) multiply the number of unique time series or log index entries, which can silently blow up storage and query cost, or make a metrics backend fall over. Keep unbounded or high-cardinality values in logs/traces (searchable, not aggregated) rather than as metric labels, and use sampling and tiered retention—full detail for a short recent window, aggregated or sampled data for longer retention.

Cost and cardinality problems are usually invisible until a single bad label change or a traffic spike multiplies them, so review label cardinality before merging instrumentation, not after the bill arrives.

### 65. What is an escalation policy, and why does it matter for MTTA/MTTR?

An escalation policy defines who is paged first, how long before it escalates to the next person or team if unacknowledged, and what happens if nobody responds—removing ambiguity about ownership during an incident. Without one, mean time to acknowledge (MTTA) and mean time to resolve (MTTR) both suffer: a page can sit unacknowledged, or land on someone without the context or access to act on it.

A good policy also accounts for cross-team dependencies—paging the owning team for a symptom that actually originates in a dependency wastes the time-to-resolution on a misrouted page.

---

## 10. Production and reliability scenarios

### 66. A pod is stuck in `CrashLoopBackOff` after a deploy. How do you diagnose it?

Check `kubectl logs --previous` (the crashed attempt, not the empty new one), `kubectl describe pod` for events (image pull failure, OOM-kill, failed probe), and confirm whether the new pod template actually differs meaningfully from the last known-good one (config, secret, image tag). Distinguish a genuine application startup failure from an environment problem (missing ConfigMap key, unreachable dependency, insufficient resources).

If the fix isn't immediate, roll back the Deployment first to restore service, then diagnose the failing version out of the production path.

### 67. A rollout causes a spike in 5xx errors. Walk through your response.

Confirm the timing correlates with the deploy, then decide fast: if error rate is clearly elevated and rising, roll back (or pause a canary) before doing deep root-cause analysis—restoring service comes first. Check whether it's uniform across all new pods (a code/config defect) or isolated to a subset (a bad node, a partial rollout artifact, a dependency only some replicas can reach).

Once mitigated, correlate with request traces, recent config/schema changes, and dependency health to find the actual cause, and add a regression test or canary check that would have caught it automatically next time.

### 68. Nodes are being OOM-killed under normal load. What do you check?

Compare actual pod memory usage against configured requests/limits—pods with no limits or with limits set too loosely allow a single pod to consume node memory that others needed. Check for a memory leak in a specific workload (rising usage over time rather than a stable working set), and check the node's own overhead (system daemons, kubelet, monitoring agents) against what was budgeted as allocatable.

Also check QoS class distribution: too many Burstable/BestEffort pods without adequate requests means the scheduler under-reserves memory relative to what actually gets used under load.

### 69. The cluster autoscaler isn't adding nodes despite pending pods. What do you inspect?

Check why the scheduler can't place the pending pods in the first place (`kubectl describe pod` events)—the autoscaler only adds nodes it believes will let a pending pod schedule, so an unsatisfiable constraint (a node selector or affinity rule no node group can satisfy, a resource request larger than any available instance type, a volume topology constraint) leaves pods pending with no amount of added capacity helping. Also check the autoscaler's own limits (max node count reached) and cloud-provider quota (instance quota exhausted) as more mundane causes.

### 70. How do you decide between scaling up and scaling out, and what does each cost operationally?

Scaling out (more replicas) improves availability and fits stateless workloads well, but only helps if the bottleneck is genuinely parallelizable load rather than a single hot resource (a shared lock, a single-writer database). Scaling up (bigger instances) can resolve a per-instance bottleneck without redesigning for distribution, but concentrates risk in fewer, larger units and eventually hits a ceiling on instance size.

In practice, most stateless services should scale out by default; scaling up is often a stopgap for a component (frequently a stateful one) that hasn't been redesigned to shard or replicate.

### 71. How would you design for regional disaster recovery, and what do RTO/RPO drive?

Start from the business requirement expressed as RTO (how long can we be down) and RPO (how much data can we lose), since those numbers—not a generic “multi-region” goal—determine the architecture: active-passive with periodic replication and a documented failover runbook for a looser RTO/RPO, versus active-active with synchronous or near-synchronous replication and automated failover for a tight one. Also plan for the failback path, not just failover—many DR plans are tested one-way and never rehearsed in reverse.

Actually rehearsing a full regional failover (a game day, not a tabletop exercise) is usually the only way to discover the untested assumption that breaks it—a hardcoded region reference, a manual step nobody automated, a dependency that doesn't actually exist in the failover region.

### 72. Cloud costs have crept up 3x without a proportional increase in traffic. How do you investigate?

Break the bill down by service/resource and compare against a traffic/usage baseline to find what grew disproportionately—often idle over-provisioned capacity, an autoscaler stuck high after a load spike it never scaled back down from, unused or forgotten resources (orphaned volumes, load balancers, snapshots), or a change in data transfer/egress patterns rather than compute itself. Cross-check recent deploys and infrastructure changes against the timing of the cost increase.

Distinguish a one-time step change (a new always-on environment, a changed instance type) from continuous growth (a leak, an unbounded retry loop generating requests, log/metric volume growth)—the fix and the urgency differ.

### 73. What distinguishes a senior DevOps/platform answer?

- Explains the mechanism (cgroups, reconciliation loop, DNS caching) rather than naming a tool.
- Treats requests/limits, quotas, and autoscaler bounds as capacity planning, not defaults to leave unset.
- Designs deployments and migrations for the version-skew window, not just the final state.
- Separates “deployed” from “released” using progressive rollout and feature flags.
- Builds alerting around actionable symptoms and rehearses failure instead of only documenting it.
- Treats infrastructure changes with the same review and rollback discipline as application code.
- Connects cost, capacity, and reliability trade-offs to the actual business requirement (RTO/RPO, traffic pattern) rather than a default architecture.

---

## 11. Rapid revision

### Must-answer questions

Before an interview, answer these without notes:

1. Container versus VM isolation?
2. Why does image layer order affect build cache efficiency?
3. What happens when a container exceeds its memory limit?
4. Requests versus limits, and what enforces each?
5. What are QoS classes, and how do they drive eviction order?
6. How does a rolling update actually work under a Deployment?
7. StatefulSet versus Deployment?
8. `ClusterIP` versus `NodePort` versus `LoadBalancer` versus `Ingress`?
9. Liveness versus readiness versus startup probes?
10. What happens during graceful pod termination?
11. HPA versus VPA versus Cluster Autoscaler?
12. What does a `PodDisruptionBudget` protect against—and not protect against?
13. Why is a Kubernetes `Secret` not automatically secure?
14. What is workload identity, and why is it safer than a static credential?
15. What does a `NetworkPolicy` do, and what is the default without one?
16. Build once and promote versus rebuild per environment—why does it matter?
17. What is supply-chain security for a CI/CD pipeline?
18. What is GitOps, and how does it differ from push-based CD?
19. Rolling versus blue-green versus canary deployment?
20. What is expand-and-contract, and why does it enable safe rolling deploys?
21. Why do database migrations need to support both old and new app versions mid-rollout?
22. Declarative versus imperative IaC?
23. Why is IaC state dangerous to lose or corrupt?
24. What is environment drift, and how do you prevent it?
25. L4 versus L7 load balancing?
26. Why put a database in a private subnet?
27. What makes an alert actionable versus alert fatigue?
28. What is a blameless postmortem for?
29. Why run chaos engineering or a game day deliberately?
30. What do RTO and RPO drive in a disaster-recovery design?

### Thirty-second summary

Containers isolate processes with namespaces and cgroups rather than virtualizing a kernel, so resource limits and image hygiene are enforced at the kernel and build level, not just in application code. Kubernetes objects—pods, Deployments, Services, Ingress—exist to give ephemeral, replicated workloads stable identity, networking, and controlled rollout, governed by requests/limits, QoS, and disruption budgets that determine what survives contention. Safe delivery depends on promoting one immutable artifact through gated stages, supply-chain integrity, and rollout strategies (rolling, blue-green, canary) paired with backward-compatible expand-and-contract changes. Infrastructure as code and GitOps bring that same reviewable, idempotent discipline to the environment itself, while observability, actionable alerting, and rehearsed failure (game days, DR drills) are what make recovery a practiced procedure instead of an improvisation.

## Official references

- [Kubernetes documentation](https://kubernetes.io/docs/home/)
- [Docker documentation](https://docs.docker.com/)
- [The Twelve-Factor App](https://12factor.net/)
- [Google SRE Book](https://sre.google/sre-book/table-of-contents/)
- [SLSA supply-chain security levels](https://slsa.dev/)
- [Terraform documentation](https://developer.hashicorp.com/terraform/docs)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [CNCF Cloud Native Glossary](https://glossary.cncf.io/)

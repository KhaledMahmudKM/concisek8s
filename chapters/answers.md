# Answers to Chapter Questions

Answers and short explanations for the **Check your understanding** questions at the end of each chapter. Try the questions first; each section links back to them.

## Chapter 1: Why Kubernetes

**1. A. Orchestrate containers across many machines: placement, failure, discovery and change**

Docker packages and runs containers. Orchestration across many hosts is the gap Kubernetes fills.

**2. C. Discovery**

Clients need a stable way to reach a workload whose address keeps changing: that is discovery.

**3. B. You describe the desired state and the cluster works continuously to make reality match it**

You state the result, such as three copies of web:1.2; control loops close any gap between that and reality.

**4. D. The cluster sees two running where three are desired and starts a third on a healthy node**

The desired state did not change, so the cluster restores it on its own.

**5. B. Change the image in the description from 1.2 to 1.3 and let the cluster work out the steps**

You change the desired state; the cluster plans the safe sequence of replacements.

**6. D. A container runtime such as containerd, through the CRI**

Kubernetes is not a container runtime; it asks a runtime, usually containerd, through the Container Runtime Interface.

**7. A. For one app on one server**

Kubernetes earns its operational cost with many services, many machines, or a need for self-healing; for one app on one server, Compose is simpler.

**8. C. Kubernetes: K, eight letters, then s**

It is a numeronym: the eight letters between K and s are replaced by the number.

[Back to the questions](01-why-kubernetes.md#check-your-understanding)

## Chapter 2: Architecture and the Control Loop

**1. D. The kube-apiserver**

Everything talks to the API server, and only the API server touches etcd: the hub-and-spoke shape.

**2. B. Raft consensus tolerates the loss of a minority of members**

With Raft, a majority must agree; three members survive one failure, five survive two.

**3. D. Filters and scores nodes, then binds the Pod to the best node through the API server**

The scheduler only decides placement and writes it back; the kubelet starts the containers.

**4. A. Watches: long-lived connections that stream object changes**

Components watch the API server and receive a stream of changes, which makes the system reactive.

**5. C. What the responsible controller last observed**

spec is what you asked for; status is what the controller observed. Controllers drive status toward spec.

**6. D. It reacts to the difference between desired and current state whenever it notices, so missed events cannot leave it wrong**

A controller acts on 'two Pods exist, three should', not on the one-time event 'a Pod died'.

**7. B. It watches the API server and sees Pods with no node assigned**

No component calls another; each watches the API server for the objects it cares about.

**8. C. kube-system**

kube-system holds the cluster's own components; default holds your objects if you name no namespace.

[Back to the questions](02-architecture.md#check-your-understanding)

## Chapter 3: A Lab Cluster with kind

**1. A. As Docker containers, each running a full node inside**

Each node is a container running containerd and the kubelet; Pods run as containers inside those node containers.

**2. C. A single-node cluster hides interesting parts, such as traffic crossing between nodes**

With several nodes you can see scheduling across nodes and traffic between them.

**3. D. A named combination of a cluster, a user and a default namespace**

You switch clusters by switching contexts.

**4. B. kubectl config current-context**

Running a command against the wrong cluster is a common and painful mistake.

**5. C. status**

The responsible controller fills in status; you write apiVersion, kind, metadata and spec.

**6. A. apps/v1**

Core objects use v1; others carry a group, such as apps/v1 for Deployments.

**7. A. kind load docker-image IMAGE --name lab**

The kind nodes run their own containerd and cannot see your Docker image store.

**8. C. Kubernetes treats latest as always pull, and the pull fails because the image is in no registry**

With latest, the default pull policy is Always, so the node tries a registry and fails.

[Back to the questions](03-lab-cluster.md#check-your-understanding)

## Chapter 4: Pods

**1. B. One network namespace (one IP, localhost) and any declared volumes**

Containers in a Pod share an IP and reach each other on localhost; they can also share declared volumes.

**2. C. They conflict: they share one network namespace**

One network namespace means one set of ports.

**3. A. Holding the Pod's network namespace open so the real containers can join it**

The kubelet creates it first; the app containers then join its namespace.

**4. A. It is gone; a controller creates a new Pod with a new name and IP**

Pods are disposable. Nothing should depend on a Pod's name or IP; Services provide stable addresses.

**5. C. Documents and names the port; the container is reachable on any port it listens on**

It is documentation for people and tools, not access control.

**6. B. kubectl logs --previous to see the output of the crashed attempt**

CrashLoopBackOff is a symptom: the container keeps crashing and the kubelet backs off restarts. The crashed run's logs show why.

**7. D. With label selectors**

Labels and selectors loosely couple everything in Kubernetes.

**8. B. As an init container with restartPolicy: Always**

That guarantees it starts before the app and keeps running for the Pod's whole life.

[Back to the questions](04-pods.md#check-your-understanding)

## Chapter 5: Deployments and Other Controllers

**1. A. ReplicaSets**

A Deployment manages ReplicaSets, and each ReplicaSet keeps N identical Pods running.

**2. C. To move between Pod templates safely, it runs an old and a new ReplicaSet during a rollout**

A ReplicaSet only keeps N copies of one template; the Deployment orchestrates the move from one to another.

**3. B. How far above and below the desired Pod count a rolling update may go**

They pace the rollout; both default to 25%.

**4. D. maxSurge 1, maxUnavailable 0**

Never remove a Pod before its replacement is Ready, at the cost of one extra Pod's resources.

**5. B. Old Pods are removed only when new Pods report Ready; without a probe, a broken version can roll out completely**

Without a readiness probe, Ready just means the container started.

**6. D. It scales a kept, old ReplicaSet back up**

Old ReplicaSets are kept at zero replicas. Undo changes the live object, not your file, so revert the file too.

**7. A. StatefulSet**

StatefulSets provide stable names, per-replica PVCs, ordered operations and stable DNS names.

**8. C. DaemonSet**

A DaemonSet adds a Pod automatically when a node joins; kube-proxy and CNI agents use them.

[Back to the questions](05-deployments.md#check-your-understanding)

## Chapter 6: The Network Model and CNI

**1. D. Pods can reach all other Pods, on any node, without NAT**

The receiver sees the sender's real Pod IP. Containers in a Pod share one IP.

**2. B. The CNI plugin**

Kubernetes defines the contract; CNI plugins such as kindnet, Calico or Cilium fulfil it.

**3. D. The Pod CIDR, split into per-node blocks**

Each node hands out Pod IPs from its own block; 10.96.0.0/16 is kind's Service CIDR.

**4. A. When a Pod's sandbox is created on a node**

The plugin then creates the Pod's eth0, assigns an IP through IPAM, and installs routes.

**5. C. A virtual cable: what enters one end comes out the other**

One end becomes eth0 in the Pod; the other stays in the node's root namespace.

**6. D. Only inside the host: Pod veth, host routing table, other Pod's veth**

Same-node traffic never leaves the host.

**7. B. Encapsulates them, usually in VXLAN or Geneve, inside UDP between node IPs**

The physical network only sees node IPs; the inner packet keeps its Pod addresses.

**8. C. Header overhead, which needs a slightly smaller MTU inside Pods**

Overlays run anywhere but add encapsulation headers; routed designs are leaner but need the network to cooperate.

[Back to the questions](06-network-model.md#check-your-understanding)

## Chapter 7: Services and kube-proxy

**1. A. One stable virtual IP (the ClusterIP) and port, with load spread across matching Pods**

Pods come and go; the Service's ClusterIP and port stay put.

**2. C. EndpointSlices**

The EndpointSlice controller writes them; a Pod is marked ready only when its readiness probe passes.

**3. D. None: it exists only as a match in rules kube-proxy installs on every node**

No interface has the ClusterIP and nothing answers ARP for it; packets to it are DNATed by netfilter rules.

**4. B. DNAT: rewrites the destination to one backend Pod's IP and target port**

From then on it is ordinary Pod-to-Pod traffic; conntrack reverses the translation for replies.

**5. C. Balancing is per connection, and gRPC keeps one long-lived connection**

The endpoint is chosen when a connection starts. Balancing per request needs an L7 proxy or a service mesh.

**6. A. nftables**

nftables uses verdict maps and incremental updates; iptables remains the default for compatibility, and IPVS is being phased out.

**7. A. The same port, from 30000 to 32767, opened on every node**

Traffic to any node's IP on that port reaches the Service. LoadBalancer builds on top of NodePort.

**8. C. It preserves the client IP, but load is only as even as Pod placement**

With Cluster (the default), nodes forward to Pods elsewhere and SNAT the source, hiding the client IP.

[Back to the questions](07-services.md#check-your-understanding)

## Chapter 8: Cluster DNS and Service Discovery

**1. B. CoreDNS, running as a Deployment in kube-system**

CoreDNS sits behind a Service still called kube-dns, at 10.96.0.10 in a kind cluster.

**2. C. To the kube-dns Service address, then DNATed by kube-proxy rules to a CoreDNS Pod**

It follows the ordinary Service path from Chapter 7.

**3. A. web.shop.svc.cluster.local**

The pattern is service.namespace.svc.cluster.local.

**4. A. The search list in resolv.conf expands it to web.shop.svc.cluster.local**

The first search domain is the Pod's own namespace.

**5. C. No: any Pod can resolve any Service name; isolation is a job for NetworkPolicies**

web.payments resolves from any namespace.

**6. B. Several queries with cluster search suffixes fail before the name is tried as written**

Names with fewer than five dots are tried with each search domain first, multiplying DNS load.

**7. D. One A record per ready Pod**

With no virtual IP, kube-proxy ignores it; StatefulSet Pods also get stable names like db-0.db.shop.svc.cluster.local.

**8. B. ClusterFirst**

The naming is unfortunate: Default inherits the node's resolver and is not the default.

[Back to the questions](08-dns.md#check-your-understanding)

## Chapter 9: Ingress and the Gateway API

**1. A. Routing by host and path, shared TLS and one address for many Services**

A LoadBalancer works at Layer 4: an IP and a port.

**2. C. Nothing: an installed Ingress controller must watch it and configure itself**

Kubernetes ships no Ingress implementation.

**3. B. Features beyond host and path were bolted on with controller-specific annotations**

Header matching, traffic splitting, timeouts and rewrites were all annotations.

**4. D. Supported and stable, but frozen: no new features**

And the community Ingress NGINX controller was retired in March 2026.

**5. B. Gateway**

GatewayClass belongs to the implementation provider; Routes belong to application teams.

**6. D. Through parentRefs**

The Gateway decides which routes it accepts.

**7. A. With weighted backendRefs, an ordinary typed field**

Weights are portable fields; moving them to 50/50 and then 0/100 completes a canary release.

**8. C. A ReferenceGrant in the target namespace**

It prevents one team from routing traffic into another team's Services without consent.

[Back to the questions](09-ingress-gateway.md#check-your-understanding)

## Chapter 10: Network Policies

**1. D. The CNI plugin**

The policy is just an API object; plugins such as Calico, Cilium or Antrea program the rules. Without support, policies are silently ignored.

**2. B. Everything**

Isolation is opt-in: a Pod becomes isolated for a direction only once some policy selects it for that direction.

**3. D. The union of what both allow**

Policies only allow; there are no deny rules, so adding a policy can only open more traffic.

**4. A. No: policies are stateful, so replies are allowed automatically**

If a connection is allowed, its return traffic is too.

**5. C. Denies all ingress to every Pod in its namespace: default deny**

It selects every Pod in the namespace and allows nothing, isolating them all.

**6. D. Both must match (AND): Pods with that label in those namespaces**

In separate elements they mean OR, which usually allows far more than intended; the YAML differs by one dash.

**7. B. DNS is blocked too, so no names resolve; allow port 53 to kube-dns**

Always allow DNS explicitly whenever you restrict egress.

**8. C. Times out, because the packets are silently dropped**

Keep short timeouts on your test probes.

[Back to the questions](10-network-policies.md#check-your-understanding)

## Chapter 11: Configuration and Storage

**1. A. Only when the Pod is replaced**

Environment variables are read once, at container start. Mounted files are refreshed in place (except with subPath).

**2. C. Put a hash of the configuration in a Pod template annotation, so a change triggers a rolling update**

Changing the template starts a normal, gradual, reversible rollout.

**3. D. Base64-encoded, which anyone can reverse**

Real protection comes from encryption at rest, tight RBAC and external secret managers.

**4. B. Encryption at rest in etcd plus tight RBAC**

Anyone who can get Secrets, or create Pods that mount them, can read them; grant both sparingly.

**5. C. emptyDir**

emptyDir is scratch space for the life of the Pod, shareable by its containers.

**6. A. A namespaced request for storage: size, access mode and optionally a StorageClass**

The control plane binds each claim to a matching PersistentVolume, often created on demand by a StorageClass.

**7. A. ReadWriteOnce**

ReadWriteOnce: read-write from one node at a time. RWX needs shared storage such as NFS or CephFS.

**8. C. The dynamically created volume, and its data, is destroyed**

Use Retain for anything you cannot afford to lose by accident.

[Back to the questions](11-config-storage.md#check-your-understanding)

## Chapter 12: Running Workloads Well

**1. B. Requests**

A node fits a Pod only if existing requests plus the new requests stay within its allocatable capacity.

**2. C. The kernel kills it (OOMKilled, exit code 137) and it is restarted**

Exceeding the CPU limit only throttles; exceeding the memory limit kills.

**3. A. CPU throttling causes hard-to-diagnose latency spikes; without a limit, containers can use idle CPU**

A memory limit is still recommended, because running out of memory is abrupt.

**4. A. The Pod is removed from Service endpoints but not restarted**

Liveness failures restart the container; readiness failures only take the Pod out of traffic.

**5. C. A brief database blip would make the kubelet restart every Pod at once**

Keep liveness about the process itself; check dependencies in readiness, if anywhere.

**6. B. Requests**

Another reason to always set requests; the metrics come from metrics-server.

**7. D. It is purely additive: anything not granted is refused**

Roles and ClusterRoles grant verbs on resources; bindings grant them to users, groups or ServiceAccounts.

**8. B. Attach an ephemeral container with kubectl debug, sharing its namespaces**

The ephemeral container brings the tools and joins the Pod's namespaces.

[Back to the questions](12-running-well.md#check-your-understanding)

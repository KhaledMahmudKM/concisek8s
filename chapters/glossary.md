# Glossary

**Access mode.** How a PersistentVolume may be mounted: ReadWriteOnce, ReadWriteOncePod, ReadOnlyMany, or ReadWriteMany.

**Annotation.** Key-value metadata on an object, for tools and humans. Unlike labels, annotations are not used for selection.

**API server (kube-apiserver).** The front door of the control plane. Every component and every user reads and writes cluster state through it.

**BestEffort, Burstable, Guaranteed.** The three Quality of Service classes, set by a Pod's requests and limits. They decide eviction order under memory pressure.

**CNI (Container Network Interface).** The plugin standard that gives each Pod its network interface and IP address, and connects Pods across nodes.

**ClusterIP.** The default Service type: a stable virtual IP reachable only inside the cluster.

**ConfigMap.** An object holding non-secret configuration as key-value pairs, consumed as environment variables or files.

**Container runtime.** The software on each node that runs containers, such as containerd, driven by the kubelet through the CRI.

**Control plane.** The components that store and manage cluster state: API server, etcd, scheduler, and controller manager.

**Controller.** A loop that watches objects and acts to make actual state match desired state.

**CoreDNS.** The cluster DNS server. It answers names such as `web.shop.svc.cluster.local`.

**CRD (CustomResourceDefinition).** A way to add new object types to the API, used by operators and by the Gateway API.

**DaemonSet.** A workload that runs one Pod on each node, or on each selected node.

**Deployment.** A workload for stateless apps. It manages ReplicaSets to provide rolling updates and rollbacks.

**Desired state.** What you declared in a manifest. Controllers work continuously to make reality match it.

**DNAT (destination NAT).** Rewriting a packet's destination address. kube-proxy uses it to turn a Service IP into a Pod IP.

**EndpointSlice.** An object listing the ready Pod addresses behind a Service. It replaces the older Endpoints API.

**etcd.** The consistent key-value store that holds all cluster state.

**externalTrafficPolicy.** A Service setting. `Local` keeps traffic on the receiving node and preserves the client IP; `Cluster` may forward to other nodes.

**Gateway API.** The successor to Ingress: Gateway, GatewayClass, and route objects such as HTTPRoute, with roles split between platform and application teams.

**Headless Service.** A Service with `clusterIP: None`. DNS returns Pod IPs directly instead of a virtual IP.

**HorizontalPodAutoscaler (HPA).** A controller that changes a workload's replica count to keep a metric near a target.

**Ingress.** The original API for HTTP routing into a cluster. It is frozen; new work uses the Gateway API.

**Init container.** A container that runs to completion before the app containers in a Pod start.

**Job, CronJob.** Workloads that run Pods to completion, once or on a schedule.

**kind (Kubernetes in Docker).** A tool that runs Kubernetes nodes as Docker containers. The lab cluster in this book.

**kube-proxy.** The node agent that programs Service forwarding rules, using iptables, nftables, or IPVS.

**kubeconfig.** The client file that lists clusters, users, and contexts for kubectl.

**kubectl.** The command-line client for the Kubernetes API.

**kubelet.** The node agent that starts and monitors the Pods assigned to its node.

**Label, selector.** Labels are key-value tags on objects; selectors pick objects by label. Services, Deployments, and policies all use them.

**Limit.** The maximum CPU or memory a container may use. CPU above it is throttled; memory above it is OOM-killed.

**LoadBalancer.** A Service type that requests an external load balancer, provided by a cloud or by cloud-provider-kind in the lab.

**Manifest.** A YAML or JSON file describing one or more objects.

**Namespace.** A scope for names, policies, quotas, and RBAC within a cluster.

**NetworkPolicy.** Rules that allow traffic to or from selected Pods. Once a Pod is selected, anything not allowed is denied.

**Node.** A machine, real or virtual, that runs Pods.

**NodePort.** A Service type that opens the same port on every node.

**OOMKilled.** The status of a container killed for exceeding its memory limit.

**Operator.** A controller plus custom resources that automate running a specific application.

**Overlay network.** A Pod network that tunnels packets between nodes, for example with VXLAN.

**PersistentVolume (PV), PersistentVolumeClaim (PVC).** A PV is a piece of storage; a PVC is a request for storage that binds to a PV.

**Pod.** The smallest deployable unit: one or more containers sharing a network namespace and volumes.

**Pod Security Admission.** The built-in check that enforces the privileged, baseline, or restricted profile per namespace.

**Probe.** A health check run by the kubelet. Liveness restarts, readiness removes from traffic, startup gates the others.

**RBAC.** Role-based access control: Roles and ClusterRoles list permissions; bindings grant them to subjects.

**Reconciliation.** One pass of a controller comparing desired and actual state and acting on the difference.

**ReplicaSet.** Keeps a set number of identical Pods running. Usually managed by a Deployment.

**Request.** The CPU and memory a container reserves. The scheduler places Pods by requests.

**Rolling update.** Replacing Pods gradually, bounded by maxSurge and maxUnavailable.

**Scheduler (kube-scheduler).** The control plane component that assigns Pods to nodes.

**Secret.** An object for sensitive values. It is base64-encoded, not encrypted, unless encryption at rest is configured.

**Service.** A stable name and virtual IP in front of a changing set of Pods.

**ServiceAccount.** The identity a Pod uses to call the Kubernetes API.

**Sidecar.** A helper container that runs alongside the app in the same Pod. Native sidecars are init containers with `restartPolicy: Always`.

**StatefulSet.** A workload giving each Pod a stable name, DNS identity, and its own volume.

**StorageClass.** Names a provisioner and its settings for creating PersistentVolumes on demand.

**Taint, toleration.** A taint repels Pods from a node unless they carry a matching toleration.

**Volume.** Storage mounted into a Pod's containers, with a lifetime set by its type.

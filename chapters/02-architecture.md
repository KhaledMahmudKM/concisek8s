# Chapter 2: Architecture and the Control Loop

A Kubernetes cluster is a set of machines, called **nodes**, split into two roles. The **control plane** stores the desired state and makes decisions. The **worker nodes** run your containers. This chapter names every component, explains what each one does, and then follows a single command through all of them.

## The big picture

<figure class="diagram">
<svg viewBox="0 0 720 380" style="width:100%;height:auto;max-width:720px" role="img" aria-label="The control plane runs etcd, the API server, the scheduler, and the controller manager. Worker nodes run the kubelet, kube-proxy, a container runtime, and Pods. kubectl and every component talk only to the API server." xmlns="http://www.w3.org/2000/svg">
<style>
.k2a{ font-family:var(--sans),sans-serif; }
.k2a .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k2a .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k2a .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k2a .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k2a .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k2a .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k2a .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k2a .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k2a .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k2a .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k2a .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k2a .s{ fill:var(--text-dim); font-size:13px; }
.k2a .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k2a .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k2a .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k2a .c{ fill:var(--text-faint); font-size:12.5px; }
.k2a .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k2a .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k2a .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k2a-a); }
.k2a .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k2a-a); marker-start:url(#k2a-s); }
.k2a .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k2a-g); }
.k2a .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k2a-a); }
.k2a .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k2a .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k2a-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k2a-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k2a-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k2a">
<rect x="20" y="36" width="680" height="112" rx="7" class="zone"/>
<text x="36" y="58" class="b">Control plane</text>
<path d="M601 78 C 601 52, 300 52, 300 76" class="ln"/>
<line x1="166" y1="104" x2="180" y2="104" class="ln"/>
<line x1="340" y1="104" x2="354" y2="104" class="ln"/>
<rect x="36" y="78" width="130" height="52" rx="7" class="pnl"/>
<text x="101.0" y="99.5" text-anchor="middle" class="b">etcd</text>
<text x="101.0" y="117.5" text-anchor="middle" class="c">state store</text>
<rect x="180" y="78" width="160" height="52" rx="7" class="acc"/>
<text x="260.0" y="99.5" text-anchor="middle" class="b">kube-apiserver</text>
<text x="260.0" y="117.5" text-anchor="middle" class="s">the front door</text>
<rect x="354" y="78" width="148" height="52" rx="7" class="pnl"/>
<text x="428.0" y="99.5" text-anchor="middle" class="b">kube-scheduler</text>
<text x="428.0" y="117.5" text-anchor="middle" class="c">picks a node</text>
<rect x="516" y="78" width="170" height="52" rx="7" class="pnl"/>
<text x="601.0" y="99.5" text-anchor="middle" class="b">controller-manager</text>
<text x="601.0" y="117.5" text-anchor="middle" class="c">runs control loops</text>
<path d="M76 222 L76 170 L226 170 L226 132" class="ar"/>
<text x="90" y="162" class="c">REST over HTTPS</text>
<rect x="20" y="222" width="112" height="48" rx="7" class="box"/>
<text x="76.0" y="241.5" text-anchor="middle" class="b">kubectl</text>
<text x="76.0" y="259.5" text-anchor="middle" class="c">you</text>
<line x1="246" y1="190" x2="256" y2="134" class="ag"/>
<line x1="520" y1="190" x2="322" y2="134" class="ag"/>
<text x="474" y="162" class="c">kubelets watch the API server</text>
<rect x="150" y="190" width="265" height="172" rx="7" class="zone"/>
<text x="166" y="212" class="b">Worker node 1</text>
<rect x="166" y="222" width="112" height="34" rx="7" class="acc"/>
<text x="222.0" y="243.5" text-anchor="middle" class="b">kubelet</text>
<rect x="288" y="222" width="112" height="34" rx="7" class="pnl"/>
<text x="344.0" y="243.5" text-anchor="middle" class="s">kube-proxy</text>
<rect x="166" y="264" width="234" height="30" rx="7" class="box"/>
<text x="283.0" y="283.5" text-anchor="middle" class="s">container runtime (containerd)</text>
<rect x="166" y="306" width="70" height="34" rx="7" class="hov"/>
<text x="201.0" y="327.5" text-anchor="middle" class="s">Pod</text>
<rect x="248" y="306" width="70" height="34" rx="7" class="hov"/>
<text x="283.0" y="327.5" text-anchor="middle" class="s">Pod</text>
<rect x="330" y="306" width="70" height="34" rx="7" class="hov"/>
<text x="365.0" y="327.5" text-anchor="middle" class="s">Pod</text>
<rect x="435" y="190" width="265" height="172" rx="7" class="zone"/>
<text x="451" y="212" class="b">Worker node 2</text>
<rect x="451" y="222" width="112" height="34" rx="7" class="acc"/>
<text x="507.0" y="243.5" text-anchor="middle" class="b">kubelet</text>
<rect x="573" y="222" width="112" height="34" rx="7" class="pnl"/>
<text x="629.0" y="243.5" text-anchor="middle" class="s">kube-proxy</text>
<rect x="451" y="264" width="234" height="30" rx="7" class="box"/>
<text x="568.0" y="283.5" text-anchor="middle" class="s">container runtime (containerd)</text>
<rect x="451" y="306" width="70" height="34" rx="7" class="hov"/>
<text x="486.0" y="327.5" text-anchor="middle" class="s">Pod</text>
<rect x="533" y="306" width="70" height="34" rx="7" class="hov"/>
<text x="568.0" y="327.5" text-anchor="middle" class="s">Pod</text>
<rect x="615" y="306" width="70" height="34" rx="7" class="hov"/>
<text x="650.0" y="327.5" text-anchor="middle" class="s">Pod</text>
</g>
</svg>
<figcaption>A Kubernetes cluster. The control plane decides; the nodes run the work. Every component, and you, talk only to the API server.</figcaption>
</figure>

The single most useful fact in this picture is the **hub-and-spoke shape**. Nothing talks to anything else directly. You talk to the API server. The scheduler talks to the API server. Every kubelet talks to the API server. The API server is the only component that talks to etcd. Once that clicks, the rest of the architecture becomes easy to reason about.

## The control plane

**etcd** is a distributed key-value store that holds the entire state of the cluster: every object you create, and the latest status reported for each one. It uses the Raft consensus algorithm, so production clusters run three or five etcd members and tolerate the loss of a minority. If etcd's data is lost, the cluster has forgotten everything, which is why etcd backups matter more than almost anything else in operations.

**kube-apiserver** is the front door. It exposes the Kubernetes API over HTTPS, authenticates and authorizes every request, validates objects, and persists them to etcd. It also lets clients **watch** objects: instead of polling, a component opens a long-lived connection and receives a stream of changes. Watches are what make the whole system reactive.

**kube-scheduler** watches for Pods that have no node assigned. For each one, it filters out nodes that cannot run it (not enough free CPU or memory, wrong labels, taints the Pod does not tolerate), scores the rest, and **binds** the Pod to the best node by writing that choice back through the API server. It does not start anything itself.

**kube-controller-manager** runs dozens of **controllers** in one process. Each controller owns one kind of object and runs a reconcile loop for it. The ReplicaSet controller keeps the right number of Pods. The Node controller notices nodes that stop reporting. The EndpointSlice controller tracks which Pods back each Service. And so on.

**cloud-controller-manager** (in cloud clusters) holds the controllers that talk to the cloud provider's API: creating load balancers for Services, attaching disks, and learning about node machines. Keeping it separate lets Kubernetes itself stay cloud-neutral.

## The worker nodes

**kubelet** is the agent on every node. It watches the API server for Pods bound to its node, asks the container runtime to start their containers, mounts their volumes, runs their health checks, and reports their status back. If the kubelet stops, the node stops being managed.

**The container runtime** actually runs containers. Kubernetes talks to it through the **Container Runtime Interface (CRI)**. The usual runtime is **containerd**, the same engine that sits under Docker, with CRI-O as a common alternative. Any OCI image works, including the ones you build with `docker build`.

**kube-proxy** programs each node's packet-filtering rules so that traffic sent to a Service's virtual IP reaches a real Pod. It is central to Part III, and Chapter 7 dissects it. Some network plugins replace kube-proxy entirely with their own data plane.

**The CNI plugin** (not drawn above) gives every Pod its network interface and IP address. Chapter 6 is devoted to it.

## The control loop

Every controller, and in a sense the whole cluster, runs the same loop.

<figure class="diagram">
<svg viewBox="0 0 720 250" style="width:100%;height:auto;max-width:720px" role="img" aria-label="The reconcile loop: observe the actual state, compare it with the desired state, act to remove the difference, then observe again." xmlns="http://www.w3.org/2000/svg">
<style>
.k2b{ font-family:var(--sans),sans-serif; }
.k2b .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k2b .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k2b .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k2b .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k2b .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k2b .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k2b .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k2b .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k2b .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k2b .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k2b .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k2b .s{ fill:var(--text-dim); font-size:13px; }
.k2b .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k2b .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k2b .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k2b .c{ fill:var(--text-faint); font-size:12.5px; }
.k2b .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k2b .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k2b .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k2b-a); }
.k2b .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k2b-a); marker-start:url(#k2b-s); }
.k2b .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k2b-g); }
.k2b .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k2b-a); }
.k2b .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k2b .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k2b-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k2b-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k2b-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k2b">
<path d="M460 48 C 560 48, 580 90, 580 138" class="ar"/>
<path d="M478 164 L242 164" class="ar"/>
<path d="M140 140 C 140 90, 160 48, 258 48" class="ar"/>
<rect x="260" y="24" width="200" height="48" rx="7" class="pnl"/>
<text x="360.0" y="43.5" text-anchor="middle" class="b">Observe</text>
<text x="360.0" y="61.5" text-anchor="middle" class="c">read the actual state</text>
<rect x="480" y="140" width="200" height="48" rx="7" class="pnl"/>
<text x="580.0" y="159.5" text-anchor="middle" class="b">Compare</text>
<text x="580.0" y="177.5" text-anchor="middle" class="c">diff against desired</text>
<rect x="40" y="140" width="200" height="48" rx="7" class="acc"/>
<text x="140.0" y="159.5" text-anchor="middle" class="b">Act</text>
<text x="140.0" y="177.5" text-anchor="middle" class="s">create, delete, update</text>
<text x="360" y="110" text-anchor="middle" class="b">the reconcile loop</text>
<text x="360" y="128" text-anchor="middle" class="c">runs forever, in every controller</text>
<text x="250" y="220" text-anchor="middle" class="ma">spec = what you want</text>
<text x="480" y="220" text-anchor="middle" class="ma">status = what is true now</text>
</g>
</svg>
<figcaption>The control loop. Every controller in Kubernetes repeats observe, compare, act, so the cluster keeps converging on the state you declared.</figcaption>
</figure>

Every Kubernetes object carries both halves of that comparison. The **`spec`** is what you asked for. The **`status`** is what the responsible controller last observed. A controller's entire job is to drive `status` toward `spec`.

Three properties fall out of this design, and they explain a lot of Kubernetes behavior:

- **Level-triggered, not edge-triggered.** A controller does not react to "a Pod died" as a one-time event. It reacts to "there are two Pods and there should be three", whenever and however it notices. Missed events cannot leave the system permanently wrong.
- **Eventually consistent.** After you change the spec, there is a short window where reality has not caught up. That is normal, not an error.
- **Composable.** Controllers stack. A Deployment controller manages ReplicaSets, ReplicaSet controllers manage Pods, and the kubelet manages containers. Each one only understands its own level.

## Following one command

Here is what happens, step by step, when you run `kubectl apply -f web.yaml` for a Deployment with three replicas.

<figure class="diagram">
<svg viewBox="0 0 720 380" style="width:100%;height:auto;max-width:720px" role="img" aria-label="Seven steps from kubectl apply to running containers: the API server stores the Deployment, controllers create a ReplicaSet and Pods, the scheduler binds them, and kubelets start them." xmlns="http://www.w3.org/2000/svg">
<style>
.k2c{ font-family:var(--sans),sans-serif; }
.k2c .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k2c .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k2c .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k2c .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k2c .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k2c .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k2c .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k2c .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k2c .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k2c .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k2c .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k2c .s{ fill:var(--text-dim); font-size:13px; }
.k2c .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k2c .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k2c .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k2c .c{ fill:var(--text-faint); font-size:12.5px; }
.k2c .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k2c .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k2c .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k2c-a); }
.k2c .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k2c-a); marker-start:url(#k2c-s); }
.k2c .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k2c-g); }
.k2c .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k2c-a); }
.k2c .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k2c .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k2c-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k2c-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k2c-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k2c">
<rect x="30" y="30" width="130" height="44" rx="7" class="box"/>
<text x="95.0" y="56.5" text-anchor="middle" class="s">kubectl</text>
<rect x="290" y="30" width="160" height="44" rx="7" class="dark"/>
<text x="370.0" y="56.5" text-anchor="middle" class="on">API server</text>
<rect x="560" y="30" width="130" height="44" rx="7" class="pnl"/>
<text x="625.0" y="56.5" text-anchor="middle" class="s">etcd</text>
<line x1="160" y1="52" x2="288" y2="52" class="ar"/>
<text x="224" y="44" text-anchor="middle" class="a">1</text>
<line x1="450" y1="52" x2="558" y2="52" class="ar"/>
<text x="504" y="44" text-anchor="middle" class="a">2</text>
<line x1="105.0" y1="148" x2="310" y2="76" class="ar2"/>
<text x="89.0" y="140" text-anchor="end" class="a">3</text>
<rect x="20" y="150" width="170" height="46" rx="7" class="pnl"/>
<text x="105.0" y="177.5" text-anchor="middle" class="s">Deployment controller</text>
<line x1="295.0" y1="148" x2="350" y2="76" class="ar2"/>
<text x="279.0" y="140" text-anchor="end" class="a">4</text>
<rect x="210" y="150" width="170" height="46" rx="7" class="pnl"/>
<text x="295.0" y="177.5" text-anchor="middle" class="s">ReplicaSet controller</text>
<line x1="465.0" y1="148" x2="390" y2="76" class="ar2"/>
<text x="481.0" y="140" class="a">5</text>
<rect x="400" y="150" width="130" height="46" rx="7" class="pnl"/>
<text x="465.0" y="177.5" text-anchor="middle" class="s">scheduler</text>
<line x1="625.0" y1="148" x2="430" y2="76" class="ar2"/>
<text x="641.0" y="140" class="a">6</text>
<rect x="550" y="150" width="150" height="46" rx="7" class="pnl"/>
<text x="625.0" y="177.5" text-anchor="middle" class="s">kubelet (node)</text>
<rect x="550" y="250" width="150" height="44" rx="7" class="acc"/>
<text x="625.0" y="276.5" text-anchor="middle" class="b">Pods running</text>
<line x1="625" y1="196" x2="625" y2="248" class="ar"/>
<text x="639" y="228" class="a">7</text>
<text x="20" y="232" class="s">1  kubectl sends the Deployment to the API server</text>
<text x="20" y="252" class="s">2  the API server validates it and stores it in etcd</text>
<text x="20" y="272" class="s">3  the Deployment controller creates a ReplicaSet</text>
<text x="20" y="292" class="s">4  the ReplicaSet controller creates 3 Pod objects</text>
<text x="20" y="312" class="s">5  the scheduler binds each Pod to a node</text>
<text x="20" y="332" class="s">6  each kubelet notices Pods bound to its node</text>
<text x="20" y="352" class="s">7  the kubelet starts the containers, reports status</text>
</g>
</svg>
<figcaption>What happens on kubectl apply. No component calls another directly: each one watches the API server and reacts to changes there.</figcaption>
</figure>

Notice that no step involves one component calling another. The Deployment controller never tells the scheduler anything. It writes a ReplicaSet to the API server, the ReplicaSet controller sees it there, and so on. Each component watches for the objects it cares about and writes its results back. That loose coupling is why you can replace or restart any one component without the others noticing.

You can watch this chain yourself. In the lab from the next chapter, run `kubectl get events --watch` in one terminal and apply a Deployment in another. You will see the scheduling, image pulls, and container starts appear in order.

## Namespaces

A **namespace** is a scope for names inside one cluster. Two teams can each have a Deployment called `web` as long as they are in different namespaces. Many policies, including access control and resource quotas, attach to namespaces, so they are also the usual boundary between teams or environments.

A fresh cluster starts with a few:

| Namespace | Holds |
|---|---|
| `default` | Your objects, if you do not name a namespace |
| `kube-system` | Cluster components: CoreDNS, kube-proxy, the CNI plugin |
| `kube-public` | Data readable by everyone, rarely used |
| `kube-node-lease` | Node heartbeats, used to detect dead nodes quickly |

Not everything is namespaced. Nodes, PersistentVolumes, and StorageClasses belong to the whole cluster.

## Summary

- The **control plane** (etcd, API server, scheduler, controller manager) stores desired state and makes decisions; **worker nodes** (kubelet, runtime, kube-proxy, CNI) run Pods.
- Everything talks through the **API server**, and only the API server touches etcd.
- Every object has a **spec** (desired) and a **status** (observed). Controllers run reconcile loops that close the gap, forever.
- A single `kubectl apply` ripples through several independent controllers, each reacting to what the previous one wrote.

Next, you will build a cluster of your own and run these commands for real.

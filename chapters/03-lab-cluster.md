# Chapter 3: A Lab Cluster with kind

You learn Kubernetes by using it. This chapter builds a real multi-node cluster on your own machine with **kind** ("Kubernetes in Docker"), installs `kubectl`, and introduces the manifests you will write for the rest of the book.

## How kind works

kind runs each Kubernetes node as a **Docker container**. Inside each node container runs a full node: containerd, the kubelet, and on the control plane node, the API server, etcd, the scheduler, and the controller manager. Your Pods then run as containers inside those node containers.

<figure class="diagram">
<svg viewBox="0 0 720 330" style="width:100%;height:auto;max-width:720px" role="img" aria-label="kind runs each Kubernetes node as a Docker container on a Docker network called kind. kubectl on your machine reaches the API server through a port published on 127.0.0.1." xmlns="http://www.w3.org/2000/svg">
<style>
.k3a{ font-family:var(--sans),sans-serif; }
.k3a .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k3a .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k3a .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k3a .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k3a .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k3a .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k3a .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k3a .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k3a .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k3a .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k3a .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k3a .s{ fill:var(--text-dim); font-size:13px; }
.k3a .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k3a .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k3a .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k3a .c{ fill:var(--text-faint); font-size:12.5px; }
.k3a .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k3a .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k3a .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k3a-a); }
.k3a .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k3a-a); marker-start:url(#k3a-s); }
.k3a .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k3a-g); }
.k3a .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k3a-a); }
.k3a .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k3a .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k3a-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k3a-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k3a-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k3a">
<rect x="20" y="36" width="680" height="280" rx="7" class="zone"/>
<text x="36" y="58" class="b">Your machine (Docker host)</text>
<rect x="36" y="80" width="150" height="46" rx="7" class="box"/>
<text x="111.0" y="98.5" text-anchor="middle" class="b">kubectl</text>
<text x="111.0" y="116.5" text-anchor="middle" class="ma">~/.kube/config</text>
<text x="36" y="160" class="ma">127.0.0.1:port</text>
<text x="36" y="178" class="c">maps to :6443</text>
<rect x="220" y="76" width="466" height="226" rx="7" class="dash"/>
<text x="236" y="98" class="a">Docker network: kind</text>
<rect x="236" y="112" width="140" height="176" rx="7" class="pnl"/>
<text x="306" y="134" text-anchor="middle" class="b">control plane</text>
<text x="306" y="152" text-anchor="middle" class="c">node container</text>
<rect x="246" y="164" width="120" height="30" rx="7" class="acc"/>
<text x="306.0" y="183.5" text-anchor="middle" class="b">API server</text>
<rect x="246" y="202" width="120" height="28" rx="7" class="box"/>
<text x="306.0" y="220.5" text-anchor="middle" class="s">containerd</text>
<rect x="246" y="240" width="56" height="26" rx="7" class="hov"/>
<text x="274.0" y="257.5" text-anchor="middle" class="s">pod</text>
<rect x="310" y="240" width="56" height="26" rx="7" class="hov"/>
<text x="338.0" y="257.5" text-anchor="middle" class="s">pod</text>
<rect x="386" y="112" width="140" height="176" rx="7" class="pnl"/>
<text x="456" y="134" text-anchor="middle" class="b">worker</text>
<text x="456" y="152" text-anchor="middle" class="c">node container</text>
<rect x="396" y="164" width="120" height="30" rx="7" class="box"/>
<text x="456.0" y="183.5" text-anchor="middle" class="s">kubelet</text>
<rect x="396" y="202" width="120" height="28" rx="7" class="box"/>
<text x="456.0" y="220.5" text-anchor="middle" class="s">containerd</text>
<rect x="396" y="240" width="56" height="26" rx="7" class="hov"/>
<text x="424.0" y="257.5" text-anchor="middle" class="s">pod</text>
<rect x="460" y="240" width="56" height="26" rx="7" class="hov"/>
<text x="488.0" y="257.5" text-anchor="middle" class="s">pod</text>
<rect x="536" y="112" width="140" height="176" rx="7" class="pnl"/>
<text x="606" y="134" text-anchor="middle" class="b">worker 2</text>
<text x="606" y="152" text-anchor="middle" class="c">node container</text>
<rect x="546" y="164" width="120" height="30" rx="7" class="box"/>
<text x="606.0" y="183.5" text-anchor="middle" class="s">kubelet</text>
<rect x="546" y="202" width="120" height="28" rx="7" class="box"/>
<text x="606.0" y="220.5" text-anchor="middle" class="s">containerd</text>
<rect x="546" y="240" width="56" height="26" rx="7" class="hov"/>
<text x="574.0" y="257.5" text-anchor="middle" class="s">pod</text>
<rect x="610" y="240" width="56" height="26" rx="7" class="hov"/>
<text x="638.0" y="257.5" text-anchor="middle" class="s">pod</text>
<path d="M186 103 C 212 103, 214 179, 244 179" class="ar"/>
</g>
</svg>
<figcaption>A kind cluster. Each node is a container, and Pods run inside those containers. Your kubectl talks to the API server through a published port.</figcaption>
</figure>

This is why kind is a good fit for this book. It needs nothing but Docker, it starts a multi-node cluster in about a minute, and because each node is a container on a Docker network, everything you learned about Docker networking still applies when you go looking for packets in Part III.

## Install the tools

You need Docker running, plus two command-line tools.

**kubectl** is the Kubernetes client. Install it from the official instructions for your OS at kubernetes.io, or with a package manager:

```bash
# macOS
brew install kubectl
# Linux (any distribution with snap)
sudo snap install kubectl --classic
kubectl version --client
```

**kind** is a single binary:

```bash
# macOS
brew install kind
# Linux, or anywhere Go is installed
go install sigs.k8s.io/kind@latest
kind version
```

## Create a multi-node cluster

A single-node cluster hides the interesting parts, such as traffic crossing between nodes. Describe a cluster with one control plane node and two workers in a small config file:

```yaml
# kind-lab.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: lab
nodes:
- role: control-plane
- role: worker
- role: worker
```

Then create it:

```bash
kind create cluster --config kind-lab.yaml
kubectl get nodes -o wide
```

You should see three nodes, `lab-control-plane`, `lab-worker`, and `lab-worker2`, each with an internal IP on the Docker network named `kind`. Confirm they really are containers:

```bash
docker ps --format '{{.Names}}\t{{.Image}}'
```

kind configures `kubectl` for you, so commands work right away. When you are done for the day, `kind delete cluster --name lab` removes everything.

> **Pinning a version.** kind picks a default Kubernetes version for its release. To choose one, add `image: kindest/node:<version>` under each node, using an image listed in the release notes of your kind version.

## kubeconfig and contexts

`kubectl` finds clusters through a **kubeconfig** file, by default `~/.kube/config`.

<figure class="diagram">
<svg viewBox="0 0 720 230" style="width:100%;height:auto;max-width:720px" role="img" aria-label="A kubeconfig file lists clusters and users; each context pairs one cluster with one user and a default namespace; current-context picks which context kubectl uses." xmlns="http://www.w3.org/2000/svg">
<style>
.k3b{ font-family:var(--sans),sans-serif; }
.k3b .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k3b .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k3b .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k3b .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k3b .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k3b .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k3b .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k3b .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k3b .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k3b .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k3b .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k3b .s{ fill:var(--text-dim); font-size:13px; }
.k3b .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k3b .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k3b .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k3b .c{ fill:var(--text-faint); font-size:12.5px; }
.k3b .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k3b .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k3b .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k3b-a); }
.k3b .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k3b-a); marker-start:url(#k3b-s); }
.k3b .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k3b-g); }
.k3b .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k3b-a); }
.k3b .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k3b .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k3b-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k3b-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k3b-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k3b">
<rect x="20" y="24" width="190" height="80" rx="7" class="pnl"/>
<text x="115.0" y="50.5" text-anchor="middle" class="b">clusters</text>
<text x="115.0" y="68.5" text-anchor="middle" class="ma">kind-kind</text>
<text x="115.0" y="86.5" text-anchor="middle" class="m">prod-eu</text>
<rect x="20" y="124" width="190" height="80" rx="7" class="pnl"/>
<text x="115.0" y="150.5" text-anchor="middle" class="b">users</text>
<text x="115.0" y="168.5" text-anchor="middle" class="ma">kind-kind</text>
<text x="115.0" y="186.5" text-anchor="middle" class="m">alice@prod</text>
<rect x="270" y="54" width="210" height="120" rx="7" class="acc"/>
<text x="375.0" y="91.5" text-anchor="middle" class="b">contexts</text>
<text x="375.0" y="109.5" text-anchor="middle" class="m">kind-kind =</text>
<text x="375.0" y="127.5" text-anchor="middle" class="s">cluster + user</text>
<text x="375.0" y="145.5" text-anchor="middle" class="s">+ namespace</text>
<rect x="540" y="90" width="160" height="48" rx="7" class="dark"/>
<text x="620.0" y="118.5" text-anchor="middle" class="on">current-context</text>
<text x="620" y="160" text-anchor="middle" class="ma">kind-kind</text>
<line x1="210" y1="64" x2="268" y2="100" class="ar"/>
<line x1="210" y1="164" x2="268" y2="130" class="ar"/>
<line x1="480" y1="114" x2="538" y2="114" class="ar"/>
</g>
</svg>
<figcaption>Inside a kubeconfig. kind adds a cluster, a user, and a context named kind-kind, and makes it the current context.</figcaption>
</figure>

A **context** is a named combination of a cluster, a user, and a default namespace. You switch clusters by switching contexts, which matters once you have more than one:

```bash
kubectl config get-contexts
kubectl config use-context kind-lab
kubectl config set-context --current --namespace=demo   # change default namespace
```

A common and painful mistake is running a command against the wrong cluster. Get into the habit of checking `kubectl config current-context` before anything destructive.

## Manifests

You can create objects with imperative commands such as `kubectl create deployment`, and they are handy for quick experiments. Real work uses **manifests**: YAML files that describe objects declaratively, live in version control, and are applied with `kubectl apply`.

<figure class="diagram">
<svg viewBox="0 0 720 280" style="width:100%;height:auto;max-width:720px" role="img" aria-label="An annotated manifest: apiVersion and kind name the object type, metadata gives its identity, spec is the desired state you write, and status is the actual state Kubernetes writes back." xmlns="http://www.w3.org/2000/svg">
<style>
.k3c{ font-family:var(--sans),sans-serif; }
.k3c .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k3c .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k3c .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k3c .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k3c .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k3c .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k3c .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k3c .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k3c .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k3c .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k3c .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k3c .s{ fill:var(--text-dim); font-size:13px; }
.k3c .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k3c .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k3c .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k3c .c{ fill:var(--text-faint); font-size:12.5px; }
.k3c .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k3c .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k3c .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k3c-a); }
.k3c .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k3c-a); marker-start:url(#k3c-s); }
.k3c .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k3c-g); }
.k3c .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k3c-a); }
.k3c .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k3c .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k3c-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k3c-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k3c-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k3c">
<rect x="20" y="20" width="330" height="246" rx="7" class="pnl"/>
<text x="40" y="48" class="m">apiVersion: apps/v1</text>
<text x="40" y="70" class="m">kind: Deployment</text>
<text x="40" y="92" class="m">metadata:</text>
<text x="60" y="114" class="m">name: web</text>
<text x="60" y="136" class="m">labels: {app: web}</text>
<text x="40" y="158" class="m">spec:</text>
<text x="60" y="180" class="m">replicas: 3</text>
<text x="60" y="202" class="m">template: ...</text>
<text x="40" y="224" class="m" style="fill:var(--text-faint)">status:</text>
<text x="60" y="246" class="m" style="fill:var(--text-faint)">readyReplicas: 3</text>
<line x1="366" y1="34" x2="366" y2="75" class="ln"/>
<line x1="360" y1="34" x2="366" y2="34" class="ln"/>
<line x1="360" y1="75" x2="366" y2="75" class="ln"/>
<text x="382" y="59.0" class="s">which API and object type</text>
<line x1="366" y1="78" x2="366" y2="141" class="ln"/>
<line x1="360" y1="78" x2="366" y2="78" class="ln"/>
<line x1="360" y1="141" x2="366" y2="141" class="ln"/>
<text x="382" y="114.0" class="s">identity: name, namespace, labels</text>
<line x1="366" y1="144" x2="366" y2="207" class="ln"/>
<line x1="360" y1="144" x2="366" y2="144" class="ln"/>
<line x1="360" y1="207" x2="366" y2="207" class="ln"/>
<text x="382" y="180.0" class="s">spec: the desired state you write</text>
<line x1="366" y1="210" x2="366" y2="251" class="ln"/>
<line x1="360" y1="210" x2="366" y2="210" class="ln"/>
<line x1="360" y1="251" x2="366" y2="251" class="ln"/>
<text x="382" y="235.0" class="s">status: written back by Kubernetes</text>
</g>
</svg>
<figcaption>The anatomy of a manifest. You write the top three parts; the cluster fills in status as it works.</figcaption>
</figure>

Every manifest has the same four top-level fields:

- **`apiVersion`** and **`kind`** say which type of object this is. Core objects use `v1`; others carry a group, such as `apps/v1` for Deployments.
- **`metadata`** identifies it: a `name`, usually a `namespace`, and `labels` (Chapter 4 explains why labels matter so much).
- **`spec`** is the desired state. Its fields depend entirely on the kind.
- **`status`** is never written by you. The responsible controller fills it in.

## Your first workload

Create this file and apply it:

```yaml
# hello.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - name: web
        image: nginx:1.27
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f hello.yaml
kubectl get pods -o wide        # three Pods, spread across the workers
kubectl delete pod <one-pod-name>
kubectl get pods                # a replacement appears within seconds
```

That last pair of commands is Chapter 2 in action: you deleted a Pod, the ReplicaSet controller saw two where it wanted three, and it created a new one.

## Using your own images

Images you build locally live in your Docker image store, but the kind nodes run their own containerd and cannot see it. Load the image into the cluster explicitly:

```bash
docker build -t myapp:0.1 .
kind load docker-image myapp:0.1 --name lab
```

Then reference `myapp:0.1` in your manifest. Avoid the `latest` tag for local images: Kubernetes treats `latest` as "always pull", and the pull fails because the image is not in any registry.

## Useful habits

A few `kubectl` habits pay off immediately:

```bash
kubectl explain deployment.spec.strategy   # built-in documentation for any field
kubectl get pods -A                        # all namespaces
kubectl get deploy hello -o yaml           # the full object, including status
kubectl diff -f hello.yaml                 # what apply would change
kubectl apply -f hello.yaml --dry-run=server
```

`kubectl explain` deserves special mention. It answers "what fields does this object have?" straight from the API server, for exactly the version you are running.

## Summary

- **kind** runs Kubernetes nodes as Docker containers; a three-node lab takes one config file and one command.
- `kubectl` uses **kubeconfig** contexts to pick a cluster, user, and namespace. Check the current context before risky commands.
- **Manifests** declare objects with `apiVersion`, `kind`, `metadata`, and `spec`; the cluster writes `status`.
- Load local images with `kind load docker-image`, and avoid `latest`.

With a cluster in hand, the next chapter looks closely at the unit everything else is built on: the Pod.

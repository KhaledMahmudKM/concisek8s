# Chapter 4: Pods

Kubernetes never schedules a bare container. Its smallest unit is the **Pod**: one or more containers that are always placed together on one node and share a network identity and storage. Understanding Pods, and labels, is the foundation for everything that follows.

## What a Pod is

<figure class="diagram">
<svg viewBox="0 0 720 270" style="width:100%;height:auto;max-width:720px" role="img" aria-label="A Pod wraps an app container and a sidecar container. Both share one network namespace, so one IP address and localhost, and they can share volumes." xmlns="http://www.w3.org/2000/svg">
<style>
.k4a{ font-family:var(--sans),sans-serif; }
.k4a .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k4a .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k4a .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k4a .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k4a .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k4a .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k4a .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k4a .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k4a .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k4a .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k4a .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k4a .s{ fill:var(--text-dim); font-size:13px; }
.k4a .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k4a .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k4a .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k4a .c{ fill:var(--text-faint); font-size:12.5px; }
.k4a .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k4a .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k4a .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k4a-a); }
.k4a .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k4a-a); marker-start:url(#k4a-s); }
.k4a .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k4a-g); }
.k4a .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k4a-a); }
.k4a .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k4a .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k4a-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k4a-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k4a-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k4a">
<rect x="150" y="30" width="420" height="222" rx="7" class="out"/>
<text x="170" y="56" class="h">Pod</text>
<text x="550" y="56" text-anchor="end" class="ma">IP 10.244.1.7</text>
<rect x="180" y="76" width="160" height="60" rx="7" class="acc"/>
<text x="260.0" y="101.5" text-anchor="middle" class="b">app container</text>
<text x="260.0" y="119.5" text-anchor="middle" class="s">listens on :8080</text>
<rect x="380" y="76" width="160" height="60" rx="7" class="acc"/>
<text x="460.0" y="101.5" text-anchor="middle" class="b">log shipper</text>
<text x="460.0" y="119.5" text-anchor="middle" class="s">sidecar container</text>
<rect x="180" y="152" width="360" height="36" rx="7" class="pnl"/>
<text x="360.0" y="174.5" text-anchor="middle" class="s">shared network namespace: one IP, localhost</text>
<rect x="180" y="198" width="360" height="36" rx="7" class="box"/>
<text x="360.0" y="220.5" text-anchor="middle" class="s">shared volume: /var/log/app</text>
</g>
</svg>
<figcaption>A Pod is one or more containers that share a network identity and storage. Kubernetes schedules, starts, and stops them together.</figcaption>
</figure>

The containers in a Pod share:

- **One network namespace.** The Pod has a single IP address, every container in it sees the same network interfaces, and they reach each other over `localhost`. Two containers in one Pod cannot both listen on port 8080.
- **Volumes**, when declared. A volume mounted into two containers lets them exchange files, as with an app writing logs that a sidecar ships elsewhere.
- **A lifecycle.** They are scheduled together, start on the same node, and are deleted together.

Under the hood, the kubelet first creates a tiny **pause container** that does nothing but hold the network namespace open. The real containers then join that namespace. You will see these if you list containers on a node in Chapter 6.

Most Pods run exactly one container. Put two containers in one Pod only when they are tightly coupled and must share a network or files, such as an app and its log shipper or a local proxy. Two services that merely talk to each other belong in separate Pods.

## A Pod manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  labels:
    app: web
spec:
  containers:
  - name: web
    image: nginx:1.27
    ports:
    - containerPort: 80
```

```bash
kubectl apply -f pod.yaml
kubectl get pod web -o wide          # node and Pod IP
kubectl logs web
kubectl exec -it web -- sh           # a shell inside the container
kubectl port-forward pod/web 8080:80 # then open http://localhost:8080
```

`containerPort` is documentation, not a firewall. The container is reachable on any port it listens on; declaring the port just names it and makes it visible to tools.

## Pods are disposable

A Pod is **not** restarted somewhere else when its node dies. It is gone, and a controller creates a brand new Pod with a new name and a new IP. This is the single most important habit to build: treat Pods as cattle, never as pets.

That is why you rarely create Pods directly, as in the manifest above. You create a controller, usually a Deployment, which creates and replaces Pods for you. It is also why nothing should ever depend on a Pod's IP address. Services, in Chapter 7, exist to provide the stable address that Pods cannot.

## The Pod lifecycle

<figure class="diagram">
<svg viewBox="0 0 720 214" style="width:100%;height:auto;max-width:720px" role="img" aria-label="Pod phases: Pending, then Running, then Succeeded or Failed. Containers inside a Running Pod can restart according to restartPolicy; repeated crashes lead to CrashLoopBackOff." xmlns="http://www.w3.org/2000/svg">
<style>
.k4b{ font-family:var(--sans),sans-serif; }
.k4b .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k4b .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k4b .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k4b .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k4b .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k4b .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k4b .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k4b .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k4b .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k4b .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k4b .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k4b .s{ fill:var(--text-dim); font-size:13px; }
.k4b .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k4b .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k4b .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k4b .c{ fill:var(--text-faint); font-size:12.5px; }
.k4b .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k4b .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k4b .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k4b-a); }
.k4b .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k4b-a); marker-start:url(#k4b-s); }
.k4b .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k4b-g); }
.k4b .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k4b-a); }
.k4b .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k4b .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k4b-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k4b-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k4b-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k4b">
<path d="M245 80 C 245 44, 305 44, 305 80" class="ar"/>
<text x="275" y="26" text-anchor="middle" class="c">containers restart per restartPolicy</text>
<rect x="20" y="80" width="130" height="44" rx="7" class="pnl"/>
<text x="85.0" y="106.5" text-anchor="middle" class="s">Pending</text>
<rect x="210" y="80" width="130" height="44" rx="7" class="acc"/>
<text x="275.0" y="106.5" text-anchor="middle" class="b">Running</text>
<rect x="430" y="40" width="150" height="40" rx="7" class="pnl"/>
<text x="505.0" y="64.5" text-anchor="middle" class="s">Succeeded</text>
<rect x="430" y="124" width="150" height="40" rx="7" class="pnl"/>
<text x="505.0" y="148.5" text-anchor="middle" class="s">Failed</text>
<line x1="150" y1="102" x2="208" y2="102" class="ar"/>
<line x1="340" y1="94" x2="428" y2="64" class="ar"/>
<line x1="340" y1="110" x2="428" y2="140" class="ar"/>
<text x="85" y="150" text-anchor="middle" class="c">scheduled? image pulled?</text>
<text x="275" y="150" text-anchor="middle" class="c">a container is up</text>
<text x="505" y="194" text-anchor="middle" class="c">repeated crashes: CrashLoopBackOff</text>
</g>
</svg>
<figcaption>The Pod lifecycle. Phases describe the Pod as a whole; restarts happen inside a Running Pod without changing its phase.</figcaption>
</figure>

A Pod moves through a small set of **phases**:

- **Pending.** Accepted by the API server, but not yet running. It may be waiting for the scheduler, or the node may still be pulling the image.
- **Running.** Bound to a node, with at least one container running or restarting.
- **Succeeded** or **Failed.** All containers have terminated, successfully or not. This matters for Jobs; long-running servers should never get here.

Inside a Running Pod, individual containers can crash and be restarted by the kubelet according to the Pod's `restartPolicy` (`Always` by default). Restarts back off exponentially, up to five minutes between attempts, and `kubectl get pods` shows the infamous **`CrashLoopBackOff`** status while it waits. That status is a symptom, not a cause; `kubectl logs --previous` shows the output of the crashed attempt.

## Labels and selectors

Labels are the glue of Kubernetes. A **label** is a key-value pair attached to any object, such as `app=web` or `env=prod`. A **selector** is a query over labels.

<figure class="diagram">
<svg viewBox="0 0 720 220" style="width:100%;height:auto;max-width:720px" role="img" aria-label="A selector app=web picks out the three Pods labelled app=web and ignores the Pods labelled app=api and app=db." xmlns="http://www.w3.org/2000/svg">
<style>
.k4c{ font-family:var(--sans),sans-serif; }
.k4c .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k4c .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k4c .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k4c .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k4c .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k4c .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k4c .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k4c .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k4c .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k4c .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k4c .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k4c .s{ fill:var(--text-dim); font-size:13px; }
.k4c .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k4c .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k4c .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k4c .c{ fill:var(--text-faint); font-size:12.5px; }
.k4c .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k4c .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k4c .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k4c-a); }
.k4c .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k4c-a); marker-start:url(#k4c-s); }
.k4c .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k4c-g); }
.k4c .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k4c-a); }
.k4c .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k4c .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k4c-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k4c-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k4c-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k4c">
<rect x="250" y="20" width="220" height="44" rx="7" class="dark"/>
<text x="360.0" y="46.5" text-anchor="middle" class="on">selector: app=web</text>
<line x1="360" y1="64" x2="80" y2="148" class="ar"/>
<line x1="360" y1="64" x2="220" y2="148" class="ar"/>
<line x1="360" y1="64" x2="500" y2="148" class="ar"/>
<rect x="20" y="150" width="120" height="52" rx="7" class="acc"/>
<text x="80.0" y="171.5" text-anchor="middle" class="b">Pod</text>
<text x="80.0" y="189.5" text-anchor="middle" class="m">app=web</text>
<rect x="160" y="150" width="120" height="52" rx="7" class="acc"/>
<text x="220.0" y="171.5" text-anchor="middle" class="b">Pod</text>
<text x="220.0" y="189.5" text-anchor="middle" class="m">app=web</text>
<rect x="300" y="150" width="120" height="52" rx="7" class="box"/>
<text x="360.0" y="171.5" text-anchor="middle" class="b">Pod</text>
<text x="360.0" y="189.5" text-anchor="middle" class="m">app=api</text>
<rect x="440" y="150" width="120" height="52" rx="7" class="acc"/>
<text x="500.0" y="171.5" text-anchor="middle" class="b">Pod</text>
<text x="500.0" y="189.5" text-anchor="middle" class="m">app=web</text>
<rect x="580" y="150" width="120" height="52" rx="7" class="box"/>
<text x="640.0" y="171.5" text-anchor="middle" class="b">Pod</text>
<text x="640.0" y="189.5" text-anchor="middle" class="m">app=db</text>
</g>
</svg>
<figcaption>Labels and selectors. Labels are free-form key=value tags; a selector is a query over them. This is how Kubernetes objects find each other.</figcaption>
</figure>

Controllers find their Pods with selectors. Services find their backends with selectors. NetworkPolicies choose which Pods they protect with selectors. Nothing in Kubernetes is linked by name or ID when a label query will do, which is what lets the pieces stay loosely coupled.

```bash
kubectl get pods -l app=web
kubectl get pods -l 'env in (prod,staging)'
kubectl label pod web tier=frontend
kubectl get pods --show-labels
```

A good habit is to label everything consistently from the start, for example with `app`, `component`, and `env`. Kubernetes also recommends a standard set under the `app.kubernetes.io/` prefix, such as `app.kubernetes.io/name`.

**Annotations** look similar but serve a different purpose. They hold non-identifying metadata, such as a build commit or tool configuration, and cannot be used in selectors.

## Init containers and sidecars

Some Pods need setup before the app starts, or a helper that runs beside it.

<figure class="diagram">
<svg viewBox="0 0 720 200" style="width:100%;height:auto;max-width:720px" role="img" aria-label="Init containers run one after another to completion; only then do the app container and its sidecar start and run alongside each other." xmlns="http://www.w3.org/2000/svg">
<style>
.k4d{ font-family:var(--sans),sans-serif; }
.k4d .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k4d .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k4d .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k4d .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k4d .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k4d .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k4d .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k4d .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k4d .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k4d .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k4d .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k4d .s{ fill:var(--text-dim); font-size:13px; }
.k4d .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k4d .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k4d .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k4d .c{ fill:var(--text-faint); font-size:12.5px; }
.k4d .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k4d .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k4d .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k4d-a); }
.k4d .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k4d-a); marker-start:url(#k4d-s); }
.k4d .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k4d-g); }
.k4d .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k4d-a); }
.k4d .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k4d .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k4d-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k4d-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k4d-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k4d">
<rect x="20" y="60" width="150" height="48" rx="7" class="pnl"/>
<text x="95.0" y="79.5" text-anchor="middle" class="b">init 1</text>
<text x="95.0" y="97.5" text-anchor="middle" class="c">wait for the db</text>
<rect x="200" y="60" width="150" height="48" rx="7" class="pnl"/>
<text x="275.0" y="79.5" text-anchor="middle" class="b">init 2</text>
<text x="275.0" y="97.5" text-anchor="middle" class="c">run migrations</text>
<rect x="390" y="40" width="310" height="90" rx="7" class="dash"/>
<rect x="406" y="60" width="136" height="48" rx="7" class="acc"/>
<text x="474.0" y="79.5" text-anchor="middle" class="b">app</text>
<text x="474.0" y="97.5" text-anchor="middle" class="s">main process</text>
<rect x="552" y="60" width="136" height="48" rx="7" class="acc"/>
<text x="620.0" y="79.5" text-anchor="middle" class="b">sidecar</text>
<text x="620.0" y="97.5" text-anchor="middle" class="s">runs alongside</text>
<line x1="170" y1="84" x2="198" y2="84" class="ar"/>
<line x1="350" y1="84" x2="388" y2="84" class="ar"/>
<text x="185" y="136" text-anchor="middle" class="c">each must succeed, in order</text>
<line x1="20" y1="160" x2="690" y2="160" class="ag"/>
<text x="700" y="182" text-anchor="end" class="c">time</text>
</g>
</svg>
<figcaption>Init containers prepare the Pod, one at a time, before the main containers start. A sidecar starts with the app and runs for the Pod's whole life.</figcaption>
</figure>

**Init containers** run before the main containers, one at a time, and each must exit successfully before the next starts. They are ideal for waiting on a dependency, running a database migration, or fetching configuration.

**Sidecars** run alongside the main container for the Pod's whole life. Kubernetes models them as init containers with `restartPolicy: Always`, which guarantees they start before the app and keep running. This native sidecar support has been stable since Kubernetes 1.33.

```yaml
spec:
  initContainers:
  - name: wait-for-db
    image: busybox:1.36
    command: ['sh', '-c', 'until nc -z db 5432; do sleep 2; done']
  - name: log-shipper              # a native sidecar
    image: fluent/fluent-bit:3.0
    restartPolicy: Always
  containers:
  - name: app
    image: myapp:0.1
```

## Summary

- A **Pod** is one or more containers sharing a network namespace (one IP, `localhost`) and optionally volumes. It is the unit Kubernetes schedules.
- Pods are **disposable**: when they die they are replaced with new Pods, new names, and new IPs. Manage them through controllers.
- Phases are Pending, Running, Succeeded, and Failed; crashing containers inside a Running Pod show as `CrashLoopBackOff`.
- **Labels and selectors** connect everything in Kubernetes. Label consistently.
- **Init containers** prepare a Pod; **sidecars** run beside the app for its whole life.

The next chapter introduces the controllers that create and replace Pods for you.

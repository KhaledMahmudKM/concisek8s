# Chapter 12: Running Workloads Well

A manifest that runs is not yet a manifest that runs **well**. This chapter covers the settings that separate the two: resource requests and limits, health probes, autoscaling, access control, and basic Pod security. It ends with a troubleshooting method that uses everything in the book.

## Requests and limits

Each container can declare how much CPU and memory it needs. CPU is measured in cores, often written in millicores (`500m` is half a core). Memory is measured in bytes (`256Mi`, `1Gi`).

```yaml
resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    memory: 512Mi
```

The two fields do different jobs.

<figure class="diagram">
<svg viewBox="0 0 720 238" style="width:100%;height:auto;max-width:720px" role="img" aria-label="The scheduler packs Pods onto a node by their requests: here three Pods requesting 1, 1.5, and 1 CPU fit on a node with 4 allocatable CPUs. At runtime, limits are enforced: CPU above its limit is throttled, memory above its limit is OOM-killed." xmlns="http://www.w3.org/2000/svg">
<style>
.k12a{ font-family:var(--sans),sans-serif; }
.k12a .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k12a .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k12a .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k12a .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k12a .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k12a .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k12a .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k12a .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k12a .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k12a .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k12a .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k12a .s{ fill:var(--text-dim); font-size:13px; }
.k12a .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k12a .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k12a .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k12a .c{ fill:var(--text-faint); font-size:12.5px; }
.k12a .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k12a .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k12a .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k12a-a); }
.k12a .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k12a-a); marker-start:url(#k12a-s); }
.k12a .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k12a-g); }
.k12a .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k12a-a); }
.k12a .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k12a .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k12a-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k12a-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k12a-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k12a">
<text x="20" y="24" class="b">Scheduling uses requests</text>
<rect x="20" y="36" width="680" height="46" rx="7" class="zone"/>
<rect x="20" y="36" width="170" height="46" rx="7" class="acc"/>
<text x="105.0" y="63.5" text-anchor="middle" class="b">Pod A: 1 CPU</text>
<rect x="190" y="36" width="255" height="46" rx="7" class="acc"/>
<text x="317.5" y="63.5" text-anchor="middle" class="b">Pod B: 1.5 CPU</text>
<rect x="445" y="36" width="170" height="46" rx="7" class="acc"/>
<text x="530.0" y="63.5" text-anchor="middle" class="b">Pod C: 1 CPU</text>
<text x="658" y="64" text-anchor="middle" class="c">free</text>
<text x="360" y="104" text-anchor="middle" class="c">node allocatable: 4 CPU</text>
<text x="20" y="142" class="b">Limits are enforced at runtime</text>
<rect x="20" y="156" width="330" height="66" rx="7" class="pnl"/>
<text x="185.0" y="184.5" text-anchor="middle" class="b">CPU above its limit</text>
<text x="185.0" y="202.5" text-anchor="middle" class="s">throttled: the container slows down</text>
<rect x="370" y="156" width="330" height="66" rx="7" class="pnl"/>
<text x="535.0" y="184.5" text-anchor="middle" class="b">memory above its limit</text>
<text x="535.0" y="202.5" text-anchor="middle" class="s">OOMKilled: the container restarts</text>
</g>
</svg>
<figcaption>Requests and limits. Requests decide placement; limits cap usage. CPU is compressible and gets throttled, memory is not and gets killed.</figcaption>
</figure>

- **Requests** are what the **scheduler** uses. A node fits a Pod only if the sum of the requests already placed there, plus the new Pod's requests, stays within the node's allocatable capacity. Requests are a reservation, whether or not the container uses them.
- **Limits** are enforced by the kernel through **cgroups** (the same mechanism Docker uses). A container that tries to use more **CPU** than its limit is **throttled**: it keeps running, more slowly. A container that uses more **memory** than its limit is **killed** by the kernel's OOM killer and restarted, with the reason `OOMKilled` and exit code 137.

Some practical guidance:

- **Always set requests.** Without them the scheduler is guessing, and nodes become overcommitted.
- **Set a memory limit,** usually equal to or a little above the request, because running out of memory is abrupt.
- **Be careful with CPU limits.** Throttling causes latency spikes that are hard to diagnose. Many teams set CPU requests but no CPU limit, so containers can use idle CPU.

Requests and limits also put each Pod in a **Quality of Service class**, which decides who is evicted first when a node runs short of memory.

<figure class="diagram">
<svg viewBox="0 0 720 182" style="width:100%;height:auto;max-width:720px" role="img" aria-label="Three QoS classes ordered by eviction priority under node memory pressure: BestEffort Pods are evicted first, then Burstable, and Guaranteed Pods last." xmlns="http://www.w3.org/2000/svg">
<style>
.k12b{ font-family:var(--sans),sans-serif; }
.k12b .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k12b .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k12b .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k12b .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k12b .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k12b .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k12b .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k12b .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k12b .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k12b .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k12b .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k12b .s{ fill:var(--text-dim); font-size:13px; }
.k12b .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k12b .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k12b .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k12b .c{ fill:var(--text-faint); font-size:12.5px; }
.k12b .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k12b .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k12b .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k12b-a); }
.k12b .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k12b-a); marker-start:url(#k12b-s); }
.k12b .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k12b-g); }
.k12b .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k12b-a); }
.k12b .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k12b .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k12b-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k12b-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k12b-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k12b">
<rect x="20" y="50" width="200" height="64" rx="7" class="box"/>
<text x="120.0" y="77.5" text-anchor="middle" class="b">BestEffort</text>
<text x="120.0" y="95.5" text-anchor="middle" class="c">no requests or limits</text>
<rect x="260" y="50" width="200" height="64" rx="7" class="pnl"/>
<text x="360.0" y="77.5" text-anchor="middle" class="b">Burstable</text>
<text x="360.0" y="95.5" text-anchor="middle" class="c">requests below limits</text>
<line x1="222" y1="82" x2="258" y2="82" class="ag"/>
<rect x="500" y="50" width="200" height="64" rx="7" class="acc"/>
<text x="600.0" y="77.5" text-anchor="middle" class="b">Guaranteed</text>
<text x="600.0" y="95.5" text-anchor="middle" class="s">requests equal limits</text>
<line x1="462" y1="82" x2="498" y2="82" class="ag"/>
<text x="120" y="32" text-anchor="middle" class="a">evicted first</text>
<text x="600" y="32" text-anchor="middle" class="c">evicted last</text>
<text x="360" y="160" text-anchor="middle" class="c">under node memory pressure, the kubelet evicts from left to right</text>
</g>
</svg>
<figcaption>Quality of Service classes. Setting requests and limits deliberately decides which Pods survive when a node runs short of memory.</figcaption>
</figure>

A **LimitRange** sets default requests and limits for a namespace, and a **ResourceQuota** caps the total a namespace may request. Together they stop one team from consuming a shared cluster.

## Health probes

The kubelet can check each container's health in three ways, and each probe has a different consequence.

<figure class="diagram">
<svg viewBox="0 0 720 240" style="width:100%;height:auto;max-width:720px" role="img" aria-label="A startupProbe gates the other probes until the app has started. After that, a failing livenessProbe restarts the container, and a failing readinessProbe removes the Pod from Service endpoints without restarting it." xmlns="http://www.w3.org/2000/svg">
<style>
.k12c{ font-family:var(--sans),sans-serif; }
.k12c .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k12c .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k12c .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k12c .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k12c .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k12c .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k12c .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k12c .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k12c .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k12c .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k12c .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k12c .s{ fill:var(--text-dim); font-size:13px; }
.k12c .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k12c .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k12c .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k12c .c{ fill:var(--text-faint); font-size:12.5px; }
.k12c .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k12c .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k12c .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k12c-a); }
.k12c .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k12c-a); marker-start:url(#k12c-s); }
.k12c .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k12c-g); }
.k12c .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k12c-a); }
.k12c .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k12c .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k12c-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k12c-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k12c-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k12c">
<rect x="20" y="82" width="190" height="64" rx="7" class="pnl"/>
<text x="115.0" y="109.5" text-anchor="middle" class="m">startupProbe</text>
<text x="115.0" y="127.5" text-anchor="middle" class="c">until the app has started</text>
<rect x="290" y="30" width="200" height="64" rx="7" class="acc"/>
<text x="390.0" y="57.5" text-anchor="middle" class="m">livenessProbe</text>
<text x="390.0" y="75.5" text-anchor="middle" class="s">is it stuck?</text>
<rect x="290" y="134" width="200" height="64" rx="7" class="acc"/>
<text x="390.0" y="161.5" text-anchor="middle" class="m">readinessProbe</text>
<text x="390.0" y="179.5" text-anchor="middle" class="s">can it serve now?</text>
<rect x="540" y="30" width="160" height="64" rx="7" class="box"/>
<text x="620.0" y="57.5" text-anchor="middle" class="c">on failure</text>
<text x="620.0" y="75.5" text-anchor="middle" class="b">restart container</text>
<rect x="540" y="134" width="160" height="64" rx="7" class="box"/>
<text x="620.0" y="161.5" text-anchor="middle" class="c">on failure</text>
<text x="620.0" y="179.5" text-anchor="middle" class="b">remove from endpoints</text>
<line x1="210" y1="104" x2="288" y2="66" class="ar"/>
<line x1="210" y1="124" x2="288" y2="160" class="ar"/>
<line x1="490" y1="62" x2="538" y2="62" class="ar"/>
<line x1="490" y1="166" x2="538" y2="166" class="ar"/>
<text x="360" y="228" text-anchor="middle" class="c">liveness and readiness start only after the startup probe succeeds</text>
</g>
</svg>
<figcaption>Three probes, three different reactions. Mixing up liveness and readiness is one of the most common causes of cascading outages.</figcaption>
</figure>

- A **liveness probe** answers "is this process stuck?" If it fails repeatedly, the kubelet **restarts the container**.
- A **readiness probe** answers "can it serve traffic right now?" If it fails, the Pod is marked not ready and **removed from Service endpoints** (Chapter 7), but not restarted. Rolling updates also wait for readiness (Chapter 5).
- A **startup probe** protects slow starters. Until it succeeds, the other two probes are not run, so a long startup is not mistaken for a hang.

```yaml
readinessProbe:
  httpGet: {path: /ready, port: 8080}
  periodSeconds: 5
  failureThreshold: 3
livenessProbe:
  httpGet: {path: /healthz, port: 8080}
  periodSeconds: 10
  failureThreshold: 3
startupProbe:
  httpGet: {path: /healthz, port: 8080}
  periodSeconds: 5
  failureThreshold: 30      # allow up to 150 s to start
```

Probes can also run a command (`exec`), open a TCP connection (`tcpSocket`), or call a gRPC health service (`grpc`).

One rule prevents a whole class of outages: **a liveness probe must never depend on anything outside the container**. If every Pod's liveness check also tests the database, a brief database blip makes the kubelet restart every Pod at once, turning a small problem into a full outage. Check dependencies in readiness, if anywhere, and keep liveness about the process itself.

## Autoscaling

The **HorizontalPodAutoscaler** (HPA) adjusts a Deployment's replica count to keep a metric near a target.

<figure class="diagram">
<svg viewBox="0 0 720 270" style="width:100%;height:auto;max-width:720px" role="img" aria-label="The HorizontalPodAutoscaler loop: metrics-server collects Pod CPU usage, the HPA controller compares it with the target and computes a desired replica count, and updates the Deployment, which adds or removes Pods." xmlns="http://www.w3.org/2000/svg">
<style>
.k12d{ font-family:var(--sans),sans-serif; }
.k12d .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k12d .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k12d .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k12d .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k12d .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k12d .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k12d .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k12d .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k12d .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k12d .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k12d .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k12d .s{ fill:var(--text-dim); font-size:13px; }
.k12d .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k12d .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k12d .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k12d .c{ fill:var(--text-faint); font-size:12.5px; }
.k12d .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k12d .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k12d .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k12d-a); }
.k12d .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k12d-a); marker-start:url(#k12d-s); }
.k12d .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k12d-g); }
.k12d .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k12d-a); }
.k12d .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k12d .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k12d-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k12d-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k12d-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k12d">
<rect x="20" y="100" width="140" height="50" rx="7" class="acc"/>
<text x="90.0" y="129.5" text-anchor="middle" class="b">Pods</text>
<rect x="220" y="24" width="170" height="54" rx="7" class="pnl"/>
<text x="305.0" y="46.5" text-anchor="middle" class="b">metrics-server</text>
<text x="305.0" y="64.5" text-anchor="middle" class="c">collects CPU, memory</text>
<rect x="450" y="96" width="180" height="58" rx="7" class="dark"/>
<text x="540.0" y="120.5" text-anchor="middle" class="on">HPA controller</text>
<text x="540.0" y="138.5" text-anchor="middle" class="on">target: 70% CPU</text>
<rect x="220" y="172" width="170" height="54" rx="7" class="pnl"/>
<text x="305.0" y="194.5" text-anchor="middle" class="b">Deployment</text>
<text x="305.0" y="212.5" text-anchor="middle" class="m">spec.replicas</text>
<path d="M90 100 C 90 60, 150 51, 218 51" class="ar"/>
<path d="M390 51 C 470 51, 540 60, 540 94" class="ar"/>
<path d="M540 154 C 540 190, 470 199, 392 199" class="ar"/>
<path d="M218 199 C 150 199, 90 190, 90 152" class="ar"/>
<text x="360" y="254" text-anchor="middle" class="ma">desired = ceil(current replicas x current usage / target)</text>
</g>
</svg>
<figcaption>The autoscaling loop is just another controller: observe a metric, compare with a target, change a replica count.</figcaption>
</figure>

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web
spec:
  scaleTargetRef: {apiVersion: apps/v1, kind: Deployment, name: web}
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target: {type: Utilization, averageUtilization: 70}
```

Utilization is measured against **requests**, which is one more reason to set them. Resource metrics come from **metrics-server**, which kind does not install. Add it, allowing the self-signed kubelet certificates kind uses:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl -n kube-system patch deployment metrics-server --type=json \
  -p '[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
kubectl top pods
```

The HPA can also scale on custom and external metrics, such as requests per second or queue length, through adapters like the Prometheus Adapter or KEDA. Two related tools work at other levels: the **Vertical Pod Autoscaler** recommends or adjusts requests, and **node autoscalers** such as Cluster Autoscaler or Karpenter add and remove nodes when Pods cannot be scheduled.

## Access control with RBAC

Every request to the API server is authenticated, then authorized. Authorization almost always uses **RBAC** (role-based access control).

<figure class="diagram">
<svg viewBox="0 0 720 236" style="width:100%;height:auto;max-width:720px" role="img" aria-label="A RoleBinding in namespace shop connects subjects, such as a ServiceAccount, a user, and a group, to a Role whose rules list allowed verbs on resources." xmlns="http://www.w3.org/2000/svg">
<style>
.k12e{ font-family:var(--sans),sans-serif; }
.k12e .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k12e .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k12e .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k12e .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k12e .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k12e .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k12e .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k12e .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k12e .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k12e .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k12e .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k12e .s{ fill:var(--text-dim); font-size:13px; }
.k12e .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k12e .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k12e .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k12e .c{ fill:var(--text-faint); font-size:12.5px; }
.k12e .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k12e .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k12e .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k12e-a); }
.k12e .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k12e-a); marker-start:url(#k12e-s); }
.k12e .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k12e-g); }
.k12e .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k12e-a); }
.k12e .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k12e .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k12e-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k12e-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k12e-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k12e">
<rect x="20" y="24" width="180" height="48" rx="7" class="box"/>
<text x="110.0" y="43.5" text-anchor="middle" class="b">ServiceAccount</text>
<text x="110.0" y="61.5" text-anchor="middle" class="m">ci-bot</text>
<line x1="200" y1="48" x2="268" y2="110" class="ar"/>
<rect x="20" y="84" width="180" height="48" rx="7" class="box"/>
<text x="110.0" y="103.5" text-anchor="middle" class="b">User</text>
<text x="110.0" y="121.5" text-anchor="middle" class="m">alice</text>
<line x1="200" y1="108" x2="268" y2="110" class="ar"/>
<rect x="20" y="144" width="180" height="48" rx="7" class="box"/>
<text x="110.0" y="163.5" text-anchor="middle" class="b">Group</text>
<text x="110.0" y="181.5" text-anchor="middle" class="m">devs</text>
<line x1="200" y1="168" x2="268" y2="110" class="ar"/>
<rect x="270" y="82" width="170" height="56" rx="7" class="acc"/>
<text x="355.0" y="105.5" text-anchor="middle" class="b">RoleBinding</text>
<text x="355.0" y="123.5" text-anchor="middle" class="s">namespace shop</text>
<rect x="500" y="62" width="200" height="96" rx="7" class="pnl"/>
<text x="600.0" y="96.5" text-anchor="middle" class="b">Role deployer</text>
<text x="600.0" y="114.5" text-anchor="middle" class="m">get, list: pods</text>
<text x="600.0" y="132.5" text-anchor="middle" class="m">update: deployments</text>
<line x1="440" y1="110" x2="498" y2="110" class="ar"/>
<text x="470" y="100" text-anchor="middle" class="c">roleRef</text>
<text x="360" y="218" text-anchor="middle" class="c">ClusterRole and ClusterRoleBinding work the same way, cluster-wide</text>
</g>
</svg>
<figcaption>RBAC in one picture. A Role says what can be done; a binding says who can do it, and where.</figcaption>
</figure>

- A **Role** lists allowed **verbs** (`get`, `list`, `watch`, `create`, `update`, `delete`, and so on) on **resources** within one namespace. A **ClusterRole** does the same cluster-wide or for cluster-scoped resources.
- A **RoleBinding** grants a Role to **subjects**: users, groups, or **ServiceAccounts**. A **ClusterRoleBinding** grants a ClusterRole across the whole cluster.
- RBAC is purely additive. There are no deny rules; anything not granted is refused.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: {name: deployer, namespace: shop}
rules:
- apiGroups: [""]
  resources: [pods]
  verbs: [get, list, watch]
- apiGroups: [apps]
  resources: [deployments]
  verbs: [get, update, patch]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: {name: ci-bot-deployer, namespace: shop}
subjects:
- kind: ServiceAccount
  name: ci-bot
  namespace: shop
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: deployer
```

**ServiceAccounts** are the identities that Pods and automation use. Every namespace has one named `default`, and each Pod receives a short-lived, automatically rotated token for its ServiceAccount. Give each application its own ServiceAccount with only the permissions it needs, and set `automountServiceAccountToken: false` for Pods that never talk to the API.

Check what an identity can do with `kubectl auth can-i`:

```bash
kubectl auth can-i update deployments -n shop --as=system:serviceaccount:shop:ci-bot
kubectl auth can-i --list -n shop --as=system:serviceaccount:shop:ci-bot
```

## Pod security

By default, containers run as whatever user the image specifies, often root, with the default Linux capabilities. Tighten that in each container's `securityContext`:

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: [ALL]
  seccompProfile: {type: RuntimeDefault}
```

To enforce such settings across a namespace, use **Pod Security Admission**, built into the API server. It applies one of three standard profiles, `privileged`, `baseline`, or `restricted`, according to a namespace label:

```bash
kubectl label namespace shop pod-security.kubernetes.io/enforce=restricted
```

Pods that violate the profile are rejected at creation time. Start with `warn` or `audit` mode to find violations without breaking anything, then switch to `enforce`.

## A troubleshooting method

When something does not work, start from what Kubernetes already knows and work down through the layers in this book.

<figure class="diagram">
<svg viewBox="0 0 720 276" style="width:100%;height:auto;max-width:720px" role="img" aria-label="A troubleshooting map from Pod status to the next check: Pending to events and resources, ImagePullBackOff to image name and pull secret, CrashLoopBackOff to previous logs, Running but not Ready to the readiness probe, and Ready but unreachable to endpoints, selectors, policies, and DNS." xmlns="http://www.w3.org/2000/svg">
<style>
.k12f{ font-family:var(--sans),sans-serif; }
.k12f .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k12f .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k12f .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k12f .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k12f .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k12f .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k12f .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k12f .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k12f .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k12f .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k12f .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k12f .s{ fill:var(--text-dim); font-size:13px; }
.k12f .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k12f .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k12f .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k12f .c{ fill:var(--text-faint); font-size:12.5px; }
.k12f .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k12f .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k12f .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k12f-a); }
.k12f .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k12f-a); marker-start:url(#k12f-s); }
.k12f .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k12f-g); }
.k12f .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k12f-a); }
.k12f .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k12f .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k12f-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k12f-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k12f-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k12f">
<rect x="20" y="116" width="160" height="44" rx="7" class="dark"/>
<text x="100.0" y="142.5" text-anchor="middle" class="on">kubectl get pods</text>
<line x1="180" y1="138" x2="238" y2="40" class="ar"/>
<rect x="240" y="20" width="190" height="40" rx="7" class="pnl"/>
<text x="335.0" y="44.5" text-anchor="middle" class="s">Pending</text>
<line x1="430" y1="40" x2="448" y2="40" class="ar"/>
<rect x="450" y="20" width="250" height="40" rx="7" class="box"/>
<text x="575.0" y="44.5" text-anchor="middle" class="s">check events, resources, PVCs</text>
<line x1="180" y1="138" x2="238" y2="90" class="ar"/>
<rect x="240" y="70" width="190" height="40" rx="7" class="pnl"/>
<text x="335.0" y="94.5" text-anchor="middle" class="s">ImagePullBackOff</text>
<line x1="430" y1="90" x2="448" y2="90" class="ar"/>
<rect x="450" y="70" width="250" height="40" rx="7" class="box"/>
<text x="575.0" y="94.5" text-anchor="middle" class="s">image name, tag, pull secret</text>
<line x1="180" y1="138" x2="238" y2="140" class="ar"/>
<rect x="240" y="120" width="190" height="40" rx="7" class="pnl"/>
<text x="335.0" y="144.5" text-anchor="middle" class="s">CrashLoopBackOff</text>
<line x1="430" y1="140" x2="448" y2="140" class="ar"/>
<rect x="450" y="120" width="250" height="40" rx="7" class="box"/>
<text x="575.0" y="144.5" text-anchor="middle" class="s">logs --previous, exit code</text>
<line x1="180" y1="138" x2="238" y2="190" class="ar"/>
<rect x="240" y="170" width="190" height="40" rx="7" class="pnl"/>
<text x="335.0" y="194.5" text-anchor="middle" class="s">Running, not Ready</text>
<line x1="430" y1="190" x2="448" y2="190" class="ar"/>
<rect x="450" y="170" width="250" height="40" rx="7" class="box"/>
<text x="575.0" y="194.5" text-anchor="middle" class="s">readiness probe, describe</text>
<line x1="180" y1="138" x2="238" y2="240" class="ar"/>
<rect x="240" y="220" width="190" height="40" rx="7" class="pnl"/>
<text x="335.0" y="244.5" text-anchor="middle" class="s">Ready, unreachable</text>
<line x1="430" y1="240" x2="448" y2="240" class="ar"/>
<rect x="450" y="220" width="250" height="40" rx="7" class="box"/>
<text x="575.0" y="244.5" text-anchor="middle" class="s">endpoints, selector, policy, DNS</text>
</g>
</svg>
<figcaption>Start from what kubectl get pods says. Each status points to a different layer, and to a different first command.</figcaption>
</figure>

The commands behind each branch:

```bash
kubectl get pods -o wide                       # status, restarts, node, IP
kubectl describe pod <pod>                     # events: scheduling, pulls, probe failures
kubectl logs <pod> --previous                  # output of the last crashed run
kubectl get events --sort-by=.lastTimestamp    # recent cluster events
kubectl get endpointslices -l kubernetes.io/service-name=<svc>   # any ready backends?
kubectl debug -it <pod> --image=nicolaka/netshoot --target=<container>
```

The last command attaches an **ephemeral container** to a running Pod, sharing its namespaces. It is the cleanest way to debug a minimal image that has no shell or network tools.

For "the app cannot reach the service" problems, walk the networking chapters in order. Does the name resolve (Chapter 8)? Does the Service have ready endpoints (Chapter 7)? Does a NetworkPolicy block the path (Chapter 10)? Can the Pods reach each other by IP at all (Chapter 6)? The first step that fails tells you which layer to fix.

## Summary

- **Requests** drive scheduling and QoS; **limits** cap usage, with CPU throttled and memory OOM-killed. Always set requests and a memory limit.
- **Liveness** restarts, **readiness** removes from traffic, **startup** protects slow starts. Keep liveness independent of other services.
- The **HPA** scales replicas on metrics from metrics-server or custom sources.
- **RBAC** grants verbs on resources to users, groups, and ServiceAccounts, namespaced or cluster-wide. Grant the least you can.
- Harden Pods with **securityContext** and enforce standards with **Pod Security Admission**.
- Troubleshoot from **`kubectl get pods`** down through the layers, and use `describe`, `logs --previous`, events, and `kubectl debug`.

That completes the tour. The Appendix collects the kubectl commands from every chapter, and the Glossary defines every term.

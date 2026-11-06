# Chapter 5: Deployments and Other Controllers

You almost never create Pods by hand. You create a **controller** that creates Pods for you and keeps them in the shape you asked for. This chapter covers the controller you will use most, the Deployment, and then the specialized ones for stateful apps, per-node agents, and batch work.

## Deployment, ReplicaSet, Pod

A **Deployment** does not manage Pods directly. It manages **ReplicaSets**, and each ReplicaSet keeps a fixed number of identical Pods running.

<figure class="diagram">
<svg viewBox="0 0 720 250" style="width:100%;height:auto;max-width:720px" role="img" aria-label="A Deployment owns a ReplicaSet, and the ReplicaSet owns three identical Pods whose names carry the ReplicaSet&#x27;s hash." xmlns="http://www.w3.org/2000/svg">
<style>
.k5a{ font-family:var(--sans),sans-serif; }
.k5a .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k5a .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k5a .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k5a .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k5a .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k5a .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k5a .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k5a .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k5a .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k5a .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k5a .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k5a .s{ fill:var(--text-dim); font-size:13px; }
.k5a .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k5a .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k5a .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k5a .c{ fill:var(--text-faint); font-size:12.5px; }
.k5a .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k5a .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k5a .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k5a-a); }
.k5a .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k5a-a); marker-start:url(#k5a-s); }
.k5a .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k5a-g); }
.k5a .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k5a-a); }
.k5a .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k5a .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k5a-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k5a-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k5a-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k5a">
<rect x="260" y="20" width="200" height="46" rx="7" class="dark"/>
<text x="360.0" y="47.5" text-anchor="middle" class="on">Deployment: web</text>
<rect x="260" y="100" width="200" height="46" rx="7" class="acc"/>
<text x="360.0" y="127.5" text-anchor="middle" class="b">ReplicaSet web-7d4f9</text>
<line x1="360" y1="66" x2="360" y2="98" class="ar"/>
<line x1="360" y1="146" x2="130" y2="182" class="ar"/>
<line x1="360" y1="146" x2="360" y2="182" class="ar"/>
<line x1="360" y1="146" x2="590" y2="182" class="ar"/>
<rect x="40" y="184" width="180" height="46" rx="7" class="box"/>
<text x="130.0" y="211.5" text-anchor="middle" class="m">web-7d4f9-x2kq8</text>
<rect x="270" y="184" width="180" height="46" rx="7" class="box"/>
<text x="360.0" y="211.5" text-anchor="middle" class="m">web-7d4f9-m9zpl</text>
<rect x="500" y="184" width="180" height="46" rx="7" class="box"/>
<text x="590.0" y="211.5" text-anchor="middle" class="m">web-7d4f9-c4hvd</text>
<text x="480" y="48" class="c">manages versions and rollouts</text>
<text x="480" y="128" class="c">keeps N identical Pods alive</text>
</g>
</svg>
<figcaption>Deployment, ReplicaSet, Pods. You manage the Deployment; it manages ReplicaSets; they manage Pods.</figcaption>
</figure>

The split exists because of updates. A ReplicaSet only knows how to keep N copies of one Pod template alive. A Deployment knows how to move from one template to another safely, which it does by running two ReplicaSets at once during a rollout. You can see the chain with:

```bash
kubectl get deploy,rs,pods -l app=hello
```

The random suffix on each ReplicaSet name is a hash of its Pod template, so every distinct template gets its own ReplicaSet.

## Scaling

Scaling is a change to one field:

```bash
kubectl scale deployment hello --replicas=5
```

In a declarative workflow you would edit `replicas` in the manifest and re-apply it instead, so that the file in version control stays the source of truth. Chapter 12 shows how to let Kubernetes choose the replica count automatically with a HorizontalPodAutoscaler.

## Rolling updates

Change anything in the Pod template, most often the image, and the Deployment starts a **rollout**:

```bash
kubectl set image deployment/hello web=nginx:1.28
kubectl rollout status deployment/hello
```

The default strategy, `RollingUpdate`, creates a new ReplicaSet and shifts Pods over gradually. Two settings control the pace:

- **`maxSurge`**: how many Pods above the desired count may exist during the rollout.
- **`maxUnavailable`**: how many Pods below the desired count are tolerated.

Both default to 25%. Setting `maxSurge: 1` and `maxUnavailable: 0` gives the most conservative rollout: capacity never drops, at the cost of one extra Pod's worth of resources.

<figure class="diagram">
<svg viewBox="0 0 720 260" style="width:100%;height:auto;max-width:720px" role="img" aria-label="A rolling update from version 1 to version 2 with replicas 3, maxSurge 1, maxUnavailable 0, shown as seven steps; at every step at least three Pods are Ready." xmlns="http://www.w3.org/2000/svg">
<style>
.k5b{ font-family:var(--sans),sans-serif; }
.k5b .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k5b .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k5b .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k5b .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k5b .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k5b .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k5b .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k5b .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k5b .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k5b .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k5b .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k5b .s{ fill:var(--text-dim); font-size:13px; }
.k5b .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k5b .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k5b .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k5b .c{ fill:var(--text-faint); font-size:12.5px; }
.k5b .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k5b .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k5b .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k5b-a); }
.k5b .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k5b-a); marker-start:url(#k5b-s); }
.k5b .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k5b-g); }
.k5b .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k5b-a); }
.k5b .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k5b .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k5b-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k5b-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k5b-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k5b">
<text x="55" y="32" text-anchor="middle" class="c">step 0</text>
<rect x="20" y="46" width="70" height="26" rx="7" class="box"/>
<text x="55.0" y="63.5" text-anchor="middle" class="s">v1</text>
<rect x="20" y="78" width="70" height="26" rx="7" class="box"/>
<text x="55.0" y="95.5" text-anchor="middle" class="s">v1</text>
<rect x="20" y="110" width="70" height="26" rx="7" class="box"/>
<text x="55.0" y="127.5" text-anchor="middle" class="s">v1</text>
<text x="55" y="196" text-anchor="middle" class="m">3 + 0</text>
<text x="153" y="32" text-anchor="middle" class="c">step 1</text>
<rect x="118" y="46" width="70" height="26" rx="7" class="box"/>
<text x="153.0" y="63.5" text-anchor="middle" class="s">v1</text>
<rect x="118" y="78" width="70" height="26" rx="7" class="box"/>
<text x="153.0" y="95.5" text-anchor="middle" class="s">v1</text>
<rect x="118" y="110" width="70" height="26" rx="7" class="box"/>
<text x="153.0" y="127.5" text-anchor="middle" class="s">v1</text>
<rect x="118" y="142" width="70" height="26" rx="7" class="acc"/>
<text x="153.0" y="159.5" text-anchor="middle" class="b">v2</text>
<text x="153" y="196" text-anchor="middle" class="m">3 + 1</text>
<text x="251" y="32" text-anchor="middle" class="c">step 2</text>
<rect x="216" y="46" width="70" height="26" rx="7" class="box"/>
<text x="251.0" y="63.5" text-anchor="middle" class="s">v1</text>
<rect x="216" y="78" width="70" height="26" rx="7" class="box"/>
<text x="251.0" y="95.5" text-anchor="middle" class="s">v1</text>
<rect x="216" y="110" width="70" height="26" rx="7" class="acc"/>
<text x="251.0" y="127.5" text-anchor="middle" class="b">v2</text>
<text x="251" y="196" text-anchor="middle" class="m">2 + 1</text>
<text x="349" y="32" text-anchor="middle" class="c">step 3</text>
<rect x="314" y="46" width="70" height="26" rx="7" class="box"/>
<text x="349.0" y="63.5" text-anchor="middle" class="s">v1</text>
<rect x="314" y="78" width="70" height="26" rx="7" class="box"/>
<text x="349.0" y="95.5" text-anchor="middle" class="s">v1</text>
<rect x="314" y="110" width="70" height="26" rx="7" class="acc"/>
<text x="349.0" y="127.5" text-anchor="middle" class="b">v2</text>
<rect x="314" y="142" width="70" height="26" rx="7" class="acc"/>
<text x="349.0" y="159.5" text-anchor="middle" class="b">v2</text>
<text x="349" y="196" text-anchor="middle" class="m">2 + 2</text>
<text x="447" y="32" text-anchor="middle" class="c">step 4</text>
<rect x="412" y="46" width="70" height="26" rx="7" class="box"/>
<text x="447.0" y="63.5" text-anchor="middle" class="s">v1</text>
<rect x="412" y="78" width="70" height="26" rx="7" class="acc"/>
<text x="447.0" y="95.5" text-anchor="middle" class="b">v2</text>
<rect x="412" y="110" width="70" height="26" rx="7" class="acc"/>
<text x="447.0" y="127.5" text-anchor="middle" class="b">v2</text>
<text x="447" y="196" text-anchor="middle" class="m">1 + 2</text>
<text x="545" y="32" text-anchor="middle" class="c">step 5</text>
<rect x="510" y="46" width="70" height="26" rx="7" class="box"/>
<text x="545.0" y="63.5" text-anchor="middle" class="s">v1</text>
<rect x="510" y="78" width="70" height="26" rx="7" class="acc"/>
<text x="545.0" y="95.5" text-anchor="middle" class="b">v2</text>
<rect x="510" y="110" width="70" height="26" rx="7" class="acc"/>
<text x="545.0" y="127.5" text-anchor="middle" class="b">v2</text>
<rect x="510" y="142" width="70" height="26" rx="7" class="acc"/>
<text x="545.0" y="159.5" text-anchor="middle" class="b">v2</text>
<text x="545" y="196" text-anchor="middle" class="m">1 + 3</text>
<text x="643" y="32" text-anchor="middle" class="c">step 6</text>
<rect x="608" y="46" width="70" height="26" rx="7" class="acc"/>
<text x="643.0" y="63.5" text-anchor="middle" class="b">v2</text>
<rect x="608" y="78" width="70" height="26" rx="7" class="acc"/>
<text x="643.0" y="95.5" text-anchor="middle" class="b">v2</text>
<rect x="608" y="110" width="70" height="26" rx="7" class="acc"/>
<text x="643.0" y="127.5" text-anchor="middle" class="b">v2</text>
<text x="643" y="196" text-anchor="middle" class="m">0 + 3</text>
<line x1="20" y1="214" x2="690" y2="214" class="ag"/>
<text x="360" y="240" text-anchor="middle" class="c">old ReplicaSet scales down only as new Pods become Ready</text>
</g>
</svg>
<figcaption>A rolling update with maxSurge 1 and maxUnavailable 0. One extra Pod is allowed, none may be missing, so capacity never drops.</figcaption>
</figure>

The key word in that figure is **Ready**. The Deployment only scales the old ReplicaSet down when new Pods report Ready, which they do when their readiness probe passes (Chapter 12). Without a readiness probe, "Ready" just means "the container started", and a broken version can roll out completely before anyone notices.

```yaml
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

The other strategy, `Recreate`, deletes all old Pods before creating new ones. It causes downtime and is only for apps that cannot run two versions at once.

## Rollback

Every rollout leaves its previous ReplicaSet behind, scaled to zero.

<figure class="diagram">
<svg viewBox="0 0 720 220" style="width:100%;height:auto;max-width:720px" role="img" aria-label="Three ReplicaSets, one per revision. Revisions 1 and 2 are kept at zero replicas; revision 3 runs three Pods. kubectl rollout undo scales revision 2 back up." xmlns="http://www.w3.org/2000/svg">
<style>
.k5c{ font-family:var(--sans),sans-serif; }
.k5c .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k5c .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k5c .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k5c .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k5c .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k5c .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k5c .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k5c .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k5c .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k5c .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k5c .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k5c .s{ fill:var(--text-dim); font-size:13px; }
.k5c .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k5c .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k5c .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k5c .c{ fill:var(--text-faint); font-size:12.5px; }
.k5c .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k5c .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k5c .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k5c-a); }
.k5c .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k5c-a); marker-start:url(#k5c-s); }
.k5c .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k5c-g); }
.k5c .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k5c-a); }
.k5c .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k5c .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k5c-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k5c-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k5c-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k5c">
<path d="M605 98 C 605 52, 360 52, 360 96" class="ad"/>
<text x="482" y="40" text-anchor="middle" class="ma">kubectl rollout undo</text>
<rect x="20" y="100" width="190" height="60" rx="7" class="box"/>
<text x="115.0" y="125.5" text-anchor="middle" class="b">revision 1</text>
<text x="115.0" y="143.5" text-anchor="middle" class="m">web:1.0, 0 Pods</text>
<rect x="265" y="100" width="190" height="60" rx="7" class="box"/>
<text x="360.0" y="125.5" text-anchor="middle" class="b">revision 2</text>
<text x="360.0" y="143.5" text-anchor="middle" class="m">web:1.1, 0 Pods</text>
<rect x="510" y="100" width="190" height="60" rx="7" class="acc"/>
<text x="605.0" y="125.5" text-anchor="middle" class="b">revision 3 (current)</text>
<text x="605.0" y="143.5" text-anchor="middle" class="m">web:1.2, 3 Pods</text>
<text x="360" y="196" text-anchor="middle" class="c">old ReplicaSets are kept at zero, so a rollback is just a scale-up</text>
</g>
</svg>
<figcaption>Revision history. Each rollout leaves its old ReplicaSet behind at zero replicas, which is what makes rollback fast.</figcaption>
</figure>

That makes rolling back nothing more than scaling an old ReplicaSet up again:

```bash
kubectl rollout history deployment/hello
kubectl rollout undo deployment/hello                 # to the previous revision
kubectl rollout undo deployment/hello --to-revision=1
```

`revisionHistoryLimit` (default 10) sets how many old ReplicaSets are kept. Note that `rollout undo` changes the live object but not your YAML file. In a declarative workflow, also revert the file, or the next `apply` will roll forward again.

## Other workload controllers

Deployments assume every replica is interchangeable. Some workloads break that assumption, and Kubernetes has a controller for each case.

<figure class="diagram">
<svg viewBox="0 0 720 280" style="width:100%;height:auto;max-width:720px" role="img" aria-label="Four workload controllers: Deployment for interchangeable replicas, StatefulSet for stable identity and storage, DaemonSet for one Pod per node, Job and CronJob for run-to-completion work." xmlns="http://www.w3.org/2000/svg">
<style>
.k5d{ font-family:var(--sans),sans-serif; }
.k5d .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k5d .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k5d .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k5d .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k5d .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k5d .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k5d .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k5d .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k5d .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k5d .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k5d .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k5d .s{ fill:var(--text-dim); font-size:13px; }
.k5d .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k5d .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k5d .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k5d .c{ fill:var(--text-faint); font-size:12.5px; }
.k5d .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k5d .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k5d .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k5d-a); }
.k5d .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k5d-a); marker-start:url(#k5d-s); }
.k5d .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k5d-g); }
.k5d .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k5d-a); }
.k5d .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k5d .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k5d-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k5d-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k5d-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k5d">
<rect x="20" y="20" width="330" height="112" rx="7" class="pnl"/>
<text x="40" y="52" class="b">Deployment</text>
<text x="40" y="80" class="s">interchangeable replicas</text>
<text x="40" y="104" class="c">web servers, APIs (stateless)</text>
<rect x="370" y="20" width="330" height="112" rx="7" class="pnl"/>
<text x="390" y="52" class="b">StatefulSet</text>
<text x="390" y="80" class="s">stable names and own storage</text>
<text x="390" y="104" class="c">databases: db-0, db-1, db-2</text>
<rect x="20" y="150" width="330" height="112" rx="7" class="pnl"/>
<text x="40" y="182" class="b">DaemonSet</text>
<text x="40" y="210" class="s">exactly one Pod per node</text>
<text x="40" y="234" class="c">log agents, CNI, kube-proxy</text>
<rect x="370" y="150" width="330" height="112" rx="7" class="pnl"/>
<text x="390" y="182" class="b">Job / CronJob</text>
<text x="390" y="210" class="s">runs to completion, or on a schedule</text>
<text x="390" y="234" class="c">batch work, backups, reports</text>
</g>
</svg>
<figcaption>Choosing a workload controller. All four create Pods; they differ in what they promise about those Pods.</figcaption>
</figure>

### StatefulSet

Databases, message brokers, and anything else that clusters with its peers needs replicas that are **not** interchangeable. A **StatefulSet** gives each replica a stable identity:

<figure class="diagram">
<svg viewBox="0 0 720 240" style="width:100%;height:auto;max-width:720px" role="img" aria-label="A StatefulSet named db creates Pods db-0, db-1, and db-2 in order, each with its own PersistentVolumeClaim that follows it across restarts." xmlns="http://www.w3.org/2000/svg">
<style>
.k5e{ font-family:var(--sans),sans-serif; }
.k5e .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k5e .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k5e .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k5e .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k5e .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k5e .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k5e .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k5e .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k5e .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k5e .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k5e .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k5e .s{ fill:var(--text-dim); font-size:13px; }
.k5e .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k5e .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k5e .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k5e .c{ fill:var(--text-faint); font-size:12.5px; }
.k5e .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k5e .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k5e .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k5e-a); }
.k5e .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k5e-a); marker-start:url(#k5e-s); }
.k5e .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k5e-g); }
.k5e .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k5e-a); }
.k5e .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k5e .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k5e-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k5e-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k5e-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k5e">
<rect x="260" y="14" width="200" height="38" rx="7" class="dark"/>
<text x="360.0" y="37.5" text-anchor="middle" class="on">StatefulSet: db</text>
<line x1="360" y1="52" x2="120" y2="98" class="ar"/>
<rect x="40" y="100" width="160" height="52" rx="7" class="acc"/>
<text x="120.0" y="121.5" text-anchor="middle" class="m">db-0</text>
<text x="120.0" y="139.5" text-anchor="middle" class="s">starts first</text>
<line x1="120" y1="152" x2="120" y2="184" class="ln"/>
<rect x="40" y="184" width="160" height="38" rx="7" class="box"/>
<text x="120.0" y="207.5" text-anchor="middle" class="ma">data-db-0</text>
<line x1="360" y1="52" x2="360" y2="98" class="ar"/>
<rect x="280" y="100" width="160" height="52" rx="7" class="acc"/>
<text x="360.0" y="121.5" text-anchor="middle" class="m">db-1</text>
<text x="360.0" y="139.5" text-anchor="middle" class="s">then second</text>
<line x1="360" y1="152" x2="360" y2="184" class="ln"/>
<rect x="280" y="184" width="160" height="38" rx="7" class="box"/>
<text x="360.0" y="207.5" text-anchor="middle" class="ma">data-db-1</text>
<line x1="360" y1="52" x2="600" y2="98" class="ar"/>
<rect x="520" y="100" width="160" height="52" rx="7" class="acc"/>
<text x="600.0" y="121.5" text-anchor="middle" class="m">db-2</text>
<text x="600.0" y="139.5" text-anchor="middle" class="s">then third</text>
<line x1="600" y1="152" x2="600" y2="184" class="ln"/>
<rect x="520" y="184" width="160" height="38" rx="7" class="box"/>
<text x="600.0" y="207.5" text-anchor="middle" class="ma">data-db-2</text>
</g>
</svg>
<figcaption>A StatefulSet gives each replica a stable name and its own volume. If db-1 is rescheduled, it comes back as db-1 with the same data.</figcaption>
</figure>

- **Stable names**: `db-0`, `db-1`, `db-2`, which survive rescheduling.
- **Stable storage**: each replica gets its own PersistentVolumeClaim from a `volumeClaimTemplates` entry, and keeps it when it moves.
- **Ordered operations**: Pods start in order and stop in reverse order by default.
- **Stable DNS names**, through a companion headless Service (Chapter 8), such as `db-0.db.default.svc.cluster.local`.

A StatefulSet does not make an app highly available by itself. The app still has to handle replication and failover; the StatefulSet just gives it the stable identities it needs to do so.

### DaemonSet

A **DaemonSet** runs exactly one Pod on every node, or on every node matching a selector, and adds one automatically when a node joins. Cluster plumbing uses DaemonSets heavily: kube-proxy, CNI agents, log collectors, and monitoring agents. List them in your lab with `kubectl get ds -A`.

### Job and CronJob

A **Job** runs Pods until a given number of them complete successfully, retrying failures up to `backoffLimit`. It suits batch work: a data import, a report, a one-off migration. A **CronJob** creates a Job on a schedule written in cron syntax:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-report
spec:
  schedule: "0 2 * * *"          # 02:00 every day
  jobTemplate:
    spec:
      backoffLimit: 3
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: report
            image: busybox:1.36
            command: ["sh", "-c", "echo building report; sleep 5"]
```

Job Pods must use `restartPolicy: OnFailure` or `Never`, since a container that always restarts would never complete.

## Summary

- A **Deployment** manages **ReplicaSets**, which manage identical **Pods**. Scaling changes one field; any template change starts a rollout.
- **Rolling updates** are paced by `maxSurge` and `maxUnavailable` and gated on Pods becoming **Ready**, so readiness probes matter.
- Old ReplicaSets are kept at zero replicas, which makes **rollback** instant.
- Use a **StatefulSet** for stable identity and storage, a **DaemonSet** for one Pod per node, and a **Job** or **CronJob** for run-to-completion work.

Your Pods are now running and self-healing. Part III answers the next question: how does anything talk to them?

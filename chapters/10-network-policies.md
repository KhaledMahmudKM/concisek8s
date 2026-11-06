# Chapter 10: Network Policies

The network model in Chapter 6 says every Pod can reach every other Pod. That is convenient and, from a security point of view, alarming: a compromised web Pod can talk straight to the database, to other teams' services, and to the cluster's own components. **NetworkPolicies** are Kubernetes' built-in firewall rules for Pod traffic. This chapter explains how they are evaluated, the patterns that work, and the traps that catch nearly everyone once.

## Who enforces them

A NetworkPolicy is just an API object. Like an Ingress, it does nothing on its own: the **CNI plugin** enforces it, usually by programming iptables, nftables, or eBPF rules on each node. Calico, Cilium, and Antrea all support NetworkPolicy. Some simple plugins, such as plain Flannel, do not, and in that case **your policies are accepted and silently ignored**. Always confirm enforcement with a quick test before you rely on a policy.

kind has shipped with enforcement out of the box since v0.24, through the `kube-network-policies` controller that runs alongside kindnet, so every example in this chapter works in your lab.

## The evaluation model

Four rules explain all NetworkPolicy behavior:

1. **Isolation is opt-in, per Pod and per direction.** A Pod with no policies selecting it accepts and sends everything. The moment any policy with `Ingress` in its `policyTypes` selects a Pod, that Pod becomes **isolated for ingress**: only traffic some policy explicitly allows gets in. Egress works the same way, independently.
2. **Policies only allow.** There are no deny rules. Every rule is an allow, and the effective policy for a Pod is the **union** of all policies that select it. Adding a policy can only ever open more traffic for already-isolated Pods.
3. **Policies are stateful.** If a connection is allowed, its reply packets are allowed too. You never write rules for return traffic.
4. **Policies are namespaced** and select Pods with label selectors, the same mechanism you met in Chapter 4.

<figure class="diagram">
<svg viewBox="0 0 720 250" style="width:100%;height:auto;max-width:720px" role="img" aria-label="Without NetworkPolicies, web, api, and db Pods can all reach each other. After a default-deny ingress policy, no Pod in the namespace accepts any incoming connection." xmlns="http://www.w3.org/2000/svg">
<style>
.k10a{ font-family:var(--sans),sans-serif; }
.k10a .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k10a .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k10a .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k10a .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k10a .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k10a .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k10a .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k10a .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k10a .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k10a .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k10a .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k10a .s{ fill:var(--text-dim); font-size:13px; }
.k10a .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k10a .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k10a .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k10a .c{ fill:var(--text-faint); font-size:12.5px; }
.k10a .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k10a .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k10a .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k10a-a); }
.k10a .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k10a-a); marker-start:url(#k10a-s); }
.k10a .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k10a-g); }
.k10a .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k10a-a); }
.k10a .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k10a .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k10a-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k10a-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k10a-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k10a">
<text x="180" y="24" text-anchor="middle" class="b">no policies: all traffic allowed</text>
<text x="540" y="24" text-anchor="middle" class="b">default deny ingress</text>
<line x1="150" y1="80" x2="198" y2="80" class="ar2"/>
<line x1="95" y1="100" x2="150" y2="158" class="ar2"/>
<line x1="255" y1="100" x2="205" y2="158" class="ar2"/>
<rect x="40" y="60" width="110" height="40" rx="7" class="acc"/>
<text x="95.0" y="84.5" text-anchor="middle" class="b">web</text>
<rect x="200" y="60" width="110" height="40" rx="7" class="acc"/>
<text x="255.0" y="84.5" text-anchor="middle" class="b">api</text>
<rect x="120" y="160" width="110" height="40" rx="7" class="acc"/>
<text x="175.0" y="184.5" text-anchor="middle" class="b">db</text>
<line x1="510" y1="80" x2="558" y2="80" class="gd"/>
<line x1="455" y1="100" x2="510" y2="158" class="gd"/>
<line x1="615" y1="100" x2="565" y2="158" class="gd"/>
<rect x="400" y="60" width="110" height="40" rx="7" class="box"/>
<text x="455.0" y="84.5" text-anchor="middle" class="s">web</text>
<rect x="560" y="60" width="110" height="40" rx="7" class="box"/>
<text x="615.0" y="84.5" text-anchor="middle" class="s">api</text>
<rect x="480" y="160" width="110" height="40" rx="7" class="box"/>
<text x="535.0" y="184.5" text-anchor="middle" class="s">db</text>
<rect x="380" y="40" width="320" height="180" rx="7" class="dash"/>
<text x="540" y="240" text-anchor="middle" class="c">nothing gets in until a policy allows it</text>
</g>
</svg>
<figcaption>The starting point. Kubernetes allows all Pod traffic by default; a default-deny policy flips each namespace to allow-list mode.</figcaption>
</figure>

## Default deny, then allow

The robust pattern is the same as for any firewall: deny everything, then allow what you need. This policy selects every Pod in its namespace (the empty `podSelector`) and allows nothing:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: shop
spec:
  podSelector: {}
  policyTypes: [Ingress]
```

Now open exactly the paths your application needs. This policy lets only the `api` Pods reach the database, and only on its port:

<figure class="diagram">
<svg viewBox="0 0 720 260" style="width:100%;height:auto;max-width:720px" role="img" aria-label="A policy selecting the db Pods allows ingress only from Pods labelled app=api on TCP port 5432, so api can reach db while web is blocked." xmlns="http://www.w3.org/2000/svg">
<style>
.k10b{ font-family:var(--sans),sans-serif; }
.k10b .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k10b .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k10b .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k10b .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k10b .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k10b .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k10b .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k10b .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k10b .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k10b .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k10b .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k10b .s{ fill:var(--text-dim); font-size:13px; }
.k10b .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k10b .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k10b .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k10b .c{ fill:var(--text-faint); font-size:12.5px; }
.k10b .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k10b .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k10b .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k10b-a); }
.k10b .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k10b-a); marker-start:url(#k10b-s); }
.k10b .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k10b-g); }
.k10b .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k10b-a); }
.k10b .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k10b .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k10b-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k10b-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k10b-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k10b">
<rect x="20" y="40" width="130" height="46" rx="7" class="pnl"/>
<text x="85.0" y="58.5" text-anchor="middle" class="b">web</text>
<text x="85.0" y="76.5" text-anchor="middle" class="m">app=web</text>
<rect x="20" y="170" width="130" height="46" rx="7" class="pnl"/>
<text x="85.0" y="188.5" text-anchor="middle" class="b">api</text>
<text x="85.0" y="206.5" text-anchor="middle" class="m">app=api</text>
<rect x="330" y="70" width="190" height="110" rx="7" class="out"/>
<text x="425" y="94" text-anchor="middle" class="c">policy selects app=db</text>
<rect x="350" y="106" width="150" height="54" rx="7" class="acc"/>
<text x="425.0" y="128.5" text-anchor="middle" class="b">db</text>
<text x="425.0" y="146.5" text-anchor="middle" class="m">app=db :5432</text>
<line x1="150" y1="192" x2="348" y2="140" class="ar"/>
<text x="255" y="200" text-anchor="middle" class="a">allowed</text>
<line x1="150" y1="62" x2="326" y2="98" class="gd"/>
<text x="240" y="64" text-anchor="middle" class="c">blocked</text>
<rect x="540" y="40" width="160" height="176" rx="7" class="pnl"/>
<text x="552" y="64" class="m">podSelector:</text>
<text x="567.0" y="86" class="m">app: db</text>
<text x="552" y="108" class="m">ingress:</text>
<text x="552" y="130" class="m">- from:</text>
<text x="567.0" y="152" class="m">- podSelector:</text>
<text x="597.0" y="174" class="m">app: api</text>
<text x="567.0" y="196" class="m">ports: [5432]</text>
</g>
</svg>
<figcaption>A targeted allow rule. The policy is attached to the destination (db) and lists who may connect to it.</figcaption>
</figure>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-api
  namespace: shop
spec:
  podSelector:
    matchLabels: {app: db}
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector:
        matchLabels: {app: api}
    ports:
    - protocol: TCP
      port: 5432
```

Policies attach to the **destination**. You do not say "web may not call db"; you say "db accepts connections from api", and everything else is refused because db is isolated.

## Peers: pods, namespaces, and IP blocks

The `from` (ingress) and `to` (egress) lists accept three kinds of peers:

- **`podSelector`**: Pods in the policy's own namespace with matching labels.
- **`namespaceSelector`**: all Pods in namespaces with matching labels. Every namespace automatically carries the label `kubernetes.io/metadata.name: <name>`, which makes selecting one namespace by name easy.
- **`ipBlock`**: a CIDR, with optional exceptions, for traffic to or from outside the cluster.

Combining the first two is where the most famous mistake lives.

<figure class="diagram">
<svg viewBox="0 0 720 276" style="width:100%;height:auto;max-width:720px" role="img" aria-label="Placing namespaceSelector and podSelector in one from element means both must match; placing them in two elements means either may match, which allows far more traffic." xmlns="http://www.w3.org/2000/svg">
<style>
.k10c{ font-family:var(--sans),sans-serif; }
.k10c .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k10c .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k10c .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k10c .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k10c .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k10c .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k10c .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k10c .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k10c .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k10c .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k10c .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k10c .s{ fill:var(--text-dim); font-size:13px; }
.k10c .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k10c .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k10c .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k10c .c{ fill:var(--text-faint); font-size:12.5px; }
.k10c .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k10c .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k10c .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k10c-a); }
.k10c .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k10c-a); marker-start:url(#k10c-s); }
.k10c .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k10c-g); }
.k10c .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k10c-a); }
.k10c .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k10c .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k10c-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k10c-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k10c-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k10c">
<text x="185" y="24" text-anchor="middle" class="b">one element: AND</text>
<text x="535" y="24" text-anchor="middle" class="b">two elements: OR</text>
<rect x="20" y="38" width="330" height="124" rx="7" class="pnl"/>
<text x="36" y="62" class="m">from:</text>
<text x="36" y="84" class="m">- namespaceSelector:</text>
<text x="66.0" y="106" class="m">matchLabels: {team: a}</text>
<text x="51.0" y="128" class="m">podSelector:</text>
<text x="66.0" y="150" class="m">matchLabels: {app: client}</text>
<line x1="185" y1="162" x2="185" y2="184" class="ag"/>
<rect x="370" y="38" width="330" height="124" rx="7" class="pnl"/>
<text x="386" y="62" class="m">from:</text>
<text x="386" y="84" class="m">- namespaceSelector:</text>
<text x="416.0" y="106" class="m">matchLabels: {team: a}</text>
<text x="386" y="128" class="m">- podSelector:</text>
<text x="416.0" y="150" class="m">matchLabels: {app: client}</text>
<line x1="535" y1="162" x2="535" y2="184" class="ag"/>
<rect x="20" y="186" width="330" height="70" rx="7" class="acc"/>
<text x="185.0" y="216.5" text-anchor="middle" class="s">allows app=client Pods</text>
<text x="185.0" y="234.5" text-anchor="middle" class="s">only inside team=a namespaces</text>
<rect x="370" y="186" width="330" height="70" rx="7" class="box"/>
<text x="535.0" y="216.5" text-anchor="middle" class="s">allows every Pod in team=a namespaces,</text>
<text x="535.0" y="234.5" text-anchor="middle" class="s">plus app=client Pods in this namespace</text>
</g>
</svg>
<figcaption>The classic NetworkPolicy trap. One dash changes AND into OR; always check which you meant.</figcaption>
</figure>

When `namespaceSelector` and `podSelector` sit in the **same list element**, both must match: Pods with that label inside those namespaces. When they sit in **separate elements**, either may match, which usually allows far more than intended. The YAML differs by a single dash. Read every `from` and `to` list twice.

`ipBlock` has a subtlety of its own: it applies to the address the policy engine sees. Traffic that went through SNAT, such as NodePort traffic with `externalTrafficPolicy: Cluster`, arrives with a node's address rather than the client's, so client-IP rules may not match the way you expect.

## Egress and the DNS trap

Egress policies restrict where Pods may connect, which limits what an attacker can reach from a compromised Pod and stops data from leaving quietly. They follow exactly the same rules, with one trap that catches everyone:

<figure class="diagram">
<svg viewBox="0 0 720 240" style="width:100%;height:auto;max-width:720px" role="img" aria-label="A Pod under default-deny egress needs two explicit egress rules: one to the cluster DNS on port 53, and one to the api Service&#x27;s Pods on port 8080." xmlns="http://www.w3.org/2000/svg">
<style>
.k10d{ font-family:var(--sans),sans-serif; }
.k10d .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k10d .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k10d .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k10d .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k10d .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k10d .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k10d .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k10d .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k10d .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k10d .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k10d .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k10d .s{ fill:var(--text-dim); font-size:13px; }
.k10d .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k10d .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k10d .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k10d .c{ fill:var(--text-faint); font-size:12.5px; }
.k10d .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k10d .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k10d .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k10d-a); }
.k10d .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k10d-a); marker-start:url(#k10d-s); }
.k10d .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k10d-g); }
.k10d .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k10d-a); }
.k10d .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k10d .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k10d-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k10d-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k10d-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k10d">
<rect x="20" y="90" width="140" height="50" rx="7" class="pnl"/>
<text x="90.0" y="110.5" text-anchor="middle" class="b">app Pod</text>
<text x="90.0" y="128.5" text-anchor="middle" class="c">egress: deny all</text>
<rect x="300" y="30" width="170" height="50" rx="7" class="acc"/>
<text x="385.0" y="50.5" text-anchor="middle" class="b">CoreDNS</text>
<text x="385.0" y="68.5" text-anchor="middle" class="m">kube-system :53</text>
<rect x="300" y="150" width="170" height="50" rx="7" class="acc"/>
<text x="385.0" y="170.5" text-anchor="middle" class="b">api Pods</text>
<text x="385.0" y="188.5" text-anchor="middle" class="m">:8080</text>
<line x1="160" y1="105" x2="298" y2="60" class="ar"/>
<text x="208" y="62" text-anchor="middle" class="c">DNS, UDP and TCP 53</text>
<line x1="160" y1="125" x2="298" y2="175" class="ar"/>
<text x="212" y="182" text-anchor="middle" class="c">TCP 8080</text>
<rect x="520" y="30" width="180" height="170" rx="7" class="pnl"/>
<text x="536" y="56" class="b">allow egress to:</text>
<text x="536" y="84" class="s">kube-dns, port 53</text>
<text x="536" y="108" class="s">app=api, port 8080</text>
<text x="536" y="140" class="c">nothing else</text>
<text x="360" y="226" text-anchor="middle" class="c">forget the DNS rule and every lookup by name fails</text>
</g>
</svg>
<figcaption>Egress default-deny. The rule people forget is DNS: without it, the app cannot even resolve the names of the services it is allowed to reach.</figcaption>
</figure>

A default-deny egress policy also blocks **DNS**. The Pod can no longer resolve any name, including the names of the Services it is allowed to reach, so the symptom looks like "everything broke". Allow DNS explicitly whenever you restrict egress:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-and-api
  namespace: shop
spec:
  podSelector:
    matchLabels: {app: web}
  policyTypes: [Egress]
  egress:
  - to:
    - namespaceSelector:
        matchLabels: {kubernetes.io/metadata.name: kube-system}
      podSelector:
        matchLabels: {k8s-app: kube-dns}
    ports:
    - {protocol: UDP, port: 53}
    - {protocol: TCP, port: 53}
  - to:
    - podSelector:
        matchLabels: {app: api}
    ports:
    - {protocol: TCP, port: 8080}
```

Note that the DNS rule puts `namespaceSelector` and `podSelector` in **one** element: CoreDNS Pods, in `kube-system` only.

## Hands on: test before you trust

Build the three-tier app in a `shop` namespace, then probe it before and after each policy:

```bash
kubectl create namespace shop
kubectl -n shop run db  --image=nginx:1.27 --labels=app=db  --port=80 --expose
kubectl -n shop run api --image=nicolaka/netshoot --labels=app=api -- sleep infinity
kubectl -n shop run web --image=nicolaka/netshoot --labels=app=web -- sleep infinity

kubectl -n shop exec web -- curl -s -m 3 http://db    # works: no policies yet
# apply default-deny-ingress, plus a version of db-allow-api using port 80
kubectl -n shop exec web -- curl -s -m 3 http://db    # times out
kubectl -n shop exec api -- curl -s -m 3 http://db    # still works
```

A blocked connection usually **times out** rather than being refused, because the packets are silently dropped. Keep a short `-m` timeout on your probes. Then list what applies with `kubectl -n shop get networkpolicy` and `kubectl -n shop describe networkpolicy db-allow-api`.

## Beyond NetworkPolicy

The NetworkPolicy API is deliberately small: namespaced, allow-only, and Layer 3 and 4. Real platforms often need more, and two directions are worth knowing:

- **Cluster-wide policy.** SIG Network is developing cluster-scoped policy resources so that platform administrators can set rules, including explicit denies, that namespace owners cannot override.
- **Plugin-specific policies.** Calico and Cilium offer their own richer policy resources with features such as explicit deny rules, policy ordering, DNS-name-based egress, and, in Cilium's case, Layer 7 rules for HTTP paths and methods. They are powerful but tie you to that plugin.

## Summary

- NetworkPolicies are **enforced by the CNI plugin**; without support they are silently ignored. kind enforces them out of the box.
- A Pod becomes **isolated** for a direction once any policy selects it for that direction; allowed traffic is the **union** of all such policies, and replies are allowed automatically.
- Start with **default deny**, then allow specific paths, attaching each rule to the destination.
- `namespaceSelector` and `podSelector` in **one element mean AND**, in **two elements mean OR**.
- Restricting egress blocks **DNS** too. Always allow port 53 to `kube-dns`.

That completes the networking part. Part IV turns to configuration, storage, and keeping workloads healthy in production.

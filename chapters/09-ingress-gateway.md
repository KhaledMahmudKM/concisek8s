# Chapter 9: Ingress and the Gateway API

A LoadBalancer Service gets traffic into the cluster, but only at Layer 4: an IP address and a port. Real web applications need Layer 7 decisions: send `shop.example.com/api` to one Service and everything else to another, terminate TLS once, split traffic between versions. Kubernetes has two APIs for this. **Ingress** is the original and is now frozen; the **Gateway API** is its successor. This chapter explains both and why the ecosystem is moving from one to the other.

## Why a shared entry point

<figure class="diagram">
<svg viewBox="0 0 720 256" style="width:100%;height:auto;max-width:720px" role="img" aria-label="On the left, three Services each get their own external load balancer. On the right, one load balancer feeds an L7 proxy that routes to three Services by host name and path." xmlns="http://www.w3.org/2000/svg">
<style>
.k9a{ font-family:var(--sans),sans-serif; }
.k9a .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k9a .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k9a .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k9a .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k9a .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k9a .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k9a .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k9a .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k9a .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k9a .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k9a .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k9a .s{ fill:var(--text-dim); font-size:13px; }
.k9a .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k9a .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k9a .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k9a .c{ fill:var(--text-faint); font-size:12.5px; }
.k9a .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k9a .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k9a .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k9a-a); }
.k9a .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k9a-a); marker-start:url(#k9a-s); }
.k9a .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k9a-g); }
.k9a .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k9a-a); }
.k9a .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k9a .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k9a-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k9a-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k9a-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k9a">
<text x="180" y="20" text-anchor="middle" class="b">one LoadBalancer per Service</text>
<text x="540" y="20" text-anchor="middle" class="b">one shared entry point</text>
<rect x="30" y="40" width="140" height="36" rx="7" class="pnl"/>
<text x="100.0" y="62.5" text-anchor="middle" class="s">load balancer</text>
<rect x="210" y="40" width="120" height="36" rx="7" class="box"/>
<text x="270.0" y="62.5" text-anchor="middle" class="s">svc a</text>
<line x1="170" y1="58" x2="208" y2="58" class="ar"/>
<rect x="30" y="100" width="140" height="36" rx="7" class="pnl"/>
<text x="100.0" y="122.5" text-anchor="middle" class="s">load balancer</text>
<rect x="210" y="100" width="120" height="36" rx="7" class="box"/>
<text x="270.0" y="122.5" text-anchor="middle" class="s">svc b</text>
<line x1="170" y1="118" x2="208" y2="118" class="ar"/>
<rect x="30" y="160" width="140" height="36" rx="7" class="pnl"/>
<text x="100.0" y="182.5" text-anchor="middle" class="s">load balancer</text>
<rect x="210" y="160" width="120" height="36" rx="7" class="box"/>
<text x="270.0" y="182.5" text-anchor="middle" class="s">svc c</text>
<line x1="170" y1="178" x2="208" y2="178" class="ar"/>
<text x="180" y="232" text-anchor="middle" class="c">three IPs, three bills, no shared TLS</text>
<rect x="380" y="100" width="100" height="36" rx="7" class="pnl"/>
<text x="430.0" y="122.5" text-anchor="middle" class="s">load balancer</text>
<rect x="510" y="88" width="100" height="60" rx="7" class="acc"/>
<text x="560.0" y="113.5" text-anchor="middle" class="b">L7 proxy</text>
<text x="560.0" y="131.5" text-anchor="middle" class="s">host, path</text>
<line x1="480" y1="118" x2="508" y2="118" class="ar"/>
<rect x="636" y="40" width="64" height="36" rx="7" class="box"/>
<text x="668.0" y="62.5" text-anchor="middle" class="s">svc a</text>
<line x1="610" y1="118" x2="634" y2="58" class="ar"/>
<rect x="636" y="100" width="64" height="36" rx="7" class="box"/>
<text x="668.0" y="122.5" text-anchor="middle" class="s">svc b</text>
<line x1="610" y1="118" x2="634" y2="118" class="ar"/>
<rect x="636" y="160" width="64" height="36" rx="7" class="box"/>
<text x="668.0" y="182.5" text-anchor="middle" class="s">svc c</text>
<line x1="610" y1="118" x2="634" y2="178" class="ar"/>
<text x="540" y="232" text-anchor="middle" class="c">one IP; routing and TLS in one place</text>
</g>
</svg>
<figcaption>Why L7 routing exists. A LoadBalancer Service per app gets expensive and leaves each team to solve TLS and routing alone.</figcaption>
</figure>

Giving every Service its own cloud load balancer works, but each one costs money and a public IP, and each team ends up solving TLS certificates, redirects, and routing separately. A shared Layer 7 proxy fixes all three: one address, one place for certificates, and routing rules that every team can contribute to.

## Ingress

An **Ingress** object is a list of HTTP routing rules: for this host and path, send traffic to that Service and port.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop
spec:
  ingressClassName: example
  tls:
  - hosts: [shop.example.com]
    secretName: shop-tls
  rules:
  - host: shop.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service: {name: api, port: {number: 8080}}
      - path: /
        pathType: Prefix
        backend:
          service: {name: web, port: {number: 80}}
```

The object does nothing by itself. Kubernetes ships no Ingress implementation. You install an **Ingress controller**, a proxy such as Traefik, HAProxy, Envoy, or a cloud provider's own, which watches Ingress objects and configures itself to match. The `ingressClassName` field says which controller should handle a given Ingress.

<figure class="diagram">
<svg viewBox="0 0 720 270" style="width:100%;height:auto;max-width:720px" role="img" aria-label="Traffic from a client enters through a load balancer to the Ingress controller&#x27;s proxy Pods, which watch Ingress objects and route /api to the api Service and everything else to the web Service." xmlns="http://www.w3.org/2000/svg">
<style>
.k9b{ font-family:var(--sans),sans-serif; }
.k9b .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k9b .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k9b .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k9b .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k9b .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k9b .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k9b .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k9b .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k9b .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k9b .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k9b .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k9b .s{ fill:var(--text-dim); font-size:13px; }
.k9b .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k9b .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k9b .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k9b .c{ fill:var(--text-faint); font-size:12.5px; }
.k9b .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k9b .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k9b .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k9b-a); }
.k9b .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k9b-a); marker-start:url(#k9b-s); }
.k9b .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k9b-g); }
.k9b .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k9b-a); }
.k9b .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k9b .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k9b-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k9b-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k9b-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k9b">
<rect x="20" y="110" width="110" height="40" rx="7" class="box"/>
<text x="75.0" y="134.5" text-anchor="middle" class="s">client</text>
<rect x="160" y="110" width="130" height="40" rx="7" class="pnl"/>
<text x="225.0" y="134.5" text-anchor="middle" class="s">LoadBalancer</text>
<rect x="320" y="90" width="160" height="80" rx="7" class="acc"/>
<text x="400.0" y="125.5" text-anchor="middle" class="b">Ingress controller</text>
<text x="400.0" y="143.5" text-anchor="middle" class="s">proxy Pods</text>
<rect x="320" y="200" width="160" height="56" rx="7" class="pnl"/>
<text x="400.0" y="223.5" text-anchor="middle" class="b">Ingress object</text>
<text x="400.0" y="241.5" text-anchor="middle" class="c">rules: host, path</text>
<line x1="400" y1="200" x2="400" y2="172" class="ag"/>
<line x1="130" y1="130" x2="158" y2="130" class="ar"/>
<line x1="290" y1="130" x2="318" y2="130" class="ar"/>
<rect x="560" y="60" width="140" height="46" rx="7" class="box"/>
<text x="630.0" y="78.5" text-anchor="middle" class="b">Service api</text>
<text x="630.0" y="96.5" text-anchor="middle" class="m">/api</text>
<rect x="560" y="150" width="140" height="46" rx="7" class="box"/>
<text x="630.0" y="168.5" text-anchor="middle" class="b">Service web</text>
<text x="630.0" y="186.5" text-anchor="middle" class="m">/</text>
<line x1="480" y1="118" x2="558" y2="84" class="ar"/>
<line x1="480" y1="142" x2="558" y2="172" class="ar"/>
</g>
</svg>
<figcaption>The Ingress model. An Ingress object is only a set of rules; an Ingress controller you install reads them and does the actual proxying.</figcaption>
</figure>

Ingress worked well for simple cases, but it had real limits. The API only understands host and path. Everything else, including header matching, traffic splitting, timeouts, rewrites, and even TCP or gRPC, was bolted on with controller-specific **annotations**. A manifest written for one controller rarely worked on another, and a single Ingress object mixed concerns that belong to different people: the platform team's TLS settings and the app team's paths.

The Ingress API itself remains supported and stable, but it is **frozen**: it gets no new features. And the most widely used controller, the community **Ingress NGINX** project, was retired in March 2026. Its existing installations keep working, but it no longer receives releases, bug fixes, or security patches. The Kubernetes project's recommendation is to move to the Gateway API.

## The Gateway API

The **Gateway API** is a set of resources under `gateway.networking.k8s.io`, developed by SIG Network as the successor to Ingress. It reached v1.0 in 2023, and version 1.5, released in February 2026, moved TLSRoute, ListenerSet, and several other features into the stable channel. Its central idea is to split routing configuration by **role**.

<figure class="diagram">
<svg viewBox="0 0 720 280" style="width:100%;height:auto;max-width:720px" role="img" aria-label="Gateway API splits responsibility by role: the infrastructure provider supplies a GatewayClass, the cluster operator creates a Gateway with listeners, and application teams attach HTTPRoutes to it." xmlns="http://www.w3.org/2000/svg">
<style>
.k9c{ font-family:var(--sans),sans-serif; }
.k9c .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k9c .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k9c .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k9c .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k9c .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k9c .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k9c .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k9c .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k9c .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k9c .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k9c .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k9c .s{ fill:var(--text-dim); font-size:13px; }
.k9c .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k9c .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k9c .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k9c .c{ fill:var(--text-faint); font-size:12.5px; }
.k9c .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k9c .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k9c .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k9c-a); }
.k9c .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k9c-a); marker-start:url(#k9c-s); }
.k9c .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k9c-g); }
.k9c .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k9c-a); }
.k9c .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k9c .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k9c-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k9c-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k9c-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k9c">
<rect x="20" y="24" width="200" height="50" rx="7" class="box"/>
<text x="120.0" y="44.5" text-anchor="middle" class="b">Infrastructure provider</text>
<text x="120.0" y="62.5" text-anchor="middle" class="c">cloud or platform vendor</text>
<rect x="20" y="118" width="200" height="50" rx="7" class="box"/>
<text x="120.0" y="138.5" text-anchor="middle" class="b">Cluster operator</text>
<text x="120.0" y="156.5" text-anchor="middle" class="c">platform team</text>
<rect x="20" y="212" width="200" height="50" rx="7" class="box"/>
<text x="120.0" y="232.5" text-anchor="middle" class="b">App developers</text>
<text x="120.0" y="250.5" text-anchor="middle" class="c">each team</text>
<rect x="300" y="24" width="200" height="50" rx="7" class="pnl"/>
<text x="400.0" y="44.5" text-anchor="middle" class="b">GatewayClass</text>
<text x="400.0" y="62.5" text-anchor="middle" class="c">which controller</text>
<rect x="300" y="118" width="200" height="50" rx="7" class="acc"/>
<text x="400.0" y="138.5" text-anchor="middle" class="b">Gateway</text>
<text x="400.0" y="156.5" text-anchor="middle" class="s">listeners :80 :443</text>
<rect x="270" y="212" width="150" height="50" rx="7" class="pnl"/>
<text x="345.0" y="232.5" text-anchor="middle" class="b">HTTPRoute</text>
<text x="345.0" y="250.5" text-anchor="middle" class="c">team A</text>
<rect x="440" y="212" width="150" height="50" rx="7" class="pnl"/>
<text x="515.0" y="232.5" text-anchor="middle" class="b">HTTPRoute</text>
<text x="515.0" y="250.5" text-anchor="middle" class="c">team B</text>
<line x1="220" y1="49" x2="298" y2="49" class="ag"/>
<line x1="220" y1="143" x2="298" y2="143" class="ag"/>
<line x1="220" y1="237" x2="268" y2="237" class="ag"/>
<line x1="400" y1="118" x2="400" y2="76" class="ar"/>
<text x="410" y="100" class="c">gatewayClassName</text>
<line x1="345" y1="212" x2="380" y2="170" class="ar"/>
<line x1="515" y1="212" x2="440" y2="170" class="ar"/>
<text x="530" y="196" class="c">parentRefs</text>
<text x="520" y="54" class="c">cluster-scoped</text>
<text x="520" y="148" class="c">where traffic enters</text>
</g>
</svg>
<figcaption>Gateway API's role model. Each team owns only its own object, and routes attach to a Gateway the way Pods attach to a node.</figcaption>
</figure>

- **GatewayClass** (cluster-scoped), owned by whoever provides the implementation, names the controller, much like a StorageClass names a storage provisioner.
- **Gateway**, owned by the cluster operator, is an actual entry point: which ports and protocols to listen on, which host names, which TLS certificates, and which namespaces may attach routes.
- **Routes**, owned by application teams, describe what to do with matching traffic. **HTTPRoute** is the most common; GRPCRoute, TLSRoute, and others cover other protocols. A route attaches to a Gateway through `parentRefs`.

Each team edits only its own object, and the Gateway decides which routes it accepts. Features that were annotations in Ingress, such as header matching, weighted backends, redirects, rewrites, and mirroring, are ordinary typed fields, so the same YAML works across implementations.

<figure class="diagram">
<svg viewBox="0 0 720 260" style="width:100%;height:auto;max-width:720px" role="img" aria-label="A Gateway listening on port 80 for shop.example.com passes traffic to an HTTPRoute with two rules: /api goes to the api Service, and / is split 90 percent to web-v1 and 10 percent to web-v2." xmlns="http://www.w3.org/2000/svg">
<style>
.k9d{ font-family:var(--sans),sans-serif; }
.k9d .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k9d .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k9d .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k9d .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k9d .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k9d .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k9d .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k9d .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k9d .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k9d .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k9d .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k9d .s{ fill:var(--text-dim); font-size:13px; }
.k9d .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k9d .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k9d .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k9d .c{ fill:var(--text-faint); font-size:12.5px; }
.k9d .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k9d .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k9d .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k9d-a); }
.k9d .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k9d-a); marker-start:url(#k9d-s); }
.k9d .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k9d-g); }
.k9d .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k9d-a); }
.k9d .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k9d .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k9d-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k9d-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k9d-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k9d">
<rect x="20" y="100" width="150" height="56" rx="7" class="acc"/>
<text x="95.0" y="123.5" text-anchor="middle" class="b">Gateway :80</text>
<text x="95.0" y="141.5" text-anchor="middle" class="m">shop.example.com</text>
<rect x="210" y="24" width="240" height="216" rx="7" class="zone"/>
<text x="226" y="48" class="b">HTTPRoute: shop</text>
<rect x="226" y="64" width="208" height="56" rx="7" class="pnl"/>
<text x="330.0" y="87.5" text-anchor="middle" class="m">match /api</text>
<text x="330.0" y="105.5" text-anchor="middle" class="s">to api:8080</text>
<rect x="226" y="148" width="208" height="76" rx="7" class="pnl"/>
<text x="330.0" y="172.5" text-anchor="middle" class="m">match /</text>
<text x="330.0" y="190.5" text-anchor="middle" class="s">90% to web-v1</text>
<text x="330.0" y="208.5" text-anchor="middle" class="s">10% to web-v2</text>
<line x1="170" y1="128" x2="208" y2="128" class="ar"/>
<rect x="520" y="70" width="180" height="44" rx="7" class="box"/>
<text x="610.0" y="96.5" text-anchor="middle" class="s">Service api</text>
<rect x="520" y="146" width="180" height="40" rx="7" class="box"/>
<text x="610.0" y="170.5" text-anchor="middle" class="s">Service web-v1</text>
<rect x="520" y="200" width="180" height="40" rx="7" class="box"/>
<text x="610.0" y="224.5" text-anchor="middle" class="s">Service web-v2</text>
<line x1="434" y1="92" x2="518" y2="92" class="ar"/>
<line x1="434" y1="176" x2="518" y2="166" class="ar"/>
<line x1="434" y1="196" x2="518" y2="218" class="ar"/>
</g>
</svg>
<figcaption>An HTTPRoute in action: path matching plus a weighted split, the basis of a canary release, in plain portable YAML.</figcaption>
</figure>

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shop-gw
spec:
  gatewayClassName: cloud-provider-kind
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces: {from: Same}
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: shop
spec:
  parentRefs:
  - name: shop-gw
  hostnames: [shop.example.com]
  rules:
  - matches:
    - path: {type: PathPrefix, value: /api}
    backendRefs:
    - name: api
      port: 8080
  - matches:
    - path: {type: PathPrefix, value: /}
    backendRefs:
    - name: web-v1
      port: 80
      weight: 90
    - name: web-v2
      port: 80
      weight: 10
```

Moving the weights from 90/10 to 50/50 and then 0/100 is a complete canary release, with no controller-specific annotations anywhere.

When a route needs to reference a Service in a different namespace, the target namespace must explicitly permit it with a **ReferenceGrant**. This prevents one team from routing traffic into another team's Services without consent.

## Hands on: a Gateway in kind

The Gateway API resources are **not** built into Kubernetes; they are CustomResourceDefinitions installed separately. Conveniently, cloud-provider-kind (Chapter 7) also runs a Gateway API controller with a GatewayClass named `cloud-provider-kind`. With it running beside your cluster:

```bash
kubectl get gatewayclass
```

If the command reports that the resource type is unknown, install the standard channel CRDs for the version you want, then restart cloud-provider-kind:

```bash
kubectl apply --server-side -f \
  https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.1/standard-install.yaml
```

Deploy two simple backends, apply the Gateway and HTTPRoute above, and wait for the Gateway to get an address:

```bash
kubectl create deployment web-v1 --image=nginx:1.27 && kubectl expose deployment web-v1 --port=80
kubectl create deployment web-v2 --image=nginx:1.28 && kubectl expose deployment web-v2 --port=80
kubectl get gateway shop-gw          # wait for an ADDRESS
GW=$(kubectl get gateway shop-gw -o jsonpath='{.status.addresses[0].value}')
curl -H 'Host: shop.example.com' http://$GW/
```

Repeat the `curl` a few dozen times and count the `Server:` headers: about one response in ten comes from nginx 1.28. Then inspect the conditions the controller writes back, which say whether each route was accepted and why not if it was refused:

```bash
kubectl describe httproute shop
```

## Choosing between them

For a new cluster, start with the **Gateway API**. It is where all new development happens, it is portable across implementations, and its role model scales to many teams. Keep using **Ingress** where it already works with a maintained controller, and plan any migration away from Ingress NGINX now. Tools such as `ingress2gateway` convert existing Ingress objects into Gateway API resources as a starting point.

## Summary

- Layer 7 entry points let many Services share **one address, one TLS setup, and one set of routing rules**.
- **Ingress** routes by host and path through a separately installed controller. The API is frozen, and the popular Ingress NGINX controller was retired in March 2026.
- The **Gateway API** splits configuration by role: **GatewayClass** (provider), **Gateway** (operator), and **HTTPRoute** and friends (app teams). Traffic splitting, header matching, and more are portable typed fields.
- In kind, **cloud-provider-kind** supplies both LoadBalancer addresses and a Gateway API controller.

Traffic can now reach your Pods from inside and outside the cluster. The last networking chapter is about stopping the traffic that should not.

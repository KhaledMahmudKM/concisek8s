# Chapter 7: Services and kube-proxy

Pods are reachable, but their IPs change every time they are replaced, and there are usually several of them behind one function. Clients need one address that stays put and spreads load across whichever Pods exist right now. That is a **Service**. This chapter explains what a Service really is on the wire, which turns out to be surprisingly little: a set of packet-rewriting rules on every node.

## A stable front for changing Pods

<figure class="diagram">
<svg viewBox="0 0 720 256" style="width:100%;height:auto;max-width:720px" role="img" aria-label="A client Pod connects to the Service web at a stable virtual IP, and the Service spreads connections across three backend Pods whose own IPs change over time." xmlns="http://www.w3.org/2000/svg">
<style>
.k7a{ font-family:var(--sans),sans-serif; }
.k7a .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k7a .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k7a .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k7a .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k7a .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k7a .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k7a .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k7a .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k7a .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k7a .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k7a .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k7a .s{ fill:var(--text-dim); font-size:13px; }
.k7a .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k7a .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k7a .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k7a .c{ fill:var(--text-faint); font-size:12.5px; }
.k7a .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k7a .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k7a .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k7a-a); }
.k7a .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k7a-a); marker-start:url(#k7a-s); }
.k7a .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k7a-g); }
.k7a .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k7a-a); }
.k7a .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k7a .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k7a-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k7a-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k7a-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k7a">
<rect x="20" y="100" width="120" height="46" rx="7" class="pnl"/>
<text x="80.0" y="127.5" text-anchor="middle" class="s">client Pod</text>
<rect x="230" y="90" width="200" height="66" rx="7" class="dark"/>
<text x="330.0" y="118.5" text-anchor="middle" class="on">Service: web</text>
<text x="330.0" y="136.5" text-anchor="middle" class="on">10.96.45.10:80</text>
<line x1="140" y1="123" x2="228" y2="123" class="ar"/>
<line x1="430" y1="123" x2="518" y2="53" class="ar"/>
<rect x="520" y="30" width="180" height="46" rx="7" class="acc"/>
<text x="610.0" y="48.5" text-anchor="middle" class="b">web-1</text>
<text x="610.0" y="66.5" text-anchor="middle" class="m">10.244.1.4</text>
<line x1="430" y1="123" x2="518" y2="123" class="ar"/>
<rect x="520" y="100" width="180" height="46" rx="7" class="acc"/>
<text x="610.0" y="118.5" text-anchor="middle" class="b">web-2</text>
<text x="610.0" y="136.5" text-anchor="middle" class="m">10.244.2.7</text>
<line x1="430" y1="123" x2="518" y2="193" class="ar"/>
<rect x="520" y="170" width="180" height="46" rx="7" class="acc"/>
<text x="610.0" y="188.5" text-anchor="middle" class="b">web-3</text>
<text x="610.0" y="206.5" text-anchor="middle" class="m">10.244.1.9</text>
<text x="180" y="194" text-anchor="middle" class="c">connects to one stable name and IP</text>
<text x="180" y="214" text-anchor="middle" class="ma">web.default.svc.cluster.local</text>
<text x="610" y="240" text-anchor="middle" class="c">Pods come and go; their IPs change</text>
</g>
</svg>
<figcaption>A Service gives a changing set of Pods one stable virtual IP and DNS name.</figcaption>
</figure>

A Service has a **selector**, a **virtual IP** called the ClusterIP, and one or more **ports**:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
  - name: http
    port: 80          # the port clients use on the ClusterIP
    targetPort: 8080  # the port the container listens on
```

`port` and `targetPort` may differ, which lets you change a container's port without touching any client. `targetPort` can also name a container port, such as `targetPort: http`.

## EndpointSlices: tracking the backends

A Service does not hold a list of Pods. The cluster maintains that list separately, in **EndpointSlice** objects.

<figure class="diagram">
<svg viewBox="0 0 720 300" style="width:100%;height:auto;max-width:720px" role="img" aria-label="The EndpointSlice controller matches the Service selector against Ready Pods and writes an EndpointSlice listing their addresses; kube-proxy on every node watches Services and EndpointSlices and programs packet rules." xmlns="http://www.w3.org/2000/svg">
<style>
.k7b{ font-family:var(--sans),sans-serif; }
.k7b .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k7b .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k7b .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k7b .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k7b .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k7b .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k7b .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k7b .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k7b .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k7b .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k7b .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k7b .s{ fill:var(--text-dim); font-size:13px; }
.k7b .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k7b .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k7b .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k7b .c{ fill:var(--text-faint); font-size:12.5px; }
.k7b .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k7b .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k7b .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k7b-a); }
.k7b .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k7b-a); marker-start:url(#k7b-s); }
.k7b .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k7b-g); }
.k7b .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k7b-a); }
.k7b .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k7b .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k7b-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k7b-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k7b-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k7b">
<rect x="20" y="30" width="190" height="56" rx="7" class="pnl"/>
<text x="115.0" y="53.5" text-anchor="middle" class="b">Service web</text>
<text x="115.0" y="71.5" text-anchor="middle" class="m">selector app=web</text>
<rect x="20" y="150" width="190" height="80" rx="7" class="pnl"/>
<text x="115.0" y="185.5" text-anchor="middle" class="b">Pods app=web</text>
<text x="115.0" y="203.5" text-anchor="middle" class="m">Ready: .1.4 .2.7 .1.9</text>
<rect x="260" y="90" width="170" height="56" rx="7" class="acc"/>
<text x="345.0" y="113.5" text-anchor="middle" class="b">EndpointSlice</text>
<text x="345.0" y="131.5" text-anchor="middle" class="s">controller</text>
<line x1="210" y1="58" x2="258" y2="106" class="ar"/>
<line x1="210" y1="190" x2="258" y2="132" class="ar"/>
<rect x="480" y="24" width="220" height="104" rx="7" class="pnl"/>
<text x="590.0" y="53.5" text-anchor="middle" class="b">EndpointSlice web-x7k2</text>
<text x="590.0" y="71.5" text-anchor="middle" class="m">10.244.1.4:8080</text>
<text x="590.0" y="89.5" text-anchor="middle" class="m">10.244.2.7:8080</text>
<text x="590.0" y="107.5" text-anchor="middle" class="m">10.244.1.9:8080</text>
<line x1="430" y1="112" x2="478" y2="92" class="ar"/>
<rect x="480" y="170" width="220" height="56" rx="7" class="acc"/>
<text x="590.0" y="193.5" text-anchor="middle" class="b">kube-proxy (every node)</text>
<text x="590.0" y="211.5" text-anchor="middle" class="s">watches Services, slices</text>
<line x1="590" y1="128" x2="590" y2="168" class="ag"/>
<rect x="480" y="250" width="220" height="38" rx="7" class="box"/>
<text x="590.0" y="273.5" text-anchor="middle" class="s">iptables / nftables rules</text>
<line x1="590" y1="226" x2="590" y2="248" class="ar"/>
</g>
</svg>
<figcaption>From selector to packet rules. Only Ready Pods are listed, which is how readiness probes take a Pod out of rotation.</figcaption>
</figure>

The EndpointSlice controller watches Services and Pods. For each Service it writes slices listing the IPs and ports of matching Pods, along with each endpoint's condition. Crucially, a Pod is only marked **ready** when its readiness probe passes. A Pod that is starting up, or failing its probe, stays out of rotation without anyone touching the Service. Slices hold up to 100 endpoints each, so a Service with a thousand Pods gets ten slices, and a change to one Pod rewrites only one small object.

```bash
kubectl get endpointslices -l kubernetes.io/service-name=web -o wide
```

You may also meet the older **Endpoints** object, one big list per Service. It has been deprecated since Kubernetes 1.33 in favor of EndpointSlices.

## What a ClusterIP really is

Here is the part that surprises people: **no machine owns the ClusterIP**. No interface has it, nothing answers ARP for it, and in the default proxy mode you cannot even ping it. It exists only as a match in packet-filtering rules that kube-proxy installs on every node.

<figure class="diagram">
<svg viewBox="0 0 720 320" style="width:100%;height:auto;max-width:720px" role="img" aria-label="A client Pod sends to the Service virtual IP. Netfilter on the client&#x27;s own node matches the VIP, picks one of three endpoints, and rewrites the destination to that Pod&#x27;s IP; replies are translated back by conntrack." xmlns="http://www.w3.org/2000/svg">
<style>
.k7c{ font-family:var(--sans),sans-serif; }
.k7c .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k7c .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k7c .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k7c .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k7c .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k7c .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k7c .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k7c .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k7c .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k7c .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k7c .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k7c .s{ fill:var(--text-dim); font-size:13px; }
.k7c .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k7c .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k7c .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k7c .c{ fill:var(--text-faint); font-size:12.5px; }
.k7c .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k7c .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k7c .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k7c-a); }
.k7c .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k7c-a); marker-start:url(#k7c-s); }
.k7c .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k7c-g); }
.k7c .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k7c-a); }
.k7c .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k7c .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k7c-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k7c-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k7c-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k7c">
<rect x="20" y="40" width="150" height="50" rx="7" class="pnl"/>
<text x="95.0" y="60.5" text-anchor="middle" class="b">client Pod</text>
<text x="95.0" y="78.5" text-anchor="middle" class="m">10.244.1.2</text>
<rect x="240" y="24" width="240" height="188" rx="7" class="zone"/>
<text x="360" y="44" text-anchor="middle" class="b">netfilter, client's node</text>
<rect x="256" y="56" width="208" height="34" rx="7" class="box"/>
<text x="360.0" y="77.5" text-anchor="middle" class="s">KUBE-SERVICES: match VIP</text>
<rect x="256" y="106" width="208" height="34" rx="7" class="box"/>
<text x="360.0" y="127.5" text-anchor="middle" class="s">KUBE-SVC: pick 1 of 3</text>
<rect x="256" y="156" width="208" height="34" rx="7" class="acc"/>
<text x="360.0" y="177.5" text-anchor="middle" class="b">KUBE-SEP: DNAT</text>
<line x1="360" y1="90" x2="360" y2="104" class="ar"/>
<line x1="360" y1="140" x2="360" y2="154" class="ar"/>
<line x1="170" y1="65" x2="254" y2="73" class="ar"/>
<rect x="550" y="148" width="150" height="50" rx="7" class="acc"/>
<text x="625.0" y="168.5" text-anchor="middle" class="b">web-2</text>
<text x="625.0" y="186.5" text-anchor="middle" class="m">10.244.2.7</text>
<line x1="464" y1="173" x2="548" y2="173" class="ar"/>
<text x="360" y="248" text-anchor="middle" class="m">before: 10.244.1.2 to 10.96.45.10:80</text>
<text x="360" y="270" text-anchor="middle" class="ma">after DNAT: 10.244.1.2 to 10.244.2.7:8080</text>
<text x="360" y="300" text-anchor="middle" class="c">conntrack remembers the mapping and reverses it for replies</text>
</g>
</svg>
<figcaption>How a ClusterIP works. The virtual IP exists only as a rule: the destination is rewritten on the client's own node before the packet is routed.</figcaption>
</figure>

When a Pod sends a packet to `10.96.45.10:80`, the packet passes through **netfilter** in the kernel of the node the client Pod runs on. A rule there matches the virtual IP and port, picks one endpoint, and performs **DNAT** (destination NAT), rewriting the destination to that Pod's real IP and target port. From then on, the packet is ordinary Pod-to-Pod traffic and follows the routes from Chapter 6. The kernel's connection tracker, **conntrack**, records the translation so that replies are rewritten back and the client only ever sees the virtual IP.

Two consequences are worth remembering:

- **Load balancing is per connection, not per request.** The endpoint is chosen when a connection starts. A client that keeps one long-lived connection, as gRPC and HTTP/2 often do, sticks to one Pod. Balancing those requests needs an L7 proxy or a service mesh.
- **Load balancing happens at the client.** Every node has the full rule set, so there is no central balancer to overload or fail.

## kube-proxy and its modes

**kube-proxy** runs on every node, usually as a DaemonSet. It watches Services and EndpointSlices and translates them into kernel rules. It has three modes:

- **iptables** (the default). Rules are chains of iptables entries, as shown below. Simple and universal, but rule updates and the first packet of a connection slow down as the number of Services grows into the thousands.
- **nftables** (generally available since 1.33). Uses the newer nftables framework and verdict maps, so lookups take roughly constant time and updates are incremental. It is the recommended mode for new clusters, although iptables remains the default for compatibility. The project plans to make nftables the default in a future release.
- **IPVS.** Uses the kernel's IP Virtual Server load balancer. It was once the answer to iptables scaling, but nftables now matches it, and IPVS is expected to be phased out.

Some CNI plugins, most notably Cilium, replace kube-proxy entirely with eBPF programs. The behavior is the same; only the machinery changes.

In iptables mode, the rules for one Service with three endpoints look like this:

<figure class="diagram">
<svg viewBox="0 0 720 270" style="width:100%;height:auto;max-width:720px" role="img" aria-label="iptables chains for a Service with three endpoints: KUBE-SERVICES jumps to KUBE-SVC-WEB, which jumps to the first endpoint with probability one third, the second with one half of the rest, and the third otherwise." xmlns="http://www.w3.org/2000/svg">
<style>
.k7d{ font-family:var(--sans),sans-serif; }
.k7d .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k7d .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k7d .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k7d .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k7d .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k7d .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k7d .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k7d .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k7d .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k7d .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k7d .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k7d .s{ fill:var(--text-dim); font-size:13px; }
.k7d .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k7d .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k7d .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k7d .c{ fill:var(--text-faint); font-size:12.5px; }
.k7d .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k7d .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k7d .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k7d-a); }
.k7d .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k7d-a); marker-start:url(#k7d-s); }
.k7d .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k7d-g); }
.k7d .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k7d-a); }
.k7d .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k7d .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k7d-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k7d-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k7d-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k7d">
<rect x="20" y="110" width="170" height="44" rx="7" class="pnl"/>
<text x="105.0" y="136.5" text-anchor="middle" class="m">KUBE-SERVICES</text>
<rect x="250" y="110" width="170" height="44" rx="7" class="acc"/>
<text x="335.0" y="136.5" text-anchor="middle" class="m">KUBE-SVC-WEB</text>
<line x1="190" y1="132" x2="248" y2="132" class="ar"/>
<text x="220" y="176" text-anchor="middle" class="ma">dst 10.96.45.10:80</text>
<rect x="500" y="30" width="200" height="52" rx="7" class="pnl"/>
<text x="600.0" y="51.5" text-anchor="middle" class="m">KUBE-SEP-1</text>
<text x="600.0" y="69.5" text-anchor="middle" class="c">DNAT 10.244.1.4</text>
<rect x="500" y="110" width="200" height="52" rx="7" class="pnl"/>
<text x="600.0" y="131.5" text-anchor="middle" class="m">KUBE-SEP-2</text>
<text x="600.0" y="149.5" text-anchor="middle" class="c">DNAT 10.244.2.7</text>
<rect x="500" y="190" width="200" height="52" rx="7" class="pnl"/>
<text x="600.0" y="211.5" text-anchor="middle" class="m">KUBE-SEP-3</text>
<text x="600.0" y="229.5" text-anchor="middle" class="c">DNAT 10.244.1.9</text>
<line x1="420" y1="124" x2="498" y2="58" class="ar"/>
<line x1="420" y1="132" x2="498" y2="136" class="ar"/>
<line x1="420" y1="140" x2="498" y2="214" class="ar"/>
<text x="452" y="66" text-anchor="middle" class="c">p = 1/3</text>
<text x="459" y="124" text-anchor="middle" class="c">p = 1/2</text>
<text x="452" y="198" text-anchor="middle" class="c">rest</text>
<text x="360" y="254" text-anchor="middle" class="c">each endpoint ends up with an equal share of new connections</text>
</g>
</svg>
<figcaption>kube-proxy's iptables rules for one Service. The probabilities cascade so that every endpoint gets one third of new connections.</figcaption>
</figure>

The probabilities cascade. The first rule matches one time in three. If it does not, the second matches half of what is left, which is again one third overall, and the last takes the rest. You can read the real rules in your lab:

```bash
docker exec lab-worker iptables -t nat -L KUBE-SERVICES -n | grep web
docker exec lab-worker iptables -t nat -L KUBE-SVC-<hash> -n
```

To try nftables mode, create a kind cluster with `networking: {kubeProxyMode: nftables}` in its config, then inspect the rules with `nft list table ip kube-proxy` inside a node.

## Service types

So far the Service is only reachable from inside the cluster. The `type` field adds ways in from outside, and each type builds on the one before.

<figure class="diagram">
<svg viewBox="0 0 720 264" style="width:100%;height:auto;max-width:720px" role="img" aria-label="Service types nest: a LoadBalancer Service is also a NodePort Service, which is also a ClusterIP Service." xmlns="http://www.w3.org/2000/svg">
<style>
.k7e{ font-family:var(--sans),sans-serif; }
.k7e .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k7e .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k7e .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k7e .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k7e .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k7e .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k7e .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k7e .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k7e .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k7e .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k7e .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k7e .s{ fill:var(--text-dim); font-size:13px; }
.k7e .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k7e .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k7e .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k7e .c{ fill:var(--text-faint); font-size:12.5px; }
.k7e .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k7e .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k7e .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k7e-a); }
.k7e .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k7e-a); marker-start:url(#k7e-s); }
.k7e .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k7e-g); }
.k7e .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k7e-a); }
.k7e .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k7e .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k7e-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k7e-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k7e-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k7e">
<rect x="20" y="16" width="680" height="234" rx="7" class="zone"/>
<text x="40" y="42" class="b">LoadBalancer</text>
<text x="160" y="42" class="c">an external IP from the cloud (or cloud-provider-kind)</text>
<rect x="60" y="60" width="600" height="176" rx="7" class="pnl"/>
<text x="80" y="86" class="b">NodePort</text>
<text x="170" y="86" class="c">a port in 30000-32767, open on every node</text>
<rect x="100" y="104" width="520" height="118" rx="7" class="acc"/>
<text x="120" y="130" class="b">ClusterIP</text>
<text x="210" y="130" class="s">a virtual IP, reachable inside the cluster</text>
<rect x="150" y="156" width="120" height="44" rx="7" class="pnl"/>
<text x="210.0" y="182.5" text-anchor="middle" class="s">Pod</text>
<rect x="300" y="156" width="120" height="44" rx="7" class="pnl"/>
<text x="360.0" y="182.5" text-anchor="middle" class="s">Pod</text>
<rect x="450" y="156" width="120" height="44" rx="7" class="pnl"/>
<text x="510.0" y="182.5" text-anchor="middle" class="s">Pod</text>
</g>
</svg>
<figcaption>Service types build on each other. Each outer type adds one more way in, and all of them end at the same ClusterIP machinery.</figcaption>
</figure>

- **ClusterIP** (default): a virtual IP reachable from inside the cluster.
- **NodePort**: additionally opens the same port, from the range 30000 to 32767, on **every** node. Traffic to any node's IP on that port is forwarded to the Service.
- **LoadBalancer**: additionally asks the cloud provider, through the cloud-controller-manager, for an external load balancer that sends traffic to the nodes. Its address appears in the Service's `status.loadBalancer`.
- **ExternalName**: no proxying at all. The Service's DNS name becomes a CNAME for an external host name.
- **Headless** (`clusterIP: None`): no virtual IP. DNS returns the Pod IPs directly. Chapter 8 explains why that is useful.

On a laptop there is no cloud to provide load balancers, so a LoadBalancer Service stays `<pending>` forever. kind fixes this with **cloud-provider-kind**, a small controller that you run beside your clusters:

```bash
go install sigs.k8s.io/cloud-provider-kind@latest   # or: brew install cloud-provider-kind
cloud-provider-kind                                  # leave it running in its own terminal
```

It watches for LoadBalancer Services and starts a load balancer container on the kind network for each one, then fills in the external IP. On macOS and Windows, where the kind network is not directly reachable from the host, run it with `--enable-lb-port-mapping`. If you want the control plane node to receive load balancer traffic too, remove its exclusion label:

```bash
kubectl label node lab-control-plane node.kubernetes.io/exclude-from-external-load-balancers-
```

## External traffic policy

When traffic enters through a NodePort or LoadBalancer, it may arrive at a node that has no Pod for the Service. What happens next is set by `externalTrafficPolicy`.

<figure class="diagram">
<svg viewBox="0 0 720 296" style="width:100%;height:auto;max-width:720px" role="img" aria-label="With externalTrafficPolicy Cluster, any node accepts traffic and forwards it, with SNAT, to a node that has a Pod, losing the client IP. With Local, only nodes with a ready Pod receive traffic and the client IP is preserved." xmlns="http://www.w3.org/2000/svg">
<style>
.k7f{ font-family:var(--sans),sans-serif; }
.k7f .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k7f .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k7f .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k7f .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k7f .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k7f .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k7f .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k7f .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k7f .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k7f .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k7f .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k7f .s{ fill:var(--text-dim); font-size:13px; }
.k7f .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k7f .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k7f .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k7f .c{ fill:var(--text-faint); font-size:12.5px; }
.k7f .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k7f .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k7f .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k7f-a); }
.k7f .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k7f-a); marker-start:url(#k7f-s); }
.k7f .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k7f-g); }
.k7f .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k7f-a); }
.k7f .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k7f .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k7f-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k7f-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k7f-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k7f">
<text x="180" y="20" text-anchor="middle" class="b">Cluster (default)</text>
<rect x="110" y="34" width="140" height="36" rx="7" class="box"/>
<text x="180.0" y="56.5" text-anchor="middle" class="s">client</text>
<rect x="40" y="120" width="130" height="92" rx="7" class="pnl"/>
<text x="105" y="142" text-anchor="middle" class="b">node A</text>
<rect x="55" y="158" width="100" height="36" rx="7" class="acc"/>
<text x="105.0" y="180.5" text-anchor="middle" class="b">Pod</text>
<rect x="190" y="120" width="130" height="92" rx="7" class="pnl"/>
<text x="255" y="142" text-anchor="middle" class="b">node B</text>
<text x="255" y="176" text-anchor="middle" class="c">no Pod</text>
<text x="540" y="20" text-anchor="middle" class="b">Local</text>
<rect x="470" y="34" width="140" height="36" rx="7" class="box"/>
<text x="540.0" y="56.5" text-anchor="middle" class="s">client</text>
<rect x="400" y="120" width="130" height="92" rx="7" class="pnl"/>
<text x="465" y="142" text-anchor="middle" class="b">node A</text>
<rect x="415" y="158" width="100" height="36" rx="7" class="acc"/>
<text x="465.0" y="180.5" text-anchor="middle" class="b">Pod</text>
<rect x="550" y="120" width="130" height="92" rx="7" class="pnl"/>
<text x="615" y="142" text-anchor="middle" class="b">node B</text>
<text x="615" y="176" text-anchor="middle" class="c">no Pod</text>
<line x1="180" y1="70" x2="245" y2="118" class="ar"/>
<line x1="190" y1="176" x2="137" y2="176" class="ar"/>
<text x="163" y="230" text-anchor="middle" class="a">SNAT</text>
<text x="180" y="256" text-anchor="middle" class="c">any node can take the traffic</text>
<text x="180" y="276" text-anchor="middle" class="c">extra hop; client IP is lost</text>
<line x1="520" y1="70" x2="470" y2="118" class="ar"/>
<line x1="580" y1="70" x2="615" y2="118" class="gd"/>
<text x="615" y="196" text-anchor="middle" class="c">skipped</text>
<text x="540" y="256" text-anchor="middle" class="c">only nodes with a ready Pod</text>
<text x="540" y="276" text-anchor="middle" class="c">no extra hop; client IP is kept</text>
</g>
</svg>
<figcaption>externalTrafficPolicy. Cluster spreads load evenly but hides the client's address; Local keeps it, at the cost of possibly uneven load.</figcaption>
</figure>

With **`Cluster`** (the default), the node forwards the traffic to a Pod elsewhere. To make sure replies come back through it, the node also rewrites the **source** address to its own (SNAT), so the Pod sees the node's IP instead of the client's. With **`Local`**, a node only accepts traffic if it has a ready local Pod; the cloud load balancer health-checks each node and skips the rest. The client IP survives, which matters for logging and access control, but load is only as even as your Pod placement.

A related field, `internalTrafficPolicy: Local`, keeps in-cluster traffic on the same node, which suits node-local agents such as a logging daemon.

## Summary

- A **Service** gives a set of Pods, chosen by selector, one stable **ClusterIP** and port.
- **EndpointSlices** list the ready backends; readiness probes control membership.
- The ClusterIP is not owned by anything. **kube-proxy** programs **DNAT** rules on every node, and **conntrack** reverses them for replies. Balancing is per connection and happens on the client's node.
- kube-proxy runs in **iptables** (default), **nftables** (recommended, GA since 1.33), or **IPVS** mode, or is replaced by an eBPF data plane.
- **NodePort** and **LoadBalancer** add external entry points; **externalTrafficPolicy** trades even load against preserving the client IP.

Services give you stable IPs, but nobody wants to hard-code IPs. The next chapter covers the DNS layer that turns Service names into addresses.

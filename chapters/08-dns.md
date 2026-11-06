# Chapter 8: Cluster DNS and Service Discovery

Services give you stable virtual IPs, but applications should not contain IP addresses at all. They should say `http://api` and let the platform work out the rest. Every Kubernetes cluster runs a DNS server for exactly this. This chapter covers how it works, how names are formed, and the one configuration default that causes most DNS surprises.

## CoreDNS and the request path

Cluster DNS is provided by **CoreDNS**, running as an ordinary Deployment in `kube-system`. It sits behind a Service that, for historical reasons, is still called `kube-dns`. In a kind cluster that Service has the address `10.96.0.10`.

<figure class="diagram">
<svg viewBox="0 0 720 256" style="width:100%;height:auto;max-width:720px" role="img" aria-label="A Pod&#x27;s resolv.conf points at the kube-dns Service, which kube-proxy spreads across CoreDNS Pods. CoreDNS answers cluster names from the API server and forwards other names upstream." xmlns="http://www.w3.org/2000/svg">
<style>
.k8a{ font-family:var(--sans),sans-serif; }
.k8a .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k8a .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k8a .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k8a .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k8a .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k8a .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k8a .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k8a .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k8a .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k8a .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k8a .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k8a .s{ fill:var(--text-dim); font-size:13px; }
.k8a .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k8a .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k8a .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k8a .c{ fill:var(--text-faint); font-size:12.5px; }
.k8a .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k8a .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k8a .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k8a-a); }
.k8a .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k8a-a); marker-start:url(#k8a-s); }
.k8a .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k8a-g); }
.k8a .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k8a-a); }
.k8a .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k8a .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k8a-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k8a-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k8a-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k8a">
<rect x="20" y="40" width="190" height="76" rx="7" class="pnl"/>
<text x="115.0" y="64.5" text-anchor="middle" class="b">app Pod</text>
<text x="115.0" y="82.5" text-anchor="middle" class="m">/etc/resolv.conf</text>
<text x="115.0" y="100.5" text-anchor="middle" class="m">nameserver 10.96.0.10</text>
<rect x="270" y="52" width="170" height="52" rx="7" class="dark"/>
<text x="355.0" y="73.5" text-anchor="middle" class="on">kube-dns Service</text>
<text x="355.0" y="91.5" text-anchor="middle" class="on">10.96.0.10:53</text>
<line x1="210" y1="78" x2="268" y2="78" class="ar"/>
<rect x="500" y="24" width="200" height="46" rx="7" class="acc"/>
<text x="600.0" y="42.5" text-anchor="middle" class="b">CoreDNS Pod</text>
<text x="600.0" y="60.5" text-anchor="middle" class="s">kube-system</text>
<rect x="500" y="84" width="200" height="46" rx="7" class="acc"/>
<text x="600.0" y="102.5" text-anchor="middle" class="b">CoreDNS Pod</text>
<text x="600.0" y="120.5" text-anchor="middle" class="s">kube-system</text>
<line x1="440" y1="70" x2="498" y2="48" class="ar"/>
<line x1="440" y1="86" x2="498" y2="106" class="ar"/>
<rect x="500" y="190" width="200" height="46" rx="7" class="pnl"/>
<text x="600.0" y="217.5" text-anchor="middle" class="s">API server</text>
<line x1="600" y1="130" x2="600" y2="188" class="ag"/>
<text x="612" y="164" class="c">watches Services</text>
<rect x="270" y="190" width="170" height="46" rx="7" class="box"/>
<text x="355.0" y="208.5" text-anchor="middle" class="b">upstream DNS</text>
<text x="355.0" y="226.5" text-anchor="middle" class="c">the node's resolver</text>
<line x1="510" y1="130" x2="442" y2="198" class="ag"/>
<text x="455" y="158" text-anchor="end" class="c">other names</text>
</g>
</svg>
<figcaption>The DNS path. Cluster names are answered from Kubernetes objects; everything else is forwarded to the DNS server the node uses.</figcaption>
</figure>

The kubelet writes every Pod's `/etc/resolv.conf` so that the Pod's resolver sends queries to that Service address. The query then follows exactly the Service path from Chapter 7: kube-proxy rules on the Pod's node DNAT it to one of the CoreDNS Pods.

CoreDNS answers in one of two ways. For names inside the cluster domain, its **kubernetes** plugin answers from Services and EndpointSlices, which it watches through the API server just like kube-proxy does. Everything else, such as `github.com`, is **forwarded** to the upstream resolver the node itself uses. The configuration lives in the `coredns` ConfigMap, called the Corefile:

```text
.:53 {
    errors
    health
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}
```

```bash
kubectl -n kube-system get deploy coredns
kubectl -n kube-system get svc kube-dns
kubectl -n kube-system get configmap coredns -o yaml
```

## How names are formed

Every Service gets an A record (and an AAAA record in IPv6 clusters) under a predictable name.

<figure class="diagram">
<svg viewBox="0 0 720 200" style="width:100%;height:auto;max-width:720px" role="img" aria-label="The fully qualified name web.shop.svc.cluster.local is made of the Service name, the namespace, the fixed label svc, and the cluster domain." xmlns="http://www.w3.org/2000/svg">
<style>
.k8b{ font-family:var(--sans),sans-serif; }
.k8b .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k8b .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k8b .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k8b .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k8b .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k8b .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k8b .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k8b .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k8b .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k8b .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k8b .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k8b .s{ fill:var(--text-dim); font-size:13px; }
.k8b .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k8b .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k8b .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k8b .c{ fill:var(--text-faint); font-size:12.5px; }
.k8b .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k8b .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k8b .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k8b-a); }
.k8b .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k8b-a); marker-start:url(#k8b-s); }
.k8b .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k8b-g); }
.k8b .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k8b-a); }
.k8b .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k8b .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k8b-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k8b-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k8b-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k8b">
<rect x="60" y="26" width="110" height="40" rx="7" class="pnl"/>
<text x="115.0" y="50.5" text-anchor="middle" class="m">web</text>
<rect x="195" y="26" width="120" height="40" rx="7" class="pnl"/>
<text x="255.0" y="50.5" text-anchor="middle" class="m">shop</text>
<rect x="340" y="26" width="100" height="40" rx="7" class="box"/>
<text x="390.0" y="50.5" text-anchor="middle" class="m">svc</text>
<rect x="465" y="26" width="190" height="40" rx="7" class="acc"/>
<text x="560.0" y="50.5" text-anchor="middle" class="m">cluster.local</text>
<text x="182" y="52" text-anchor="middle" class="b">.</text>
<text x="327" y="52" text-anchor="middle" class="b">.</text>
<text x="452" y="52" text-anchor="middle" class="b">.</text>
<text x="115" y="90" text-anchor="middle" class="c">Service name</text>
<text x="255" y="90" text-anchor="middle" class="c">namespace</text>
<text x="390" y="90" text-anchor="middle" class="c">always svc</text>
<text x="560" y="90" text-anchor="middle" class="c">cluster domain</text>
<text x="60" y="130" class="m">web</text>
<text x="330" y="130" class="s">from the same namespace</text>
<text x="60" y="154" class="m">web.shop</text>
<text x="330" y="154" class="s">from any namespace</text>
<text x="60" y="178" class="m">web.shop.svc.cluster.local</text>
<text x="330" y="178" class="s">fully qualified, from anywhere</text>
</g>
</svg>
<figcaption>Anatomy of a Service name. The search list in each Pod lets you use the short forms.</figcaption>
</figure>

The cluster domain is `cluster.local` unless the cluster was built with something else. Named ports also get **SRV** records, such as `_http._tcp.web.shop.svc.cluster.local`, which return the port number along with the target.

The short forms work because of the **search list** in each Pod's `resolv.conf`. For a Pod in namespace `shop` it looks like this:

```text
nameserver 10.96.0.10
search shop.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

So a query for `web` is expanded to `web.shop.svc.cluster.local` and answered from the Pod's own namespace. A query for `web.payments` becomes `web.payments.svc.cluster.local`, reaching across namespaces. Namespaces do not isolate DNS: any Pod can resolve any Service name. Isolation, if you want it, is a job for NetworkPolicies (Chapter 10).

## The ndots trap

The line `options ndots:5` deserves attention. It tells the resolver: if a name has **fewer than five dots**, try it with each search domain before trying it as written.

<figure class="diagram">
<svg viewBox="0 0 720 290" style="width:100%;height:auto;max-width:720px" role="img" aria-label="Looking up api from namespace shop succeeds on the first search domain. Looking up example.com, which has fewer than five dots, first tries three cluster suffixes that all fail before the real name is tried." xmlns="http://www.w3.org/2000/svg">
<style>
.k8c{ font-family:var(--sans),sans-serif; }
.k8c .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k8c .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k8c .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k8c .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k8c .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k8c .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k8c .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k8c .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k8c .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k8c .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k8c .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k8c .s{ fill:var(--text-dim); font-size:13px; }
.k8c .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k8c .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k8c .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k8c .c{ fill:var(--text-faint); font-size:12.5px; }
.k8c .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k8c .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k8c .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k8c-a); }
.k8c .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k8c-a); marker-start:url(#k8c-s); }
.k8c .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k8c-g); }
.k8c .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k8c-a); }
.k8c .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k8c .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k8c-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k8c-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k8c-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k8c">
<text x="170" y="24" text-anchor="middle" class="b">lookup: api</text>
<rect x="20" y="40" width="300" height="34" rx="7" class="acc"/>
<text x="32" y="62" class="m">api.shop.svc.cluster.local</text>
<text x="310" y="62" text-anchor="end" class="b">hit</text>
<text x="170" y="100" text-anchor="middle" class="c">answered on the first try</text>
<text x="530" y="24" text-anchor="middle" class="b">lookup: example.com (1 dot, ndots is 5)</text>
<rect x="360" y="40" width="340" height="34" rx="7" class="box"/>
<text x="372" y="62" class="m">example.com.shop.svc.cluster.local</text>
<text x="690" y="62" text-anchor="end" class="c">miss</text>
<rect x="360" y="80" width="340" height="34" rx="7" class="box"/>
<text x="372" y="102" class="m">example.com.svc.cluster.local</text>
<text x="690" y="102" text-anchor="end" class="c">miss</text>
<rect x="360" y="120" width="340" height="34" rx="7" class="box"/>
<text x="372" y="142" class="m">example.com.cluster.local</text>
<text x="690" y="142" text-anchor="end" class="c">miss</text>
<rect x="360" y="160" width="340" height="34" rx="7" class="acc"/>
<text x="372" y="182" class="m">example.com</text>
<text x="690" y="182" text-anchor="end" class="b">hit</text>
<text x="530" y="216" text-anchor="middle" class="c">three wasted queries before the real one</text>
<text x="360" y="252" text-anchor="middle" class="m">search shop.svc.cluster.local svc.cluster.local cluster.local</text>
<text x="360" y="274" text-anchor="middle" class="m">options ndots:5</text>
</g>
</svg>
<figcaption>The ndots trap. Any name with fewer than five dots is tried against every search domain first. A trailing dot, as in example.com., skips the search list.</figcaption>
</figure>

That is perfect for short cluster names and wasteful for external ones. Every lookup of `api.github.com`, which has two dots, first produces three failed queries against the cluster suffixes. Usually this just costs a few milliseconds, but in a busy cluster it multiplies DNS load several times over and makes CoreDNS a bottleneck.

Several fixes exist, from simplest to broadest:

- Write external names **fully qualified with a trailing dot** (`api.github.com.`) in configuration that you control.
- Lower `ndots` for one Pod with `dnsConfig`:

```yaml
spec:
  dnsConfig:
    options:
    - name: ndots
      value: "2"
```

- Run **NodeLocal DNSCache**, a DaemonSet that puts a caching resolver on every node, so repeated and failed lookups are answered locally instead of crossing the network to CoreDNS.

## Headless Services

Sometimes a client should not be load-balanced through a single virtual IP. It wants the address of every backend, to pick one itself or to talk to all of them. A **headless Service**, declared with `clusterIP: None`, provides that.

<figure class="diagram">
<svg viewBox="0 0 720 260" style="width:100%;height:auto;max-width:720px" role="img" aria-label="A normal Service name resolves to one ClusterIP. A headless Service name resolves to the IPs of every ready Pod, and StatefulSet Pods behind it also get individual names." xmlns="http://www.w3.org/2000/svg">
<style>
.k8d{ font-family:var(--sans),sans-serif; }
.k8d .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k8d .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k8d .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k8d .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k8d .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k8d .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k8d .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k8d .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k8d .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k8d .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k8d .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k8d .s{ fill:var(--text-dim); font-size:13px; }
.k8d .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k8d .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k8d .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k8d .c{ fill:var(--text-faint); font-size:12.5px; }
.k8d .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k8d .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k8d .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k8d-a); }
.k8d .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k8d-a); marker-start:url(#k8d-s); }
.k8d .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k8d-g); }
.k8d .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k8d-a); }
.k8d .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k8d .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k8d-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k8d-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k8d-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k8d">
<text x="170" y="20" text-anchor="middle" class="b">normal Service</text>
<text x="550" y="20" text-anchor="middle" class="b">headless (clusterIP: None)</text>
<rect x="20" y="32" width="300" height="38" rx="7" class="box"/>
<text x="170.0" y="55.5" text-anchor="middle" class="m">dig web.shop.svc.cluster.local</text>
<rect x="20" y="100" width="300" height="38" rx="7" class="acc"/>
<text x="170.0" y="123.5" text-anchor="middle" class="m">A 10.96.45.10 (the ClusterIP)</text>
<line x1="170" y1="70" x2="170" y2="98" class="ar"/>
<text x="170" y="164" text-anchor="middle" class="c">kube-proxy then picks a Pod</text>
<rect x="400" y="32" width="300" height="38" rx="7" class="box"/>
<text x="550.0" y="55.5" text-anchor="middle" class="m">dig db.shop.svc.cluster.local</text>
<rect x="400" y="100" width="300" height="82" rx="7" class="acc"/>
<text x="550.0" y="127.5" text-anchor="middle" class="m">A 10.244.1.4</text>
<text x="550.0" y="145.5" text-anchor="middle" class="m">A 10.244.2.7</text>
<text x="550.0" y="163.5" text-anchor="middle" class="m">A 10.244.1.9</text>
<line x1="550" y1="70" x2="550" y2="98" class="ar"/>
<text x="550" y="212" text-anchor="middle" class="ma">db-0.db.shop.svc.cluster.local</text>
<text x="550" y="236" text-anchor="middle" class="c">StatefulSet Pods also get their own names</text>
</g>
</svg>
<figcaption>Normal versus headless Services. Headless DNS hands the client every Pod address, so the client can choose, which is what clustered databases need.</figcaption>
</figure>

With no virtual IP, kube-proxy ignores the Service entirely, and DNS returns one A record per ready Pod instead. When the Service is the `serviceName` of a StatefulSet (Chapter 5), each Pod also gets its own stable name, such as `db-0.db.shop.svc.cluster.local`. This is how clustered databases find their peers: `db-1` can always reach `db-0` by name, even after `db-0` has been rescheduled with a new IP.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db
spec:
  clusterIP: None
  selector:
    app: db
  ports:
  - port: 5432
```

## DNS policy

A Pod's `dnsPolicy` controls how its `resolv.conf` is built:

| Policy | Behavior |
|---|---|
| `ClusterFirst` (default) | Use cluster DNS; forward other names upstream |
| `Default` | Inherit the node's resolver; no cluster names |
| `ClusterFirstWithHostNet` | Cluster DNS for Pods using `hostNetwork: true` |
| `None` | Ignore everything and use only `dnsConfig` |

The naming is unfortunate: `Default` is **not** the default. `ClusterFirst` is.

## Hands on: debugging DNS

Most "the network is broken" reports in Kubernetes turn out to be DNS. Use the `netshoot` Pod from Chapter 6 to check each step:

```bash
kubectl exec -it netshoot -- cat /etc/resolv.conf
kubectl exec -it netshoot -- dig +search web            # uses the search list
kubectl exec -it netshoot -- dig web.default.svc.cluster.local
kubectl exec -it netshoot -- dig @10.96.0.10 example.com # ask CoreDNS directly
kubectl -n kube-system logs deploy/coredns
```

If a Service name does not resolve, check that the Service exists in the namespace you think it does. If it resolves but connections fail, the problem is not DNS. Check the EndpointSlices (are any Pods ready?) and then any NetworkPolicies.

## Summary

- **CoreDNS** runs in `kube-system` behind the `kube-dns` Service. Pods query it through ordinary Service routing.
- Services are named `service.namespace.svc.cluster.local`; the Pod's **search list** makes `service` and `service.namespace` work too.
- **`ndots:5`** makes short names convenient and external names expensive. Use trailing dots, `dnsConfig`, or NodeLocal DNSCache when it matters.
- A **headless Service** (`clusterIP: None`) returns Pod IPs instead of a virtual IP and gives StatefulSet Pods stable individual names.
- DNS does not isolate namespaces; NetworkPolicies do.

Everything so far has been inside the cluster. The next chapter brings traffic in from the outside world.

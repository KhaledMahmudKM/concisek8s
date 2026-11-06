# Chapter 6: The Network Model and CNI

Docker's default networking gives each container a private address behind NAT on one host. Kubernetes asks for something very different and much simpler to program against: a flat network where every Pod has a real IP and can reach every other Pod directly. This chapter explains that model, then follows packets through the Linux plumbing that implements it.

## The model: three rules

Kubernetes does not ship a network implementation. It defines a contract, and plugins fulfil it.

<figure class="diagram">
<svg viewBox="0 0 720 270" style="width:100%;height:auto;max-width:720px" role="img" aria-label="Two nodes with two Pods each. Every Pod has its own IP from its node&#x27;s Pod CIDR, and any Pod can reach any other Pod directly, without NAT." xmlns="http://www.w3.org/2000/svg">
<style>
.k6a{ font-family:var(--sans),sans-serif; }
.k6a .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k6a .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k6a .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k6a .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k6a .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k6a .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k6a .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k6a .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k6a .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k6a .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k6a .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k6a .s{ fill:var(--text-dim); font-size:13px; }
.k6a .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k6a .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k6a .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k6a .c{ fill:var(--text-faint); font-size:12.5px; }
.k6a .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k6a .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k6a .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k6a-a); }
.k6a .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k6a-a); marker-start:url(#k6a-s); }
.k6a .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k6a-g); }
.k6a .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k6a-a); }
.k6a .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k6a .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k6a-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k6a-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k6a-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k6a">
<rect x="20" y="36" width="320" height="168" rx="7" class="zone"/>
<text x="36" y="60" class="b">node 1</text>
<text x="324" y="60" text-anchor="end" class="ma">10.244.1.0/24</text>
<rect x="380" y="36" width="320" height="168" rx="7" class="zone"/>
<text x="396" y="60" class="b">node 2</text>
<text x="684" y="60" text-anchor="end" class="ma">10.244.2.0/24</text>
<line x1="162" y1="114" x2="198" y2="114" class="ar2"/>
<line x1="322" y1="114" x2="398" y2="114" class="ar2"/>
<line x1="522" y1="114" x2="558" y2="114" class="ar2"/>
<rect x="40" y="90" width="120" height="48" rx="7" class="acc"/>
<text x="100.0" y="109.5" text-anchor="middle" class="b">Pod A</text>
<text x="100.0" y="127.5" text-anchor="middle" class="m">10.244.1.2</text>
<rect x="200" y="90" width="120" height="48" rx="7" class="acc"/>
<text x="260.0" y="109.5" text-anchor="middle" class="b">Pod B</text>
<text x="260.0" y="127.5" text-anchor="middle" class="m">10.244.1.3</text>
<rect x="400" y="90" width="120" height="48" rx="7" class="acc"/>
<text x="460.0" y="109.5" text-anchor="middle" class="b">Pod C</text>
<text x="460.0" y="127.5" text-anchor="middle" class="m">10.244.2.5</text>
<rect x="560" y="90" width="120" height="48" rx="7" class="acc"/>
<text x="620.0" y="109.5" text-anchor="middle" class="b">Pod D</text>
<text x="620.0" y="127.5" text-anchor="middle" class="m">10.244.2.6</text>
<rect x="40" y="152" width="280" height="34" rx="7" class="box"/>
<text x="180.0" y="173.5" text-anchor="middle" class="s">kubelet and node agents</text>
<rect x="400" y="152" width="280" height="34" rx="7" class="box"/>
<text x="540.0" y="173.5" text-anchor="middle" class="s">kubelet and node agents</text>
<text x="130" y="238" text-anchor="middle" class="s">1  every Pod gets its own IP</text>
<text x="360" y="238" text-anchor="middle" class="s">2  Pod to Pod on any node, no NAT</text>
<text x="590" y="238" text-anchor="middle" class="s">3  agents reach local Pods</text>
</g>
</svg>
<figcaption>The Kubernetes network model in three rules. The model says what must be true; the CNI plugin decides how.</figcaption>
</figure>

1. **Every Pod gets its own IP address.** Containers in a Pod share it (Chapter 4).
2. **Pods can reach all other Pods, on any node, without NAT.** The source address a Pod sees on an incoming packet is the real Pod IP of the sender.
3. **Agents on a node**, such as the kubelet, **can reach every Pod on that node.**

The payoff is that Pods behave like hosts on a network. An application can bind a port and advertise its own IP, and no one has to manage port mappings the way `docker run -p` requires. The cost is that someone has to make thousands of short-lived IPs routable across many machines. That someone is the CNI plugin.

Two address ranges matter in every cluster. The **Pod CIDR** (in kind, `10.244.0.0/16` by default) is split into per-node blocks, usually `/24`s, from which each node hands out Pod IPs. The **Service CIDR** (in kind, `10.96.0.0/16`) holds the virtual IPs of Chapter 7. Neither range should overlap your real networks.

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,PODCIDR:.spec.podCIDR
```

## CNI: who wires up a Pod

The **Container Network Interface (CNI)** is a small specification: a plugin is an executable that the container runtime calls with a command (`ADD`, `DEL`, `CHECK`) and a JSON configuration. When a Pod lands on a node, this happens:

<figure class="diagram">
<svg viewBox="0 0 720 290" style="width:100%;height:auto;max-width:720px" role="img" aria-label="When a Pod is scheduled, the kubelet asks containerd to create a sandbox with an empty network namespace, then the runtime calls the CNI plugin, which creates a veth pair, assigns an IP, adds routes, and returns the IP." xmlns="http://www.w3.org/2000/svg">
<style>
.k6b{ font-family:var(--sans),sans-serif; }
.k6b .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k6b .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k6b .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k6b .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k6b .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k6b .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k6b .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k6b .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k6b .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k6b .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k6b .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k6b .s{ fill:var(--text-dim); font-size:13px; }
.k6b .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k6b .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k6b .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k6b .c{ fill:var(--text-faint); font-size:12.5px; }
.k6b .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k6b .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k6b .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k6b-a); }
.k6b .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k6b-a); marker-start:url(#k6b-s); }
.k6b .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k6b-g); }
.k6b .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k6b-a); }
.k6b .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k6b .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k6b-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k6b-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k6b-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k6b">
<rect x="20" y="40" width="140" height="46" rx="7" class="pnl"/>
<text x="90.0" y="58.5" text-anchor="middle" class="b">kubelet</text>
<text x="90.0" y="76.5" text-anchor="middle" class="c">Pod scheduled here</text>
<rect x="190" y="40" width="150" height="46" rx="7" class="pnl"/>
<text x="265.0" y="58.5" text-anchor="middle" class="b">containerd</text>
<text x="265.0" y="76.5" text-anchor="middle" class="c">creates sandbox</text>
<rect x="370" y="40" width="170" height="46" rx="7" class="pnl"/>
<text x="455.0" y="58.5" text-anchor="middle" class="b">network namespace</text>
<text x="455.0" y="76.5" text-anchor="middle" class="c">empty, no interfaces</text>
<rect x="570" y="40" width="130" height="46" rx="7" class="acc"/>
<text x="635.0" y="58.5" text-anchor="middle" class="b">CNI plugin</text>
<text x="635.0" y="76.5" text-anchor="middle" class="s">called with ADD</text>
<line x1="160" y1="63" x2="188" y2="63" class="ar"/>
<line x1="340" y1="63" x2="368" y2="63" class="ar"/>
<line x1="540" y1="63" x2="568" y2="63" class="ar"/>
<line x1="265" y1="86" x2="285" y2="128" class="ag"/>
<rect x="190" y="130" width="190" height="64" rx="7" class="box"/>
<text x="285.0" y="157.5" text-anchor="middle" class="m">config: /etc/cni/net.d</text>
<text x="285.0" y="175.5" text-anchor="middle" class="m">binaries: /opt/cni/bin</text>
<line x1="635" y1="86" x2="635" y2="128" class="ar"/>
<rect x="410" y="130" width="290" height="118" rx="7" class="pnl"/>
<text x="426" y="152" class="b">The plugin then:</text>
<text x="426" y="176" class="s">1  creates a veth pair</text>
<text x="426" y="196" class="s">2  moves one end into the Pod</text>
<text x="426" y="216" class="s">3  assigns an IP (IPAM)</text>
<text x="426" y="236" class="s">4  adds routes</text>
<path d="M408 236 L85 236 L85 88" class="ag"/>
<text x="250" y="262" text-anchor="middle" class="c">the Pod IP is returned and reported to the API server</text>
</g>
</svg>
<figcaption>Wiring up a Pod. Kubernetes itself never configures Pod networking; it delegates that to whichever CNI plugin the cluster runs.</figcaption>
</figure>

The runtime creates the Pod's sandbox, the pause container from Chapter 4, which owns a fresh network namespace with nothing in it but loopback. It then reads the CNI configuration in `/etc/cni/net.d/` and runs the plugin binary from `/opt/cni/bin/`. The plugin creates the Pod's `eth0`, assigns it an address through its **IPAM** (IP address management) component, and installs routes. When the Pod is deleted, the runtime calls `DEL` and the plugin cleans up.

Because the interface is this small, there are many plugins with very different designs:

| Plugin | Approach |
|---|---|
| **kindnet** | kind's default: veth per Pod, plain routes between nodes |
| **Flannel** | simple overlay (usually VXLAN) between nodes |
| **Calico** | routed with BGP, or overlay; strong NetworkPolicy support |
| **Cilium** | eBPF data plane; can replace kube-proxy entirely |
| **Cloud VPC CNIs** | Pods get addresses straight from the cloud network (AWS VPC CNI, Azure CNI) |

## Inside one node

Here is what the plugin builds. A **veth pair** is a virtual cable: whatever enters one end comes out the other. One end becomes `eth0` inside the Pod's namespace; the other stays in the node's root namespace with a generated name.

<figure class="diagram">
<svg viewBox="0 0 720 320" style="width:100%;height:auto;max-width:720px" role="img" aria-label="Inside one node: each Pod&#x27;s eth0 is one end of a veth pair whose other end sits in the host namespace. The host routing table sends each local Pod IP to its veth and remote Pod CIDRs to the other node." xmlns="http://www.w3.org/2000/svg">
<style>
.k6c{ font-family:var(--sans),sans-serif; }
.k6c .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k6c .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k6c .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k6c .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k6c .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k6c .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k6c .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k6c .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k6c .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k6c .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k6c .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k6c .s{ fill:var(--text-dim); font-size:13px; }
.k6c .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k6c .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k6c .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k6c .c{ fill:var(--text-faint); font-size:12.5px; }
.k6c .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k6c .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k6c .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k6c-a); }
.k6c .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k6c-a); marker-start:url(#k6c-s); }
.k6c .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k6c-g); }
.k6c .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k6c-a); }
.k6c .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k6c .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k6c-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k6c-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k6c-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k6c">
<rect x="20" y="20" width="680" height="290" rx="7" class="zone"/>
<text x="36" y="42" class="b">node lab-worker (host network namespace)</text>
<rect x="40" y="56" width="200" height="64" rx="7" class="acc"/>
<text x="140.0" y="83.5" text-anchor="middle" class="b">Pod A netns</text>
<text x="140.0" y="101.5" text-anchor="middle" class="m">eth0 10.244.1.2</text>
<rect x="260" y="56" width="200" height="64" rx="7" class="acc"/>
<text x="360.0" y="83.5" text-anchor="middle" class="b">Pod B netns</text>
<text x="360.0" y="101.5" text-anchor="middle" class="m">eth0 10.244.1.3</text>
<rect x="500" y="56" width="180" height="64" rx="7" class="hov"/>
<text x="590.0" y="83.5" text-anchor="middle" class="b">eth0 (node)</text>
<text x="590.0" y="101.5" text-anchor="middle" class="m">172.18.0.3</text>
<line x1="140" y1="120" x2="140" y2="150" class="ln"/>
<line x1="360" y1="120" x2="360" y2="150" class="ln"/>
<text x="150" y="140" class="c">veth pair</text>
<text x="370" y="140" class="c">veth pair</text>
<rect x="60" y="150" width="160" height="30" rx="7" class="box"/>
<text x="140.0" y="169.5" text-anchor="middle" class="m">vethA1b2</text>
<rect x="280" y="150" width="160" height="30" rx="7" class="box"/>
<text x="360.0" y="169.5" text-anchor="middle" class="m">vethC3d4</text>
<line x1="140" y1="180" x2="140" y2="200" class="ln"/>
<line x1="360" y1="180" x2="360" y2="200" class="ln"/>
<line x1="590" y1="120" x2="590" y2="200" class="ln"/>
<rect x="40" y="200" width="640" height="96" rx="7" class="pnl"/>
<text x="56" y="222" class="b">host routing table</text>
<text x="56" y="246" class="m">10.244.1.2 dev vethA1b2</text>
<text x="400" y="246" class="c">Pod A, on this node</text>
<text x="56" y="266" class="m">10.244.1.3 dev vethC3d4</text>
<text x="400" y="266" class="c">Pod B, on this node</text>
<text x="56" y="286" class="m">10.244.2.0/24 via 172.18.0.4</text>
<text x="400" y="286" class="c">Pods on lab-worker2</text>
</g>
</svg>
<figcaption>One node up close, in the point-to-point style kind uses. Each Pod hangs off a veth pair, and plain Linux routes do the rest.</figcaption>
</figure>

kind's plugin, kindnet, uses the point-to-point style shown here: each Pod's host-side veth gets a host route for that one Pod IP. Other plugins attach all the host-side veths to a Linux bridge (often called `cni0`) and route the whole node CIDR to the bridge. Either way, the forwarding decisions are ordinary Linux routing, which means ordinary Linux tools can inspect them.

Traffic between two Pods on the **same node** never leaves the host: Pod A sends to its default gateway, the packet comes out of its veth into the host, the host's routing table points at Pod B's veth, and the packet goes in.

## Across nodes

When the destination Pod lives on another node, the host needs a route for that node's Pod CIDR.

<figure class="diagram">
<svg viewBox="0 0 720 300" style="width:100%;height:auto;max-width:720px" role="img" aria-label="A packet from Pod A on lab-worker to Pod B on lab-worker2 leaves through the veth, is routed by the host to 172.18.0.4 over the kind Docker network, then routed by the second host into Pod B&#x27;s veth, with Pod IPs unchanged." xmlns="http://www.w3.org/2000/svg">
<style>
.k6d{ font-family:var(--sans),sans-serif; }
.k6d .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k6d .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k6d .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k6d .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k6d .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k6d .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k6d .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k6d .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k6d .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k6d .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k6d .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k6d .s{ fill:var(--text-dim); font-size:13px; }
.k6d .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k6d .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k6d .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k6d .c{ fill:var(--text-faint); font-size:12.5px; }
.k6d .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k6d .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k6d .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k6d-a); }
.k6d .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k6d-a); marker-start:url(#k6d-s); }
.k6d .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k6d-g); }
.k6d .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k6d-a); }
.k6d .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k6d .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k6d-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k6d-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k6d-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k6d">
<text x="360" y="24" text-anchor="middle" class="ma">src 10.244.1.2, dst 10.244.2.5: unchanged end to end, no NAT</text>
<rect x="20" y="40" width="320" height="190" rx="7" class="zone"/>
<text x="36" y="62" class="b">lab-worker</text>
<text x="324" y="62" text-anchor="end" class="m">172.18.0.3</text>
<rect x="380" y="40" width="320" height="190" rx="7" class="zone"/>
<text x="396" y="62" class="b">lab-worker2</text>
<text x="684" y="62" text-anchor="end" class="m">172.18.0.4</text>
<rect x="40" y="74" width="140" height="46" rx="7" class="acc"/>
<text x="110.0" y="92.5" text-anchor="middle" class="b">Pod A</text>
<text x="110.0" y="110.5" text-anchor="middle" class="m">10.244.1.2</text>
<rect x="540" y="74" width="140" height="46" rx="7" class="acc"/>
<text x="610.0" y="92.5" text-anchor="middle" class="b">Pod B</text>
<text x="610.0" y="110.5" text-anchor="middle" class="m">10.244.2.5</text>
<rect x="40" y="144" width="280" height="66" rx="7" class="pnl"/>
<text x="180.0" y="172.5" text-anchor="middle" class="b">route lookup</text>
<text x="180.0" y="190.5" text-anchor="middle" class="m">10.244.2.0/24 via 172.18.0.4</text>
<rect x="400" y="144" width="280" height="66" rx="7" class="pnl"/>
<text x="540.0" y="172.5" text-anchor="middle" class="b">route lookup</text>
<text x="540.0" y="190.5" text-anchor="middle" class="m">10.244.2.5 dev veth9f2e</text>
<line x1="110" y1="120" x2="110" y2="142" class="ar"/>
<rect x="20" y="248" width="680" height="34" rx="7" class="box"/>
<text x="36" y="270" class="s">Docker network: kind</text>
<path d="M230 210 L230 264 L470 264 L470 212" class="ar"/>
<line x1="610" y1="144" x2="610" y2="122" class="ar"/>
</g>
</svg>
<figcaption>Crossing nodes with direct routing. Because kind's nodes share one Docker network, each node just routes the other node's Pod CIDR to it.</figcaption>
</figure>

The packet keeps its original Pod source and destination addresses the whole way. This is the "no NAT" rule in action. The only question a plugin has to answer is how the network between nodes learns where each Pod CIDR lives. There are three broad answers:

- **Direct routing.** Each node gets a route to every other node's Pod CIDR via that node's IP. This works when the nodes share a Layer 2 segment, as kind's nodes do, or when the plugin advertises the routes with BGP to the physical network, as Calico can.
- **Overlay.** The plugin encapsulates Pod packets, usually in VXLAN or Geneve, and sends them node to node as ordinary UDP. The physical network only ever sees node IPs.
- **Native cloud addressing.** The cloud network itself knows every Pod IP, because Pods get their addresses from the VPC.

<figure class="diagram">
<svg viewBox="0 0 720 280" style="width:100%;height:auto;max-width:720px" role="img" aria-label="Two ways to carry Pod traffic between nodes: direct routing keeps the original packet, while an overlay wraps it in an outer IP and UDP VXLAN header addressed node to node." xmlns="http://www.w3.org/2000/svg">
<style>
.k6e{ font-family:var(--sans),sans-serif; }
.k6e .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k6e .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k6e .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k6e .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k6e .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k6e .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k6e .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k6e .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k6e .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k6e .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k6e .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k6e .s{ fill:var(--text-dim); font-size:13px; }
.k6e .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k6e .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k6e .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k6e .c{ fill:var(--text-faint); font-size:12.5px; }
.k6e .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k6e .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k6e .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k6e-a); }
.k6e .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k6e-a); marker-start:url(#k6e-s); }
.k6e .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k6e-g); }
.k6e .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k6e-a); }
.k6e .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k6e .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k6e-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k6e-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k6e-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k6e">
<text x="180" y="30" text-anchor="middle" class="b">Routed (no encapsulation)</text>
<rect x="60" y="46" width="240" height="32" rx="7" class="box"/>
<text x="180.0" y="66.5" text-anchor="middle" class="s">Ethernet</text>
<rect x="60" y="84" width="240" height="32" rx="7" class="pnl"/>
<text x="180.0" y="104.5" text-anchor="middle" class="s">IP: Pod A to Pod B</text>
<rect x="60" y="122" width="240" height="32" rx="7" class="box"/>
<text x="180.0" y="142.5" text-anchor="middle" class="s">TCP + data</text>
<text x="180" y="190" text-anchor="middle" class="c">needs routes for Pod CIDRs</text>
<text x="180" y="208" text-anchor="middle" class="c">on the network (L2, BGP, or cloud)</text>
<text x="540" y="30" text-anchor="middle" class="b">Overlay (VXLAN)</text>
<rect x="410" y="46" width="260" height="30" rx="7" class="box"/>
<text x="540.0" y="65.5" text-anchor="middle" class="s">outer Ethernet</text>
<rect x="410" y="82" width="260" height="30" rx="7" class="acc"/>
<text x="540.0" y="101.5" text-anchor="middle" class="b">outer IP: node to node</text>
<rect x="410" y="118" width="260" height="30" rx="7" class="acc"/>
<text x="540.0" y="137.5" text-anchor="middle" class="b">UDP 4789 + VXLAN</text>
<rect x="410" y="154" width="260" height="30" rx="7" class="pnl"/>
<text x="540.0" y="173.5" text-anchor="middle" class="s">inner IP: Pod A to Pod B</text>
<rect x="410" y="190" width="260" height="30" rx="7" class="box"/>
<text x="540.0" y="209.5" text-anchor="middle" class="s">TCP + data</text>
<text x="540" y="248" text-anchor="middle" class="c">works over any IP network</text>
<text x="540" y="266" text-anchor="middle" class="c">costs about 50 bytes per packet</text>
</g>
</svg>
<figcaption>Routed versus overlay. Overlays trade a little overhead for independence from the underlying network; this is the same VXLAN used in data center fabrics.</figcaption>
</figure>

If you have read the Data Center Fabric book, the right-hand stack will look familiar: it is the same VXLAN encapsulation, with each node acting as a VTEP. The tradeoffs are the same too. Overlays run anywhere but add header overhead and need a slightly smaller MTU inside Pods; routed designs are leaner but need the network to cooperate.

## Hands on: look at the plumbing

Run a throwaway debugging Pod with a full set of network tools, and pin it to a node so you know where to look:

```bash
kubectl run netshoot --image=nicolaka/netshoot \
  --overrides='{"spec":{"nodeName":"lab-worker"}}' -- sleep infinity
kubectl exec -it netshoot -- ip addr show eth0
kubectl exec -it netshoot -- ip route
```

Inside the Pod you will see a single `eth0` with a `10.244.x.y` address and a default route. Now look at the same node from the host side. Remember that a kind node is a Docker container, so `docker exec` drops you into it:

```bash
docker exec lab-worker ip route          # one route per local Pod, one per remote node CIDR
docker exec lab-worker ip -br link       # the host-side veth interfaces
docker exec lab-worker crictl pods       # the Pod sandboxes containerd is running
```

Finally, prove the "no NAT" rule. Start a second Pod on the other worker, then ping it from `netshoot`. If you capture on the destination node with `docker exec lab-worker2 tcpdump -ni any icmp` (install `tcpdump` in the node first if it is missing), the source address you see is `netshoot`'s Pod IP, not a node address.

## Summary

- The Kubernetes **network model**: every Pod has an IP, Pods reach each other without NAT, and node agents reach local Pods.
- **CNI plugins** implement it. The runtime calls the plugin when a Pod starts; the plugin creates a veth pair, assigns an IP, and installs routes.
- On one node, traffic moves through **veth pairs and host routes** (or a bridge). Between nodes, it is **routed directly**, **encapsulated in an overlay**, or carried by the **cloud network**.
- All of it is ordinary Linux networking, so `ip route`, `ip link`, and `tcpdump` on a node show you exactly what is happening.

Pod IPs are reachable but not stable. The next chapter adds the stable layer on top: Services.

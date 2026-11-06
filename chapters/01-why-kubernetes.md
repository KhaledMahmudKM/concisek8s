# Chapter 1: Why Kubernetes

Docker solved a real problem: packaging an application with everything it needs, so it runs the same everywhere. But running a container is only half the job. Production systems run hundreds or thousands of containers across dozens of machines, and someone has to decide where each one goes, notice when one dies, and keep the whole system matching what it is supposed to be. That job is **orchestration**, and Kubernetes is the orchestrator that won.

## From one host to many

On a single host, Docker is all you need. You run a few containers, you can see all of them, and when one stops you restart it. The moment you have many hosts, a new set of questions appears.

<figure class="diagram">
<svg viewBox="0 0 720 250" style="width:100%;height:auto;max-width:720px" role="img" aria-label="Running a few containers on one host is easy, but on fifty hosts you must answer placement, restart, discovery, and update questions yourself." xmlns="http://www.w3.org/2000/svg">
<style>
.k1a{ font-family:var(--sans),sans-serif; }
.k1a .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k1a .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k1a .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k1a .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k1a .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k1a .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k1a .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k1a .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k1a .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k1a .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k1a .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k1a .s{ fill:var(--text-dim); font-size:13px; }
.k1a .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k1a .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k1a .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k1a .c{ fill:var(--text-faint); font-size:12.5px; }
.k1a .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k1a .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k1a .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k1a-a); }
.k1a .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k1a-a); marker-start:url(#k1a-s); }
.k1a .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k1a-g); }
.k1a .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k1a-a); }
.k1a .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k1a .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k1a-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k1a-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k1a-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k1a">
<text x="135" y="26" text-anchor="middle" class="h">One host: easy</text>
<rect x="20" y="40" width="230" height="190" rx="7" class="zone"/>
<text x="135" y="72" text-anchor="middle" class="ma">docker run ...</text>
<rect x="40" y="92" width="58" height="32" rx="7" class="acc"/>
<text x="69.0" y="112.5" text-anchor="middle" class="b">web</text>
<rect x="106" y="92" width="58" height="32" rx="7" class="acc"/>
<text x="135.0" y="112.5" text-anchor="middle" class="b">api</text>
<rect x="172" y="92" width="58" height="32" rx="7" class="acc"/>
<text x="201.0" y="112.5" text-anchor="middle" class="b">db</text>
<rect x="40" y="140" width="190" height="70" rx="7" class="pnl"/>
<text x="135.0" y="170.5" text-anchor="middle" class="b">one Docker host</text>
<text x="135.0" y="188.5" text-anchor="middle" class="c">you can see everything</text>
<text x="500" y="26" text-anchor="middle" class="h">Fifty hosts: hard</text>
<rect x="310" y="40" width="62" height="34" rx="7" class="pnl"/>
<text x="341.0" y="61.5" text-anchor="middle" class="s">node</text>
<rect x="388" y="40" width="62" height="34" rx="7" class="pnl"/>
<text x="419.0" y="61.5" text-anchor="middle" class="s">node</text>
<rect x="466" y="40" width="62" height="34" rx="7" class="pnl"/>
<text x="497.0" y="61.5" text-anchor="middle" class="s">node</text>
<rect x="544" y="40" width="62" height="34" rx="7" class="pnl"/>
<text x="575.0" y="61.5" text-anchor="middle" class="s">node</text>
<rect x="622" y="40" width="62" height="34" rx="7" class="pnl"/>
<text x="653.0" y="61.5" text-anchor="middle" class="s">node</text>
<rect x="310" y="86" width="62" height="34" rx="7" class="pnl"/>
<text x="341.0" y="107.5" text-anchor="middle" class="s">node</text>
<rect x="388" y="86" width="62" height="34" rx="7" class="pnl"/>
<text x="419.0" y="107.5" text-anchor="middle" class="s">node</text>
<rect x="466" y="86" width="62" height="34" rx="7" class="pnl"/>
<text x="497.0" y="107.5" text-anchor="middle" class="s">node</text>
<rect x="544" y="86" width="62" height="34" rx="7" class="pnl"/>
<text x="575.0" y="107.5" text-anchor="middle" class="s">node</text>
<rect x="622" y="86" width="62" height="34" rx="7" class="pnl"/>
<text x="653.0" y="107.5" text-anchor="middle" class="s">node</text>
<text x="310" y="160" class="s">Which node has room for this container?</text>
<text x="310" y="182" class="s">What restarts it when it crashes?</text>
<text x="310" y="204" class="s">How do clients find it after it moves?</text>
<text x="310" y="226" class="s">How do I update 50 copies with no downtime?</text>
</g>
</svg>
<figcaption>Containers solve packaging. Running many of them across many machines raises new questions that Docker alone does not answer.</figcaption>
</figure>

Each question is a small distributed-systems problem:

- **Placement.** Which machine has enough free CPU and memory for this container, and which ones should it avoid?
- **Failure.** Containers crash, and whole machines die. Something must notice and start replacements somewhere healthy.
- **Discovery.** If a container moves to a new machine, it gets a new IP address. Clients need a stable way to reach it anyway.
- **Change.** Updating a service means replacing every copy without dropping traffic, and being able to go back if the new version is broken.

You could script answers to all of these. Many teams did, and those scripts slowly became their own unreliable orchestrator. Kubernetes is the shared, battle-tested version of that work.

## Declarative, not imperative

The most important idea in Kubernetes is not a feature. It is a style of operation.

With Docker you work **imperatively**: you issue commands, one step at a time. "Run this container. Now run another. That one died, run it again." You are the one who notices problems and reacts.

Kubernetes is **declarative**. You write down the result you want, such as "three copies of `web:1.2`, each with half a CPU", and hand that description to the cluster. Kubernetes then works continuously to make reality match it.

<figure class="diagram">
<svg viewBox="0 0 720 290" style="width:100%;height:auto;max-width:720px" role="img" aria-label="Imperative operation means you issue each step and react to failures; declarative operation means you state the desired result and Kubernetes keeps correcting reality toward it." xmlns="http://www.w3.org/2000/svg">
<style>
.k1b{ font-family:var(--sans),sans-serif; }
.k1b .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k1b .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k1b .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k1b .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k1b .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k1b .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k1b .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k1b .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k1b .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k1b .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k1b .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k1b .s{ fill:var(--text-dim); font-size:13px; }
.k1b .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k1b .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k1b .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k1b .c{ fill:var(--text-faint); font-size:12.5px; }
.k1b .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k1b .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k1b .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k1b-a); }
.k1b .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k1b-a); marker-start:url(#k1b-s); }
.k1b .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k1b-g); }
.k1b .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k1b-a); }
.k1b .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k1b .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k1b-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k1b-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k1b-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k1b">
<text x="20" y="26" class="b">Imperative: you issue every step</text>
<rect x="20" y="40" width="200" height="42" rx="7" class="box"/>
<text x="120.0" y="65.5" text-anchor="middle" class="m">docker run web (x3)</text>
<rect x="260" y="40" width="200" height="42" rx="7" class="box"/>
<text x="360.0" y="65.5" text-anchor="middle" class="s">a copy died? run again</text>
<rect x="500" y="40" width="200" height="42" rx="7" class="box"/>
<text x="600.0" y="65.5" text-anchor="middle" class="s">host full? move it</text>
<line x1="220" y1="61" x2="258" y2="61" class="ag"/>
<line x1="460" y1="61" x2="498" y2="61" class="ag"/>
<text x="360" y="108" text-anchor="middle" class="c">You are the control loop. Every failure waits for you.</text>
<text x="20" y="146" class="b">Declarative: you state the goal</text>
<rect x="20" y="160" width="200" height="56" rx="7" class="box"/>
<text x="120.0" y="183.5" text-anchor="middle" class="m">replicas: 3</text>
<text x="120.0" y="201.5" text-anchor="middle" class="m">image: web:1.2</text>
<rect x="260" y="160" width="200" height="56" rx="7" class="dark"/>
<text x="360.0" y="192.5" text-anchor="middle" class="on">Kubernetes</text>
<rect x="500" y="160" width="200" height="56" rx="7" class="pnl"/>
<text x="600.0" y="183.5" text-anchor="middle" class="b">actual state</text>
<text x="600.0" y="201.5" text-anchor="middle" class="s">3 copies running</text>
<line x1="220" y1="188" x2="258" y2="188" class="ar"/>
<line x1="460" y1="188" x2="498" y2="188" class="ar"/>
<path d="M600 216 C 600 262, 360 262, 360 218" class="ar"/>
<text x="480" y="282" text-anchor="middle" class="a">observe, compare, correct: forever</text>
</g>
</svg>
<figcaption>Imperative versus declarative. With Kubernetes you write down the desired state, and a control loop keeps making the real state match it.</figcaption>
</figure>

This changes how failure feels. If a machine dies at 3 a.m., you do not get paged to restart containers. The cluster sees that only two copies are running where you asked for three, and it starts a third on a healthy machine. The desired state did not change, so the cluster restored it.

The same idea explains updates. You do not tell Kubernetes how to roll out version 1.3. You change the image in your description from `1.2` to `1.3`, and the cluster works out the safe sequence of steps to get there.

> **The core loop.** Every part of Kubernetes follows the same pattern: observe the current state, compare it with the desired state, act to close the gap, repeat forever. Chapter 2 shows the machinery behind this loop.

## What Kubernetes takes off your plate

Put the questions and the declarative style together, and you get the list of jobs an orchestrator does for you.

<figure class="diagram">
<svg viewBox="0 0 720 250" style="width:100%;height:auto;max-width:720px" role="img" aria-label="Kubernetes in the centre with six responsibilities around it: scheduling, self-healing, scaling, service discovery, rolling updates, and configuration." xmlns="http://www.w3.org/2000/svg">
<style>
.k1c{ font-family:var(--sans),sans-serif; }
.k1c .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k1c .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k1c .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k1c .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k1c .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k1c .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k1c .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k1c .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k1c .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k1c .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k1c .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k1c .s{ fill:var(--text-dim); font-size:13px; }
.k1c .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k1c .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k1c .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k1c .c{ fill:var(--text-faint); font-size:12.5px; }
.k1c .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k1c .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k1c .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k1c-a); }
.k1c .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k1c-a); marker-start:url(#k1c-s); }
.k1c .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k1c-g); }
.k1c .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k1c-a); }
.k1c .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k1c .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k1c-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k1c-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k1c-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k1c">
<line x1="250" y1="45" x2="290" y2="125" class="ln"/>
<line x1="250" y1="125" x2="290" y2="125" class="ln"/>
<line x1="250" y1="205" x2="290" y2="125" class="ln"/>
<line x1="470" y1="45" x2="430" y2="125" class="ln"/>
<line x1="470" y1="125" x2="430" y2="125" class="ln"/>
<line x1="470" y1="205" x2="430" y2="125" class="ln"/>
<rect x="20" y="20" width="230" height="50" rx="7" class="pnl"/>
<text x="135.0" y="40.5" text-anchor="middle" class="b">Scheduling</text>
<text x="135.0" y="58.5" text-anchor="middle" class="c">places containers on nodes</text>
<rect x="20" y="100" width="230" height="50" rx="7" class="pnl"/>
<text x="135.0" y="120.5" text-anchor="middle" class="b">Self-healing</text>
<text x="135.0" y="138.5" text-anchor="middle" class="c">replaces what fails</text>
<rect x="20" y="180" width="230" height="50" rx="7" class="pnl"/>
<text x="135.0" y="200.5" text-anchor="middle" class="b">Scaling</text>
<text x="135.0" y="218.5" text-anchor="middle" class="c">runs more or fewer copies</text>
<rect x="470" y="20" width="230" height="50" rx="7" class="pnl"/>
<text x="585.0" y="40.5" text-anchor="middle" class="b">Service discovery</text>
<text x="585.0" y="58.5" text-anchor="middle" class="c">stable names and addresses</text>
<rect x="470" y="100" width="230" height="50" rx="7" class="pnl"/>
<text x="585.0" y="120.5" text-anchor="middle" class="b">Rolling updates</text>
<text x="585.0" y="138.5" text-anchor="middle" class="c">new versions, no downtime</text>
<rect x="470" y="180" width="230" height="50" rx="7" class="pnl"/>
<text x="585.0" y="200.5" text-anchor="middle" class="b">Configuration</text>
<text x="585.0" y="218.5" text-anchor="middle" class="c">settings and secrets outside images</text>
<rect x="290" y="100" width="140" height="50" rx="7" class="dark"/>
<text x="360.0" y="129.5" text-anchor="middle" class="on">Kubernetes</text>
</g>
</svg>
<figcaption>What an orchestrator does. Each of these is a job you would otherwise script and babysit yourself.</figcaption>
</figure>

You will meet each of these in the chapters ahead: scheduling and self-healing in Chapter 2, scaling and rolling updates in Chapter 5, discovery in Chapters 7 and 8, and configuration in Chapter 11.

## What Kubernetes is not

It helps to be clear about the boundaries too:

- **It is not a container runtime.** Kubernetes does not run containers itself. It asks a runtime on each machine, usually **containerd**, to do that through a standard interface called the CRI. The images you build with Docker run unchanged.
- **It is not a PaaS.** It does not build your code, run your CI, or pick your database. It gives you building blocks, and platforms are built on top of it.
- **It is not small.** Kubernetes has a real learning curve and real operational weight. For one app on one server, Docker Compose is often the better tool. Kubernetes earns its cost when you have many services, many machines, or a need for self-healing and zero-downtime change.

## A short history

Kubernetes came out of Google, which had run containers at enormous scale for years on an internal system called Borg. In 2014 Google open-sourced a new system built on those lessons, named after the Greek word for a ship's helmsman. In 2015 it was donated to the newly formed Cloud Native Computing Foundation (CNCF), which still governs it. The name is often shortened to **K8s**: K, then eight letters, then s.

Today every major cloud offers managed Kubernetes (EKS, GKE, AKS, and others), and the same API runs on a laptop, in a data center, or at the edge. That portability is a large part of why it won: learn the API once and it applies everywhere.

## Summary

- Docker packages and runs containers; Kubernetes **orchestrates** them across many machines.
- At scale you need answers for placement, failure, discovery, and change. Kubernetes provides them.
- Kubernetes is **declarative**: you describe the desired state, and control loops keep reality matching it.
- It delegates running containers to a runtime such as containerd, so your Docker images work as they are.

The next chapter opens up a cluster and shows which components do each of these jobs.

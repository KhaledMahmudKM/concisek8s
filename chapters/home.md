# Concise Book on Kubernetes

A concise, diagram-rich guide for readers who know Docker.

<img src="cover.svg" alt="Cover Image" width="100%" >

You can build an image and run a container. This book picks up from there and explains how Kubernetes runs containers across a whole fleet of machines: how it decides where they go, keeps them alive, connects them, exposes them, and updates them without downtime.

It assumes no prior Kubernetes knowledge, but it does assume you are comfortable with Docker: images, containers, ports, volumes, and networks. If those feel shaky, the Concise Book on Docker covers them.

The networking chapters go deeper than most introductions. They follow real packets through veth pairs, routes, and kube-proxy rules, because that is where Kubernetes most often surprises people and where most production problems live.

---

## How this book is organized

**Part I, Foundations.** Why orchestration exists, how a cluster is put together, and how to run one on your laptop with kind.

**Part II, Workloads.** Pods, the smallest unit Kubernetes runs, and the controllers that manage them: Deployments for stateless apps, plus StatefulSets, DaemonSets, and Jobs.

**Part III, Networking.** The Kubernetes network model and CNI plugins, Services and kube-proxy, cluster DNS, traffic from outside through Ingress and the Gateway API, and NetworkPolicies for segmentation.

**Part IV, Configuration and Operations.** ConfigMaps, Secrets, and persistent storage, then the practices that keep workloads healthy: resource requests and limits, probes, autoscaling, RBAC, and troubleshooting.

The **Appendix** is a kubectl reference, and the **Glossary** defines every term.

---

## How to read it

Every chapter leans on diagrams. Read them closely; a packet path or a control loop is far easier to follow as a picture.

Every example is a manifest or a command you can run in the lab cluster from Chapter 3. Type them in, break things on purpose, and watch how the cluster reacts. That is the fastest way to build an accurate mental model.

Kubernetes changes quickly. The book reflects Kubernetes 1.36 and 1.37 and the Gateway API v1.5 era. Where a detail is version-specific, the text says so.

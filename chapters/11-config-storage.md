# Chapter 11: Configuration and Storage

Images should be the same in every environment; what changes is configuration, credentials, and data. Kubernetes keeps all three outside the image. **ConfigMaps** hold settings, **Secrets** hold sensitive values, and **volumes**, especially **PersistentVolumes**, hold data that must outlive a container. This chapter covers each, with the behavior that matters in practice.

## ConfigMaps

A **ConfigMap** is a set of key-value pairs, where a value can be anything from a single word to a whole configuration file. The limit is 1 MiB per object.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: info
  app.yaml: |
    server:
      port: 8080
      timeout: 30s
```

You can also create one straight from files: `kubectl create configmap app-config --from-file=app.yaml --from-literal=LOG_LEVEL=info`.

A container consumes it in two ways, and the difference matters.

<figure class="diagram">
<svg viewBox="0 0 720 250" style="width:100%;height:auto;max-width:720px" role="img" aria-label="A ConfigMap and a Secret feed a container in two ways: as environment variables, read once when the container starts, and as files in a mounted volume, which the kubelet updates when the source changes." xmlns="http://www.w3.org/2000/svg">
<style>
.k11a{ font-family:var(--sans),sans-serif; }
.k11a .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k11a .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k11a .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k11a .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k11a .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k11a .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k11a .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k11a .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k11a .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k11a .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k11a .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k11a .s{ fill:var(--text-dim); font-size:13px; }
.k11a .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k11a .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k11a .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k11a .c{ fill:var(--text-faint); font-size:12.5px; }
.k11a .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k11a .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k11a .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k11a-a); }
.k11a .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k11a-a); marker-start:url(#k11a-s); }
.k11a .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k11a-g); }
.k11a .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k11a-a); }
.k11a .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k11a .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k11a-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k11a-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k11a-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k11a">
<rect x="20" y="30" width="200" height="76" rx="7" class="pnl"/>
<text x="120.0" y="54.5" text-anchor="middle" class="b">ConfigMap app-config</text>
<text x="120.0" y="72.5" text-anchor="middle" class="m">LOG_LEVEL: info</text>
<text x="120.0" y="90.5" text-anchor="middle" class="m">app.yaml: |...</text>
<rect x="20" y="144" width="200" height="76" rx="7" class="pnl"/>
<text x="120.0" y="168.5" text-anchor="middle" class="b">Secret db-cred</text>
<text x="120.0" y="186.5" text-anchor="middle" class="m">password: &lt;base64&gt;</text>
<text x="120.0" y="204.5" text-anchor="middle" class="m">type: Opaque</text>
<rect x="360" y="16" width="340" height="220" rx="7" class="out"/>
<text x="380" y="42" class="h">Pod</text>
<rect x="380" y="60" width="300" height="64" rx="7" class="acc"/>
<text x="530.0" y="87.5" text-anchor="middle" class="b">as environment variables</text>
<text x="530.0" y="105.5" text-anchor="middle" class="s">read once, when the container starts</text>
<rect x="380" y="146" width="300" height="70" rx="7" class="box"/>
<text x="530.0" y="167.5" text-anchor="middle" class="b">as files in a volume</text>
<text x="530.0" y="185.5" text-anchor="middle" class="m">/etc/app/app.yaml</text>
<text x="530.0" y="203.5" text-anchor="middle" class="c">updated in place when the source changes</text>
<line x1="220" y1="60" x2="378" y2="86" class="ar"/>
<line x1="220" y1="80" x2="378" y2="170" class="ar"/>
<line x1="220" y1="176" x2="378" y2="104" class="ar"/>
<line x1="220" y1="196" x2="378" y2="196" class="ar"/>
</g>
</svg>
<figcaption>Two ways in. Environment variables are simple but frozen at start; mounted files can change while the Pod runs.</figcaption>
</figure>

```yaml
spec:
  containers:
  - name: app
    image: myapp:0.1
    envFrom:
    - configMapRef: {name: app-config}      # every key becomes an env var
    volumeMounts:
    - name: config
      mountPath: /etc/app                    # every key becomes a file
  volumes:
  - name: config
    configMap: {name: app-config}
```

**Environment variables** are read once, when the container starts. Changing the ConfigMap does nothing until the Pod is replaced. **Mounted files** are refreshed by the kubelet in place, usually within a minute, so an app that re-reads its config file picks up changes without a restart. (Files mounted with `subPath` are the exception: they never update.)

In practice, most teams want config changes to roll out like code changes, gradually and reversibly. A common pattern is to put a hash of the configuration into a Pod template annotation, so any config change also changes the template and triggers a normal rolling update. Marking a ConfigMap `immutable: true` prevents accidental edits altogether and reduces load on the API server.

## Secrets

A **Secret** looks almost identical to a ConfigMap and is consumed the same ways. The differences are in handling: Secrets can be restricted separately with RBAC, are kept off disk on nodes (mounted Secret volumes are held in memory), and can be encrypted at rest. Several built-in types exist, such as `kubernetes.io/tls` for certificates and `kubernetes.io/dockerconfigjson` for registry credentials.

```bash
kubectl create secret generic db-cred --from-literal=password='s3cret'
kubectl get secret db-cred -o yaml        # the value is base64, not encrypted
```

<figure class="diagram">
<svg viewBox="0 0 720 212" style="width:100%;height:auto;max-width:720px" role="img" aria-label="Four layers around a Secret value: base64 in the object is only encoding; encryption at rest protects it in etcd; RBAC controls who can read it; an external secret manager can hold the real source of truth." xmlns="http://www.w3.org/2000/svg">
<style>
.k11b{ font-family:var(--sans),sans-serif; }
.k11b .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k11b .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k11b .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k11b .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k11b .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k11b .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k11b .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k11b .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k11b .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k11b .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k11b .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k11b .s{ fill:var(--text-dim); font-size:13px; }
.k11b .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k11b .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k11b .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k11b .c{ fill:var(--text-faint); font-size:12.5px; }
.k11b .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k11b .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k11b .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k11b-a); }
.k11b .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k11b-a); marker-start:url(#k11b-s); }
.k11b .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k11b-g); }
.k11b .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k11b-a); }
.k11b .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k11b .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k11b-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k11b-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k11b-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k11b">
<rect x="14" y="60" width="166" height="70" rx="7" class="box"/>
<text x="97.0" y="90.5" text-anchor="middle" class="b">base64 in the object</text>
<text x="97.0" y="108.5" text-anchor="middle" class="c">encoding, not encryption</text>
<rect x="192" y="60" width="166" height="70" rx="7" class="pnl"/>
<text x="275.0" y="90.5" text-anchor="middle" class="b">encryption at rest</text>
<text x="275.0" y="108.5" text-anchor="middle" class="c">protects the copy in etcd</text>
<line x1="180" y1="95" x2="190" y2="95" class="ar"/>
<rect x="370" y="60" width="166" height="70" rx="7" class="pnl"/>
<text x="453.0" y="90.5" text-anchor="middle" class="b">RBAC</text>
<text x="453.0" y="108.5" text-anchor="middle" class="c">who may read Secrets</text>
<line x1="358" y1="95" x2="368" y2="95" class="ar"/>
<rect x="548" y="60" width="166" height="70" rx="7" class="acc"/>
<text x="631.0" y="90.5" text-anchor="middle" class="b">external secret store</text>
<text x="631.0" y="108.5" text-anchor="middle" class="s">the real source of truth</text>
<line x1="536" y1="95" x2="546" y2="95" class="ar"/>
<text x="360" y="30" text-anchor="middle" class="b">what actually protects a Secret</text>
<text x="360" y="172" text-anchor="middle" class="c">base64 alone hides nothing: anyone who can read the object can decode it</text>
<text x="360" y="196" text-anchor="middle" class="m">echo cGFzc3dvcmQ= | base64 -d</text>
</g>
</svg>
<figcaption>A Secret is a distribution mechanism, not a vault. Its protection comes from encryption at rest, RBAC, and where the value really lives.</figcaption>
</figure>

The most important fact about Secrets is what they are **not**. The values are only **base64-encoded**, which anyone can reverse. Real protection comes from the layers around the object:

- **Encryption at rest.** The API server can encrypt Secrets before storing them in etcd, using an encryption configuration or a cloud key management service. Managed Kubernetes services usually offer this as a setting.
- **RBAC.** Anyone who can `get` Secrets in a namespace can read them all, and anyone who can create Pods there can mount them. Grant both sparingly.
- **An external source of truth.** Tools such as the External Secrets Operator or the Secrets Store CSI driver sync values from a real secret manager (Vault, AWS Secrets Manager, and others) into the cluster. Sealed Secrets lets you commit encrypted Secrets to Git safely.

Never commit a plain Secret manifest to version control. Its "encryption" is a single `base64 -d` away from plain text.

## Volumes

A container's own filesystem disappears whenever the container restarts. **Volumes** attach storage with a longer life. Kubernetes supports many types, and they differ mainly in how long the data lives.

<figure class="diagram">
<svg viewBox="0 0 720 214" style="width:100%;height:auto;max-width:720px" role="img" aria-label="What survives each event: the container filesystem is lost when the container restarts; an emptyDir survives restarts but is deleted with the Pod; a network-backed PersistentVolume survives Pod deletion and node loss." xmlns="http://www.w3.org/2000/svg">
<style>
.k11d{ font-family:var(--sans),sans-serif; }
.k11d .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k11d .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k11d .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k11d .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k11d .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k11d .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k11d .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k11d .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k11d .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k11d .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k11d .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k11d .s{ fill:var(--text-dim); font-size:13px; }
.k11d .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k11d .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k11d .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k11d .c{ fill:var(--text-faint); font-size:12.5px; }
.k11d .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k11d .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k11d .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k11d-a); }
.k11d .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k11d-a); marker-start:url(#k11d-s); }
.k11d .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k11d-g); }
.k11d .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k11d-a); }
.k11d .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k11d .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k11d-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k11d-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k11d-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k11d">
<text x="330" y="30" text-anchor="middle" class="b">container restarts</text>
<text x="470" y="30" text-anchor="middle" class="b">Pod deleted</text>
<text x="610" y="30" text-anchor="middle" class="b">node lost</text>
<text x="20" y="77" class="s">container filesystem</text>
<rect x="270" y="50" width="120" height="40" rx="7" class="box"/>
<text x="330.0" y="74.5" text-anchor="middle" class="s">lost</text>
<rect x="410" y="50" width="120" height="40" rx="7" class="box"/>
<text x="470.0" y="74.5" text-anchor="middle" class="s">lost</text>
<rect x="550" y="50" width="120" height="40" rx="7" class="box"/>
<text x="610.0" y="74.5" text-anchor="middle" class="s">lost</text>
<text x="20" y="131" class="s">emptyDir volume</text>
<rect x="270" y="104" width="120" height="40" rx="7" class="acc"/>
<text x="330.0" y="128.5" text-anchor="middle" class="b">kept</text>
<rect x="410" y="104" width="120" height="40" rx="7" class="box"/>
<text x="470.0" y="128.5" text-anchor="middle" class="s">lost</text>
<rect x="550" y="104" width="120" height="40" rx="7" class="box"/>
<text x="610.0" y="128.5" text-anchor="middle" class="s">lost</text>
<text x="20" y="185" class="s">PersistentVolume (network disk)</text>
<rect x="270" y="158" width="120" height="40" rx="7" class="acc"/>
<text x="330.0" y="182.5" text-anchor="middle" class="b">kept</text>
<rect x="410" y="158" width="120" height="40" rx="7" class="acc"/>
<text x="470.0" y="182.5" text-anchor="middle" class="b">kept</text>
<rect x="550" y="158" width="120" height="40" rx="7" class="acc"/>
<text x="610.0" y="182.5" text-anchor="middle" class="b">kept</text>
</g>
</svg>
<figcaption>Volume lifetimes. Choose the volume type by how long the data must outlive the container.</figcaption>
</figure>

- **`emptyDir`**: a scratch directory created when the Pod starts and deleted when the Pod is removed. Containers in the same Pod can share it, and it survives container restarts. Set `medium: Memory` for a RAM-backed tmpfs.
- **`configMap`, `secret`, `projected`**: configuration and credentials as files, covered above.
- **`hostPath`**: a directory on the node itself. It ties a Pod to one node and exposes the node's filesystem, so it is for system agents only, never for applications.
- **`persistentVolumeClaim`**: durable storage, the subject of the rest of this chapter.

## PersistentVolumes and claims

Durable storage is split into two objects so that applications never have to know where storage comes from.

<figure class="diagram">
<svg viewBox="0 0 720 222" style="width:100%;height:auto;max-width:720px" role="img" aria-label="A Pod mounts a PersistentVolumeClaim; the claim names a StorageClass, whose provisioner creates a PersistentVolume backed by a real disk, and the claim binds to that volume." xmlns="http://www.w3.org/2000/svg">
<style>
.k11c{ font-family:var(--sans),sans-serif; }
.k11c .box{ fill:var(--bg-soft); stroke:var(--border); stroke-width:1.5; }
.k11c .pnl{ fill:var(--panel); stroke:var(--border); stroke-width:1.5; }
.k11c .hov{ fill:var(--panel-hover); stroke:var(--border); stroke-width:1.5; }
.k11c .acc{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.6; }
.k11c .dark{ fill:var(--accent); stroke:var(--accent); stroke-width:1.6; }
.k11c .out{ fill:none; stroke:var(--accent); stroke-width:2; }
.k11c .dash{ fill:none; stroke:var(--accent); stroke-width:1.6; stroke-dasharray:6 4; }
.k11c .zone{ fill:none; stroke:var(--border); stroke-width:1.5; stroke-dasharray:6 4; }
.k11c .tint{ fill:var(--accent-dim); stroke:var(--accent); stroke-width:1.2; opacity:0.28; }
.k11c .h{ fill:var(--text); font-size:15px; font-weight:700; }
.k11c .b{ fill:var(--text); font-size:13px; font-weight:700; }
.k11c .s{ fill:var(--text-dim); font-size:13px; }
.k11c .a{ fill:var(--accent); font-size:13px; font-weight:700; }
.k11c .m{ fill:var(--text); font-size:12.5px; font-family:var(--mono),monospace; }
.k11c .ma{ fill:var(--accent); font-size:12.5px; font-family:var(--mono),monospace; }
.k11c .c{ fill:var(--text-faint); font-size:12.5px; }
.k11c .on{ fill:var(--bg); font-size:13px; font-weight:700; }
.k11c .ln{ stroke:var(--border); stroke-width:1.5; fill:none; }
.k11c .ar{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k11c-a); }
.k11c .ar2{ stroke:var(--accent); stroke-width:2; fill:none; marker-end:url(#k11c-a); marker-start:url(#k11c-s); }
.k11c .ag{ stroke:var(--text-faint); stroke-width:1.8; fill:none; marker-end:url(#k11c-g); }
.k11c .ad{ stroke:var(--accent); stroke-width:1.8; fill:none; stroke-dasharray:6 4; marker-end:url(#k11c-a); }
.k11c .gd{ stroke:var(--text-faint); stroke-width:1.6; fill:none; stroke-dasharray:4 4; }
.k11c .x{ stroke:var(--text-faint); stroke-width:2.4; fill:none; }
</style>
<defs><marker id="k11c-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--accent)"/></marker><marker id="k11c-s" viewBox="0 0 10 10" refX="1" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M10 0 L0 5 L10 10 z" fill="var(--accent)"/></marker><marker id="k11c-g" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0 0 L10 5 L0 10 z" fill="var(--text-faint)"/></marker></defs>
<g class="k11c">
<rect x="380" y="18" width="220" height="50" rx="7" class="pnl"/>
<text x="490.0" y="38.5" text-anchor="middle" class="b">StorageClass standard</text>
<text x="490.0" y="56.5" text-anchor="middle" class="c">a provisioner</text>
<rect x="20" y="110" width="130" height="56" rx="7" class="box"/>
<text x="85.0" y="133.5" text-anchor="middle" class="b">Pod</text>
<text x="85.0" y="151.5" text-anchor="middle" class="c">mounts /data</text>
<rect x="200" y="110" width="150" height="56" rx="7" class="acc"/>
<text x="275.0" y="133.5" text-anchor="middle" class="b">PVC data</text>
<text x="275.0" y="151.5" text-anchor="middle" class="m">10Gi, RWO</text>
<rect x="420" y="110" width="150" height="56" rx="7" class="pnl"/>
<text x="495.0" y="133.5" text-anchor="middle" class="b">PV pvc-8f3a</text>
<text x="495.0" y="151.5" text-anchor="middle" class="m">10Gi</text>
<rect x="600" y="110" width="100" height="56" rx="7" class="box"/>
<text x="650.0" y="133.5" text-anchor="middle" class="b">real disk</text>
<text x="650.0" y="151.5" text-anchor="middle" class="c">cloud or local</text>
<line x1="150" y1="138" x2="198" y2="138" class="ar"/>
<line x1="350" y1="138" x2="418" y2="138" class="ar2"/>
<line x1="570" y1="138" x2="598" y2="138" class="ar"/>
<line x1="300" y1="108" x2="400" y2="70" class="ag"/>
<line x1="500" y1="68" x2="500" y2="108" class="ar"/>
<text x="384" y="158" text-anchor="middle" class="c">bound</text>
<text x="275" y="200" text-anchor="middle" class="c">you write the claim</text>
<text x="495" y="200" text-anchor="middle" class="c">the cluster makes the volume</text>
</g>
</svg>
<figcaption>Dynamic provisioning. The claim describes what you need; the StorageClass decides how to supply it.</figcaption>
</figure>

- A **PersistentVolume (PV)** is a piece of real storage, such as a cloud disk or an NFS export, represented as a cluster-wide object.
- A **PersistentVolumeClaim (PVC)** is a namespaced request for storage: a size, an access mode, and optionally a StorageClass. The control plane **binds** each claim to a matching volume.
- A **StorageClass** names a **provisioner** that creates volumes on demand. With **dynamic provisioning**, you never create PVs by hand: you write a claim, and the provisioner makes a volume to satisfy it.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 10Gi
  # storageClassName omitted: use the cluster's default class
```

A Pod then mounts the claim by name, under `volumes:` with `persistentVolumeClaim: {claimName: data}`. If the Pod is deleted and recreated, even on another node, it gets the same volume back. That is exactly what StatefulSets automate with their `volumeClaimTemplates`.

The **access mode** describes how the volume may be mounted:

| Mode | Meaning |
|---|---|
| `ReadWriteOnce` (RWO) | Read-write by Pods on **one node** at a time |
| `ReadWriteOncePod` (RWOP) | Read-write by **one Pod** only |
| `ReadOnlyMany` (ROX) | Read-only by many nodes |
| `ReadWriteMany` (RWX) | Read-write by many nodes; needs shared storage such as NFS or CephFS |

Block storage such as cloud disks is almost always RWO. If several replicas truly need to write to the same files, you need a shared filesystem, and it is often a sign that the application should use a database or object storage instead.

Two more settings appear in every real StorageClass. The **reclaim policy** decides what happens to a dynamically created volume when its claim is deleted: `Delete` (the default) destroys the disk, while `Retain` keeps it for manual recovery. Use `Retain` for anything you cannot afford to lose by accident. The **volume binding mode** `WaitForFirstConsumer` delays creating the volume until a Pod using it is scheduled, so the disk is created in the same zone as the node.

## Hands on: storage in kind

kind ships with a simple provisioner (local-path) and a default StorageClass named `standard`, which creates volumes as directories on the node:

```bash
kubectl get storageclass
kubectl apply -f pvc.yaml                       # the claim above
kubectl get pvc data                            # Pending: waits for a consumer
kubectl run writer --image=busybox:1.36 --overrides='{"spec":{"volumes":[{"name":"d","persistentVolumeClaim":{"claimName":"data"}}],"containers":[{"name":"writer","image":"busybox:1.36","command":["sh","-c","date >> /data/log; sleep 3600"],"volumeMounts":[{"name":"d","mountPath":"/data"}]}]}}'
kubectl get pvc data                            # now Bound
kubectl delete pod writer                       # the claim and its data remain
```

Because local-path storage lives on one node's disk, a Pod using it is pinned to that node. Real clusters use network-backed storage so that volumes can follow Pods across nodes.

## Summary

- **ConfigMaps** hold configuration; consume them as **environment variables** (fixed at start) or **mounted files** (updated in place).
- **Secrets** are base64, **not encrypted**. Protect them with encryption at rest, tight RBAC, and an external secret manager.
- Choose a **volume** by lifetime: container filesystem, `emptyDir`, or a **PersistentVolume**.
- Applications request storage with a **PVC**; a **StorageClass** provisions a **PV** on demand. Mind the **access mode** and **reclaim policy**.

The final chapter brings everything together into the practices that keep workloads healthy in production.

# Appendix: kubectl Reference

The commands used throughout the book, grouped by task. Most accept `-n <namespace>` to pick a namespace, or `-A` for all namespaces.

## Cluster and context

| Command | Purpose |
|---|---|
| `kind create cluster --config kind-lab.yaml` | Create the lab cluster (Chapter 3) |
| `kind delete cluster --name lab` | Delete it |
| `kubectl cluster-info` | Show the API server address |
| `kubectl get nodes -o wide` | Nodes, versions, internal IPs |
| `kubectl config get-contexts` | List kubeconfig contexts |
| `kubectl config use-context kind-lab` | Switch context |
| `kubectl config set-context --current --namespace=shop` | Change the default namespace |
| `kubectl api-resources` | Every resource type, short name, and API group |
| `kubectl explain deployment.spec.strategy` | Field documentation from the API itself |

## Viewing objects

| Command | Purpose |
|---|---|
| `kubectl get pods -o wide` | Pods with node and IP |
| `kubectl get all` | Common workload objects in a namespace |
| `kubectl get pods -l app=web` | Filter by label |
| `kubectl get pods --show-labels` | Show all labels |
| `kubectl get deploy web -o yaml` | Full object as stored |
| `kubectl get pods -w` | Watch for changes |
| `kubectl get pods -o jsonpath='{.items[*].status.podIP}'` | Extract fields |
| `kubectl describe pod <pod>` | Human summary plus recent events |
| `kubectl get events --sort-by=.lastTimestamp` | Recent events in order |

## Creating and changing

| Command | Purpose |
|---|---|
| `kubectl apply -f file.yaml` | Create or update declaratively (Chapter 2) |
| `kubectl apply -f dir/` | Apply every manifest in a folder |
| `kubectl diff -f file.yaml` | Preview what apply would change |
| `kubectl delete -f file.yaml` | Delete what the file describes |
| `kubectl create deployment web --image=nginx --dry-run=client -o yaml` | Generate a manifest skeleton |
| `kubectl edit deploy web` | Edit the live object in an editor |
| `kubectl label pod <pod> tier=fe` | Add or change a label |
| `kubectl scale deploy web --replicas=5` | Change the replica count |

## Deployments and rollouts

| Command | Purpose |
|---|---|
| `kubectl set image deploy/web web=nginx:1.27` | Change the image, starting a rollout |
| `kubectl rollout status deploy/web` | Wait for the rollout to finish |
| `kubectl rollout history deploy/web` | List revisions |
| `kubectl rollout undo deploy/web` | Roll back to the previous revision |
| `kubectl rollout restart deploy/web` | Replace all Pods gradually |
| `kubectl get rs` | ReplicaSets behind a Deployment |

## Inside Pods

| Command | Purpose |
|---|---|
| `kubectl logs <pod>` | Container output |
| `kubectl logs <pod> -c <container> -f` | Follow one container |
| `kubectl logs <pod> --previous` | Output of the last crashed run |
| `kubectl logs -l app=web --prefix` | Logs from every matching Pod |
| `kubectl exec -it <pod> -- sh` | Shell in a container |
| `kubectl debug -it <pod> --image=nicolaka/netshoot --target=<c>` | Ephemeral debug container |
| `kubectl debug node/lab-worker -it --image=busybox` | Debug Pod on a node |
| `kubectl cp <pod>:/path ./local` | Copy files out of a container |
| `kubectl run tmp --rm -it --image=busybox:1.36 -- sh` | Throwaway Pod for tests |

## Networking

| Command | Purpose |
|---|---|
| `kubectl expose deploy web --port=80` | Create a ClusterIP Service |
| `kubectl get svc` | Services, ClusterIPs, ports |
| `kubectl get endpointslices -l kubernetes.io/service-name=web` | Backends behind a Service |
| `kubectl port-forward svc/web 8080:80` | Reach a Service from your machine |
| `kubectl run t --rm -it --image=busybox:1.36 -- nslookup web` | Test DNS (Chapter 8) |
| `kubectl get gateway,httproute` | Gateway API objects (Chapter 9) |
| `kubectl get networkpolicy` | Policies in a namespace (Chapter 10) |
| `docker exec lab-worker iptables -t nat -L KUBE-SERVICES -n` | kube-proxy rules on a kind node (Chapter 7) |

## Configuration, storage, and security

| Command | Purpose |
|---|---|
| `kubectl create configmap app-config --from-file=app.yaml` | ConfigMap from a file |
| `kubectl create secret generic db-cred --from-literal=password=x` | Secret from a literal |
| `kubectl get pvc,pv,storageclass` | Storage objects |
| `kubectl top pods` / `kubectl top nodes` | Live usage (needs metrics-server) |
| `kubectl get hpa` | Autoscaler targets and replicas |
| `kubectl auth can-i list secrets --as=<user>` | Check a permission |
| `kubectl auth can-i --list -n shop` | Everything you can do in a namespace |
| `kubectl create serviceaccount ci-bot` | A new identity for automation |

## Output flags worth knowing

- `-o wide`: extra columns, such as node and IP.
- `-o yaml` and `-o json`: the full object.
- `-o name`: just `kind/name`, handy in scripts.
- `-o jsonpath=...` and `-o custom-columns=...`: pick exact fields.
- `--dry-run=client -o yaml`: print what would be created, without creating it.

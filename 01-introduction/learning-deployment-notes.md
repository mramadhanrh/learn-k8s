# 01 - Introduction to Kubernetes Pods & Deployments

Notes from today's hands-on learning session with minikube.

## 1. Pod lifecycle

A pod's `.status.phase` moves through these values:

| Phase       | Meaning                                                            |
| ----------- | ------------------------------------------------------------------ |
| `Pending`   | Accepted, not all containers running yet (scheduling / image pull) |
| `Running`   | Bound to a node, at least one container running                    |
| `Succeeded` | All containers exited 0, won't restart (typical for Jobs)          |
| `Failed`    | All containers terminated, at least one failed                     |
| `Unknown`   | State can't be determined (node unreachable)                       |

Container-level states (visible via `kubectl describe pod`):

| State        | Meaning                                                                   |
| ------------ | ------------------------------------------------------------------------- |
| `Waiting`    | Not running yet — check `Reason` (`ImagePullBackOff`, `CrashLoopBackOff`) |
| `Running`    | Executing                                                                 |
| `Terminated` | Finished/killed — check exit code (`Completed`, `Error`, `OOMKilled`)     |

### Full flow

1. **Scheduling** — `kube-scheduler` picks a node
2. **Init containers** — run sequentially, must finish before app containers start
3. **Main containers start** — run in parallel, `postStart` hook fires (best-effort)
4. **Readiness probes** — gate whether the pod receives Service traffic (can be `Running` but not `Ready`)
5. **Liveness probes** — failing this restarts the container in place (→ `CrashLoopBackOff` if repeated)
6. **Startup probes** — delay liveness/readiness checks for slow-starting apps
7. **Termination** — `Terminating` → `preStop` hook → SIGTERM → `terminationGracePeriodSeconds` (default 30s) → SIGKILL if still alive

## 2. Pod vs Deployment — which to use

- **Pod** — smallest deployable unit. If it dies, nothing recreates it. No self-healing, scaling, or rolling updates.
- **Deployment** — manages a set of identical Pods via a ReplicaSet. Self-heals, scales, handles rolling updates/rollbacks.

**Rule of thumb: use a Deployment for anything long-running.** Bare Pods are for one-off debugging or when another controller manages the lifecycle.

### Minimal Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

Key rule: `template.metadata.labels` must match `spec.selector.matchLabels`, or the Deployment fails to adopt its own pods.

### Other controller types (for later)

- **StatefulSet** — stable network identity + storage per pod (databases)
- **DaemonSet** — one pod per node (log collectors, monitoring agents)
- **Job** — run to completion (batch tasks)
- **CronJob** — scheduled Jobs

## 3. Checking pod status

```bash
kubectl get pods                    # quick status
kubectl get pods -w                 # watch live
kubectl get pods -o wide            # + node, pod IP
kubectl describe pod <name>         # container states + Events (most useful for debugging)
kubectl logs <name>                 # container logs
kubectl logs <name> --previous      # logs from before last crash
```

Debugging workflow: `get pods` (spot bad status) → `describe pod` (read Events, find Reason) → `logs --previous` (see what happened before the crash).

## 4. A key misconception: pods don't restart, they get replaced

Pods are **immutable and disposable**. There are two distinct things that look similar but aren't:

- **Container restarts** (same pod name, `RESTARTS` count increases) — happens on liveness probe failure or process crash (e.g. our `/crash` or `/hang` routes)
- **Pod gets replaced** (new pod name, new IP, `RESTARTS: 0` on the new one) — happens when a pod is deleted, a node dies, or you scale down/up. The Deployment's ReplicaSet notices the gap between desired replicas and actual replicas, and creates a new pod object to close it.

This is the **reconciliation loop** pattern — the same desired-state-vs-actual-state pattern that shows up in distributed systems generally (relevant to etcd/CAP theorem reading).

### Testing this yourself

```bash
kubectl delete pod <pod-name>              # triggers replacement (new name)
kubectl exec -it <pod-name> -- kill 1      # simulates a crash inside the pod
kubectl scale deployment <name> --replicas=0   # then back up
```

## 5. minikube service vs kubectl port-forward

|                                      | `minikube service`                                                   | `kubectl port-forward`                                                         |
| ------------------------------------ | -------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| Works on                             | Services only                                                        | Pods, Deployments, or Services                                                 |
| Requires `NodePort`                  | Yes                                                                  | No — works with `ClusterIP` (the default)                                      |
| How it connects                      | Tunnels into minikube's node, then through real `kube-proxy` routing | Talks to the API server, which proxies straight to one pod                     |
| Portable to real clusters (e.g. EKS) | No — minikube-specific                                               | Yes — identical command everywhere                                             |
| Load balances across replicas        | **Yes** — real Service datapath                                      | **No** — pinned to one pod for the whole tunnel, even when targeting a Service |

**Why minikube needs its own tool at all:** minikube's "node" is usually just a Docker container on your host (Docker driver), not a real machine with a routable IP. A NodePort isn't reachable the normal way, so minikube provides its own tunnel into that container's network.

**Practical takeaway:** build muscle memory on `kubectl port-forward` — it's what actually transfers to production-like environments (e.g. EKS at work). `minikube service` is a local-dev-only convenience.

## 6. ClusterIP vs NodePort

- **ClusterIP** (default) — virtual IP reachable only from inside the cluster network. Used for internal pod-to-pod / service-to-service traffic.
- **NodePort** — everything ClusterIP does, plus opens a port (30000–32767) on every node, reachable from outside the cluster. This is what `minikube service` requires to work.

NodePort is a superset of ClusterIP, not a separate mechanism — every NodePort Service still has a ClusterIP underneath.

```bash
# Change a Service's type cleanly (edit YAML directly to make it stick across re-applies)
kubectl delete svc <name>
kubectl expose deployment <name> --port=80 --type=NodePort
```

## 7. Debugging crash loops — checklist

1. `kubectl describe pod <name>` → check `Events` at the bottom
   - `Failed to pull image` → bad image reference or typo (e.g. `bitnami:node` instead of `bitnami/node:20`)
   - `Back-off restarting failed container` after a successful pull → the image has no long-running process to execute (e.g. a bare runtime image with no app/entrypoint)
   - `ErrImageNeverPull` → `imagePullPolicy: Never` set but the image was never loaded into minikube (`minikube image load <name>`), or never built locally at all
2. `kubectl logs <name> --previous` → see what happened right before the crash
3. Check image existence at each layer if using a locally-built image:
   ```bash
   docker images | grep <name>       # exists on host?
   minikube image ls | grep <name>   # exists inside minikube?
   ```

**Simplest fix, and the one we landed on:** skip the local build/load pipeline entirely and use a published image built for testing — e.g. `ealen/echo-server` or `mendhak/http-https-echo` (both Express-based, return request/hostname info as JSON out of the box). No `imagePullPolicy: Never`, no host/minikube Docker context mismatch to debug.

## 8. Cleanup commands

```bash
kubectl delete deployment <name>      # cascade deletes ReplicaSet + pods
kubectl delete service <name>
kubectl delete -f deployment.yaml     # deletes everything defined in the file
kubectl delete deployment,service -l app=<label>   # delete by label

# Nuclear option — reset minikube entirely
minikube delete
minikube start
```

## Reference: test app used — ealen/echo-server

Published Express-based image, no local build/load pipeline needed:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-server
  labels:
    app: echo-server
spec:
  replicas: 5
  selector:
    matchLabels:
      app: echo-server
  template:
    metadata:
      labels:
        app: echo-server
    spec:
      containers:
        - name: echo-server
          image: ealen/echo-server:latest
          ports:
            - containerPort: 80
```

```bash
kubectl apply -f deployment.yaml
kubectl expose deployment echo-server --port=80 --type=ClusterIP
```

To include the pod name (hostname) in the response body:

```bash
kubectl set env deployment/echo-server ECHO_INCLUDE_ENV=true
```

This triggers a rollout; afterward the JSON response includes an `env` section with `HOSTNAME` = pod name — useful for observing load balancing across replicas.

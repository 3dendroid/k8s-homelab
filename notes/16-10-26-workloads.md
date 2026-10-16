# 16-10-26 — Workloads (week 1)

Hands-on with core workload objects on the HA cluster. All experiments ran in the `work` namespace, which was cleaned up at the end.

## Architecture recap

- `kubelet` and `containerd` are systemd services. kubelet starts the control plane as **static pods** from `/etc/kubernetes/manifests` (empty on workers).
- `create` is imperative (`AlreadyExists`, exit 1 on re-run). `apply` is declarative and idempotent (`unchanged`, exit 0). Git is the single source of truth: a manual `scale` is configuration drift and is reverted by the next `apply`.

## Pod

- Most pod fields are immutable, so a change means delete and re-apply.
- Failing **liveness** probe → container restarts. Failing **readiness** probe → pod is `0/1` and removed from the EndpointSlice.
- Experiment: wrong liveness port gave a `1/1` pod with a growing restart count.
- QoS classes: Guaranteed (requests = limits), Burstable, BestEffort (evicted first).
- Default `not-ready`/`unreachable` tolerations are 300s, so pods leave a dead node after about 5 minutes.

## Deployment

- Deployment → ReplicaSet → Pod. A new pod template creates a new ReplicaSet (`pod-template-hash`). Scaling does not.
- RollingUpdate with 3 replicas: `maxSurge=1`, `maxUnavailable=0` (new pod up first, old one removed after).
- `rollout restart | status | history | undo`. `undo` reuses the old ReplicaSet (`revisionHistoryLimit: 10`).

## Service

- Selects pods by labels. The EndpointSlice contains only Ready pods. kube-proxy (nftables) DNATs ClusterIP → pod IP.
- NodePort answers on every node, including nodes without the pod (`externalTrafficPolicy: Cluster`). Verified via a control-plane node IP.

## StatefulSet

- Stable ordinal names. Ordered start. **Scale-down in reverse order, one at a time** (verified 5 → 2: `-4`, `-3`, `-2`).
- A recreated pod keeps its name but gets a new IP → address pods via the headless service DNS name (`<pod>.<headless-svc>.<ns>.svc.cluster.local`), never by IP.
- Why one at a time: clustered stateful apps (etcd, Kafka) depend on **quorum** (N/2 + 1).

## DaemonSet

- No `replicas`: one pod per eligible node.
- Without a toleration, pods run only on workers because of the taint `node-role.kubernetes.io/control-plane:NoSchedule`. Adding the toleration → 5 pods.
- Default update strategy (`maxSurge: 0`, `maxUnavailable: 1`): old pod deleted before the new one starts, node by node. Each node has a short window without the agent.

## Lessons learned

- Deleting a Deployment does not delete its Service. Clean up with the same manifests you applied (`kubectl delete -f`).
- `kubectl -n get pods -A` fails: `-n` consumes the next word as the namespace name.
- Read the output before re-running a command.
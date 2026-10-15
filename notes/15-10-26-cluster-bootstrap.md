# 15.10.26 — Cluster bootstrap (kubeadm, Calico, NodeLocalDNS, join)

Result: 3 control-plane nodes behind a VIP (keepalived + haproxy), 2 workers, Calico CNI,
kube-proxy in nftables mode, NodeLocalDNS. In-cluster DNS verified from a pod.

## 1. First control-plane node (cp1)

kubeadm config (from the course, unchanged):

- `controlPlaneEndpoint: "192.168.218.140:7443"` — the VIP + haproxy, not cp1's own address.
  This endpoint is baked into every kubeconfig; pointing it at a single node would make the
  whole HA setup pointless.
- `kubelet.clusterDNS: 169.254.25.10` — the NodeLocalDNS address. `kubeadm config validate`
  warns that the default would be `10.96.0.10`; this is intentional.
- `kubeProxy.mode: nftables`, `podSubnet: 10.244.0.0/16`.

```bash
kubeadm config validate --config ~/manifests/kubeadm-config.yaml && echo VALID
sudo kubeadm init --config ~/manifests/kubeadm-config.yaml --upload-certs
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes        # NotReady until a CNI is installed
```

The `kubeadm init` output contains secrets (bootstrap token, certificate key). They are not
stored in this repository.

### Calico

- Install the Tigera operator first, then the `Installation` resource. Applying the
  `Installation` first fails with `no matches for kind "Installation"` — the CRD does not
  exist until the operator is installed.
- Before Calico: CoreDNS pods are `Pending` and `/etc/cni/net.d/` is empty.
  After: CoreDNS `Running`, CNI config present, node `Ready`.

### kube-proxy / NodeLocalDNS

```bash
kubectl -n kube-system logs ds/kube-proxy | grep "Using nftables Proxier"
sudo nft list tables                  # ip kube-proxy, ip6 kube-proxy present
ip -br a | grep 169.254               # nodelocaldns dummy interface (state DOWN is normal)
```

Note: my `nft list tables` output has fewer legacy `ip nat/filter/...` tables than the course
reference. Behaviour (service routing, DNS) was verified instead.

## 2. Joining more control-plane nodes (cp2, cp3)

```bash
# on cp1
sudo kubeadm token create --print-join-command        # token + CA cert hash
sudo kubeadm init phase upload-certs --upload-certs   # certificate key (valid 2h)
# on cp2 / cp3
sudo kubeadm join 192.168.218.140:7443 --token <REDACTED> \
  --discovery-token-ca-cert-hash sha256:<REDACTED> \
  --control-plane --certificate-key <REDACTED>
```

| Value | Purpose | Lifetime |
|---|---|---|
| token | the node authenticates to the cluster | 24h |
| CA cert hash | the cluster authenticates to the node (prevents MITM) | static |
| certificate key | decrypts control-plane certs from the `kubeadm-certs` Secret | 2h |

Then spread CoreDNS across nodes:

```bash
kubectl rollout restart deployment/coredns -n kube-system
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
```

haproxy stats: all three `kubernetes-api-backend` servers `UP` (`L4OK`), round-robin visible
in session counters.

## 3. Worker nodes (w1, w2)

```bash
sudo kubeadm join 192.168.218.140:7443 --token <REDACTED> \
  --discovery-token-ca-cert-hash sha256:<REDACTED>
kubectl get nodes -o wide             # 5 nodes Ready
```

DaemonSets (calico-node, kube-proxy, node-local-dns, csi-node-driver) land on new nodes
automatically. Pods with `hostNetwork: true` show the node IP; regular pods get `10.244.x.x`.

## 4. In-cluster DNS test

```bash
kubectl run dnstest --rm -it --restart=Never --image=busybox:1.36 -- \
  nslookup kubernetes.default.svc.cluster.local
# Server: 169.254.25.10 -> 10.96.0.1

kubectl run dnstest --restart=Never --image=busybox:1.36 -- nslookup google.com
kubectl get pod dnstest && kubectl logs dnstest && kubectl delete pod dnstest
```

Path: pod → NodeLocalDNS → CoreDNS → infra dnsmasq → upstream.

## Troubleshooting log

| Symptom | Cause | Fix |
|---|---|---|
| `couldn't validate the identity of the API Server ... none of the public keys are pinned` | CA hash copied from the course instead of my own cluster | take all join values from my cp1 (`--print-join-command`) |
| `[ERROR NumCPU]: the number of available CPUs 1 is less than the required 2` | cloned VM had 1 vCPU | set 2 vCPU in VirtualBox (also fixed on workers); no `kubeadm reset` needed — preflight fails before any change |
| `localhost:8080 ... connection refused` from kubectl on cp2/cp3 | no `~/.kube/config` | copy `admin.conf` (cluster-admin — lab only) |
| test pod `FailedScheduling: untolerated taint` | only control-plane nodes existed; they carry `control-plane:NoSchedule` | run the test after workers join |
| `NXDOMAIN` for `kubernetes.default` | busybox `nslookup` ignores search domains | use the FQDN |
| `openssl: command not found` / `tar: command not found` | Minimal Install | install the package |

## Takeaways

- Read the error before retrying; a repeated command gives the same result.
- Never reuse example values from a guide — they belong to someone else's cluster.
- With 2 etcd members, quorum is 2: join the third control-plane node right away.
- When an interactive pod finishes too fast for `-it`, run it detached and read `kubectl logs`.
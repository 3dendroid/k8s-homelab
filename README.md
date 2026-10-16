# k8s-homelab

Highly available Kubernetes cluster built from scratch with **kubeadm** on VirtualBox VMs — a hands-on learning project on my way from QA to DevOps.

Based on the course *Junior-01* by Artur Kryukov (CC BY-NC-SA 4.0). Manifests in `manifests/` are from the course; notes are my own.

## Topology

| Host | IP | Role |
|---|---|---|
| — | 192.168.218.140 | VIP (kube-apiserver via haproxy :7443) |
| cp1–cp3.local.lan | .141–.143 | control plane (keepalived + haproxy, stacked etcd) |
| w1–w2.local.lan | .144–.145 | workers |
| infra.local.lan | .13 | DNS (dnsmasq) + NFS |

Rocky Linux 10, containerd, Kubernetes v1.37, Calico CNI, kube-proxy in nftables mode, NodeLocalDNS.

## Progress

- [x] Test stand: golden image, static IPs, FQDN hostnames
- [x] Infra VM: dnsmasq (local.lan zone) + NFS share
- [x] Node preparation: swap, SELinux, kernel modules, sysctl, containerd
- [x] HA control-plane endpoint: keepalived + haproxy, failover tested
- [x] `kubeadm init` + Calico + NodeLocalDNS
- [x] Join control-plane and worker nodes — 5 nodes `Ready`, in-cluster DNS verified
- [x] Workloads: namespaces, pods, deployments, services, statefulsets, daemonsets

## Notes

- [15-10-26 — Cluster bootstrap](notes/15-10-26-cluster-bootstrap.md)
- [16-10-26 — Workloads](notes/16-10-26-workloads.md)

## Lessons learned

- A fix is only verified after reproducing the failure (e.g. reboot test caught a dnsmasq boot-order race).
- Untested HA is just an assumption: VIP failover and preemption were tested live.
- The CA cert hash in `kubeadm join` really protects you: a wrong hash makes the node refuse to join.
- Deleting a Deployment leaves its Service behind: clean up with the same manifests you applied.
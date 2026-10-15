# k8s-homelab

Highly available Kubernetes cluster built from scratch with **kubeadm** on VirtualBox VMs — a hands-on learning project on my way from QA to DevOps.

Based on the course *Junior-01* by Artur Kryukov (CC BY-NC-SA 4.0). Manifests in `manifests/` are from the course; notes are my own.

## Topology

| Host | IP | Role |
|---|---|---|
| — | 192.168.218.140 | VIP (kube-apiserver via haproxy :7443) |
| cp1–cp3.local.lan | .141–.143 | control plane (keepalived + haproxy) |
| w1–w2.local.lan | .144–.145 | workers |
| infra.local.lan | .13 | DNS (dnsmasq) + NFS |

Rocky Linux 10, containerd, Kubernetes v1.37, kube-proxy in nftables mode.

## Progress

- [x] Test stand: golden image, static IPs, FQDN hostnames
- [x] Infra VM: dnsmasq (local.lan zone) + NFS share
- [x] Node preparation: swap, SELinux, kernel modules, sysctl, containerd
- [x] HA control-plane endpoint: keepalived + haproxy, failover tested
- [ ] `kubeadm init` + Calico
- [ ] Join control-plane and worker nodes

## Lessons learned

- A fix is only verified after reproducing the failure (e.g. reboot test caught a dnsmasq boot-order race).
- Untested HA is just an assumption: VIP failover and preemption were tested live.
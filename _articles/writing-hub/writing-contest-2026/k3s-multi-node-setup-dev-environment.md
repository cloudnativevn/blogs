---
layout: post
title: "Setting Up a Multi-Node k3s Cluster: A Practical Guide for Dev Environments"
date: 2026-04-15
author: "Andrew"
tags:
  - kubernetes
  - k3s
  - devops
  - writing-hub
excerpt: "A hands-on walkthrough of setting up a 3-node k3s cluster from scratch — including master and worker installation, cluster verification, local kubectl configuration, and practical cluster management tips."
---

## Introduction

When you need a Kubernetes environment that mirrors production but without the overhead of a full cluster, k3s is hard to beat. It's a single binary under 100 MB, uses roughly 512 MB of RAM on the server node, and supports the same `kubectl`, Helm, and Kustomize tooling you'd use on any other Kubernetes distribution.

I recently set up a 3-node k3s cluster on bare-metal servers to run a multi-service dev environment — AI inference workloads, a telephony stack, databases, and message queues. This article walks through exactly what I did: installing the control plane, joining worker nodes, verifying the cluster, and the commands I reach for most often to keep it running.

---

## Cluster Architecture

The setup uses three Ubuntu 24 bare-metal servers, each with a 4-core CPU and 16 GB RAM:

```
Node 1 (master)   192.168.x.10   — control plane
Node 2 (worker)   192.168.x.11   — application workloads
Node 3 (worker)   192.168.x.12   — application workloads
```

One design decision worth noting upfront: k3s ships with **Traefik** as its default ingress controller. I disabled it at install time to use ingress-nginx instead, which matches what runs in production. This avoids configuration drift between environments. To do this, pass `--disable traefik` during the k3s install.

---

## Step 1: Install the Master Node

SSH into Node 1 and run:

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik" sh -
```

Once the install script completes, verify the service is running:

```bash
systemctl status k3s
```

k3s writes a kubeconfig to `/etc/rancher/k3s/k3s.yaml`. You can use it locally on the master or copy it to your workstation (covered in Step 4).

Finally, grab the node join token — you'll need it to add worker nodes:

```bash
cat /var/lib/rancher/k3s/server/node-token
```

### Firewall Requirements

Before moving on, make sure the following ports are open between all nodes:

| Port | Protocol | Purpose |
|------|----------|---------|
| 6443 | TCP | Kubernetes API server |
| 8472 | UDP | Flannel VXLAN (pod networking) |
| 10250 | TCP | Kubelet metrics |

---

## Step 2: Join Worker Nodes

SSH into Node 2 (then repeat for Node 3) and run:

```bash
curl -sfL https://get.k3s.io | \
  K3S_URL=https://<master-ip>:6443 \
  K3S_TOKEN=<token-from-master> \
  sh -
```

Verify the agent is running on each worker:

```bash
systemctl status k3s-agent
```

If you want to pin certain workloads to specific nodes — for example, isolating database-adjacent services — label the nodes:

```bash
kubectl label node <node-name> role=infra
kubectl label node <node-name> role=app
```

You can then use `nodeSelector` in your pod specs to target those labels.

---

## Step 3: Verify the Cluster

Back on the master (or from your local machine after Step 4), check that all nodes are `Ready`:

```bash
kubectl get nodes -o wide
```

Expected output:

```
NAME     STATUS   ROLES                  AGE   VERSION
node-1   Ready    control-plane,master   2m    v1.x.x+k3s1
node-2   Ready    <none>                 1m    v1.x.x+k3s1
node-3   Ready    <none>                 1m    v1.x.x+k3s1
```

Check that the core system pods are healthy:

```bash
kubectl get pods -n kube-system
```

Run a quick smoke test to confirm scheduling works across all nodes:

```bash
kubectl run nginx --image=nginx --port=80
kubectl get pods -o wide   # confirm the pod lands on a worker node
kubectl delete pod nginx
```

---

## Step 4: Configure Local kubectl Access

k3s writes its kubeconfig to `/etc/rancher/k3s/k3s.yaml` on the master. Copy it to your workstation:

```bash
scp user@<master-ip>:/etc/rancher/k3s/k3s.yaml ~/.kube/config
```

The file defaults to `127.0.0.1` as the server address — update it to the master's real IP:

```bash
sed -i 's/127.0.0.1/<master-ip>/g' ~/.kube/config
```

Test it:

```bash
kubectl get nodes
```

If you already have a kubeconfig with other clusters, merge the k3s context in rather than overwriting the file entirely. The `KUBECONFIG` env var supports colon-separated paths:

```bash
export KUBECONFIG=~/.kube/config:~/.kube/k3s-config
kubectl config view --merge --flatten > ~/.kube/merged-config
```

---

## Step 5: Useful Commands for Daily Cluster Management

These are the commands I use most often after the cluster is up.

### Monitoring

```bash
# Node resource usage (requires metrics-server — see Lessons Learned)
kubectl top nodes

# Pod resource usage across a namespace
kubectl top pods -n <namespace>

# Watch pod status in real time
kubectl get pods -n <namespace> -w
```

### Logs and Debugging

```bash
# Stream logs, last 100 lines
kubectl logs <pod> -n <namespace> --tail=100 -f

# Logs from a previous (crashed) container instance
kubectl logs <pod> -n <namespace> --previous

# Describe a pod to see events and failure reasons
kubectl describe pod <pod> -n <namespace>
```

### Restarting and Cleaning Up

```bash
# Rolling restart a deployment (no downtime)
kubectl rollout restart deployment/<name> -n <namespace>

# Force delete a pod stuck in Terminating
kubectl delete pod <pod> -n <namespace> --grace-period=0 --force

# Remove all pods in Error or Completed state
kubectl delete pods -n <namespace> --field-selector=status.phase=Failed
```

### Node Maintenance

```bash
# Safely evict all pods from a node before maintenance
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data

# Bring the node back after maintenance
kubectl uncordon <node>

# Check k3s service status directly on a node
systemctl status k3s          # on master
systemctl status k3s-agent    # on workers
```

---

## Lessons Learned

Three issues I ran into that are worth knowing before you start:

| Issue | Root Cause | Fix |
|-------|-----------|-----|
| Pods stuck in `OOMKilled` after the first big deploy | Total memory *limits* across all pods (summed up to ~21 Gi) exceeded the physical RAM available on the node (16 GB) | Plan resource limits before deploying: `sum(limits) ≤ node RAM`. Scale down non-critical workloads temporarily if needed. |
| Worker node randomly goes `NotReady` | Flannel VXLAN traffic (UDP 8472) was blocked between nodes by the firewall | Open UDP 8472 on all nodes. Use `kubectl describe node <node>` to spot network plugin errors quickly. |
| `kubectl top` returns `error: Metrics API not available` | k3s does not bundle metrics-server by default | Install it: `kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml` |

---

## Key Takeaways

- k3s installs in under 2 minutes and is fully compatible with standard Kubernetes tooling — a solid choice for dev and staging environments.
- Disable Traefik at install time (`--disable traefik`) if your production environment uses ingress-nginx, to keep the two environments consistent.
- Before deploying a large workload stack, add up the memory *limits* across all pods and compare against available node RAM. Exceeding this causes `OOMKilled` pods that are easy to misdiagnose.
- Treat the node join token like a credential — anyone with it can add nodes to your cluster.
- Use `kubectl drain` before any node maintenance to move workloads safely, and `kubectl uncordon` to bring the node back.

---

## References

- [k3s Documentation](https://docs.k3s.io/)
- [k3s Installation Options](https://docs.k3s.io/installation/configuration)
- [Kubernetes metrics-server](https://github.com/kubernetes-sigs/metrics-server)
- [Kubernetes Node Management](https://kubernetes.io/docs/concepts/architecture/nodes/)

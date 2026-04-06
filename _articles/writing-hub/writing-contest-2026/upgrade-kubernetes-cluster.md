---
layout: post
title: "Kubernetes Cluster Upgrade Tips & Hands-on Guide for Air-Gapped Environments with Kubespray"
date: 2026-04-03
author: "Toi Pham Van"
tags: [writing-contest-2026-term1, kubernetes, kubespray, air-gapped, tutorial]
excerpt: "Upgrading Kubernetes in an air-gapped environment is a different beast from a standard upgrade. This guide covers the gotchas, preparation steps, and a complete hands-on walkthrough of upgrading from v1.26.11 to v1.27.7 with Kubespray — so you can do it safely, even without internet access."
---

Upgrading Kubernetes is never trivial — but doing it in an **air-gapped environment** adds an entirely different layer of complexity. No `docker pull`. No `apt-get install`. Every binary, image, and dependency must be accounted for before the upgrade window opens. Miss one artifact and the playbook stalls mid-run, leaving your cluster in a partially upgraded state.

This article is structured in two parts:

1. **Upgrade fundamentals** — the principles and tips that apply regardless of tooling or environment.
2. **Hands-on walkthrough** — a real upgrade from **v1.26.11 → v1.27.7** using Kubespray in a fully isolated network, with practical observations from the field.

---

## Part 1: Before You Begin

### Key Considerations

Before starting any upgrade, keep these principles in mind:

- **Understand the Changelog & Release Notes**: Read breaking changes and deprecated features carefully.
- **Check compatibility**: Ensure all cluster components — add-ons, third-party tools, and applications — are compatible with the new K8S version.
- **Back up your data**: An ETCD snapshot is mandatory before proceeding.
- **Monitor throughout**: Watch logs, resource usage, and node/pod status during the entire upgrade process.
- **Upgrade nodes one at a time** (staggered/rolling) to minimize downtime.
- **Upgrade one minor version at a time**: e.g., v1.20 → v1.21 → v1.22. Never skip versions.

### Why You Should Never Skip Versions

This is a common question. There are three key reasons:

1. **API Deprecation lifecycle**: Version N+1 typically marks an API as deprecated but still functional. Later versions actually remove it — at which point existing manifests break immediately and cause downtime.
2. **ETCD schema migration**: ETCD's internal schema can change between versions. Tools like `kubeadm` do not support skipping the data migration logic (N → N+2). Attempting this can corrupt the cluster state entirely.
3. **Version Skew Policy**: Kubernetes enforces specific version skew constraints between components. Violating these can result in undefined behavior.

**The rule is simple: v1.26 → v1.27 → v1.28. Never v1.26 → v1.28.**

---

### What You're Actually Upgrading

| Component | Focus Area | Critical Notes |
|---|---|---|
| Control Plane | API Server, Scheduler, Controller Manager | Upgrade sequentially (e.g., `1.29 -> 1.30`). Never skip minor versions. |
| Worker Nodes | Kubelet, Kube-proxy | Ensure kubelet versions do not exceed the API server version. |
| Etcd Cluster | Data Store | Perform a manual backup before starting. Highly sensitive to disk I/O and latency. |
| Container Runtime | Containerd, CRI-O, Docker | Verify the CRI compatibility matrix for the target Kubernetes version. |
| Networking (CNI) | Calico, Flannel, Cilium | Ensure the CNI plugin supports the new Kubernetes version to avoid `Network Unavailable` errors. |
| Cluster Add-ons | CoreDNS, Kube-proxy, Metrics Server | Often overlooked; check for deprecated API versions in your manifests and Helm charts. |

---

### High-Level Upgrade Steps

1. **Prepare and validate the upgrade plan**  
   Review release notes, verify compatibility, scan for deprecated APIs, back up ETCD, and prepare offline artifacts.

2. **Upgrade the control plane nodes**  
   Upgrade the primary control plane first, then roll through the remaining control plane nodes.

3. **Upgrade worker nodes**  
   Drain and upgrade worker nodes in batches, or one node at a time.

4. **Upgrade client tools**  
   Update tools such as `kubectl` to match the target cluster version.

5. **Update manifests and resources**  
   Migrate deprecated APIs and make sure manifests remain compatible with the new Kubernetes version.

6. **Verify cluster health**  
   Confirm node and pod status, control plane health, networking, storage, and application stability.

---

### Tip: Emulation Mode (K8S v1.32+)

Starting from K8S v1.32, two new flags allow you to simulate a newer version **before** actually upgrading:

- `--emulated-version`: Forces the K8S cluster component to run the new binary while emulating the old version's behavior.
- `--min-compatibility-version`: Sets the lowest version that cluster components are committed to being compatible with.

This is useful for pre-flight compatibility checks before committing to a real upgrade.

---

## Part 2: Hands-On — Upgrading v1.26.11 → v1.27.7 in an Air-Gapped Environment

### The Environment

- **Cluster**: Multi-node, control plane HA (3 masters), several worker nodes
- **Network**: Fully air-gapped — no outbound internet from any cluster node
- **Tooling**: Kubespray v2.23.1 (the release that supports K8S v1.27.7)
- **Container runtime**: containerd
- **CNI**: Calico

---

### Step 1: Scan for Deprecated APIs

Before touching the cluster, find out what will break. Use [kubent (kube-no-trouble)](https://github.com/doitintl/kube-no-trouble) — it scans live cluster resources and Helm release manifests against the target version's removal rules.

```bash
kubent --target-version 1.27.0
```

```
11:39AM INF Initializing collectors and retrieving data
11:39AM INF Target K8s version is 1.27.0
11:39AM INF Retrieved 66 resources from collector name=Cluster
11:39AM INF Retrieved 322 resources from collector name="Helm v3"
11:39AM INF Loaded ruleset name=deprecated-1-27.rego
__________________________________________________________________________________________
>>> Deprecated APIs to be removed in future <<<
------------------------------------------------------------------------------------------
KIND                  NAMESPACE     NAME                  API_VERSION                       REPLACE_WITH (SINCE)
VolumeSnapshotClass   <undefined>   cinder-csi-snapshot   snapshot.storage.k8s.io/v1beta1   snapshot.storage.k8s.io/v1 (1.21.0)
```

One finding: a `VolumeSnapshotClass` resource still using the deprecated `snapshot.storage.k8s.io/v1beta1` API. It won't break immediately in v1.27, but scheduling the migration now avoids surprises in the next upgrade cycle.

> **Run kubent early** — ideally weeks before the upgrade, not the night before. This gives your team time to update Helm charts or manifests without rushing.

---

### Step 2: Back Up ETCD

This is the only true safety net. If the upgrade goes wrong, your recovery path is: restore ETCD snapshot → recreate cluster state. Without the backup, you have no recovery path.

```bash
# Verify ETCD cluster health before taking the snapshot
etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/admin-master1.pem \
  --key=/etc/ssl/etcd/ssl/admin-master1-key.pem \
  endpoint health

# Take the snapshot
etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/admin-master1.pem \
  --key=/etc/ssl/etcd/ssl/admin-master1-key.pem \
  snapshot save backup-etcd-$(date +%Y%m%d).db
```

Store the snapshot somewhere outside the cluster — on a jump server or a separate storage system. An ETCD backup sitting on the ETCD node itself is not a backup.

---

### Step 3: Prepare Kubespray

Check out the Kubespray release that supports your target K8S version. For v1.27.7, that's Kubespray **v2.23.1**:

```bash
git checkout v2.23.1

# Install Python dependencies (via private PyPI mirror in air-gapped environments)
pip install --no-cache-dir -r requirements.txt
```

**Update the inventory carefully.** Don't blindly copy the old inventory into the new version. Kubespray's variable names and group structures change between releases. The recommended approach:

1. Start from the new version's `inventory/sample` as a baseline.
2. Diff your old inventory against the new sample to identify changed or removed variables.
3. Apply your site-specific values (node IPs, credentials, custom flags) on top.

Skipping this step is one of the most common sources of subtle upgrade failures — you may be setting variables that no longer exist, or missing new required ones.

---

### Step 4: Prepare Offline Artifacts (Air-Gap Specific)

This is the part that makes air-gapped upgrades uniquely challenging, and where most teams underestimate the effort.

Kubespray ships with helper scripts under `contrib/offline/` that handle artifact management:

```
contrib/offline/
├── generate_list.sh                      # Generates the full list of required binaries & images
├── manage-offline-files.sh               # Downloads binaries and packages them as tar
└── manage-offline-container-images.sh    # Pulls container images and saves them as tar
```

**On a machine with internet access**, run the following before the upgrade window:

```bash
cd contrib/offline

# Generate the artifact manifest for the target versions
./generate_list.sh

# Download all required binaries (kubectl, kubeadm, cni plugins, etc.)
./manage-offline-files.sh

# Pull all required container images and save to tar
./manage-offline-container-images.sh create
```

**Transfer the artifacts** into the air-gapped environment via USB drive, secure file transfer, or whatever your organization's secure transport mechanism is.

**Inside the air-gapped environment**, stand up the local artifact server before running the upgrade:

```bash
# Serve binaries via a local nginx container
./manage-offline-files.sh

# Load images into the local registry
./manage-offline-container-images.sh register
```

> **Note**: Run `generate_list.sh` against the *new* Kubespray version (v2.23.1), not the old one. The artifact list changes between Kubespray releases. Also, cross-check the generated image list against what's already in your local registry — you may only need to transfer the delta.

---

### Step 5: Run the Upgrade

**Refresh node facts first.** Stale facts from a previous Ansible run can cause the upgrade playbook to make wrong decisions about node state:

```bash
ansible-playbook playbooks/facts.yml -b \
  -i inventory/private_hlc/inventory.ini \
  -e "ansible_user=vt_admin \
      ansible_password=$ANSIBLE_SSH_PASS \
      become_method=su \
      ansible_become_password=$ANSIBLE_BECOME_PASS"
```

**Upgrade the control plane and ETCD first, one node at a time:**

```bash
ansible-playbook upgrade-cluster.yml -b \
  -i inventory/private_hlc/inventory.ini \
  -e kube_version=v1.27.7 \
  -e upgrade_node_confirm=true \
  -e upgrade_node_post_upgrade_confirm=true \
  -e serial=1 \
  --limit "kube_control_plane:etcd"
```

Here's what each key parameter does and *why* it matters:

| Parameter | Value | Why it matters |
|---|---|---|
| `kube_version` | `v1.27.7` | Target version. Must match what's in your offline artifact set. |
| `serial` | `1` | Overrides Kubespray's default of 20% of nodes at once. With `serial=1`, you upgrade one node, verify it, then move to the next. Slower, but far safer. |
| `upgrade_node_confirm` | `true` | Pauses the playbook *before* upgrading each node, requiring you to type `yes`. Use this to manually inspect cluster state before each node drains. |
| `upgrade_node_post_upgrade_confirm` | `true` | Pauses *after* upgrading each node while it's still cordoned. Inspect logs and pod status before the node is uncordoned and rejoins the pool. |
| `--limit kube_control_plane:etcd` | — | Restricts execution to control plane and ETCD nodes only. Workers are not touched in this run. |

> After each control plane node completes, check the cluster before confirming the next node: `kubectl get nodes`, `kubectl get pods -A | grep -v Running`. Any `CrashLoopBackOff` or stuck `Pending` pods are a signal to stop and investigate before continuing.

**Upgrade worker nodes after all control plane nodes are healthy:**

```bash
ansible-playbook upgrade-cluster.yml -b \
  -i inventory/private_hlc/inventory.ini \
  -e kube_version=v1.27.7 \
  -e upgrade_node_confirm=true \
  -e upgrade_node_post_upgrade_confirm=true \
  --limit "kube_node"
```

For large clusters, you can upgrade workers in named batches by adjusting `--limit`:

```bash
# Upgrade a specific batch of workers
--limit "node1:node2:node3"
```

This gives you control over the blast radius — upgrade a small group, validate, then proceed to the next batch.

---

### Post-Upgrade Checklist

Work through these systematically before declaring the upgrade complete:

- [ ] All nodes in `Ready` state: `kubectl get nodes`
- [ ] All system pods running: `kubectl get pods -n kube-system`
- [ ] Component versions match target: `kubectl version`, `kubectl get nodes -o wide`
- [ ] CNI healthy: pod-to-pod and pod-to-service communication working
- [ ] CSI healthy: existing PVCs still bound and accessible
- [ ] Control plane logs clean: no unexpected errors in `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`
- [ ] Application workloads stable: no unexpected restarts since the upgrade

**Migrate deprecated APIs.**  For the `VolumeSnapshotClass` found in Step 1:

```bash
# Export the current resource
kubectl get volumesnapshotclass cinder-csi-snapshot -o yaml > volumesnapshotclass.yaml

# Convert to the new API version
# Note: kubectl convert is a plugin, not built into kubectl
kubectl convert \
  -f volumesnapshotclass.yaml \
  --output-version snapshot.storage.k8s.io/v1 \
  -o yaml > volumesnapshotclass-v1.yaml

# Apply the migrated resource
kubectl apply -f volumesnapshotclass-v1.yaml
```

For Helm-managed resources with deprecated API versions, use [helm-mapkubeapis](https://github.com/helm/helm-mapkubeapis) to update the Helm release metadata without having to re-install the chart.

---

## Key Takeaways

1. **Always back up ETCD first** — No exceptions. It is the single most important safety net.
2. **Scan for deprecated APIs early** — Use `kubent` well in advance to give yourself time to migrate manifests before the upgrade window.
3. **Prepare offline artifacts thoroughly** — In air-gapped environments, a single missing image or binary can block the entire process. Verify the artifact list before starting.
4. **Upgrade rolling, one node at a time** — Especially for control planes, `serial=1` lets you catch problems early and roll back if needed.
5. **Validate after each node** — Don't rush to the next node before confirming the current one is stable.

---

## References

- [Kubespray - Upgrade documentation](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/operations/upgrades.md)
- [Kubernetes - Cluster Upgrade](https://kubernetes.io/docs/tasks/administer-cluster/cluster-upgrade/)
- [Kubernetes - kubeadm upgrade](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- [Kubernetes - Version Skew Policy](https://kubernetes.io/docs/setup/release/version-skew-policy/)
- [Kubernetes - Compatibility Versions](https://kubernetes.io/docs/concepts/cluster-administration/compatibility-version/)
- [kube-no-trouble (kubent)](https://github.com/doitintl/kube-no-trouble)
- [helm-mapkubeapis](https://github.com/helm/helm-mapkubeapis)
- [Medium - Kubernetes upgrade using Kubespray](https://medium.com/h7w/kubernetes-upgrade-using-kubespray-c354e184059b)
- [AWS EKS - Cluster Upgrade Best Practices](https://docs.aws.amazon.com/eks/latest/best-practices/cluster-upgrades.html)
- [GKE - Best practices for upgrading clusters](https://docs.cloud.google.com/kubernetes-engine/docs/best-practices/upgrading-clusters)

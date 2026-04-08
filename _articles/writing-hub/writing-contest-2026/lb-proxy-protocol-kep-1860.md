---
layout: post
title: "kube-proxy Intercepts LoadBalancer IP: Root Cause, Debug, and Fix for Proxy Protocol on Kubernetes"
date: 2026-04-04
author: "andrew"
tags: [writing-contest-2026-term1, kubernetes, proxy-protocol, kube-proxy, networking, kep-1860]
excerpt: "How we debugged kube-proxy bypassing our Load Balancer when Proxy Protocol was enabled on VKS — the hostname workaround, and the proper fix with KEP-1860 ipMode: Proxy."
---

## 1. Context: NLB → Nginx Ingress on VKS

Our system runs on **VNG Kubernetes Service (VKS)** — a managed Kubernetes service by [GreenNode.ai](https://greennode.ai). The traffic exposure model is common among many teams:

```text
Internet
    │
    v
┌───────────────────────────────────┐
│   vLB Network Load Balancer       │  <-- L4, TCP passthrough, Public IP
│   (auto-provisioned by VKS)       │
└─────────────────┬─────────────────┘
                  │
                  v
┌───────────────────────────────────┐
│   Nginx Ingress Controller        │  <-- L7, TLS termination
│   (inside cluster)                │      route by host / path
└─────────┬──────────────┬──────────┘
          │              │
          v              v
     Service A      Service B
     (api.*)        (admin.*)
```

When a `LoadBalancer` Service is created for Nginx Ingress, VKS automatically provisions an NLB and sets a public IP in `status.loadBalancer.ingress`:

```bash
kubectl get svc -n ingress-nginx nginx-ingress-controller
# NAME                       TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)
# nginx-ingress-controller   LoadBalancer   10.96.45.12   103.245.252.75   80:31080/TCP,443:31443/TCP
```

`103.245.252.75` is the NLB's public IP — the address that `api.company.com` points to.

---

## 2. The Requirement: Real Client IP at the Backend

The team needed backend services to log the **real client IP** for audit, rate limiting, and geo-blocking. Without extra configuration, the backend only sees the Nginx pod IP, not the actual client.

The solution was to enable **Proxy Protocol** following [GreenNode's guide](https://docs.vngcloud.vn/vng-cloud-document/vks/network/working-with-nlb/preserve-source-ip-when-using-nlb-and-nginx-ingress-controller). With this enabled, the NLB injects a special header at the start of each TCP stream:

> **Proxy Protocol**: A network protocol developed by HAProxy that operates at the TCP layer. The load balancer prepends a header to each TCP stream containing the original client IP before forwarding to the backend.

```text
PROXY TCP4 <client-ip> <nlb-ip> <client-port> <nlb-port>\r\n
```

Nginx reads this header to get the real client IP and forwards it via `X-Forwarded-For`. After following the docs and testing from outside — the real client IP appeared in logs. **Everything looked fine.**

---

## 3. The Incident: Smoke Tests Start Failing

After enabling Proxy Protocol, Nginx Ingress logs began showing strange errors:

```bash
kubectl logs -n ingress-nginx nginx-ingress-controller-xxx | grep "api.company.com"
# 2024/01/15 10:23:41 [error] 123#123: *456 SSL_do_handshake() failed
#   (SSL: error:14094418:SSL routines:ssl3_read_bytes:tlsv1 alert unknown ca)
#   while SSL handshaking, client: 10.244.2.15, server: 0.0.0.0:443
```

The **smoke test** — a Kubernetes Job that runs after every deploy to verify the full production path by calling `api.company.com` directly — started failing completely:

```bash
kubectl logs smoke-test-job-a1b2c3
# [FAIL] GET https://api.company.com/healthz
# curl: (56) Recv failure: Connection reset by peer
#
# [FAIL] GET https://api.company.com/v1/products
# curl: (52) Empty reply from server
```

Meanwhile, tests from a laptop outside the cluster passed perfectly. **Same domain, same endpoint — outside OK, inside fail.**

---

## 4. Debugging: Where Does the Traffic Go?

### Observation from the smoke test pod

Connections were reset immediately, not timed out — a sign that the server was actively closing the connection, not a network issue.

### tcpdump on the node

```bash
tcpdump -i any host 103.245.252.75 -n
# Result: NO packets to/from 103.245.252.75 !
```

Traffic destined for the NLB IP **never left the node**.

### tcpdump on the NLB (via GreenNode support)

We asked the GreenNode support team to capture traffic on the NLB while the smoke test was running. Result: **no connections from the cluster IP were seen at the NLB at all.** The NLB never saw the smoke test requests.

**Conclusion:** Traffic from the smoke test pod to `103.245.252.75` was being swallowed somewhere on the node.

### Checking iptables

```bash
iptables -t nat -L KUBE-SERVICES -n | grep 103.245.252.75
# KUBE-SVC-XXXXXXXXXXXXXX  tcp  --  0.0.0.0/0  103.245.252.75  tcp dpt:443
# KUBE-SVC-YYYYYYYYYYYYYY  tcp  --  0.0.0.0/0  103.245.252.75  tcp dpt:80
```

**Found it.** kube-proxy had created iptables rules for `103.245.252.75`. Any traffic from inside the cluster to this IP was being intercepted and DNAT'd directly to the Nginx pod — **bypassing the NLB entirely**.

```bash
iptables -t nat -L KUBE-SVC-XXXXXXXXXXXXXX -n
# KUBE-SEP-ZZZZZZ  tcp -- DNAT to 10.244.1.8:443  (Nginx pod IP)
```

---

## 5. Root Cause: kube-proxy LB IP Interception Is Intentional

This is not a bug. The [Kubernetes blog officially explains](https://kubernetes.io/blog/2023/12/18/kubernetes-1-29-feature-loadbalancer-ip-mode-alpha/) this is deliberate behavior for two reasons:

**Reason 1 — Traffic path optimization.** Without interception, in-cluster traffic to the LB would take a hairpin path: `Pod → exit node → NLB → back to cluster → Nginx pod`. With interception, kube-proxy short-circuits this: `Pod → kube-proxy DNAT → Nginx pod`. Saves latency and bandwidth when the LB is just a forwarder.

**Reason 2 — Health-check packet handling.** Some LBs send health-check packets with the LB IP as the destination. Without iptables rules, these packets have nowhere to go.

### When does interception become a problem?

When the LB has **additional processing logic** that packets must pass through:

```text
With Proxy Protocol:
  Pod → [kube-proxy DNAT] → Nginx  FAIL
  (NLB never sees the packet → no PP header injected)
  (Nginx expects PP header → rejects connection)
```

This is exactly our case: Nginx was configured with `use-proxy-protocol: "True"` — meaning **every incoming connection must have a PP header**. Connections from pods had no header → Nginx rejected them immediately.

```text
================================================================
AFTER: Proxy Protocol ON (per GreenNode docs)
================================================================

External client                  Smoke test pod (inside cluster)
      │                                       │
      v                                       │ calls 103.245.252.75
┌─────────────┐                               │
│  vLB NLB    │                               │ kube-proxy DNAT
│  +PP header │                               │ (same shortcut!)
└──────┬──────┘                               │
       │ TCP + PP header                      │ (no PP header)
       v                                      v
┌──────────────────────┐          ┌──────────────────────┐
│    Nginx Ingress     │          │    Nginx Ingress     │
│ use-proxy-protocol:  │          │ use-proxy-protocol:  │
│        true          │          │        true          │
│  reads PP header OK  │          │  NO PP header !!     │
└──────────┬───────────┘          └──────────────────────┘
           v                                  v
     Backend OK                    Connection reset !!
```

---

## 6. The Solutions

After identifying the root cause, we worked with the GreenNode support team on two approaches.

### Option 1: Use hostname instead of IP for the LB

kube-proxy only creates iptables rules based on **IPs** from `status.loadBalancer.ingress[].ip`. If a hostname is used instead, kube-proxy cannot create rules, so traffic follows the real routing path → goes through the NLB → PP header is injected.

> This is why AWS NLBs never have this problem — they always return a hostname in `status.loadBalancer.ingress`, never an IP.

For cloud providers that return an IP, a wildcard DNS service like [nip.io](https://nip.io) can wrap it: `103.245.252.75.nip.io` resolves to `103.245.252.75` with no configuration needed.

GreenNode updated their controller to set hostname instead of IP:

```yaml
# Before
status:
  loadBalancer:
    ingress:
    - ip: "103.245.252.75"

# After (workaround)
status:
  loadBalancer:
    ingress:
    - hostname: "103.245.252.75.nip.io"
```

**Result:** Smoke tests passed. Real client IPs appeared in logs.

**Trade-offs:** In-cluster traffic now takes the longer path through the NLB instead of being DNAT'd directly; `EXTERNAL-IP` shows a hostname instead of an IP; depends on an external DNS service.

### Option 2: `ipMode: Proxy` — The Official Fix (K8s 1.29+)

The [Kubernetes blog acknowledges](https://kubernetes.io/blog/2023/12/18/kubernetes-1-29-feature-loadbalancer-ip-mode-alpha/) the hostname workaround is a "makeshift solution" and introduced `ipMode` in K8s 1.29:

```yaml
status:
  loadBalancer:
    ingress:
    - ip: "103.245.252.75"
      ipMode: "Proxy"    # kube-proxy will NOT create iptables rules for this IP
```

| K8s Version | Status |
|---|---|
| 1.29 | Alpha |
| 1.30 | Beta |
| **1.32** | **Stable / GA** |

**Our approach:** Hostname workaround for clusters running K8s < 1.29, migrating to `ipMode: Proxy` on upgrade.

---

## 7. What About Cilium?

If the cluster uses **Cilium** as a kube-proxy replacement, does this problem still occur?

**Yes — and it manifests even earlier.**

Cilium uses socket-level load balancing via eBPF — intercepting at the `connect()` syscall, before packets even enter the network stack. But the mechanism is identical: Cilium reads `status.loadBalancer.ingress[].ip` and builds eBPF maps from it.

```text
kube-proxy (iptables):
  Packet → iptables DNAT → Backend pod   (intercept at L3/L4)

Cilium socket-LB (eBPF):
  connect(LB_IP) → eBPF hook → redirect → Backend pod   (intercept at socket layer)
```

Importantly, **neither kube-proxy nor Cilium processes `ingress[].hostname`**. So:
- The nip.io workaround works with both
- `ipMode: Proxy` is the proper fix for both

This makes the hostname workaround a **universal fix** — independent of the CNI plugin in use.

---

## 8. Key Takeaways

1. **When debugging K8s networking: don't trust application logs, trace actual packets.**

   ```bash
   iptables -t nat -L KUBE-SERVICES -n | grep <lb-ip>
   tcpdump -i any host <lb-ip> -n -v
   ```

2. **kube-proxy intercepting LB IPs is intentional** — it optimizes the traffic path. But when the LB has additional processing (TLS termination, Proxy Protocol, WAF...), bypassing it breaks those features.

3. **The hostname workaround is universal** — works with kube-proxy (iptables/IPVS) and Cilium (eBPF), since both only process `ingress[].ip`.

4. **Migrate to `ipMode: Proxy` when possible** — the clean, transparent solution that requires no DNS workarounds.

---

## References

1. [GreenNode.ai Docs: Preserve Source IP when using NLB and Nginx Ingress Controller](https://docs.vngcloud.vn/vng-cloud-document/vks/network/working-with-nlb/preserve-source-ip-when-using-nlb-and-nginx-ingress-controller.md)
2. [Kubernetes Blog: Load Balancer IP Mode for Services (K8s 1.29 alpha)](https://kubernetes.io/blog/2023/12/18/kubernetes-1-29-feature-loadbalancer-ip-mode-alpha/)
3. [KEP-1860: Service LoadBalancer IP Mode](https://github.com/kubernetes/enhancements/issues/1860)
4. [Deckhouse Issue #14657: No more domain workaround needed for Proxy Protocol on K8s 1.32+](https://github.com/deckhouse/deckhouse/issues/14657)
5. [Cilium Docs: Kubernetes Without kube-proxy](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/)
6. [nip.io — Wildcard DNS for any IP Address](https://nip.io)

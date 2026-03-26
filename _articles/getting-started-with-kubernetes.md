---
layout: post
title: "Getting Started with Kubernetes in Vietnam"
date: 2026-03-20
author: Cloud Native Vietnam
tags: [kubernetes, tutorial]
---

Kubernetes has become the de facto standard for container orchestration. In this post, we'll walk through the basics of getting started with Kubernetes and how the Vietnamese tech community is adopting it.

## Why Kubernetes?

Kubernetes provides a powerful platform for automating deployment, scaling, and management of containerized applications. Here's why it matters:

- **Scalability** — Automatically scale your applications based on demand
- **Self-healing** — Restart containers that fail, replace and reschedule containers
- **Service discovery** — Built-in DNS and load balancing for your services
- **Rolling updates** — Deploy changes without downtime

## Local Development Setup

The easiest way to get started is with a local Kubernetes cluster:

```bash
# Install minikube
brew install minikube

# Start a local cluster
minikube start

# Verify it's running
kubectl cluster-info
```

## Next Steps

Join the Cloud Native Vietnam community to learn more about Kubernetes and connect with other practitioners in Vietnam.

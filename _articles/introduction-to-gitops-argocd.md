---
layout: post
title: "Introduction to GitOps with Argo CD"
date: 2026-03-15
author: Cloud Native Vietnam
tags: [gitops, ci-cd]
---

GitOps is a modern approach to continuous delivery that uses Git as the single source of truth for infrastructure and application configuration. Argo CD is one of the most popular GitOps tools in the cloud native ecosystem.

## What is GitOps?

GitOps applies DevOps best practices — such as version control, collaboration, compliance, and CI/CD — to infrastructure automation. The core idea is simple:

- **Declarative** — Your entire system is described declaratively
- **Versioned** — The desired state is stored in Git
- **Automated** — Changes are automatically applied to the system
- **Self-healing** — The actual state converges to the desired state

## Why Argo CD?

Argo CD is a declarative, GitOps continuous delivery tool for Kubernetes that is part of the CNCF ecosystem. It continuously monitors your Git repository and ensures your cluster state matches the desired state defined in Git.

Stay tuned for a hands-on tutorial in our next post!

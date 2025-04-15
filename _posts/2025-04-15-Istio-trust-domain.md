---
title: "Understanding Trust Domains in Istio: Isolating Security Boundaries"
date: 2025-04-14
tags: [Istio, Service Mesh, Security, Trust Domain, Multi-Cluster, Kubernetes]
categories: [Kubernetes, Istio]
author: "Nizam Uddin"
layout: post
---

# Introduction
When running Istio in a **multi-cluster** or **multi-network** setup, securing communication between services isn't just about mTLS encryption. It's also about **who you trust** and **how you isolate identities**. That's where `trustDomain` comes in.

In this post, we'll explore what **"Isolation of security boundaries"** means in Istio, and how **different trust domains** enable secure, identity-aware communication across clusters.

---

##  What Is a Security Boundary?

A **security boundary** is a logical or physical separation between systems or workloads that prevents unrestricted access across them. In Kubernetes and Istio, it's the line that defines which workloads can talk to each other and which ones can't—**unless explicitly allowed**.

---

##  Trust Domains: Your Mesh's Identity Boundary

In Istio, every workload gets a **SPIFFE ID** like:

```text
spiffe://<trust-domain>/ns/<namespace>/sa/<service-account>
```

If two clusters use the same `trustDomain`, they inherently **trust each other's workload identities**.

> But when clusters have **different trust domains**, Istio treats their identities as **separate and untrusted**, unless told otherwise.

---

##  Without Isolation: A Single Trust Domain Mesh

- All workloads trust each other by default
- A compromised service in `cluster-1` can spoof an identity in `cluster-2`
- No boundary between tenants or teams

### Example:
```text
spiffe://mesh.local/ns/team-a/sa/app-a
spiffe://mesh.local/ns/team-b/sa/app-b
```

Both identities are in the same `mesh.local` trust space.

---

##  With Isolation: Multi Trust Domains per Cluster

![Istio Trust Domain Isolation](/assets/img/posts/istio-trust-domain.png)

Use different trust domains:

- `cluster-1` uses `trustDomain: team-a.gke.local`
- `cluster-2` uses `trustDomain: team-b.eks.local`

Now, **cross-cluster communication is denied by default**.

```text
spiffe://team-a.gke.local/ns/finance/sa/payroll
spiffe://team-b.eks.local/ns/marketing/sa/analytics
```

To allow selected communication, you can configure `trustDomainAliases`:

```yaml
meshConfig:
  trustDomain: team-a.gke.local
  trustDomainAliases:
    - team-b.eks.local
```

---

##  Benefits of Trust Domain Isolation

| Feature                      | With Isolation       | Without Isolation     |
|-----------------------------|----------------------|------------------------|
| Identity Spoofing           | Prevented            | Possible               |
| Blast Radius of Compromise  | Smaller              | Larger                 |
| Fine-grained Policy Control | Easier               | Harder                 |
| Tenant Separation           | Strong               | Weak                   |

---

##  Real-World Scenario: Hybrid Cloud Identity Federation

Company X has:
- `cluster-1` in GKE for Finance Team → `team-a.gke.local`
- `cluster-2` in EKS for Marketing Team → `team-b.eks.local`

They want mTLS across clusters but still enforce:
- Trust boundaries per team
- Identity validation
- Selective cross-access (e.g., auth service)

Using trust domain aliases, they federate identities **without full trust**.

---

## Istio Install Example with Isolation

### Cluster 1:
```bash
istioctl install --context=cluster1 \
  --set values.global.meshID=mesh1 \
  --set values.global.network=network1 \
  --set meshConfig.trustDomain="team-a.gke.local" \
  --set meshConfig.trustDomainAliases="{team-b.eks.local}"
```

### Cluster 2:
```bash
istioctl install --context=cluster2 \
  --set values.global.meshID=mesh1 \
  --set values.global.network=network2 \
  --set meshConfig.trustDomain="team-b.eks.local" \
  --set meshConfig.trustDomainAliases="{team-a.gke.local}"
```

---

##  Summary

Trust domains in Istio are more than just configuration values—they're **powerful security boundaries**. By assigning **unique trust domains per cluster**, you:

- Enforce secure identity boundaries
- Prevent impersonation across teams
- Enable safe, federated service-to-service communication

If you're working with multi-cluster Istio in production, consider `trustDomain` not optional, but **essential**.


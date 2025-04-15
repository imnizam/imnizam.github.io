---
title: "Manually Federating Trust Between Istio Clusters with Different Root CAs"
date: 2025-04-15
categories: [Istio, Kubernetes]
tags: [istio, multicluster, trustdomain, spiffe, security]
toc: true
author: "Nizam Uddin"
layout: post
---

# Introduction
When deploying Istio across multiple clusters—especially in a multi-network, multi-primary setup—it's often desirable to isolate **security boundaries** using **different root CAs** per cluster. But what if you still need **cross-cluster communication**? Let's walk through **manually federating trust** between clusters with different CAs, step-by-step.

##  Use Case

You have:
- **Cluster A** with root CA `cluster-a-root.pem` and `trustDomain: cluster-a.example`
- **Cluster B** with root CA `cluster-b-root.pem` and `trustDomain: cluster-b.example`
- Both clusters are part of the same **mesh** (shared `meshID`)
- You want **mutual TLS and SPIFFE-based identity verification** between clusters

---

##  Setup Overview

Each cluster runs Istio with its **own root CA**, but you explicitly configure each side to **trust the other's root cert**, enabling secure **cross-cluster mTLS**.

```plaintext
┌─────────────┐           mTLS           ┌─────────────┐
│ Cluster A   │ ───────────────────────► │ Cluster B   │
│ trustDomain │      Federated Trust    │ trustDomain │
│ cluster-a   │ ◄─────────────────────── │ cluster-b   │
└─────────────┘                         └─────────────┘
```

---

## 1- Export Root CAs from Each Cluster

Get each cluster’s root cert from the Istio CA secret:

```bash
# From Cluster A
kubectl -n istio-system get secret cacerts -o jsonpath="{.data['root-cert\.pem']}" | base64 -d > cluster-a-root.pem

# From Cluster B
kubectl -n istio-system get secret cacerts -o jsonpath="{.data['root-cert\.pem']}" | base64 -d > cluster-b-root.pem
```

---

## 2- Create ConfigMap to Share Remote Root CA

Create a ConfigMap in each cluster to hold the other cluster’s root cert.

```bash
# In Cluster A (add B's cert)
kubectl create configmap cluster-b-root-cert -n istio-system \
  --from-file=cluster-b-root.pem

# In Cluster B (add A's cert)
kubectl create configmap cluster-a-root-cert -n istio-system \
  --from-file=cluster-a-root.pem
```

---

## 3- Inject Extra Root Certs into Sidecars

Mount the shared root cert into the sidecar and instruct Istio to use it.

### In Helm/IstioOperator:

```yaml
values:
  global:
    proxy:
      env:
        ISTIO_META_EXTRA_ROOT_CERTS: /etc/istio/extra-certs
      extraVolumeMounts:
        - name: extra-certs
          mountPath: /etc/istio/extra-certs
          readOnly: true
  pilot:
    extraVolumes:
      - name: extra-certs
        configMap:
          name: cluster-b-root-cert
```

> Do the equivalent in Cluster B using `cluster-a-root-cert`.

---

## 4- Enable Cross-Trust via `trustDomainAliases`

Create a `PeerAuthentication` policy to explicitly **allow trust** from the other cluster.

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: allow-cluster-b
  namespace: default
spec:
  mtls:
    mode: STRICT
  selector:
    matchLabels:
      app: my-app
  trustDomainAliases:
    - cluster-b.example
```

And similarly in Cluster B for `cluster-a.example`.

---

##  Optional: Fine-Tune by Gateway or Namespace

You can scope trust down to specific workloads or gateways for **extra control**, keeping the rest of the mesh isolated.

---

##  Result

- Cross-cluster **mTLS and workload identity** verification works.
- Clusters maintain **independent root CAs** (strong isolation).
- Trust is **explicit and auditable**.

---

##  Summary

You now have a secure multi-cluster Istio mesh with **custom trust federation** and **strict identity boundaries**. This is perfect for regulated, multi-tenant, or hybrid environments.
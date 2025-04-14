---
title: How Istio Init Changes iptables in Pod Namespace
date: 2025-04-14
categories: [Kubernetes, Networking]
tags: [istio, iptables, service-mesh, kubernetes]
---

In an Istio-enabled Kubernetes cluster, sidecar injection introduces smart traffic routing magic 🧙‍♂️ using `iptables`. But how exactly does this work under the hood in the **Pod’s network namespace**?

Let’s peel back the layers and look at what the **`istio-init` container** does and how it rewires network traffic using `iptables`. Bonus: there's a **diagram below** for that visual “aha!” moment 📊.

---

## 🧠 What is `istio-init`?

When Istio injects a sidecar (Envoy proxy), it also injects an **`istio-init` container**. This is an **init container** that runs before the app container and sets up `iptables` rules.

These rules are designed to **redirect inbound and outbound traffic** from the app to the sidecar proxy so that Istio can enforce routing, telemetry, security policies, and more.

---

## 🧪 What Does It Change in `iptables`?

The `istio-init` container runs as `NET_ADMIN` and configures `iptables` in the Pod’s **network namespace** to:

- Intercept **inbound traffic** destined for the application
- Intercept **outbound traffic** initiated by the application
- Redirect both through the **Envoy sidecar**

It adds custom chains like:

- `ISTIO_REDIRECT`
- `ISTIO_INBOUND`
- `ISTIO_OUTPUT`

These chains get **hooked into existing chains** (`PREROUTING`, `OUTPUT`, etc.) to transparently reroute traffic.

---

## 🔄 Traffic Flow Explained

Here’s the simplified flow of both inbound and outbound traffic:

### 🔁 Inbound Traffic

1. Packet enters the Pod
2. Hits `iptables` `PREROUTING` chain
3. Matches a rule that jumps to `ISTIO_INBOUND`
4. Traffic gets redirected to Envoy (sidecar proxy)
5. Envoy forwards to the app container

### 🚀 Outbound Traffic

1. App initiates outbound connection
2. Packet hits the `OUTPUT` chain
3. A rule checks whether the destination is external
4. If yes, jumps to `ISTIO_OUTPUT`
5. Redirected to Envoy
6. Envoy forwards it to destination

---

## 📦 Complete Block Diagram

![Istio iptables flow](/assets/img/posts/istio-iptables-flow-diagram.png)

> A visual representation of how Istio manipulates iptables chains inside the Pod namespace.

🧭 **Legend**:
- Blue: Pod components
- Purple: iptables chains
- Arrows: Traffic flow directions

---

## 🧰 Key `iptables` Chains Created by Istio

When Istio injects a sidecar into a Pod, `istio-init` container runs, it sets up several custom chains in the **NAT table** of iptables. And — it **rewires the Pod's traffic** using iptables to enforce mesh policies transparently. Let’s dive into how this is done under the hood! 🛠️

---

| Chain              | Role                                      |
|--------------------|-------------------------------------------|
| `ISTIO_INBOUND`     | Handles all incoming connections          |
| `ISTIO_IN_REDIRECT` | Redirects incoming traffic to Envoy       |
| `ISTIO_OUTPUT`      | Processes all outgoing connections         |
| `ISTIO_REDIRECT`    | Redirects outbound traffic to Envoy       |

---

### ✅ Inbound Traffic Flow (Green Path 1–5)

This is how **external traffic to the Pod** is intercepted and routed via the sidecar proxy:

1️⃣ **Traffic hits `PREROUTING` chain**  
```bash
iptables -t nat -A PREROUTING -p tcp -j ISTIO_INBOUND
```

2️⃣ **Forwarded to `ISTIO_INBOUND`**  
```bash
iptables -t nat -A ISTIO_INBOUND -p tcp --dport 8080 -j ISTIO_IN_REDIRECT
```

3️⃣ **Routed to `ISTIO_IN_REDIRECT`**  
```bash
iptables -t nat -A ISTIO_IN_REDIRECT -p tcp -j REDIRECT --to-port 15006
```

4️⃣ **Redirected to Envoy on port `15006`**

5️⃣ **Envoy → Application**  
Envoy processes the traffic and forwards it to the app on its original port (e.g., `localhost:8080`)

---

### 🚀 Outbound Traffic Flow (Red Dashed Path A–E)

This is how **outgoing traffic from the app** gets rerouted:

🅰️ **Application sends traffic → hits `OUTPUT` chain**

🅱️ **Routed to `ISTIO_OUTPUT`**  
```bash
iptables -t nat -A OUTPUT -p tcp -j ISTIO_OUTPUT
```

🅲 **Sent to `ISTIO_REDIRECT`**  
```bash
iptables -t nat -A ISTIO_OUTPUT -p tcp -j ISTIO_REDIRECT
```

🅳 **Redirected to Envoy**  
```bash
iptables -t nat -A ISTIO_REDIRECT -p tcp -j REDIRECT --to-port 15001
```

🅴 **Envoy forwards to destination**  
Envoy handles retries, policies, and then sends the traffic on its way.

---

### 🚧 Bypass Rules (Blue Dotted Path)

Not all traffic goes through Envoy — some is **excluded intentionally**:

```bash
# Skip localhost traffic to specific ports
iptables -t nat -A ISTIO_OUTPUT -d 127.0.0.1/32 -p tcp --dport 15000 -j RETURN

# Skip traffic from Envoy itself (UID 1337)
iptables -t nat -A ISTIO_OUTPUT -m owner --uid-owner 1337 -j RETURN

# Skip Kubernetes service CIDR (example CIDR)
iptables -t nat -A ISTIO_OUTPUT -d 10.96.0.0/16 -j RETURN

# Skip Pod’s own IP
iptables -t nat -A ISTIO_OUTPUT -d $POD_IP/32 -j RETURN
```

---

### 🧩 Key Implementation Details

- **Transparent Interception**: App code stays untouched — all traffic rerouting is at the network level.
- **Port Exclusion**: You can configure ports to skip Envoy if needed.
- **UID-based Exclusion**: Prevents infinite loops by excluding traffic from Envoy (UID 1337).
- **Init Container Setup**: All iptables rules are configured *before* the app starts.

---

### 🧠 TL;DR

The `istio-init` container sets up `iptables` magic that allows Istio to:

- Intercept all **inbound and outbound** TCP traffic
- Redirect it to the Envoy sidecar
- Apply **policies, telemetry, retries, and mTLS**
- Do it all **without modifying your application**

---

### ⚠️ Important Notes

- `istio-init` **only runs once** during Pod startup
- These rules are **local to the Pod namespace**
- This redirection is **transparent** to the app

---

## 🧪 Verify It Yourself

To inspect these rules inside a running Pod (with privileges), run:

```bash
iptables -t nat -L -n -v
```

Look for chains like `ISTIO_INBOUND` and rules redirecting to port `15001` or `15006` (the Envoy listener ports).

---

## ✅ Conclusion

Thanks to `istio-init`, all traffic flows through Envoy without touching a line of app code. This setup enables powerful Istio features like telemetry, mTLS, retries, and traffic shifting 🚀.

Stay meshy! 🕸️
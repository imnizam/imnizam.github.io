---
title: Understanding iptables Chains — With a Smile!
date: 2025-04-14 12:00:00 +0000
categories: [Linux, Networking]
tags: [iptables, networking, Linux]
author: "Nizam Uddin"
layout: post
---

# Introduction
Ever run `iptables -L` and felt like you just opened the Matrix? All those chains, tables, and rules… what do they *really* mean?

Let’s break it down -

---

##  What’s a Chain?

A **chain** in `iptables` is like a **checkpoint** where packets are checked and rules are applied. Depending on the chain, we might:
- Allow the packet
- Modify it
- Or block it

Chains are grouped into **tables**, each designed for a specific kind of job. Let’s meet these tables and their trusty chains!

---

##  `filter` Table — The Bodyguard 

This is the default table, and it’s all about saying “yay” or “nay” to packets.

| Chain     | What it does |
|-----------|--------------|
| `INPUT`   | Handles packets coming **into** your system |
| `OUTPUT`  | Handles packets **leaving** from your system |
| `FORWARD` | Handles packets just **passing through** your system |

**Use this table** to set up classic firewall rules like:
```bash
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```

---

##  `nat` Table — The Identity Changer

This one handles **Network Address Translation**, changing where packets appear to come from or where they're going.

| Chain        | What it does |
|--------------|--------------|
| `PREROUTING` | Alters destination before routing |
| `POSTROUTING`| Alters source after routing |
| `OUTPUT`     | Alters locally created packets |

Use this when doing things like:
```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

---

##  `mangle` Table — The Packet Stylist 

Need to tweak packet headers or mark packets for special treatment? This is your table.

| Chain        | What it does |
|--------------|--------------|
| `PREROUTING` | Change incoming packets early |
| `INPUT`      | Change packets meant for this host |
| `FORWARD`    | Change packets passing through |
| `OUTPUT`     | Change locally created packets |
| `POSTROUTING`| Change packets just before they leave |

Example:
```bash
iptables -t mangle -A PREROUTING -p tcp --dport 80 -j MARK --set-mark 1
```

---

##  `raw` Table — The Early Bird

The **raw** table acts *before* connection tracking. It's rarely used, but super powerful when you need to bypass conntrack.

| Chain        | What it does |
|--------------|--------------|
| `PREROUTING` | Intercept external traffic early |
| `OUTPUT`     | Intercept local traffic early |

---

##  `security` Table — The SELinux Bouncer

This is where SELinux and other MAC systems apply their policies.

| Chain     | What it does |
|-----------|--------------|
| `INPUT`   | Security rules for incoming packets |
| `OUTPUT`  | Security rules for outgoing packets |
| `FORWARD` | Security rules for passed-through packets |

>  This one’s only active if you’re using SELinux or similar MAC systems.

---

##  Custom Chains — Modular Magic 

You can define your own chains to keep things clean!

```bash
iptables -N MY_CHAIN
iptables -A INPUT -j MY_CHAIN
```

This keeps your main chains neat and your logic reusable. 💡

---

##  How to View Your Chains

Want to peek into what’s going on?

```bash
iptables -t filter -L -n
iptables -t nat -L -n
iptables -t mangle -L -n
iptables -t raw -L -n
iptables -t security -L -n
```

Or dump everything:
```bash
iptables-save
```

---

##  TL;DR — Chain Cheat Sheet

| Table   | Chains                          | What It’s For                      |
|---------|----------------------------------|-------------------------------------|
| filter  | INPUT, OUTPUT, FORWARD           | Allow/block traffic                 |
| nat     | PREROUTING, POSTROUTING, OUTPUT  | NAT, port forwarding, masquerading |
| mangle  | All 5 chains                     | Header tweaks, marking, TTL        |
| raw     | PREROUTING, OUTPUT               | Pre-conntrack, advanced control     |
| security| INPUT, OUTPUT, FORWARD           | SELinux enforcement                 |

---

##  You Did It!

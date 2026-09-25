---
title: "B3 Networks — NetBird Zero Trust VPN"
author: "Levi Nguyen"
last_modified: "Sep 25, 2026"
source: github
tags: [access-management, netbird, vpn, infrastructure]
---

# B3 Networks — NetBird Zero Trust VPN

**Status:** Active Production  
**Last Verified:** 2026-09-25  
**Managed by:** TechOps / DevOps  
**Web Dashboard:** `https://netbird.b3networks.com`

---

## 1. Overview

**NetBird** is B3 Networks' enterprise self-hosted Zero Trust Network Access (ZTNA) solution built on the WireGuard mesh overlay protocol. It replaces legacy static VPNs (`wg-easy`, OpenVPN) and Squid HTTP/SOCKS proxies for corporate access across AWS, GCP, and third-party partner networks.

### Key Features
* **Google Workspace SSO:** Users authenticate directly with `@b3networks.com` Google accounts. No static configuration files (`.ovpn`, `.conf`) or private key distribution required.
* **Default-Deny Zero Trust Model:** Strict group-based access control. Network paths, ports, and subnets are completely blocked unless explicitly allowed by an Access Control Policy.
* **Hybrid Cloud Coverage:** Seamless connectivity across AWS Primary VPC, AWS Ops-Tool VPC, AWS EKS Stable clusters, and GCP Strato & Nexus production environments.
* **Dynamic Domain Routing:** Supports wildcard domain routing (`*.internal.b3networks.com`) with local DNS interception and Route53 private resolution.

---

## 2. Documentation Directory

This repository contains the complete documentation suite for B3 Networks NetBird deployment:

| Document | Target Audience | Description |
|---|---|---|
| 📘 **[NetBird Configuration Reference](NetBird-Config.md)** | DevOps / TechOps / SecOps | **Authoritative technical specification:** Docker Compose stacks, server specifications, STUN/Relay parameters, Routing Peer requirements, Network definitions, and Access Control policy matrices. |
| 🛡️ **[NetBird Administrator Guide](NetBird-Admin-Guide.md)** | DevOps / TechOps Admins | **Operations runbook:** User onboarding/offboarding workflows, group assignments, routing peer setup, setup keys, and automated backup/disaster recovery procedures. |
| 💻 **[NetBird End-User Guide](NetBird-User-Guide.md)** | All B3 Staff & Contractors | **End-user manual:** Client installation (macOS, Windows, Linux, Mobile), Google SSO login, connection verification, and common troubleshooting tips. |

---

## 3. Infrastructure Summary

### 3.1 Control Plane Servers

| Component | Instance ID | Private IP | Public IP | Function |
|---|---|---|---|---|
| **Management + Signal** | `i-0ebee8d9ae0102850` | `172.21.134.27` | `13.250.164.169` | Dashboard UI, Signal coordinator, PostgreSQL 16 store, Traefik SSL termination (`netbird.b3networks.com`). |
| **Relay Server** | `i-09b9ff45ef4ed866f` | `172.21.130.196` | `18.136.115.100` | Fallback WebSocket TCP 443 relay & STUN UDP 3478 (`relay.netbird.b3networks.com`). |

### 3.2 Routing Peers (Gateways)

| Node Name | Platform | Private IP | Served Subnets & Domains |
|---|---|---|---|
| **`netbird-routing-peer-hoiio`** | AWS Singapore (`ap-southeast-1`) | `172.21.151.54` | • `172.21.0.0/16` (Primary Prod VPC)<br>• `172.22.0.0/16` (EKS Stable Nodes & `wss-dev`)<br>• `*.internal.b3networks.com`<br>• Partner Allowed Domains |
| **`netbird-routing-peer-ops`** | AWS Singapore (`ap-southeast-1`) | `10.8.2.253` (EIP: `18.140.112.255`) | • `10.8.0.0/16` (Ops-Tool VPC, Bastions, CI/CD runners) |
| **`apse1-strato-prod-twingates-vm`** | GCP Singapore (`asia-southeast1`) | `10.31.32.5` | • `10.31.0.0/16` (GCP Strato Prod)<br>• `10.32.0.0/16` (GCP Nexus Prod) |

---

## 4. Key Accessible Resources

| Resource | Scope / Address | Access Group |
|---|---|:---:|
| **Kibana Logs** | `https://kibana.internal.b3networks.com` | `Common`, `DevOps`, `Ops` |
| **Grafana Monitoring** | `https://grafana.internal.b3networks.com` | `Common`, `DevOps`, `Ops` |
| **Alertmanager** | `https://alertmanager.internal.b3networks.com` | `Common`, `DevOps`, `Ops` |
| **AgentDuet WebSocket Dev** | `ws://wss-dev.internal.b3networks.com:8080` | `Common`, `DevOps` |
| **AWS Primary VPC** | `172.21.0.0/16` & `172.22.0.0/16` | `DevOps`, `Ops` |
| **AWS Ops-Tool VPC** | `10.8.0.0/16` | `Ops`, `DevOps` |
| **GCP Strato & Nexus Prod** | `10.31.0.0/16` & `10.32.0.0/16` | `nexus-ux-team`, `DevOps` |
| **Partner Integrations** | `*.greateasternlife.com`, `*.kpmg.com.sg`, `*.telcoflow.com` | `B3-Partner` |

---

# Contact Points

- TechOps
  - Levi Nguyen (levinguyen@b3networks.com)
  - Luk Huynh (luk@b3networks.com)
- DevOps
  - Huy Nguyen
  - Hieu Dao
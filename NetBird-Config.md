---
title: "NetBird — Infrastructure, Network & Access Configuration Reference"
author: "Levi Nguyen"
last_modified: "Sep 25, 2026"
source: github
tags: [access-management, infrastructure, vpn]
---

# NetBird — Infrastructure, Network & Access Configuration Reference

**Status:** Active Production  
**Last Verified:** 2026-09-25  
**Managed by:** TechOps / DevOps  
**Dashboard:** `https://netbird.b3networks.com`

---

## 1. Background

- NetBird is B3 Networks' self-hosted Zero Trust VPN built on top of the WireGuard mesh overlay protocol.
- It completely replaces legacy static VPNs (`wg-easy`, OpenVPN) and Squid HTTP/SOCKS proxies for corporate access.
- NetBird operates on a strict **Default-Deny** zero-trust model: no network path, port, or subnet is reachable unless explicitly defined as a Resource and granted by an Access Control Policy.
- This document serves as the authoritative technical reference for all NetBird server components, routing peers, VPC subnet mappings, DNS wildcard routing, and access control policies.

---

## 2. Objective

- Provide an unambiguous architectural blueprint of the NetBird self-hosted cluster (Management, Signal, Relay, and Routing Peers).
- Detail host configurations, Docker Compose specifications, and Linux kernel requirements.
- Document all active corporate Networks, Resources (CIDR subnets vs. Wildcard Domains), and Group Access Policies across AWS and GCP.
- Establish operational troubleshooting guidelines and record key architectural decisions (e.g., elimination of legacy proxy domains).

---

## 3. High-Level Architecture & Traffic Topology

NetBird establishes an encrypted WireGuard mesh overlay network where each connected peer receives an overlay IP within the `100.x.x.x` carrier-grade NAT range. Routing Peers act as internal gateways to forward traffic into private AWS and GCP subnets via Linux kernel NAT masquerading.

```mermaid
flowchart TD
    subgraph Clients["User Devices (macOS / Windows / Linux)"]
        Client1["DevOps Staff (Full Subnets)"]
        Client2["Ops Staff (Infra Subnets)"]
        Client3["General Staff / Common (Web & Internal Domains)"]
        Client4["UX / QA Team (GCP Nexus / Strato)"]
    end

    subgraph ControlPlane["NetBird Control Plane (AWS Singapore - 172.21.0.0/16)"]
        Mgmt["Management & Signal Server\n(netbird.b3networks.com:443)\nInstance: i-0ebee8d9ae0102850\nIP: 172.21.134.27"]
        Relay["Dedicated Relay Server\n(relay.netbird.b3networks.com:443)\nInstance: i-09b9ff45ef4ed866f\nIP: 172.21.130.196 / EIP: 18.136.115.100"]
    end

    subgraph DataPlane["Data Plane — Routing Peers (Gateways)"]
        PeerAWSPrimary["AWS Primary VPC Routing Peer\nip-172-21-151-54 (172.21.151.54)\nInstance: i-05b651be2838d49f7"]
        PeerAWSOps["AWS Ops-Tool VPC Routing Peer\nip-10-8-2-253 (10.8.2.253)\nInstance: i-0910c032f3d21eba5"]
        PeerGCP["GCP Strato & Nexus Routing Peer\napse1-strato-prod-twingates-vm\nIP: 10.31.32.5"]
    end

    subgraph CorporateNetworks["Target Corporate Subnets & Services"]
        AWSProd["AWS Primary Prod VPC (172.21.0.0/16)\n• RDS Databases\n• Kibana, Grafana, Alertmanager\n• *.internal.b3networks.com"]
        AWSEKS["AWS EKS Stable Subnets (172.22.0.0/16)\n• wss-dev.internal.b3networks.com (8080)\n• Sandbox WebSocket Nodes"]
        AWSOps["AWS Ops-Tool VPC (10.8.0.0/16)\n• Bastion Hosts\n• Jenkins CI/CD Runners"]
        GCPStrato["GCP Strato Prod VPC (10.31.0.0/16)\n• Strato Kubernetes Nodes"]
        GCPNexus["GCP Nexus Prod VPC (10.32.0.0/16)\n• Nexus Booking Agent / UX Testing"]
        PartnerDomains["Partner Allowed Domains\n• *.greateasternlife.com\n• *.kpmg.com.sg\n• *.telcoflow.com"]
    end

    Clients -- "1. Google SSO Auth & Key Exchange" --> Mgmt
    Clients -. "Fallback when P2P blocked (Symmetric NAT)" .-> Relay
    Clients == "Direct WireGuard Tunnel (P2P UDP 51820)" ==> PeerAWSPrimary
    Clients == "Direct WireGuard Tunnel (P2P UDP 51820)" ==> PeerAWSOps
    Clients == "Direct WireGuard Tunnel (P2P UDP 51820)" ==> PeerGCP

    PeerAWSPrimary -->|iptables MASQUERADE| AWSProd
    PeerAWSPrimary -->|iptables MASQUERADE| AWSEKS
    PeerAWSPrimary -->|iptables MASQUERADE| PartnerDomains
    PeerAWSOps -->|iptables MASQUERADE| AWSOps
    PeerGCP -->|iptables MASQUERADE| GCPStrato
    PeerGCP -->|iptables MASQUERADE| GCPNexus
```

---

## 4. Control Plane Infrastructure (Management & Relay)

### 4.1 Server Inventory

| Component | Instance ID | Private IP | Public IP | Instance Type | OS / Kernel | DNS Endpoint |
|---|---|---|---|---|---|---|
| **Management + Signal** | `i-0ebee8d9ae0102850` | `172.21.134.27` | `13.250.164.169` | `t4g.medium` (ARM64) | Amazon Linux 2023 | `netbird.b3networks.com` |
| **Relay Server** | `i-09b9ff45ef4ed866f` | `172.21.130.196` | `18.136.115.100` | `t4g.medium` (ARM64) | Amazon Linux 2023 | `relay.netbird.b3networks.com` |

---

### 4.2 Management Server Configuration

* **Working Directory:** `/home/ec2-user/`
* **Configuration File:** `/home/ec2-user/config.yaml`
* **Storage Engine:** SQLite (stored under `/var/lib/netbird/`)

#### Docker Compose Stack (`/home/ec2-user/docker-compose.yml`)

| Container Name | Image Tag | Purpose | Exposed Ports / Protocol |
|---|---|---|---|
| `netbird-server` | `netbirdio/netbird-server:latest` | Management API & Signal coordinator | `80/tcp` (internal proxy), `33073/udp` |
| `netbird-traefik` | `traefik:v3.6` | Reverse proxy, SSL termination (Let's Encrypt) | `80/tcp`, `443/tcp` |
| `netbird-dashboard` | `netbirdio/dashboard:latest` | Web Management Dashboard UI | Internal HTTP via Traefik |
| `netbird-crowdsec` | `crowdsecurity/crowdsec:v1.7.7` | Security log monitoring & brute-force protection | Internal |
| `netbird-proxy` | `netbirdio/reverse-proxy:latest` | Internal reverse proxy handler | Internal |

#### Critical `config.yaml` Parameters

```yaml
server:
  listenAddress: ":80"
  exposedAddress: "https://netbird.b3networks.com:443"
  authSecret: "<SECRET_HASH>"
  dataDir: "/var/lib/netbird"
  store:
    engine: "sqlite"
    encryptionKey: "<ENCRYPTION_KEY>"
  stuns:
    - uri: "stun:stun.l.google.com:19302"
      proto: "udp"
  relays:
    addresses:
      - "rels://relay.netbird.b3networks.com:443"
    credentialsTTL: "12h"
    secret: "<MATCHES_AUTH_SECRET>"
  auth:
    issuer: "https://netbird.b3networks.com/oauth2"
    signKeyRefreshEnabled: true
    dashboardRedirectURIs:
      - "https://netbird.b3networks.com/nb-auth"
      - "https://netbird.b3networks.com/nb-silent-auth"
    cliRedirectURIs:
      - "http://localhost:53000/"
```

> [!IMPORTANT]
> **Key Configuration Directives:**
> 1. **STUN Configuration:** Google STUN (`stun:stun.l.google.com:19302`) is explicitly configured to bypass Bug [#6324](https://github.com/netbirdio/netbird/discussions/6324), where the embedded STUN server silently drops UDP binding requests.
> 2. **External Relay Binding:** Local embedded relay is disabled (`Relay: false`). All fallback relay traffic is routed to the dedicated relay instance via `rels://relay.netbird.b3networks.com:443`.
> 3. **Secret Parity:** `relays.secret` on the management node **must exactly match** `NB_AUTH_SECRET` on the relay server.

---

### 4.3 Relay Server Configuration

* **Working Directory:** `/home/ec2-user/`
* **Docker Service:** `netbirdio/relay:latest`

```yaml
services:
  relay:
    image: netbirdio/relay:latest
    container_name: netbird-relay
    restart: unless-stopped
    ports:
      - '443:443'
      - '80:80'
    environment:
      - NB_LOG_LEVEL=info
      - NB_LISTEN_ADDRESS=:443
      - NB_EXPOSED_ADDRESS=rels://relay.netbird.b3networks.com:443
      - NB_AUTH_SECRET=<MATCHES_MANAGEMENT_SECRET>
      - NB_LETSENCRYPT_DOMAINS=relay.netbird.b3networks.com
      - NB_LETSENCRYPT_EMAIL=devops@b3networks.com
      - NB_LETSENCRYPT_DATA_DIR=/data/letsencrypt
    volumes:
      - relay_data:/data
```

| Port | Protocol | Purpose |
|---|---|---|
| `443` | TCP / WebSocket | Encrypted Relay tunnel when symmetric NAT blocks direct WireGuard P2P UDP. |
| `3478` | UDP | STUN discovery for peer ICE negotiation. |
| `80` | TCP | Let's Encrypt HTTP-01 challenge verification. |

---

## 5. Routing Peers (Network Gateways)

Routing peers act as network bridges between the WireGuard overlay network (`100.x.x.x`) and the target private VPCs/subnets.

### 5.1 Routing Peer Inventory

| Hostname / Node Name | Platform / Region | Instance ID / Private IP | Forwarded Subnets / CIDRs | Masquerade (NAT) |
|---|---|---|---|:---:|
| **`netbird-routing-peer-hoiio`**<br>(`ip-172-21-151-54`) | AWS Singapore<br>(`ap-southeast-1`) | `i-05b651be2838d49f7`<br>`172.21.151.54` | • `172.21.0.0/16` (Primary Prod VPC)<br>• `172.22.0.0/16` (EKS Stable / wss-dev)<br>• `172.23.0.0/16` (Secondary Prod VPC)<br>• Partner Wildcard Domains | **Enabled** |
| **`netbird-routing-peer-ops`**<br>(`ip-10-8-2-253`) | AWS Singapore<br>(`ap-southeast-1`) | `i-0910c032f3d21eba5`<br>`10.8.2.253` (EIP: `18.140.112.255`) | • `10.8.0.0/16` (Ops-Tool VPC) | **Enabled** |
| **`apse1-strato-prod-twingates-vm`** | GCP Singapore<br>(`asia-southeast1`) | GCP VM<br>`10.31.32.5` | • `10.31.0.0/16` (GCP Strato Prod)<br>• `10.32.0.0/16` (GCP Nexus Prod) | **Enabled** |
| **`routing-peer-exp-172.28`** | AWS Singapore<br>(`ap-southeast-1`) | `vpc-06c6ea0fc33bbe767`<br>`172.28.x.x` | • `172.28.0.0/16` (B3-EXP Experimental VPC) | **Enabled** |

---

### 5.2 Host System Prerequisites for Routing Peers

Every Linux instance acting as a NetBird Routing Peer must have kernel IP forwarding enabled and `iptables` installed:

```bash
# 1. Enable IPv4 packet forwarding in kernel
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/99-netbird.conf
sudo sysctl -p /etc/sysctl.d/99-netbird.conf

# 2. Ensure iptables package is installed (NetBird manages the NETBIRD-RT-NAT chain)
# For Amazon Linux / RHEL:
sudo dnf install -y iptables
# For Ubuntu / Debian:
sudo apt-get install -y iptables

# 3. Connect to Management server using Setup Key
sudo netbird up \
  --management-url https://netbird.b3networks.com \
  --setup-key <SETUP_KEY_FROM_DASHBOARD>

# 4. Verify NAT Masquerade rule is populated
sudo iptables -t nat -L NETBIRD-RT-NAT -n -v
```

> [!NOTE]
> **Masquerade (NAT) Function:** NetBird applies `MASQUERADE` on the routing peer's egress interface. Destination target instances (e.g., RDS databases, EKS pods) see incoming requests originating from the Routing Peer's private IP (`172.21.151.54`, `10.8.2.253`, or `10.31.32.5`), eliminating the need to modify route tables or security groups across the entire VPC.

---

## 6. Dashboard Configuration: Networks & Resources

NetBird networks and resources are managed in the Web Dashboard under **Networks**.

### 6.1 Active Corporate Networks Table

| Network Name | Routing Peer | Resources (Domains / CIDRs) | Assigned Groups | Operational Purpose |
|---|---|---|---|---|
| **`b3 internal resources`** | `ip-172-21-151-54` | `*.internal.b3networks.com` | `Common` | Corporate internal web tooling (Kibana, Grafana, JIT, Alertmanager, Jenkins) and internal WebSocket services (`wss-dev`). |
| **`b3 partner resources`** | `ip-172-21-151-54` | `*.greateasternlife.com`<br>`*.kpmg.com.sg`<br>`*.telcoflow.com` | `B3-Partner` | Split-tunnel egress for third-party enterprise partner integrations. *(See Section 6.2 for removed domains)*. |
| **`gcp-strato-prod`** | `apse1-strato-prod-twingates-vm` | `10.31.0.0/16` | `DevOps` | Production Kubernetes worker nodes and database instances in GCP Strato. |
| **`gcp-nexus-prod`** | `apse1-strato-prod-twingates-vm` | `10.32.0.0/16` | `nexus-ux-team`, `DevOps` | Production Nexus environment for automated booking agent QA and testing. |
| **`ops-tool`** | `ip-10-8-2-253` | `10.8.0.0/16` | `Ops`, `DevOps` | Bastion jump hosts, Jenkins CI/CD runners, and infrastructure management tooling. |

---

### 6.2 Key Architectural Decisions & Lessons Learned

#### 1. Wildcard Domain Routing vs. Static IP Subnets
* **Domain Routing (`*.internal.b3networks.com`):** When a user requests an internal domain (e.g. `wss-dev.internal.b3networks.com`), NetBird intercepts the DNS query locally, resolves the target IP (`172.22.11.10`) via the AWS Route53 private hosted zone, and dynamically injects a temporary single-host route into the client's routing table.
* **Benefit:** Eliminates the administrative overhead of manually configuring individual static CIDRs for every internal service or microservice.

#### 2. Critical Removal of `*.b3networks.com` and `*.gstatic.com`
During the migration from legacy Squid proxy to NetBird, two domains were initially carried over but caused production incidents and were permanently **deleted**:

> [!WARNING]
> **Why `*.b3networks.com` must NEVER be added to NetBird:**
> - Matching the company's root wildcard `*.b3networks.com` caused NetBird to intercept **ALL** corporate public endpoints (such as `api.b3networks.com`, `portal.b3networks.com`, and `chat02.b3networks.com`).
> - Instead of resolving to public Cloudflare/ALB VIPs, traffic was forcefully routed through internal AWS routing peers, where split-horizon DNS resolved them to internal private IPs that blocked public traffic.
> - **Rule:** Internal domains must use `*.internal.b3networks.com`. Public domains must go directly over the Internet.

> [!NOTE]
> **Why `*.gstatic.com` was removed:**
> - In the legacy Squid HTTP/SOCKS proxy, browsers failed to load Google Fonts and UI icons unless `*.gstatic.com` was whitelisted.
> - NetBird is a WireGuard layer-3 split-tunnel VPN, not an HTTP proxy. Routing public Google CDN assets through corporate VPN tunnels introduced unnecessary latency and bandwidth waste.

---

## 7. Access Control Policy & Group Matrix

NetBird enforces a **Default-Deny** security posture. Traffic between peer groups is blocked unless permitted by an active policy:

| Policy Name | Source Group | Destination Group / Resource | Protocol & Ports | Purpose |
|---|---|---|:---:|---|
| **`Common-Internal-Access`** | `Common` | `*.internal.b3networks.com` | `ALL` | Grants all B3 Google SSO authenticated staff access to internal web tools and WebSocket dev. |
| **`DevOps-Full-Mesh`** | `DevOps` | `DevOps`, AWS & GCP Subnets | `ALL` | Full administrative access to `172.21.0.0/16`, `172.22.0.0/16`, `10.8.0.0/16`, `10.31.0.0/16`, `10.32.0.0/16`. |
| **`Ops-Infra-Access`** | `Ops` | `172.21.0.0/16`, `10.8.0.0/16` | `ALL` | TechOps access to AWS Primary VPC and Ops-tool VPC infrastructure. |
| **`Nexus-Team-Access`** | `nexus-ux-team` | `10.32.0.0/16` (`gcp-nexus-prod`) | `ALL` | QA and UX testers testing Nexus booking agent services. |
| **`Partner-Routing`** | `B3-Partner` | `b3 partner resources` | `ALL` | Partner integration route access. |
| **`Route Access`** | `Common`, `DevOps`, `Ops`, `nexus-ux-team` | `Routing Peers` | `ALL` | **Mandatory:** Allows client nodes to receive subnet route advertisements from routing peers. |

---

## 8. Operational & Troubleshooting Cheatsheet

### 8.1 Server Operations (Management & Relay)

```bash
# Verify Management Docker stack
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Check Management Server logs
docker logs netbird-server 2>&1 | tail -n 30

# Verify External Relay binding
docker logs netbird-server 2>&1 | grep -i relay
# Expected output: Relay: false, Relay addresses: [rels://relay.netbird.b3networks.com:443]

# Verify Relay TLS certificate
curl -v https://relay.netbird.b3networks.com/
# Expected: HTTP 404 page not found + SSL certificate verify ok
```

---

### 8.2 Client Operations (Verification & Cache Reset)

#### macOS Client (Terminal)

```bash
# 1. View client connection status and peer count
netbird status --detail

# 2. View selected/active networks and resolved IPs
netbird networks list

# 3. Cleanly refresh network route table
netbird networks deselect all
netbird networks select all

# 4. Full daemon restart to flush route cache
netbird down && netbird up

# 5. Test internal DNS resolution and endpoint reachability
curl -Iv http://wss-dev.internal.b3networks.com:8080
```

#### Windows Client (PowerShell / GUI)

```powershell
# 1. GUI Method:
# Click NetBird icon in System Tray (near taskbar clock) -> Toggle Disconnect -> Wait 3s -> Toggle Connect.

# 2. PowerShell CLI:
netbird down; netbird up

# 3. Re-select networks:
netbird networks deselect all
netbird networks select all

# 4. Restart Windows NetBird Service (Administrative PowerShell):
Restart-Service NetBird
```

---

# Contact Points

- TechOps
  - Levi Nguyen (levinguyen@b3networks.com)
  - Luk Huynh (luk@b3networks.com)
- DevOps
  - Hieu Dao
  - Sang Ngo
  - Vinh Dang
  - Huy Nguyen

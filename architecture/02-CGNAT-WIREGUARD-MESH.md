# CGNAT Bypass via Tailscale & Headscale WireGuard Mesh

> **Document Version**: 1.0  
> **Classification**: Public Engineering Showcase  
> **Topic**: Network Architecture, WireGuard Mesh Overlays, Zero-Trust Infrastructure

---

## 1. The Regional Cloud Dilemma: Carrier-Grade NAT (CGNAT)

When regional software agencies and developers attempt to host production workloads on local bare-metal hardware, they invariably encounter severe ISP network constraints:

```
                                  THE CGNAT WALL
  Regional ISP User               ISP CGNAT Gateway            Bare-Metal Server
  (100.64.0.0/10)                 (Shared Public IP)           (Pontianak Node)
        │                                 │                           │
        ├── Inbound Port 80/443/22 ───────┼─── BLOCKED / NO INBOUND ──┤ (Unreachable)
        │                                 │                           │
```

### Key Obstacles:
1. **Inbound Traffic Blocking**: Consumer and commercial broadband lines in regional Indonesia sit behind Carrier-Grade NAT (RFC 6598: `100.64.0.0/10`). Inbound ports (80, 443, 22) are blocked at the ISP core router.
2. **Exorbitant Static IP Costs**: Leasing dedicated public IPv4 subnets from regional telecom providers incurs recurring monthly overhead ($50 - $150/month per IP), destroying the economic feasibility of self-hosted cloud platforms.
3. **Dynamic IP Churn**: Non-static connections rotate IP addresses unpredictably, disrupting active TCP sessions, SSL certificates, and DNS records.

---

## 2. The REGtee Mesh Solution

REGtee Cloud overcomes CGNAT by establishing a **Zero-Trust Encrypted WireGuard Mesh Overlay** coordinated by a lightweight self-hosted control plane:

```
                    ┌─────────────────────────────────────────┐
                    │          JAKARTA GATEWAY VPS            │
                    │   (Tier-4 DC, Public Static IPv4/IPv6)  │
                    │                                         │
                    │  ┌───────────────────────────────────┐  │
                    │  │ Headscale Control Plane (Coord)   │  │
                    │  └─────────────────┬─────────────────┘  │
                    │                    │                    │
                    │  ┌─────────────────┴─────────────────┐  │
                    │  │ Caddy Ingress Reverse Proxy       │  │
                    │  └─────────────────┬─────────────────┘  │
                    └────────────────────┼────────────────────┘
                                         │
              WireGuard Tunnel           │           DERP Relay Fallback
         ┌───────────────────────────────┴───────────────────────────────┐
         │                                                               │
         ▼                                                               ▼
┌─────────────────────────────────┐             ┌─────────────────────────────────┐
│       PONTIANAK NODE 1          │             │       PONTIANAK NODE 2          │
│   (Behind Regional ISP CGNAT)   │             │   (Behind Regional ISP CGNAT)   │
│                                 │             │                                 │
│ - Tailscale Daemon (Client)     │◄───────────►│ - Tailscale Daemon (Client)     │
│ - Proxmox Hypervisor (LXC/KVM)  │ Direct Peer │ - Dagger CI/CD Build Engine     │
│ - Virtual Enclaves (100.64.0.x) │ UDP Tunnel  │ - Private Docker Registry       │
└─────────────────────────────────┘             └─────────────────────────────────┘
```

### Core Architecture Components:

1. **Self-Hosted Headscale Control Plane (Jakarta)**:
   - Deployed on an unthrottled, low-latency cloud gateway in Jakarta.
   - Replaces proprietary Tailscale SaaS servers with an independent, open-source control plane.
   - Issues cryptographic node keys, distributes WireGuard peer endpoints, and manages Access Control Lists (ACLs).

2. **Tailscale WireGuard Data Plane (Bare-Metal Nodes)**:
   - Runs as a lightweight background daemon (`tailscaled`) on hypervisors and build workers in Pontianak.
   - Establishes point-to-point WireGuard cryptographic tunnels directly to peers or to the Jakarta gateway.

3. **NAT Traversal & UDP Hole-Punching**:
   - Tailscale utilizes STUN (Session Traversal Utilities for NAT) and interactive UDP hole-punching to establish direct, peer-to-peer tunnels between nodes whenever possible.
   - If symmetric NAT prevents direct peer connection, traffic seamlessly flows through an encrypted **DERP (Designated Encrypted Relay for Packets)** node hosted in Jakarta with minimal latency degradation (<35ms RTT).

---

## 3. Dynamic End-to-End Routing Flow

When an external user accesses an application hosted on a private Pontianak LXC container:

```
Step 1: External Client requests `https://my-app.example.com`
        │
        ▼
Step 2: DNS resolves to Jakarta Gateway Public IP (e.g., 198.51.100.10)
        │
        ▼
Step 3: Caddy Ingress receives TLS handshake, terminates SSL via automated Let's Encrypt
        │
        ▼
Step 4: Caddy inspects dynamic route table (in-memory JSON configured via AMQP)
        Matches `my-app.example.com` -> Target: `100.64.0.10:8080` (LXC Container Mesh IP)
        │
        ▼
Step 5: Packet enters the encrypted WireGuard kernel interface on the Jakarta Gateway
        │
        ▼
Step 6: Traverses the encrypted WireGuard mesh tunnel across the Java Sea to Pontianak
        │
        ▼
Step 7: Reaches target Debian LXC container on Node `cluster-node-01` (100.64.0.10:8080)
        Zero public ports exposed on the physical server!
```

---

## 4. Zero-Trust Access Control Lists (ACLs)

Network security is enforced cryptographically at the overlay layer using Headscale ACL policies:

```json
{
  "acls": [
    {
      "action": "accept",
      "src": ["tag:ingress-gateway"],
      "dst": ["tag:tenant-workload:80", "tag:tenant-workload:443", "tag:tenant-workload:8080"]
    },
    {
      "action": "accept",
      "src": ["tag:ssh-bastion"],
      "dst": ["tag:tenant-workload:22"]
    },
    {
      "action": "accept",
      "src": ["tag:app-builder"],
      "dst": ["tag:registry:5000"]
    }
  ],
  "tagOwners": {
    "tag:ingress-gateway": ["devops@cloud.example.internal"],
    "tag:ssh-bastion": ["devops@cloud.example.internal"],
    "tag:tenant-workload": ["devops@cloud.example.internal"],
    "tag:app-builder": ["devops@cloud.example.internal"]
  }
}
```

### Security Benefits:
- **Default-Deny Isolation**: Tenants cannot sniff or access neighboring tenants' containers over the mesh.
- **Micro-Segmentation**: Containers only listen to the authorized Ingress Gateway for HTTP traffic and the SSH Bastion for remote terminal sessions.
- **Immune to Port Scanners**: The physical bare-metal nodes in Pontianak expose 0 open public ports to the Internet, effectively neutralizing DDoS and botnet port-scanning attacks.

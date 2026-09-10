# 100% Data Sovereignty & UU PDP Compliance Blueprint

> **Document Version**: 1.0  
> **Classification**: Public Engineering Showcase  
> **Topic**: Regulatory Compliance, Data Sovereignty, UU No. 27 Tahun 2022 (UU PDP)

---

## 1. Regulatory Context: Indonesia's UU PDP Mandate

Enacted under **Undang-Undang Republik Indonesia Nomor 27 Tahun 2022 tentang Pelindungan Data Pribadi (UU PDP)**, Indonesian institutions, software enterprises, and financial technology services are legally bound to protect customer personal data. Key statutory requirements include:

1. **Strict Data Controller Accountability**: Organizations must guarantee data processing occurs within clear legal boundaries and verifiable technical perimeters.
2. **Restrictions on Cross-Border Transfers (Pasal 56)**: Personal data transfers abroad require the recipient country to have equal or higher personal data protection standards, binding corporate agreements, or explicit individual consent.
3. **Severe Penalties for Non-Compliance**: Fines reaching up to 2% of annual turnover, administrative sanctions, and criminal liability for negligent data breaches.

---

## 2. The Hyperscaler Sovereignty Trap

When Indonesian software agencies deploy applications on global cloud providers (AWS, GCP, Azure, DigitalOcean):
- **Uncontrolled Data Egress**: Database backups, system metrics, logs, and telemetry are frequently routed to overseas regions (e.g. Singapore, Tokyo, or US regions) without explicit administrator awareness.
- **Foreign Jurisdiction Subjection**: Data stored on foreign hyperscalers is subject to foreign legal instruments (such as the US CLOUD Act), which can compel cloud vendors to disclose extraterritorial data without Indonesian judicial oversight.
- **Currency & Cost Volatility**: Operating expenses fluctuate continuously based on USD/IDR exchange rate movements and arbitrary egress bandwidth fees.

---

## 3. How REGtee Enforces 100% Sovereignty

REGtee Cloud is architected around a strict principle: **Zero Tenant Data Leaves the Local Bare-Metal Perimeter**.

```
                         REGULATORY DATA BOUNDARY
  ┌────────────────────────────────────────────────────────────────────────┐
  │                           PONTIANAK CLUSTER                            │
  │                  (100% Local Sovereign Data Perimeter)                 │
  │                                                                        │
  │  ┌────────────────────────┐              ┌──────────────────────────┐  │
  │  │ Local ZFS Storage Pool │              │   PostgreSQL Master DB   │  │
  │  │ (Encrypted Tenant Data)│              │  (Local Tenant Records)  │  │
  │  └───────────▲────────────┘              └────────────▲─────────────┘  │
  │              │                                        │                │
  │  ┌───────────┴────────────────────────────────────────┴─────────────┐  │
  │  │           Proxmox VE Virtual Hypervisor Enclaves                 │  │
  │  │          (LXC Containers & KVM Virtual Machines)                 │  │
  │  └───────────────────────────────────▲──────────────────────────────┘  │
  │                                      │                                 │
  │                        Private Docker Registry Engine                  │
  │                         (Locally Built OCI Images)                     │
  └──────────────────────────────────────┼─────────────────────────────────┘
                                         │
                        Encrypted WireGuard Mesh Tunnel
                           (Transport Layer Transit)
                                         │
  ┌──────────────────────────────────────▼─────────────────────────────────┐
  │                         JAKARTA GATEWAY TIER                           │
  │                   (Stateless Ephemeral Relay Only)                     │
  │                                                                        │
  │  - Zero Disk Storage for Tenant Payloads                               │
  │  - Zero In-Memory Logging of Personal Data (PII Redaction)             │
  │  - Ephemeral TLS Termination & Mesh Packet Forwarding                  │
  └────────────────────────────────────────────────────────────────────────┘
```

### Architectural Guardrails:

1. **Stateless Gateway Policy**:
   The Jakarta Gateway VPS acts exclusively as an ephemeral packet router (Caddy reverse proxy and Headscale coordinator). It maintains **zero persistent storage** for tenant databases, user files, or application code.
2. **Local ZFS-on-Linux Storage**:
   Tenant volumes are provisioned on local enterprise NVMe ZFS pools directly on the Pontianak hypervisor nodes. Data at rest is encrypted using native ZFS AES-256-GCM encryption keys.
3. **Private In-Cluster Container Registry**:
   Application container images built via the Dagger CI/CD pipeline are pushed directly to a local, private container registry hosted inside the Pontianak cluster (`localhost:5000` on the private mesh), avoiding third-party image hosts like Docker Hub or GitHub Packages.
4. **Automated Audit Logging**:
   All administrative actions (SSH sessions, API calls, VM restarts) generate cryptographic tamper-evident audit trails stored in an append-only ledger, enabling full audit compliance during regulatory inspections.

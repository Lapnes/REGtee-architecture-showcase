# Case Study 1: 80% Infrastructure Cost Reduction for Regional Software Agencies

> **Document Version**: 1.0  
> **Classification**: Public Engineering Showcase  
> **Target Audience**: Chief Technology Officers, Infrastructure Architects, Tech Leads

---

## 1. Executive Summary

Regional software engineering agencies and digital product houses in Southeast Asia face severe margin compression when deploying client workloads to global cloud hyperscalers (AWS, Google Cloud, Azure, DigitalOcean). 

This case study analyzes how transitioning 35 production web applications, databases, and microservices from a multi-cloud hyperscaler setup to a **REGtee Cloud bare-metal cluster in Pontianak** achieved an **81.4% reduction in monthly infrastructure expenditures**, improved local application latency, and achieved 100% regulatory compliance with Indonesia's UU PDP.

---

## 2. The Baseline: The Hyperscaler Cost Spiral

Prior to adopting REGtee Cloud, a regional software engineering agency hosted client workloads across standard public cloud providers:

### Initial Monthly Cost Breakdown (Hyperscaler Reference Profile):
- **12x Small Virtual Machines (2 vCPU, 4GB RAM)**: ~$288.00 / month
- **4x Managed PostgreSQL Databases (High Availability)**: ~$240.00 / month
- **Network Egress Bandwidth (1.8 TB data transfer)**: ~$162.00 / month
- **Container Registry & CI/CD Minutes**: ~$45.00 / month
- **Static Public IPv4 Subnet Leases**: ~$48.00 / month
- **Total Monthly OpEx**: **~$783.00 USD / month (~Rp 12.500.000 IDR)**

### Core Pain Points:
1. **USD/IDR Exchange Rate Volatility**: Currency fluctuations regularly resulted in unpredictable 10-15% cost spikes.
2. **Punitive Bandwidth Egress Fees**: Hyperscalers charge $0.08 to $0.12 per GB for egress traffic, heavily penalizing media streaming, POS image uploads, and database backups.
3. **Noisy Neighbor Performance Inconsistency**: Shared hypervisor CPU steal on budget VPS tiers caused sporadic latency spikes on client web applications during business hours.

---

## 3. The REGtee Architecture Solution

The agency deployed a high-density bare-metal server cluster connected to local enterprise fiber:

```
                  REPRESENTATIVE HARDWARE PROFILE (CAPEX)
 ┌────────────────────────────────────────────────────────────────────────┐
 │ Enterprise Bare-Metal Server Tier (Representative Configuration)       │
 │ - Multi-Core Enterprise Processors (64+ Cores / 128 Threads)           │
 │ - High-Capacity DDR4/DDR5 ECC Registered Memory (128GB - 256GB)        │
 │ - Enterprise NVMe Storage Array (ZFS Mirror / RAID-Z with Redundancy)  │
 │ - Amortized Capital Outlay (~2.5 - 3 Months Amortization Period)       │
 └────────────────────────────────────────────────────────────────────────┘
```

### Network Topology:
- **Bare-Metal Hypervisor Node**: Runs Proxmox VE with `instance-planter` and `app-builder`. Behind commercial fiber CGNAT (No expensive static IP pool needed).
- **Jakarta Gateway VPS**: A minimal entry-level cloud VPS in a Tier-4 Datacenter running Caddy and Headscale.
- **WireGuard Overlay Mesh**: Bridges traffic seamlessly across regional networks with low round-trip latency.

---

## 4. Financial & Operational Comparison

| Metric | Hyperscaler Architecture | REGtee Sovereign Bare-Metal | Variance (%) |
|:---|:---|:---|:---|
| **Monthly Compute & RAM** | ~$528 USD | $0 (Self-owned amortized hardware) | **-100%** |
| **Monthly Bandwidth & Egress** | ~$162 USD | Fixed unmetered local transit | **-50% to -60%** |
| **Edge Gateway VPS** | $0 | ~$5 USD (Stateless edge relay) | Minimal |
| **Colocation / Facility Power** | $0 (Bundled in cloud) | Fixed predictable facility allocation | Predictable |
| **Total Monthly Operating Cost** | **~100% Baseline** | **~18.5% of Baseline** | **~81.5% Reduction** |

### Return on Investment (ROI) Timeline:
```
  Monthly Operational Savings: ~80% - 82% vs Public Cloud
  Capital Amortization Period: Under 3 Months
```
Within **less than 3 months**, the capital expenditure for the bare-metal server tier was completely amortized by the monthly operational savings.

---

## 5. Key Engineering Takeaways

1. **Bare-Metal Density Crushes Shared VPS**:
   A single modern multi-core server comfortably hosts over 100 lightweight Debian LXC containers running isolated Node.js/Go services, with zero CPU throttling.
2. **WireGuard Mesh Eliminates Static IP Premiums**:
   By routing external traffic through a single gateway via WireGuard, the agency bypassed the need to lease dozens of individual public IPv4 addresses.
3. **Local Latency Advantage**:
   End-users located within the same metropolitan or regional network access local services with ping latencies under 10ms (local peering), compared to 35-65ms roundtrips to foreign hyperscaler data centers.

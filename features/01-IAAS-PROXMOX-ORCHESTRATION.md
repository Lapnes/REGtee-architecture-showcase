# IaaS: Bare-Metal Virtualization via Proxmox VE & Terraform IaC

> **Document Version**: 1.0  
> **Classification**: Public Engineering Showcase  
> **Service**: `services/instance-planter`  
> **Topic**: Infrastructure-as-a-Service, LXC Containers, KVM Virtual Machines, OpenTofu / Terraform

---

## 1. Overview & Dual Virtualization Engine

REGtee Cloud's IaaS subsystem, orchestrated by the `instance-planter` Go microservice, provides automated bare-metal provisioning on **Proxmox VE** hypervisor clusters. It offers developers two compute virtualization modes:

```
                      PROXMOX VE HYPERVISOR ARCHITECTURE
 ┌────────────────────────────────────────────────────────────────────────┐
 │                        Bare-Metal Physical Node                        │
 │           (Dual AMD EPYC / Intel Xeon, Enterprise NVMe, Linux Kernel)   │
 ├──────────────────────────────────┬─────────────────────────────────────┤
 │    Mode A: Lightweight LXC       │          Mode B: Full KVM           │
 │  (Shared Host Linux Kernel)      │    (Hardware-Assisted Hypervisor)   │
 │                                  │                                     │
 │  ┌────────────────────────────┐  │  ┌───────────────────────────────┐  │
 │  │ App Container (Debian/Ubuntu) │  │ Guest OS (Custom Linux / BSD) │  │
 │  ├────────────────────────────┤  │  ├───────────────────────────────┤  │
 │  │ Cgroups v2 & Linux Namespaces│  │ Virtual CPU, RAM, Disk Emulat │  │
 │  ├────────────────────────────┤  │  ├───────────────────────────────┤  │
 │  │ RAM Overhead: < 25MB       │  │ Dedicated Kernel & Memory Buff  │  │
 │  │ Boot Time: < 1.5 Seconds   │  │ Boot Time: 15 - 30 Seconds       │  │
 │  └────────────────────────────┘  │  └───────────────────────────────┘  │
 └──────────────────────────────────┴─────────────────────────────────────┘
```

### Virtualization Mode Comparison:

| Feature | Lightweight Container (LXC) | Virtual Machine (KVM) |
|:---|:---|:---|
| **Ideal For** | Web apps, microservices, databases, PaaS runtimes | Custom kernel modules, Docker-in-Docker, nested hypervisors |
| **Boot Latency** | 1.2 to 2.5 seconds | 15 to 35 seconds |
| **Base RAM Usage** | ~18MB to 30MB | ~256MB to 512MB |
| **Storage Layer** | Native ZFS sub-volumes (Instant snapshots) | Raw/QCOW2 disk images on ZFS |
| **Network Device** | `veth` interface bridged to internal mesh | `virtio-net` hardware emulated NIC |

---

## 2. Declarative Infrastructure-as-Code (IaC) Pipeline

Unlike traditional control panels that store server states only in relational databases, `instance-planter` implements an **Automated IaC Pipeline**:

```
 [AMQP Event: instance.provision]
               │
               ▼
 ┌───────────────────────────┐
 │ `instance-planter` Engine │ ─── 1. Allocates unique cluster VMID (e.g., 101)
 └─────────────┬─────────────┘ ─── 2. Generates declarative `main.tf` HCL configuration
               │               ─── 3. Injects user Ed25519 public SSH keys into Cloud-Init
               ▼
 ┌───────────────────────────┐
 │ OpenTofu / Terraform CLI  │ ─── 4. Executes `tofu apply -auto-approve` against Proxmox API
 └─────────────┬─────────────┘ ─── 5. Commits verified state file (`terraform.tfstate`)
               │
               ▼
 ┌───────────────────────────┐
 │ Proxmox VE REST API v2    │ ─── 6. Creates and starts container with allocated ZFS storage
 └───────────────────────────┘
```

### Generated Terraform HCL Specification (Sample):

```hcl
terraform {
  required_providers {
    proxmox = {
      source  = "telmate/proxmox"
      version = "2.9.14"
    }
  }
}

resource "proxmox_lxc" "tenant_workload" {
  target_node  = "cluster-node-01"
  vmid         = 101
  hostname     = "demo-app-container"
  ostemplate   = "local:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst"
  cores        = 2
  memory       = 1024
  swap         = 512
  start        = true
  unprivileged = true

  rootfs {
    storage = "local-zfs"
    size    = "10G"
  }

  network {
    name     = "eth0"
    bridge   = "vmbr0"
    ip       = "100.64.0.10/24"
    gw       = "100.64.0.1"
    firewall = true
  }

  ssh_public_keys = <<-EOT
    ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIExampleMasterBastionKey
    ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIExampleUserPublicKey
  EOT
}
```

---

## 3. High-Performance Hypervisor API Direct Fallback

In addition to declarative Terraform execution, `instance-planter` includes an ultra-fast, native Go client for the **Proxmox VE REST API v2** (`/api2/json`):
- **Bypasses Process Execution Overhead**: Provisions lightweight containers in sub-second API roundtrips.
- **Atomic Cloud-Init Injection**: Injects SSH keys, network configurations, and initial users via native `/nodes/{node}/lxc/{vmid}/config` endpoints.
- **Resource Sentinel**: Continuous background goroutine polls CPU, RAM, and Disk I/O metrics via Proxmox RRD (Round-Robin Database) APIs, emitting telemetry over AMQP.

# System Architecture & Monorepo Topology

> **Document Version**: 1.0  
> **Classification**: Public Engineering Showcase  
> **Status**: Verified in Live Bare-Metal & Hypervisor Validation

---

## 1. High-Level Architecture Topology

REGtee Cloud distributes its workload across two distinct operational boundaries connected by an encrypted WireGuard mesh network:

```
                                  PUBLIC INTERNET
                                         │
                 ┌───────────────────────┴───────────────────────┐
                 │                                               │
         [HTTPS / Ingress]                               [SSH / Bastion]
                 │                                               │
                 ▼                                               ▼
   ┌───────────────────────────┐                   ┌───────────────────────────┐
   │       Caddy Ingress       │                   │    SSH Gateway Bastion    │
   │  Dynamic TLS Reverse Proxy│                   │ Native TCP SSH (Port 2222)│
   └─────────────┬─────────────┘                   └─────────────┬─────────────┘
                 │                                               │
                 │              JAKARTA GATEWAY TIER             │
                 └───────────────────────┬───────────────────────┘
                                         │
                    ┌────────────────────┴────────────────────┐
                    │  Headscale Control Plane (Mesh Coord)   │
                    └────────────────────┬────────────────────┘
                                         │
         ======================= WIREGUARD MESH OVERLAY =======================
                                (100.64.0.0/10 CGNAT Safe)
                                         │
                 ┌───────────────────────┴───────────────────────┐
                 │                                               │
                 ▼                                               ▼
   ┌───────────────────────────┐                   ┌───────────────────────────┐
   │       Core REST API       │◄─────────────────►│    RabbitMQ AMQP Broker   │
   │   Control Plane Engine    │                   │   Topic Exchanges & DLQ   │
   └─────────────┬─────────────┘                   └─────────────┬─────────────┘
                 │                                               │
                 ├───────────────────────────────┬───────────────┴─────────────┐
                 ▼                               ▼                             ▼
   ┌───────────────────────────┐   ┌───────────────────────────┐ ┌───────────────────────────┐
   │     Instance Planter      │   │        App Builder        │ │    Saga Orchestrator      │
   │  Proxmox LXC/VM Provision │   │  Dagger SDK + Nixpacks    │ │ Distributed Transactions  │
   └─────────────┬─────────────┘   └─────────────┬─────────────┘ └───────────────────────────┘
                 │                               │
                 ▼                               ▼
   ┌───────────────────────────┐   ┌───────────────────────────┐
   │  Proxmox VE Hypervisors   │   │  Private Docker Registry  │
   │   Bare-Metal Virtual Encl │   │  Local Container Storage  │
   └───────────────────────────┘   └───────────────────────────┘
                                PONTIANAK CLUSTER
```

---

## 2. The 7 Isolated Microservices

The backend is composed of seven fully decoupled Go microservices managed via Go Workspaces (`go.work`):

| Microservice | Primary Role | Core Dependencies | Primary Interface |
|:---|:---|:---|:---|
| **`core-api`** | Central Control Plane, RBAC, User Invoicing, Billing, Public REST Endpoints | Gin, GORM, PostgreSQL 16, Valkey | HTTP REST (`:8068`), AMQP Publisher |
| **`instance-planter`** | Hypervisor Worker for LXC and KVM lifecycle, automated OpenTofu/Terraform IaC generation | Proxmox VE API, OpenTofu, Go AMQP | AMQP Consumer (`instance.provision`, `instance.destroy`) |
| **`app-builder`** | PaaS Build Grid, containerized Nixpacks builder engine, vulnerability scanner | Dagger SDK v0.18.9, Docker Engine, Trivy | AMQP Consumer (`build.trigger`, `build.cancel`) |
| **`ssh-gateway`** | Dual-mode Bastion: Native TCP SSH Bastion and In-Browser Web Terminal | Go `crypto/ssh`, Gorilla WebSocket, Xterm.js | TCP (`:2222`), WSS (`:8022`) |
| **`ingress-controller`**| Dynamic routing manager synchronizing edge reverse proxy routes | Caddy Admin API (`:2019`), AMQP | AMQP Consumer (`ingress.sync`, `ingress.delete`) |
| **`network-controller`**| Mesh overlay controller managing node keys and Zero-Trust ACL policies | Headscale REST API, AMQP | AMQP Consumer (`network.enroll`, `network.acl`) |
| **`orchestrator`** | Distributed Saga coordinator enforcing multi-step provisioning workflows | RabbitMQ, Valkey State Machine | AMQP Orchestrator (`saga.*`) |

---

## 3. The 4-Layer Clean Architecture Standard

To guarantee long-term maintainability, zero test flakiness, and complete independence from external frameworks, every service strictly adheres to **Robert C. Martin's 4 Clean Architecture Layers**:

```
 ┌─────────────────────────────────────────────────────────────┐
 │ Layer 4: Frameworks & Drivers                               │
 │ (Gin Engine, GORM Driver, Proxmox SDK, Dagger Client, Bun)  │
 │  ┌───────────────────────────────────────────────────────┐  │
 │  │ Layer 3: Interface Adapters                           │  │
 │  │ (HTTP Handlers, AMQP Consumers/Producers, Repos)      │  │
 │  │  ┌─────────────────────────────────────────────────┐  │  │
 │  │  │ Layer 2: Application Use Cases                  │  │  │
 │  │  │ (Interactors, Business Logic, Port Interfaces)  │  │  │
 │  │  │  ┌───────────────────────────────────────────┐  │  │  │
 │  │  │  │ Layer 1: Domain & Entities                │  │  │  │
 │  │  │  │ (Pure Structs, Constants, Domain Errors)  │  │  │  │
 │  │  │  └───────────────────────────────────────────┘  │  │  │
 │  │  └─────────────────────────────────────────────────┘  │  │
 │  └───────────────────────────────────────────────────────┘  │
 └─────────────────────────────────────────────────────────────┘
               Dependencies Point Inward (Layer N -> Layer N-1)
```

### Layer Rules:
1. **Layer 1 (Domain & Entities)**: Contains pure business structs and domain error constants. Zero third-party imports.
2. **Layer 2 (Application Use Cases)**: Interactors executing business workflows. Defines repository and external gateway interfaces (Dependency Inversion Principle).
3. **Layer 3 (Interface Adapters)**: Adapts data to/from external forms. Contains Gin HTTP handlers, RabbitMQ event consumers, SQL database repositories, and response presenters.
4. **Layer 4 (Frameworks & Drivers)**: Specific infrastructure integrations: GORM connection setups, physical Proxmox HTTP clients, Dagger execution daemons, and OS signals.

---

## 4. Zero-Coupling Monorepo Boundaries

The repository enforces strict architectural boundaries:
- **Zero Cross-Service Imports**: Code in `services/core-api` can **NEVER** import from `services/instance-planter`.
- **Shared Code Purity**: Code in `packages/go-shared` contains **zero business logic**—only generic logging, error helpers, and AMQP connection wrappers.
- **Contract-Driven Communication**: Services communicate exclusively through asynchronously published event schemas defined in `contracts/amqp/` or REST APIs defined in `contracts/api/swagger.yaml`.
- **Database Isolation**: Relational database tables are owned by `core-api`. Worker services do not perform arbitrary schema migrations on central tables.

---

## 5. Frontend Architecture (Astro v5 + React 19 Islands)

The user-facing control plane is built using **Astro v5** and **React 19 Islands** running on the native **Bun 1.4** runtime:
- **Fast Static Delivery**: Marketing and documentation pages are rendered ahead-of-time with near-zero JavaScript payload.
- **Interactive Islands**: Dynamic dashboards (resource monitoring charts, live container stats, in-browser terminals) hydrate as isolated React 19 components only where needed.
- **Tailwind CSS v4**: Ultra-fast utility compilation with a custom sovereign cloud dark-mode design system.

# Case Study 2: Enforcing Clean Architecture and Zero Coupling in Go Monorepos

> **Document Version**: 1.0  
> **Classification**: Public Engineering Showcase  
> **Topic**: Software Architecture, Clean Architecture, Dependency Inversion, Go Workspaces

---

## 1. The Monorepo Antipattern: The "Big Ball of Mud"

In distributed systems built as monorepos, development teams often start with good intentions but quickly fall into the **Microservices Coupling Trap**:

```
  THE MONOREPO COUPLING TRAP (UNSTRUCTURED MONOREPO):
  ┌───────────────────────┐          ┌───────────────────────┐
  │  services/core-api    │─────────►│ services/app-builder  │ (Direct Import Violation!)
  └───────────┬───────────┘          └───────────┬───────────┘
              │                                  │
              │  Imports internal GORM struct    │  Directly executes
              │                                  │  another service's DB query
              ▼                                  ▼
  ┌──────────────────────────────────────────────────────────┐
  │              Shared Global Database & Models             │
  │  (Result: Circular dependencies, un-testable codebase)  │
  └──────────────────────────────────────────────────────────┘
```

When services directly import each other's internal modules or share database entity structs:
- A change to a database column in one service breaks compilation across unrelated services.
- True independent deployment becomes impossible.
- Mocking infrastructure for unit testing becomes extraordinarily painful.

---

## 2. The REGtee Standard: 4-Layer Clean Architecture

REGtee Cloud enforces **Robert C. Martin's 4-Layer Clean Architecture** across all 7 Go microservices:

```
  services/instance-planter/
  ├── internal/
  │   ├── domain/               ◄── Layer 1: Domain Entities & Errors (Pure Go)
  │   │   ├── instance.go
  │   │   └── errors.go
  │   ├── usecase/              ◄── Layer 2: Business Logic & Port Interfaces
  │   │   ├── interactor.go
  │   │   └── ports.go
  │   ├── delivery/             ◄── Layer 3: Handlers & Consumers
  │   │   ├── amqp/
  │   │   └── http/
  │   └── repository/           ◄── Layer 4: Adapters (Proxmox SDK, Terraform, DB)
  │       ├── proxmox/
  │       └── terraform/
  └── cmd/
      └── server/main.go        ◄── Composition Root & Dependency Injection
```

### Strict Architectural Principles:

1. **The Inward Dependency Rule**: Source code dependencies point strictly inward toward the domain entities. Layer 1 never imports from Layer 2; Layer 2 never imports from Layer 3 or 4.
2. **Ports and Adapters (Dependency Inversion Principle)**:
   Layer 2 use cases declare **interfaces** for external operations. Concrete infrastructure drivers in Layer 3/4 implement these interfaces.
3. **Zero Cross-Service Imports**:
   A service inside `services/` is strictly forbidden from importing code from another service under `services/`. Communication occurs exclusively via AMQP messages or HTTP APIs.

---

## 3. Concrete Implementation: Dependency Inversion in Go

### Step 1: Port Interface Declared in Application Layer (Layer 2)
Notice that the business interactor has zero knowledge of Proxmox, Terraform, or HTTP protocols:

```go
// File: services/instance-planter/internal/usecase/ports.go
package usecase

import (
    "context"
    "regtee/instance-planter/internal/domain"
)

// HypervisorPort defines the outbound interface for virtual compute operations.
// Layer 2 depends ONLY on this interface, not on Proxmox or Terraform.
type HypervisorPort interface {
    CreateContainer(ctx context.Context, req domain.CreateInstanceRequest) (*domain.InstanceResult, error)
    DestroyContainer(ctx context.Context, vmid int) error
    GetTelemetry(ctx context.Context, vmid int) (*domain.TelemetryMetrics, error)
}

// EventPublisherPort defines the outbound event publishing boundary.
type EventPublisherPort interface {
    PublishStatusUpdate(ctx context.Context, event domain.StatusUpdatedEvent) error
}
```

### Step 2: Use Case Interactor Executes Domain Logic (Layer 2)
```go
// File: services/instance-planter/internal/usecase/interactor.go
package usecase

import (
    "context"
    "fmt"
    "regtee/instance-planter/internal/domain"
)

type InstanceInteractor struct {
    hypervisor HypervisorPort
    publisher  EventPublisherPort
}

func NewInstanceInteractor(h HypervisorPort, p EventPublisherPort) *InstanceInteractor {
    return &InstanceInteractor{hypervisor: h, publisher: p}
}

func (uc *InstanceInteractor) ProvisionInstance(ctx context.Context, req domain.CreateInstanceRequest) error {
    // 1. Enforce business domain validation
    if err := req.Validate(); err != nil {
        return fmt.Errorf("%w: %s", domain.ErrValidationFailed, err.Error())
    }

    // 2. Delegate creation to the abstract port
    result, err := uc.hypervisor.CreateContainer(ctx, req)
    if err != nil {
        return fmt.Errorf("hypervisor creation failed: %w", err)
    }

    // 3. Emit domain event via publisher port
    return uc.publisher.PublishStatusUpdate(ctx, domain.StatusUpdatedEvent{
        InstanceID: req.InstanceID,
        Status:     domain.StatusRunning,
        IPAddress:  result.AllocatedIP,
    })
}
```

### Step 3: Infrastructure Adapter Implements Port (Layer 4)
The concrete Proxmox client implements the `HypervisorPort` without leaking hypervisor details into the business core:

```go
// File: services/instance-planter/internal/repository/proxmox/lxc_adapter.go
package proxmox

import (
    "context"
    "regtee/instance-planter/internal/domain"
    "regtee/instance-planter/internal/usecase"
)

// Ensure compile-time interface implementation
var _ usecase.HypervisorPort = (*ProxmoxLXCAdapter)(nil)

type ProxmoxLXCAdapter struct {
    client *ProxmoxClient
    node   string
}

func (a *ProxmoxLXCAdapter) CreateContainer(ctx context.Context, req domain.CreateInstanceRequest) (*domain.InstanceResult, error) {
    // Calls physical Proxmox VE /api2/json endpoints
    resp, err := a.client.PostLXC(ctx, a.node, map[string]interface{}{
        "vmid":     req.VMID,
        "ostemplate": req.Template,
        "cores":    req.Cores,
        "memory":   req.MemoryMB,
    })
    if err != nil {
        return nil, err
    }
    return &domain.InstanceResult{AllocatedIP: resp.IP}, nil
}
```

---

## 4. Architectural Verification & Results

By adopting this disciplined structure across the monorepo:
1. **100% Test Coverage of Core Business Logic**: Interactors can be tested with pure Go mock structs in microseconds, without needing a live Proxmox server or RabbitMQ broker.
2. **Provider Agnosticism**: If a new hypervisor (e.g. Incus, OpenStack, or KubeVirt) is introduced in the future, developers write a new adapter implementing `HypervisorPort` with **zero modifications to business logic**.
3. **Independent Compilation**: Each Go module in `services/` contains its own `go.mod`, coordinated at the root via `go.work`. Build times remain lightning fast through Go's module caching.

# AMQP 0-9-1 Messaging Protocols & Schemas

> **Document Version**: 1.0  
> **Classification**: Public Engineering Showcase  
> **Broker**: RabbitMQ 3.13 (AMQP 0-9-1 Protocol)

---

## 1. Broker Topology & Exchange Conventions

Cross-service asynchronous communication across the 7 Go microservices is standardized on RabbitMQ Topic Exchanges with dedicated Dead-Letter Exchanges:

```
                  ┌─────────────────────────────────────────┐
                  │   Topic Exchange: `regtee.events`       │
                  └────────────────────┬────────────────────┘
                                       │
         ┌─────────────────────────────┼─────────────────────────────┐
         │ Routing Key:                │ Routing Key:                │ Routing Key:
         │ `instance.*`                │ `build.*`                   │ `ingress.*`
         ▼                             ▼                             ▼
┌─────────────────────────┐   ┌─────────────────────────┐   ┌─────────────────────────┐
│ instance-planter.queue  │   │ app-builder.queue       │   │ ingress-controller.queue│
└───────────┬─────────────┘   └───────────┬─────────────┘   └───────────┬─────────────┘
            │ Nack (failed)               │ Nack (failed)               │ Nack (failed)
            ▼                             ▼                             ▼
┌─────────────────────────┐   ┌─────────────────────────┐   ┌─────────────────────────┐
│ instance-planter.dlx    │   │ app-builder.dlx         │   │ ingress-controller.dlx  │
│ ──► *.dlq (Dead Letter) │   │ ──► *.dlq (Dead Letter) │   │ ──► *.dlq (Dead Letter) │
└─────────────────────────┘   └─────────────────────────┘   └─────────────────────────┘
```

---

## 2. Event Routing Catalog

| Event Name | Routing Key | Publisher | Consumer(s) | Description |
|:---|:---|:---|:---|:---|
| **Instance Provision** | `instance.provision` | `core-api` / `orchestrator` | `instance-planter` | Triggers LXC/VM creation on Proxmox |
| **Instance Terminated** | `instance.destroy` | `core-api` / `orchestrator` | `instance-planter` | Destroys container and purges ZFS storage |
| **Instance Status** | `instance.status.updated` | `instance-planter` | `core-api`, `orchestrator` | Notifies state transition (`RUNNING`, `STOPPED`) |
| **Build Trigger** | `build.trigger` | `core-api` / `orchestrator` | `app-builder` | Initiates Dagger/Nixpacks container compilation |
| **Build Finished** | `build.finished` | `app-builder` | `core-api`, `orchestrator` | Reports build artifact OCI image tag or failure |
| **Ingress Synchronize** | `ingress.sync` | `core-api` / `orchestrator` | `ingress-controller`| Registers dynamic reverse proxy route in Caddy |
| **Network Enroll** | `network.enroll` | `core-api` / `orchestrator` | `network-controller`| Registers WireGuard node key in Headscale |

---

## 3. Concrete Payload Schemas

### 1. `instance.provision`
Published by the Control Plane to request hypervisor resource allocation:

```json
{
  "event_id": "evt_01J7KXYZ89ABCDEF01234567",
  "correlation_id": "corr_3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "timestamp": "2026-09-10T21:00:00Z",
  "payload": {
    "instance_id": "inst_01J7KXYZ89ABCDEF",
    "workspace_id": "ws_alpha_corp",
    "hostname": "api-server-01",
    "node": "cluster-node-01",
    "vmid": 101,
    "cores": 2,
    "memory_mb": 1024,
    "disk_gb": 10,
    "template": "debian-12-standard",
    "network": {
      "ip_address": "100.64.0.10/24",
      "gateway": "100.64.0.1",
      "bridge": "vmbr0"
    },
    "ssh_public_keys": [
      "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIExampleBastionKey devops@bastion.internal",
      "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIExampleUserKey developer@example.com"
    ]
  }
}
```

### 2. `build.trigger`
Published to trigger an ephemeral container compilation job:

```json
{
  "event_id": "evt_01J7KZZA12BCDEF89012345",
  "correlation_id": "corr_7bb85f64-5717-4562-b3fc-8c963f66bb12",
  "timestamp": "2026-09-10T21:01:00Z",
  "payload": {
    "build_id": "bld_01J7KZZA12BC",
    "app_id": "app_pos_frontend",
    "git_repository": "https://github.com/example-org/demo-frontend.git",
    "git_branch": "main",
    "git_commit_sha": "7a3f89b1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7",
    "target_registry": "localhost:5000",
    "image_name": "ws_alpha_corp/pos-frontend",
    "image_tag": "7a3f89b",
    "env_vars": {
      "NODE_ENV": "production",
      "API_BASE_URL": "https://api.example.com"
    }
  }
}
```

### 3. `ingress.sync`
Published to register or update reverse proxy routing rules:

```json
{
  "event_id": "evt_01J7KBB889ABCDEF01234567",
  "correlation_id": "corr_9cc85f64-5717-4562-b3fc-1d963f66cc34",
  "timestamp": "2026-09-10T21:02:00Z",
  "payload": {
    "route_id": "route_pos_frontend",
    "domain": "demo.apps.example.com",
    "target_upstream": "100.64.0.10:8080",
    "enable_tls": true,
    "websocket_support": true,
    "custom_headers": {
      "X-Forwarded-Proto": "https"
    }
  }
}
```

---

## 4. Delivery Semantics & Idempotency Rules

1. **At-Least-Once Delivery**: Publishers request Publisher Confirms (`confirm.select`) from RabbitMQ before marking records as dispatched in PostgreSQL.
2. **Mandatory Idempotency**: Consumers verify the `event_id` against a deduplication cache (Valkey / Redis) prior to executing operations.
3. **Graceful Nack Handling**: Unhandled exceptions trigger `amqp.Delivery.Nack(false, false)`, immediately transferring the message to the `.dlx` exchange for operator analysis, preventing unyielding queue head-of-line blocking.

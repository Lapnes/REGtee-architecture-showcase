# REST API Specifications & Architecture

> **Document Version**: 1.0  
> **Classification**: Public Engineering Showcase  
> **Service**: `services/core-api`  
> **Interface**: OpenAPI 3.0 / Swagger 2.0 Compliant

---

## 1. Design Principles & Standards

The **`core-api`** service serves as the central control plane for REGtee Cloud. It strictly adheres to modern enterprise API design standards:

- **JSON First**: All request bodies and responses are encoded as UTF-8 JSON.
- **Stateless Authentication**: Uses signed JSON Web Tokens (JWT) using the `HMAC-SHA256` algorithm.
- **Role-Based Access Control (RBAC)**: Workspace boundaries enforce granular permissions (`Owner`, `Admin`, `Member`, `Viewer`).
- **Idempotent State Mutators**: Mutating requests accept an optional `Idempotency-Key` HTTP header.
- **ISO 8601 Timestamps**: All temporal fields are formatted in UTC (e.g. `2026-09-10T21:00:00Z`).

---

## 2. Global Response Envelope & Error Taxonomy

### Successful Response Format
All successful responses return HTTP 200 (OK), 201 (Created), or 202 (Accepted) wrapped in a standard envelope:

```json
{
  "success": true,
  "data": {
    "id": "inst_01J7KXYZ89ABCDEF",
    "name": "production-api-01",
    "status": "RUNNING",
    "ip_address": "100.64.0.10",
    "created_at": "2026-09-10T21:00:00Z"
  },
  "message": "Instance fetched successfully"
}
```

### Standardized Error Format
Errors return appropriate HTTP status codes (400, 401, 403, 404, 409, 422, 500) with detailed machine-readable diagnostic codes:

```json
{
  "success": false,
  "error": {
    "code": "RESOURCE_QUOTA_EXCEEDED",
    "message": "Workspace RAM limit reached (Allocated: 8192MB, Maximum: 8192MB).",
    "details": {
      "workspace_id": "ws_alpha_corp",
      "current_memory_mb": 8192,
      "requested_memory_mb": 1024
    }
  },
  "timestamp": "2026-09-10T21:00:00Z"
}
```

---

## 3. Core API Endpoint Catalog

### Authentication & Identity (`/api/v1/auth`)
- `POST /api/v1/auth/register` — Create new developer account
- `POST /api/v1/auth/login` — Authenticate and retrieve JWT Bearer token
- `GET /api/v1/auth/me` — Retrieve current authenticated user profile
- `POST /api/v1/auth/ssh-keys` — Register developer Ed25519 public SSH keys

### Workspace & RBAC (`/api/v1/workspaces`)
- `GET /api/v1/workspaces` — List workspaces accessible to user
- `POST /api/v1/workspaces` — Create new isolated team workspace
- `GET /api/v1/workspaces/{id}/members` — List members and assigned RBAC roles
- `POST /api/v1/workspaces/{id}/invites` — Invite collaborator with role assignment

### Virtual Compute & IaaS (`/api/v1/instances`)
- `GET /api/v1/instances` — List compute instances in workspace
- `POST /api/v1/instances` — Provision new LXC container or KVM virtual machine (Asynchronous, returns 202 Accepted)
- `GET /api/v1/instances/{id}` — Get instance status, hypervisor node, and IP details
- `POST /api/v1/instances/{id}/actions` — Execute power actions (`start`, `stop`, `restart`, `rebuild`)
- `POST /api/v1/instances/{id}/terminal-token` — Generate single-use OTK for in-browser Web Terminal
- `DELETE /api/v1/instances/{id}` — Teardown container, purge storage, and release IP

### PaaS Application Builds (`/api/v1/apps`)
- `POST /api/v1/apps` — Link Git repository and configure deployment blueprint
- `POST /api/v1/apps/{id}/builds` — Trigger manual Nixpacks/Dagger build pipeline
- `GET /api/v1/apps/{id}/builds/{build_id}/logs` — Stream real-time build and deployment logs
- `POST /api/v1/apps/{id}/rollback` — Rollback to previous immutable container release

### Billing & Wallet (`/api/v1/billing`)
- `GET /api/v1/billing/wallet` — Retrieve current REG Credit balance (in IDR)
- `POST /api/v1/billing/topup` — Initiate Midtrans Snap payment transaction (QRIS, VA, E-Wallet)
- `POST /api/v1/billing/webhook` — Midtrans payment notification webhook receiver (HMAC verified)
- `GET /api/v1/billing/invoices` — List itemized monthly tax invoices

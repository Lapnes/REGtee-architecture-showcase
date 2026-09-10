# Dynamic Ingress: Automated Reverse Proxy & SSL via Caddy

> **Document Version**: 1.0  
> **Classification**: Public Engineering Showcase  
> **Service**: `services/ingress-controller`  
> **Topic**: Edge Ingress, Reverse Proxy, Caddy Admin API, Automated Let's Encrypt TLS

---

## 1. Overview & Problem Statement

In a multi-tenant cloud platform, applications and services are provisioned, scaled, and deleted dynamically. Traditional reverse proxies (such as Nginx or Apache) typically require writing static configuration files to disk followed by executing a configuration reload command (`nginx -s reload`):
- **Worker Process Churn**: Frequent reloads drop active TCP keep-alive connections and WebSocket pipes.
- **Race Conditions**: Concurrent tenant provisioning attempts can corrupt configuration files.
- **Slow Certificate Provisioning**: Manual certbot hooks delay application readiness.

REGtee Cloud eliminates these issues by integrating **Caddy 2** with a dedicated Go microservice: **`ingress-controller`**.

```
 [Tenant Deploys App: `app.demo.example.com`]
                      │
                      ▼
        ┌───────────────────────────┐
        │    Core REST API Server   │
        └─────────────┬─────────────┘
                      │ AMQP Event: `ingress.sync`
                      ▼
        ┌───────────────────────────┐
        │ `ingress-controller` Svc  │
        └─────────────┬─────────────┘
                      │
                      │ HTTP POST JSON to Caddy Admin API
                      │ `http://localhost:2019/config/apps/http/servers/srv0/routes`
                      ▼
        ┌───────────────────────────┐
        │   Caddy Edge Ingress      │ ─── 1. Inserts route in-memory (< 2ms)
        │   (Jakarta Gateway)       │ ─── 2. Issues Let's Encrypt TLS cert in background
        └─────────────┬─────────────┘ ─── 3. Proxies traffic to `100.64.0.10:8080` via Mesh
                      │
        ┌─────────────┴─────────────┐
        ▼                           ▼
  [Client Web Request]        [Let's Encrypt ACME]
```

---

## 2. In-Memory Dynamic Route Management

Instead of editing Caddyfiles on disk, `ingress-controller` communicates directly with Caddy's in-memory JSON config engine via its native **Admin API (`localhost:2019`)**:

```go
// Construct Caddy dynamic route definition
route := CaddyRoute{
    Match: []CaddyMatch{
        {Host: []string{domainName}},
    },
    Handle: []CaddyHandle{
        {
            Handler: "reverse_proxy",
            Upstreams: []CaddyUpstream{
                {Dial: fmt.Sprintf("%s:%d", targetMeshIP, targetPort)},
            },
        },
    },
    Terminal: true,
}

// Atomic in-memory route insertion without dropping active connections
err := caddyClient.InsertRoute(ctx, routeID, route)
```

### Key Advantages:
1. **Sub-Millisecond Updates**: New tenant domains become routable within milliseconds.
2. **Zero Connection Drops**: Active WebSocket terminal sessions and large file uploads are never interrupted.
3. **Automated ACME TLS**: Caddy automatically contacts Let's Encrypt or ZeroSSL to solve HTTP-01 or TLS-ALPN-01 challenges, provisioning HTTPS certificates without human intervention.

---

## 3. Dual Domain Strategy: Platform Subdomains & Custom Domains

REGtee supports two domain routing categories:

| Domain Type | Pattern | Validation & TLS |
|:---|:---|:---|
| **Platform Subdomains** | `{project}.apps.example.com` | Pre-validated wildcard DNS record pointing to Jakarta Gateway. TLS certificate is pre-issued or instantaneous. |
| **Custom Tenant Domains** | `app.custom-domain.example.com` | Tenant configures a `CNAME` pointing to `gateway.example.com`. Caddy On-Demand TLS automatically validates and provisions certificates upon first request. |

---

## 4. Edge Health Checking & Circuit Breaking

The Ingress Controller configures upstream health checks within Caddy:
- Active HTTP health check probes test container endpoints (`/healthz` or `/`) every 10 seconds.
- If a tenant application crashes or enters an unhealthy state, Caddy immediately returns a branded, user-friendly `503 Service Temporarily Unavailable` error page rather than hanging indefinitely.
- Health recovery events are published back to RabbitMQ to update the frontend dashboard status badge in real time.

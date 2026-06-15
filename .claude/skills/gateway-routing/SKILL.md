---
name: gateway-routing
description: >
  Use this skill whenever a service endpoint needs to be exposed, changed, or protected through
  the YarpApiGateway in this e-commerce solution. Trigger it for requests like "expose this
  through the gateway", "add a route for the new service", "the endpoint returns 404 through the
  gateway but works directly", "rate-limit this", "change the proxy path", or any reverse-proxy
  concern. A common bug is adding a new endpoint that works on the service directly but is
  unreachable via the gateway because no YARP route was added — this skill prevents that.
---

# API Gateway Routing (YARP)

`YarpApiGateway` is the single entry point that reverse-proxies to the backend services and
applies cross-cutting policies like rate limiting. Routes and clusters are configured in the
gateway's `appsettings.json` under the `ReverseProxy` section.

## Existing routes (the pattern to follow)
- `/catalog-service/{**catch-all}`  → `http://localhost:6000/`
- `/basket-service/{**catch-all}`   → `http://localhost:6001/`
- `/ordering-service/{**catch-all}` → `http://localhost:6003/`

The `ordering-service` route has a **fixed-window rate limiter: 5 requests / 10 seconds**.

> If a new endpoint is reachable directly (e.g. `:6001/foo`) but 404s through the gateway, the
> cause is almost always a missing or mismatched route/cluster — start there.

## Step 1 — Add or edit a route + cluster

In the gateway `appsettings.json` `ReverseProxy` section, mirror an existing pair:

```json
"ReverseProxy": {
  "Routes": {
    "payment-route": {
      "ClusterId": "payment-cluster",
      "Match": { "Path": "/payment-service/{**catch-all}" }
    }
  },
  "Clusters": {
    "payment-cluster": {
      "Destinations": {
        "destination1": { "Address": "http://localhost:6004/" }
      }
    }
  }
}
```
Rules:
- The `{**catch-all}` catch-all forwards the remainder of the path to the service.
- The destination `Address` must match the service's reachable address — `localhost:600x` when
  running services locally, or the **compose service name** (e.g. `http://payment-api:8080/`)
  when routing inside the Docker network. Use whichever scheme the existing routes use for the
  environment you're targeting.
- Keep route ids and cluster ids consistent with the existing `*-route` / `*-cluster` naming.

## Step 2 — Path transforms (if the service path differs)

If the upstream service expects a path without the `/payment-service` prefix and YARP isn't
already stripping it, add a transform:

```json
"payment-route": {
  "ClusterId": "payment-cluster",
  "Match": { "Path": "/payment-service/{**catch-all}" },
  "Transforms": [ { "PathRemovePrefix": "/payment-service" } ]
}
```
Check how the existing routes handle this and stay consistent — don't introduce a different
prefix convention.

## Step 3 — Rate limiting (optional)

Rate limiters are defined in the gateway `Program.cs` (`AddRateLimiter` with a fixed-window
policy) and attached to a route. To protect a route, attach the policy the same way the
ordering route does:

```json
"payment-route": {
  "ClusterId": "payment-cluster",
  "Match": { "Path": "/payment-service/{**catch-all}" },
  "RateLimiterPolicy": "fixed"
}
```
Tune window/permit-limit in `Program.cs` to mirror the existing `5 requests / 10 seconds`
policy unless the task specifies otherwise.

## Step 4 — Remember the gateway isn't in docker-compose

`YarpApiGateway` runs separately (local profile port `5004`), not as part of
`docker-compose.yml`. When testing gateway routes, run the gateway from Visual Studio / CLI and
ensure the target services are up.

## Step 5 — Verify

```bash
# service directly:
curl http://localhost:6004/payment/ping
# through the gateway:
curl http://localhost:5004/payment-service/payment/ping
```
Both should return the same result. For rate limiting, fire more than the allowed requests in
the window and confirm you get HTTP 429.

## Checklist
- [ ] Route + cluster added/edited in the gateway `appsettings.json`, naming consistent.
- [ ] Destination address matches the target environment (local port vs compose service name).
- [ ] Path prefix handling consistent with existing routes (transform if needed).
- [ ] Rate-limiter policy attached if required, tuned like the existing policy.
- [ ] Verified the endpoint works both directly and through the gateway.

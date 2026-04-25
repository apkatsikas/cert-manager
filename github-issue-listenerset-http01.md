# HTTP-01 ACME challenges fail silently when ListenerSet has no HTTP listener

## Summary

When using [Gateway API ListenerSets](https://gateway-api.sigs.k8s.io/geps/gep-1762/) for multi-tenant TLS termination, cert-manager's ACME HTTP-01 solver creates an HTTPRoute whose `parentRef` points to the ListenerSet. If the ListenerSet carries only HTTPS/TLS listeners — which is the common multi-tenant pattern — the solver HTTPRoute gains no ready listener, the ACME challenge never receives traffic, and the certificate issuance stalls with no actionable error surfaced to the user.

This issue proposes a small cascade rule to fix this, along with a draft PR that implements and tests it.

---

## Background: ListenerSets and multi-tenancy

`ListenerSet` (Gateway API v1.3.0+, `gateway.networking.k8s.io/v1`) is designed for exactly this topology:

```
envoy-gateway-system/
  Gateway "shared-gateway"
    listeners:
      - name: http   port: 80   protocol: HTTP   ← owned by infra team

team-a/
  ListenerSet "team-a-ls"  →  parentRef: shared-gateway
    listeners:
      - name: https  port: 443  protocol: HTTPS  ← owned by team-a

team-b/
  ListenerSet "team-b-ls"  →  parentRef: shared-gateway
    listeners:
      - name: https  port: 443  protocol: HTTPS  ← owned by team-b
```

The infra team owns the Gateway and its shared HTTP listener. Tenant teams own their own ListenerSets and declare TLS there. No team touches another team's resources.

---

## Current behavior

cert-manager discovers a TLS listener on the ListenerSet and creates a Certificate. For ACME HTTP-01 challenges it then creates a solver HTTPRoute with `parentRef` set to **the ListenerSet**:

```yaml
# solver HTTPRoute created by cert-manager
spec:
  parentRefs:
  - kind: ListenerSet
    name: team-a-ls
    namespace: team-a   # ← the ListenerSet
```

Because `team-a-ls` has no HTTP listener, the HTTPRoute reports:

```
status:
  parents:
  - conditions:
    - type: Accepted
      status: "True"
    - type: ResolvedRefs
      status: "True"
  routeParentStatuses:
  - conditions:
    - type: NoReadyListeners
      status: "True"   # ← no HTTP listener on the ListenerSet
```

No HTTP traffic reaches the solver pod. The ACME challenge times out. cert-manager surfaces a vague `order not ready` or `challenge failed` event — there is no warning that points to the misconfigured parentRef.

### The workaround today is ergonomically bad

The only escape is to replicate an HTTP listener on **every** ListenerSet:

```yaml
# team-a/listenerset.yaml — forced workaround
spec:
  listeners:
  - name: http        # ← every tenant must reproduce this
    protocol: HTTP
    port: 80
  - name: https
    protocol: HTTPS
    port: 443
    tls: ...
```

The HTTP listener is real and functional — but it belongs on the shared Gateway, not copied into every tenant namespace. In a multi-tenant cluster this means:
- The infra team cannot provide a "TLS-only ListenerSet" template — every instantiation needs an HTTP listener that is only there to satisfy cert-manager, not to serve application traffic.
- Replicating a port 80 listener across every tenant ListenerSet is not equivalent to having one on the Gateway: depending on the gateway implementation it may require additional IP allocations, open unnecessary ports in tenant namespaces, or conflict with existing Gateway-level HTTP routing rules.
- The clean separation ListenerSets are designed to provide — infra owns shared listeners, tenants declare only what their app needs — is undermined at exactly the layer cert-manager should be most transparent about.

This defeats the purpose of the shared-Gateway multi-tenant model.

---

## Proposed behavior

When cert-manager builds the solver HTTPRoute `parentRef` for a ListenerSet-owned Certificate, apply this cascade:

1. **ListenerSet has an HTTP listener** → attach the solver HTTPRoute to the **ListenerSet** (existing behavior, unchanged).
2. **ListenerSet has no HTTP listener, but the parent Gateway has an HTTP listener** → attach the solver HTTPRoute to the **Gateway** (new fallback).
3. **Neither has an HTTP listener** → parentRef is set to the ListenerSet, exactly as today (no change in behavior).

This means the working topology requires **zero changes** to ListenerSet configuration. The infra team declares one HTTP listener on the Gateway; tenants declare only TLS listeners on their ListenerSets; cert-manager routes ACME traffic through the Gateway's HTTP listener automatically.

### Result

```yaml
# solver HTTPRoute after fix — parentRef falls back to the Gateway
spec:
  parentRefs:
  - kind: Gateway
    name: shared-gateway
    namespace: envoy-gateway-system   # ← correct cross-namespace reference
```

```
status:
  routeParentStatuses:
  - conditions:
    - type: Accepted
      status: "True"
    # NoReadyListeners gone — Gateway has an HTTP listener
```

---

## Scope

The change is confined to the ListenerSet controller path and does not affect Ingress, Gateway (non-ListenerSet), or DNS-01 flows. The cascade is evaluated at reconcile time using the already-fetched parent Gateway object — no new API calls are required.

---

## Draft PR

A working implementation with unit tests is attached in the accompanying PR. The tests cover all three branches of the cascade (ListenerSet HTTP, Gateway HTTP fallback, neither).

---

## Related

- Gateway API GEP-1762 (ListenerSets): https://gateway-api.sigs.k8s.io/geps/gep-1762/
- Gateway API v1.3.0 release notes (ListenerSet GA)
- cert-manager ListenerSet tracking issue: (link if exists)

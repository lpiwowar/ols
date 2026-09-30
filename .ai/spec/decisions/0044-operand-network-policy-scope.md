# 0044: Operand NetworkPolicy Scope and Egress Exceptions

**Status:** Accepted for planned implementation
**Applies to:** lightspeed-operator, lightspeed-hub, lightspeed-agentic-operator
**Tracking:** OLS-4171; monitoring-ingress gap OLS-3943

## Context

OLS operators already reconcile namespace-scoped ingress NetworkPolicies for many operands. The older OLS-4171 epic also calls for egress restrictions and prefers AdminNetworkPolicy, but several current operands contact user-configured or dynamically discovered external endpoints. A static destination allow-list for those pods would break supported behavior. RHOKP and OpenShift MCP have ServiceMonitors but lack the matching cluster-Prometheus ingress rules.

## Decision

Continue using standard namespace-scoped NetworkPolicy for the operand changes. Add scoped Prometheus ingress for RHOKP and standalone MCP. Add egress restrictions only where required outbound connections are absent or bounded: PostgreSQL, the two console-plugin pods, RHOKP, and the local alerts adapter. Keep PostgreSQL's existing narrow ingress. Require target-cluster validation of Kubernetes API access before enforcing the local adapter's egress policy.

Do not impose OLS-managed egress restrictions on the app-server pod, standalone MCP server, OTel Collector, agentic sandbox pods, or hub-managed multicluster alerts adapter. Their provider, tool, tracing, or spoke AlertManager destinations vary, and standard NetworkPolicy cannot select external destinations by hostname. Each exception is explicit non-coverage, not an assertion of full HPSTRAT-104 compliance. Confirm separately whether standard NetworkPolicy is acceptable for the operand portion of HPSTRAT-104. The operator-in-OLM-bundle portion is tracked by OCPSTRAT-3087.

The normative behavior, individual exception rationales, ownership, and tests are in `../what/operand-network-policies.md`.

## Alternatives Considered

### Strict egress policy for every operand

Rejected for this change: a fixed allow-list would block administrator-configured providers, MCP/Helm destinations, tracing backends, or newly registered spoke routes. An apparent restriction with broad allow-all escape rules does not meet the intent of least privilege.

### Require an egress proxy or administrator-maintained CIDRs for every external destination

Deferred: it could allow tighter controls, but introduces a new product-wide configuration and availability dependency. It requires a separate design and validation for every supported external client.

### Replace all operand policies with AdminNetworkPolicy

Deferred pending the HPSTRAT-104 operand-scope acceptance decision. Cluster-scoped priority and explicit administrative Allow/Deny change ownership and conflict behavior, not just resource syntax. Existing namespace policies remain the compatible operand baseline until that decision is made.

## Consequences

- Prometheus gains only the RHOKP/MCP metrics ingress that their ServiceMonitors require.
- A subset of OLS operands gains egress isolation; the named exceptions do not. Other administrator policies can still affect any pod.
- Local alerts polling must not be broken by Kubernetes API Service-address translation; successful cluster validation is required before its egress restriction is enforced.
- The hub-managed adapter and sandbox pod exceptions cross repo boundaries but introduce no new handoff API or CRD field.
- Full HPSTRAT-104 compliance remains unverified until policy-type acceptance and the scope of egress exceptions are resolved with the owning stakeholders.

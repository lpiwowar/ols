# 0044 — Bidirectional A2A Federation Through Dynamic In-Cluster Discovery

## Status

Accepted for planned implementation; follow-up to OLS-2841.

## Context

OLS is an OpenShift/Kubernetes domain agent. Other Lightspeed products, such as OpenStack Lightspeed, own complementary domain knowledge and tools. MCP is appropriate for an agent calling a tool, but it does not describe Agent Cards, task lifecycle, streaming, or authorization between independently operated agents.

The earlier A2A spike scoped OLS as an A2A server. That enables an external orchestrator to call OLS but cannot let an OLS conversation delegate to a specialized peer. A static allowlist would provide that client role but would require an administrator to configure every peer and would not meet the requirement that a supported Lightspeed agent automatically joins when it appears in the OpenShift cluster.

Delegation can disclose data and may lead to mutations. A trust decision based only on an Agent Card or Kubernetes Service label would permit impersonation, and sender-side consent must not become a way to bypass the receiver's authorization controls.

## Decision

OLS will implement both A2A server and client roles. Supported Lightspeed products automatically enroll through Kubernetes Service discovery in administrator-configured namespaces. The OLS operator maintains a runtime peer registry and validates each peer against a supported-product/version catalog, workload identity, TLS identity, Agent Card identity, compatible A2A interface/version, and endpoint health. Agent Cards describe capabilities; they are not by themselves authority to trust an endpoint.

OLS will route only among eligible peers. Deterministic policy checks precede capability/domain ranking. When a peer offers a material advantage, OLS presents the selected peer, rationale, requested task, and exact context proposed for disclosure. The user must explicitly approve that individual delegation. A decline keeps the request local and sends no network request.

Federation permits informational and mutating work. The sender's consent authorizes only disclosure and task submission. Each receiving product independently authenticates the caller and applies its normal authorization, policy, and approval controls before executing a mutation. Workload identities authenticate A2A calls; no user, browser, Kubernetes, MCP, or model-provider credential is forwarded.

The normative lifecycle, configuration, security controls, and repository boundaries are defined in `../what/a2a-federation.md`.

## Consequences

- OLS and supported peers such as OpenStack Lightspeed can delegate to one another without product-specific clients or static per-peer configuration.
- The operator gains a discovery controller, product catalog, identity validation, registry status, and associated RBAC/network-policy responsibilities.
- The service gains registry-aware routing, consent state, durable task lifecycle/cancellation, and A2A-specific workload authentication.
- The console gains a clear consent surface that makes the peer and disclosed context visible before delegation.
- Receiver-controlled authorization prevents federation from becoming a cross-product privilege-escalation path, but products must implement compatible task and approval state reporting.

## Rejected Alternatives

### Static administrator-configured peers

Rejected because it requires per-peer URLs, credentials, and allowlists and does not automatically incorporate supported Lightspeed products discovered in the cluster.

### Trust every discovered Service or Agent Card

Rejected because labels and Card contents are advertisements, not authenticated product identity. A supported-product catalog plus workload and transport identity validation is required.

### Use a central registry service or a registry CR as the source of truth

Rejected for the initial release because Kubernetes Services, endpoints, and controller reconciliation already provide the in-cluster lifecycle signal. A registry service adds a highly available component and separate credentials; a registry CR couples every participating product to an OLS-specific API.

### Use MCP to expose one product as another's tool

Rejected because MCP does not provide the remote agent's Agent Card, task lifecycle, asynchronous status, or agent-level authorization semantics.

### Implement only an OLS A2A server

Rejected because it does not allow an OLS user request to invoke OpenStack Lightspeed or another specialist.

### Forward the requesting user's Kubernetes token to peers

Rejected because tokens are audience/cluster-specific, create confused-deputy risk, and disclose credentials to independently operated products.

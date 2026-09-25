# A2A Federation

OpenShift Lightspeed (OLS) can collaborate with independently deployed, supported Lightspeed products and other supported domain agents using the Agent2Agent (A2A) protocol. A2A is the boundary between agents; MCP remains the boundary between an agent and its tools. A peer is an opaque service: OLS never imports or assumes its prompts, tools, models, data stores, or deployment topology.

This capability is planned as a follow-up to OLS-2841. That spike covered OLS as an A2A server only; federation requires both server and client roles, dynamic in-cluster discovery, and a user-consent delegation flow.

## Behavioral Rules

### Discovery and Trust

1. An enabled OLS instance MUST implement both A2A roles: it publishes an Agent Card and task endpoint for inbound delegation, and it can delegate to eligible remote A2A agents. Either peer may initiate a request.
2. A supported participating product MUST publish an in-cluster Kubernetes Service carrying the A2A discovery labels, an Agent Card endpoint, and its serving workload identity. The discovery metadata MUST identify its product and version but MUST NOT contain credentials, internal topology, tool inventories, prompt text, or customer data.
3. The OLS operator MUST watch only the namespaces allowed by `OLSConfig` for A2A discovery Services. It MUST reconcile discovery idempotently and maintain a runtime peer registry from Services, endpoints, Agent Cards, and health observations; individual peer URLs MUST NOT be configured in `OLSConfig`.
4. A discovered peer is eligible only when its product and version appear in the operator-maintained supported-product catalog; its Service, serving workload identity, TLS identity, and Agent Card identity agree with that catalog; its Agent Card advertises a compatible A2A interface/version; and its endpoint is healthy. Discovery labels or an Agent Card alone MUST NOT establish trust.
5. All supported Lightspeed products matching the catalog and namespace policy MUST enroll automatically. An invalid, incompatible, untrusted, or unhealthy peer MUST remain visible in registry status and metrics with its reason, but MUST NOT be available to routing or receive a task.
6. OLS and peers MUST authenticate A2A calls as Kubernetes workloads using a receiver-validated workload identity and the configured transport trust roots. Browser sessions, Kubernetes user tokens, MCP credentials, LLM credentials, and caller-supplied bearer tokens MUST NOT be forwarded. A sender's human identity, if included, is informational only and never grants remote permissions.

### Routing and Consent

7. The router MUST first apply deterministic eligibility controls: registry health and trust, skill/input-mode compatibility, data-sharing policy, delegation-chain loop prevention, and hop limit. It MUST then rank remaining peers using declared capability/domain metadata and an intent assessment.
8. OLS MUST propose delegation only when an eligible peer offers a material domain advantage over local handling. A request that does not meet that threshold MUST remain local.
9. Before an outbound network request, OLS MUST present the user with the selected peer, routing rationale, requested task, and the exact attachments/context proposed for disclosure. The user MUST explicitly approve or decline that specific delegation.
10. A decline MUST keep processing local, MUST NOT contact the peer, and MUST NOT automatically attempt another peer. Approval authorizes only the approved disclosure and task submission; it MUST NOT grant remote execution permissions or bypass any receiving-agent policy.
11. A delegation sends a bounded task brief containing the user request, selected remote skill, explicitly approved context, correlation ID, remaining deadline, and delegation-chain metadata. OLS MUST redact and classify the brief using its inbound-query safeguards before sending it.
12. OLS MUST preserve a bounded delegation chain and reject a request that names itself in that chain. The default maximum is one remote hop; an administrator may lower but not raise that limit in the initial release. Task IDs are local to their issuing agent and MUST NOT be treated as globally valid identifiers.

### A2A Task Lifecycle

13. OLS MUST negotiate a versioned, mutually compatible A2A interface from the peer Agent Card and use the selected interface over HTTPS. It MUST reject unsupported cards, interfaces, versions, or authentication requirements at validation time; it MUST NOT silently downgrade a protocol or transport.
14. The initial profile MUST support task submission, streaming updates when the peer advertises streaming, task retrieval, and idempotent cancellation. Polling is permitted only as a fallback for a peer that does not advertise streaming. Push notifications, arbitrary protocol extensions, and file upload are not part of the initial profile unless separately specified.
15. OLS MUST persist the local-to-remote task mapping, authenticated peer identity, correlation ID, user-consent outcome, task state, deadline, and terminal outcome. It MUST not persist unredacted peer artifacts outside the existing transcript and retention policy.
16. An inbound A2A request MUST create an isolated task owned by the authenticated peer workload. It MUST run through OLS's normal validation, redaction, quota, RAG, and authorization paths, and MUST NOT share browser conversations or user history. The final answer or task result MUST be returned as an A2A artifact, never as internal chain-of-thought or credentials.
17. Federation supports both informational and mutating work. A sender's consent to delegate MUST NOT substitute for the receiving product's authorization, policy checks, or approval gates. A receiving product MUST apply its own controls before a mutation; if it requires approval, the A2A task MUST report a waiting-for-approval state and MUST NOT silently act.
18. An inbound cancellation MUST stop associated local work, produce an A2A cancelled terminal state, and prevent later artifacts. Cancellation MUST be idempotent. A remote timeout, peer loss, protocol error, or task failure MUST become a bounded failed delegation result and MUST NOT fail unrelated local tools.
19. Remote output is untrusted external content. Before it is streamed to a user, reinjected into a model, or stored in conversation/transcript data, it MUST be bounded, redacted, and processed by the same model-visible external-content inspection contract as MCP tool results.

### Configuration, Observability, and Operations

20. `OLSConfig` owns A2A enablement, discovery namespace scope, supported-product/version catalog, compatible A2A profiles, transport trust roots, task/resource limits, data-sharing policy, and maximum delegation hops. Credentials and private CAs MUST be referenced by Kubernetes Secrets/ConfigMaps, never embedded in the CR. It MUST NOT contain individual peer URLs or per-peer credentials.
21. A configuration, discovery metadata, workload identity, trust-root, or Agent Card change MUST cause peer eligibility to be re-evaluated. A removed, changed, unhealthy, or incompatible peer MUST be withdrawn immediately from new routing; existing tasks MUST continue only according to their deadline and cancellation rules.
22. Every inbound and outbound task MUST emit audit and telemetry events containing peer/product identity, discovered Service identity, selected skill, routing-rationale category, consent outcome, task IDs, interface/version, state/outcome, latency, and correlation ID. Request and artifact content MUST follow existing redaction, retention, and collection policy; credentials MUST never be recorded.

## End-to-End Flows

### Peer discovery and routing

1. A supported Lightspeed product creates or updates its labeled A2A Service, endpoint, and Agent Card in an allowed namespace.
2. The OLS operator observes the Service, fetches and validates its Agent Card, validates its catalog and workload/TLS identity, and writes the resulting availability and reason to the runtime registry/status.
3. For a normal OLS request, the router deterministically filters registry entries and ranks compatible peers. It retains local handling if no peer provides a material domain advantage.
4. When routing proposes a peer, the console presents its identity, rationale, task, and exact disclosure set. Declining resumes local handling; approval records consent and creates the task.
5. The A2A client submits the redacted, approved, bounded task brief using its workload identity. It consumes streaming task updates when available or polls when streaming is unavailable.

### Inbound delegation and execution

1. A trusted peer discovers OLS through its A2A Service and validates OLS's Agent Card.
2. OLS authenticates the caller workload and creates an isolated inbound task.
3. OLS performs its normal validation, authorization, policy, and safety processing. For a requested mutation, it performs its own approval workflow before execution.
4. OLS returns status and final artifacts through the negotiated A2A interface. The caller applies its own untrusted-content protections before showing or consuming the result.

## Repository Ownership

| Repository | Owns |
| --- | --- |
| **lightspeed-service** | A2A server/client, Agent Card generation, task persistence/lifecycle, registry consumption, capability routing, consent handoff, external-content handling, workload-auth integration, metrics/traces. |
| **lightspeed-operator** | OLSConfig A2A API, supported-product catalog, discovery controller, Service/route/NetworkPolicy generation, Secret/CA mounting, config reload, registry status conditions, RBAC. |
| **lightspeed-console** | Delegation proposal/approval, exact-context disclosure view, task state, routing rationale, and peer attribution. It owns no peer credentials or peer-administration UI in the initial release. |
| **Supported peer products (for example OpenStack Lightspeed)** | Their A2A Service/Card, workload identity, client/server behavior, local authorization and approval policy, task execution, and product-specific skills. |

## Verification

- Discovery tests cover Service creation/removal/change, namespace scope, identity/Card mismatch, unsupported product/version, health transitions, and idempotent reconciliation.
- Routing tests cover deterministic eligibility, capability/domain ranking, local handling, consent presentation, approval/decline behavior, loop prevention, and context minimization.
- Interoperability tests run reciprocal OLS and OpenStack Lightspeed deployments for task submission, streaming and polling, task retrieval, cancellation, timeout, and isolation.
- Mutation tests prove sender approval alone cannot execute remote work and that the receiver's authorization and approval gate remains mandatory.
- Security tests prove credential non-forwarding, namespace/RBAC isolation, workload-identity rejection, content bounds/inspection, audit correlation, and unavailable-peer fault isolation.

## Explicit Non-Goals

- A public or organization-wide agent registry outside the OpenShift cluster.
- Model-directed endpoint selection or endpoint discovery outside the validated Kubernetes registry.
- A universal identity or user-token-forwarding scheme.
- Sharing private conversation history, raw cluster data, prompts, or chain-of-thought by default.
- Supporting every A2A transport, extension, modality, push-notification feature, or file-transfer feature in the initial release.

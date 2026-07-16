# Cross-Domain Multi-Agent Identity and Authorization — CNCF/OSS Reference

This POC specifies a cross-domain, multi-agent identity and authorization architecture for enterprise agentic AI workflows, built entirely on **open-source and CNCF components**. The reference scenario is a vulnerability remediation pipeline that spans two trust domains, an autonomous coding agent and a spawned remediation sub-agent, and two IdP boundaries — all coordinated through a unified, standards-grounded trust architecture with no proprietary dependencies.

The architecture composes four standards-grounded layers:

- **AGNTCY-issued W3C Verifiable Credentials** for portable agent capability attestation, discoverable via **AGNTCY dir** and projected as a **Client ID Metadata Document (CIMD)** carrying the badge inline as `vc+jwt`.
- **IETF Identity Assertion JWT Authorization Grant (ID-JAG)** via **Keycloak** token exchange (RFC 8693) for IdP-mediated cross-domain assertion between federated realms.
- **OPA / Cedar policy engine** at the **Envoy AI Gateway** (via `ext_authz`) as the Policy Decision Point — fine-grained, declarative, task-based authorization enforced at every trust boundary, evaluating `action ∈ intent`, scope-subset, act-chain integrity, and delegation depth.
- **End-to-end subject propagation and intent binding** for cryptographically auditable delegation chains, with intent carried in the VC / access token and re-verified at each PDP. *(A Transaction Token Service for intra-domain propagation is deferred to a later phase.)*

Discovery uses **AGNTCY dir**; the DNS-anchored **ANS** is noted as the future internet-scale naming layer, not a build dependency for this PoC.

## Usecase : Cross-Domain, Multi-Agent, Agent-driven Vulnerability Remediation (OSS Stack)

### High Level Business Workflow

Sarah, a security engineer at **Org A**, delegates a vulnerability scan to an **OpenCode** agent running on her Kubernetes-managed workstation. The workflow that follows traverses two trust domains and two agents:

- **Org A Code Scanner (OpenCode):** Scans Org A's internal repository, identifies a CVE in a dependency, and decides to file a remediation ticket in Org B. The scan runs locally — no cross-domain authorization needed (Phase A).
- **Cross-domain hop — Org A → Org B:** The agent fetches and verifies its AGNTCY badge (discovered via `dir` / CIMD), then presents Sarah's Org A ID token as subject and the badge as `actor_token` to Org B's federated **Keycloak** realm, receives an ID-JAG, and exchanges it for a scoped access token. The request passes **Envoy + OPA** at Org B's ingress, which enforces badge signature, scope-subset, and `action ∈ intent` before the ticket is created (Phase B).
- **Org B Triage & Remediation Agent:** Picks up the incident, plans remediation, and determines a sub-agent is needed for dependency resolution and pull-request creation.

POC Workflow Expansion (if time permits):

- **Sub-agent spawn:** The triage agent spawns a Remediation Sub-Agent. The sub-agent receives a narrower AGNTCY badge with a nested act chain (Sarah → OpenCode Scanner → Triage → Remediation Sub-Agent) and a delegation-depth counter, its capabilities a provable subset of its parent's.
- **Sub-agent action:** The sub-agent performs a second token exchange, presents the chained identity, and — after a second Envoy + OPA check (delegation depth, sub-scope subset, `action ∈ intent`) — opens a pull request in **Gitea**.
- **Audit chain complete:** The PR appears in Org A's repository with full provenance back to Sarah's original delegation. **OpenTelemetry** traces link every hop — delegation, token exchanges, PDP decisions, and resource calls — under a shared trace ID, giving a complete, queryable causal audit.

### High Level Architecture Overview and Components

<img width="1600" height="1120" alt="image" src="https://github.com/user-attachments/assets/3ba33241-05e8-4bb8-a1b3-2c8af4be0f92" />

**Stack (all OSS / CNCF):** Kubernetes (KinD) · Envoy AI Gateway · OPA / Cedar · Keycloak · OpenTelemetry · AGNTCY Identity Service · AGNTCY dir · OpenCode.

**Standards:** W3C Verifiable Credentials · RFC 8693 Token Exchange · ID-JAG · Client ID Metadata Document (CIMD). *(Transaction Tokens for agentic context — later phase.)*

### Architecture Workflow Diagram

<img width="1680" height="1500" alt="image" src="https://github.com/user-attachments/assets/5bbc80c6-c744-44c6-b6aa-7215af24306b" />


The end-to-end sequence — local scan, discovery, cross-domain token exchange, PDP enforcement, sub-agent spawn, and audit — is in [`docs/diagrams/sequence.mmd`](docs/diagrams/sequence.mmd) (renders on GitHub / VS Code / mermaid.live).

### Phasing

| Phase | Deliverable | Proves |
|---|---|---|
| 0 | KinD + Envoy AI GW + OPA + 1 Keycloak realm | request → Envoy → OPA → ALLOW/DENY |
| 1 | AGNTCY identity + dir + CIMD projection | register → badge → resolve → verify |
| 2 | OpenCode + OPA gate (single domain) | bounded-privilege, intent-scoped agent |
| 3 | 2nd realm + federation + 8693/ID-JAG + Envoy/OPA at B | cross-domain crossing, intent re-checked |
| 4 | remediation sub-agent + narrowed badge + PR | multi-agent bounded delegation |
| 5 (TBD) | Transaction Token Service + full OTel | intra-domain propagation + causal audit |

## About

POC — CNCF/OSS reference architecture for cross-domain, bounded-privilege, agent-driven vulnerability remediation. Design-stage; implementation follows the phased plan above.

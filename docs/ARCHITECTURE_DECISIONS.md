# NEXORA — Architecture Decision Records

> **Project:** NEXORA — AIOps Observability Platform
> **Hackathon:** CICADA 2K26
> **Document Type:** Architecture Decision Records (ADR)
> **Status:** All decisions are **[Proposed]** — no code has been implemented or experimentally validated.
> **Last Updated:** 2026-11-20

---

**Related Documents:**
- [ARCHITECTURE.md](ARCHITECTURE.md) — System architecture and component design
- [METHODOLOGY.md](METHODOLOGY.md) — Engineering methodology and principles
- [THREAT_MODEL.md](THREAT_MODEL.md) — Threat model and security analysis

---

## Table of Contents

- [ADR Summary Table](#adr-summary-table)
- [ADR-001: Two-Tier Privacy-Preserving Architecture](#adr-001-two-tier-privacy-preserving-architecture)
- [ADR-002: Strict Allow-List as Primary Privacy Mechanism](#adr-002-strict-allow-list-as-primary-privacy-mechanism)
- [ADR-003: Local Anomaly Detection Before Transmission](#adr-003-local-anomaly-detection-before-transmission)
- [ADR-004: Local Buffering with Retry During Network Outages](#adr-004-local-buffering-with-retry-during-network-outages)
- [ADR-005: Egress Proof as a First-Class Feature](#adr-005-egress-proof-as-a-first-class-feature)
- [ADR-006: Cross-Layer Correlation over Single-Layer Detection](#adr-006-cross-layer-correlation-over-single-layer-detection)
- [ADR-007: Explainability as a Requirement for Recommendations](#adr-007-explainability-as-a-requirement-for-recommendations)
- [ADR-008: Human Approval Required for High-Risk Remediation Actions](#adr-008-human-approval-required-for-high-risk-remediation-actions)
- [ADR-009: Fail-Closed Privacy Behavior on Policy Errors](#adr-009-fail-closed-privacy-behavior-on-policy-errors)
- [ADR-010: Telemetry Normalization Before Privacy Filtering](#adr-010-telemetry-normalization-before-privacy-filtering)

---

## ADR Summary Table

| ADR | Title | Status | Primary Domain |
|-----|-------|--------|---------------|
| [ADR-001](#adr-001-two-tier-privacy-preserving-architecture) | Two-Tier Privacy-Preserving Architecture | Proposed | Architecture |
| [ADR-002](#adr-002-strict-allow-list-as-primary-privacy-mechanism) | Strict Allow-List as Primary Privacy Mechanism | Proposed | Privacy |
| [ADR-003](#adr-003-local-anomaly-detection-before-transmission) | Local Anomaly Detection Before Transmission | Proposed | Analytics |
| [ADR-004](#adr-004-local-buffering-with-retry-during-network-outages) | Local Buffering with Retry During Network Outages | Proposed | Reliability |
| [ADR-005](#adr-005-egress-proof-as-a-first-class-feature) | Egress Proof as a First-Class Feature | Proposed | Auditability |
| [ADR-006](#adr-006-cross-layer-correlation-over-single-layer-detection) | Cross-Layer Correlation over Single-Layer Detection | Proposed | Analytics |
| [ADR-007](#adr-007-explainability-as-a-requirement-for-recommendations) | Explainability as a Requirement for Recommendations | Proposed | AI/ML |
| [ADR-008](#adr-008-human-approval-required-for-high-risk-remediation-actions) | Human Approval Required for High-Risk Remediation Actions | Proposed | Operations |
| [ADR-009](#adr-009-fail-closed-privacy-behavior-on-policy-errors) | Fail-Closed Privacy Behavior on Policy Errors | Proposed | Privacy |
| [ADR-010](#adr-010-telemetry-normalization-before-privacy-filtering) | Telemetry Normalization Before Privacy Filtering | Proposed | Data Pipeline |

---

## ADR-001: Two-Tier Privacy-Preserving Architecture

**Date:** 2026-11-20
**Status:** \[Proposed\]

### Context

Enterprise observability platforms have traditionally been built on a centralized SaaS model: all logs, metrics, traces, and events are collected and transmitted to a cloud-hosted platform where storage, analysis, and visualization occur. This design is operationally convenient but introduces significant risks in regulated or security-sensitive environments.

Raw telemetry routinely contains sensitive material that must not leave the customer's environment, including:

- **Credentials and secrets** embedded in log lines (e.g., accidentally logged API keys, database passwords in connection strings)
- **Personally Identifiable Information (PII)** such as usernames, email addresses, session tokens, or customer identifiers in application logs
- **Internal network topology** including IP address ranges, hostname patterns, and service names that reveal the shape of the internal environment
- **Proprietary business logic** observable from application traces and performance profiles

Transmitting raw telemetry to a third-party SaaS platform creates compliance exposure under regulations such as GDPR, HIPAA, and SOC 2, and poses a data sovereignty risk for customers in jurisdictions with strict data residency requirements. At the same time, a purely on-premises solution foregoes the analytical capabilities, centralized cross-tenant intelligence, and operational simplicity that a SaaS model provides.

A design is required that delivers SaaS-level observability value while keeping sensitive raw data entirely within the customer's control.

### Decision

Adopt a **two-tier architecture** as the foundational structural pattern for NEXORA:

- **Tier 1 — Local Collector (Customer Environment):** Deployed and operated entirely within the customer's infrastructure boundary. Responsible for telemetry collection, normalization, privacy filtering, local anomaly detection, local buffering, and egress proof recording. The Local Collector transmits *only explicitly approved, privacy-filtered metadata* to Tier 2. Raw telemetry is never transmitted.

- **Tier 2 — Centralized SaaS Platform:** Operates exclusively on approved metadata received from Tier 1. Responsible for secure ingestion, cross-layer correlation, explainable recommendation generation, alert management, health visualization, and egress proof display.

The **data boundary** is explicit and enforced at the egress point of Tier 1: nothing crosses the boundary unless it has passed through the privacy filtering and allow-list pipeline. This boundary is architecturally verifiable via the Egress Proof mechanism (see [ADR-005](#adr-005-egress-proof-as-a-first-class-feature)).

> [!IMPORTANT]
> The two-tier boundary is the foundational privacy guarantee of NEXORA. Every other privacy decision in this document reinforces this boundary. Any design change that moves raw telemetry processing to Tier 2 fundamentally undermines the privacy model.

### Alternatives Considered

| Alternative | Description | Reason Not Selected |
|-------------|-------------|---------------------|
| **Fully Centralized SaaS** | All raw telemetry transmitted to SaaS for processing | Transmits PII, secrets, and internal topology; compliance exposure; data sovereignty violation |
| **Agent-Only On-Premises (No SaaS)** | Fully local deployment, no central platform | Loses cross-tenant intelligence, centralized correlation, and operational dashboard capabilities |
| **Encrypted Raw Data Upload** | Encrypt raw telemetry before transmission to SaaS | Encryption protects content in transit but the SaaS platform must decrypt to process; metadata and field names may still be exposed; does not eliminate data sovereignty concerns; a compromised SaaS is still a data breach |

### Advantages

- Raw telemetry and sensitive data never leave the customer's infrastructure
- Compliance exposure is significantly reduced: the data subject to external regulation is limited to approved, privacy-filtered metadata
- Customers retain full control and visibility over what is transmitted (supported by [ADR-005](#adr-005-egress-proof-as-a-first-class-feature))
- SaaS platform receives only metadata it is designed to process, reducing its own attack surface
- The two-tier model enables flexible deployment: Tier 1 can be deployed on-premises, in a private cloud, or in a customer-managed public cloud VPC

### Trade-offs

- Requires deploying and maintaining a Local Collector component inside the customer environment, adding operational complexity
- Local Collector must be kept updated and healthy; a failed collector means a loss of observability coverage
- End-to-end system complexity increases compared to a monolithic SaaS deployment
- Local resource consumption (CPU, memory, disk) on customer infrastructure for Tier 1 processing

### Consequences

- All subsequent ADRs are scoped within this two-tier model; decisions about filtering, detection, buffering, and egress proof apply specifically to the data boundary between Tier 1 and Tier 2
- Component ownership is split: privacy-sensitive components (normalization, filtering, detection) are Tier 1; analytical and presentation components are Tier 2
- Deployment, packaging, and update mechanisms must be designed separately for Tier 1 (customer-managed) and Tier 2 (SaaS-managed)
- See [ARCHITECTURE.md](ARCHITECTURE.md) for the component-level design that implements this decision

---

## ADR-002: Strict Allow-List as Primary Privacy Mechanism

**Date:** 2026-11-20
**Status:** \[Proposed\]

### Context

Having established that only approved metadata may cross the Tier 1–Tier 2 boundary (see [ADR-001](#adr-001-two-tier-privacy-preserving-architecture)), a mechanism must be chosen to determine which fields and values are permitted in the outgoing payload.

Two broad approaches exist for field-level filtering: **deny-listing** (block known sensitive fields, pass everything else) and **allow-listing** (explicitly enumerate permitted fields, block everything else).

Deny-list approaches are operationally familiar — they resemble traditional secret-scanning tools, WAF rules, and log redaction configurations — but they carry a fundamental security weakness: they are only as complete as the list of known sensitive fields. In heterogeneous enterprise environments, telemetry is produced by hundreds of different applications, middleware systems, operating systems, and custom services, each with its own schema and naming conventions. It is not feasible to enumerate every field that could ever carry sensitive data across this diversity. A single unanticipated field name — a novel ORM version that logs query parameters differently, a custom middleware that injects authorization headers — creates a data leak that may go undetected.

> [!WARNING]
> Deny-list filtering has a systematic failure mode: it leaks data for any sensitive field not on the list. In adversarial and compliance contexts, this is an unacceptable risk posture. Unknown fields must be blocked by default, not passed through.

### Decision

Use **strict allow-listing** as the primary privacy enforcement mechanism in the Privacy Filtering Engine (Tier 1). The allow-list is the **single source of truth** for what may appear in outgoing metadata payloads:

- Only fields that are explicitly enumerated in the current allow-list policy are permitted in the outgoing payload
- Any field not present in the allow-list is **rejected and dropped**, regardless of its apparent content
- The policy is versioned; the active policy version is recorded in the Egress Proof (see [ADR-005](#adr-005-egress-proof-as-a-first-class-feature))
- Policy evaluation failures cause the event to be **blocked entirely** (see [ADR-009](#adr-009-fail-closed-privacy-behavior-on-policy-errors))
- The allow-list is maintained as a structured, auditable policy artifact, not embedded in application code

This is a **fail-closed** security posture: the default outcome for any unknown or unrecognized field is rejection, not transmission.

### Alternatives Considered

| Alternative | Description | Reason Not Selected |
|-------------|-------------|---------------------|
| **Deny-List Filtering** | Block a list of known sensitive field names; pass all others | Systematically incomplete; unknown sensitive fields leak; maintenance burden grows with environment diversity |
| **Regex-Only Secret Detection** | Scan field values for patterns matching known secret formats (API keys, passwords) | Only catches known secret formats; does not handle PII, topology, or arbitrary sensitive business data; misses novel secret formats |
| **Manual Field Selection per Event** | Operators manually tag events for transmission | Not operationally scalable; human error rate is unacceptable in automated telemetry pipelines |

### Advantages

- **Fail-closed security posture:** unknown fields cannot accidentally reach Tier 2; the default is rejection, not transmission
- **Auditability:** the allow-list is an explicit, versioned, reviewable document that can be inspected by security teams and auditors
- **Predictable behavior:** the filtering outcome is deterministic and testable; there are no heuristics or probabilistic judgments in the primary filtering path
- **Reduced attack surface on Tier 2:** the SaaS platform receives only fields it was explicitly designed to handle, simplifying its own validation requirements

### Trade-offs

- Allow-list must be maintained and updated as telemetry schemas evolve; a missing field means a legitimate signal is blocked
- Initial allow-list population requires a careful audit of all desired metadata fields across all telemetry sources
- Operators may experience friction when new telemetry fields are added to sources — these must be explicitly added to the allow-list before they propagate to Tier 2
- Requires a schema governance process to manage allow-list evolution

### Consequences

- The Strict Allow-List Policy Engine becomes a critical Tier 1 component that must be tested, versioned, and deployed with rigorous change control
- Allow-list policy updates must be treated as security-sensitive changes requiring review before deployment
- Schema governance for telemetry normalization (see [ADR-010](#adr-010-telemetry-normalization-before-privacy-filtering)) directly impacts the allow-list: normalized field names are the allow-list's vocabulary
- See [THREAT_MODEL.md](THREAT_MODEL.md) for the threat analysis of allow-list bypass scenarios

---

## ADR-003: Local Anomaly Detection Before Transmission

**Date:** 2026-11-20
**Status:** \[Proposed\]

### Context

A core value proposition of NEXORA is anomaly detection: identifying deviations from normal behavior across infrastructure, application, network, and security layers. However, the two-tier privacy model (see [ADR-001](#adr-001-two-tier-privacy-preserving-architecture)) creates a constraint: if anomaly detection is performed exclusively by the Tier 2 SaaS platform, the platform must receive raw or richly detailed telemetry to make accurate detections. This directly conflicts with the privacy boundary.

Anomaly detection requires access to the full telemetry context: metric values over time, log message content, event sequences, and resource utilization patterns. Much of this content is sensitive and would not pass the allow-list (see [ADR-002](#adr-002-strict-allow-list-as-primary-privacy-mechanism)). Sending only pre-filtered metadata to SaaS for detection would significantly reduce detection capability.

A mechanism is needed to deliver anomaly intelligence to Tier 2 without transmitting the sensitive underlying telemetry.

### Decision

Deploy a **Local Anomaly Detection Engine** as a Tier 1 component within the customer environment. This engine:

- Runs lightweight anomaly detection algorithms against the full, unfiltered telemetry in-environment, where it is permissible to access raw data
- Produces **anomaly signals** — structured metadata records describing detected anomalies (severity, affected component, detection type, timestamp, statistical context) — rather than raw telemetry
- Anomaly signals are themselves evaluated by the allow-list (see [ADR-002](#adr-002-strict-allow-list-as-primary-privacy-mechanism)) before transmission; they are designed to contain only approved metadata fields
- Raw telemetry values that triggered the anomaly remain in Tier 1 and are not included in the outgoing signal

The separation is fundamental: **the detection logic runs against raw data locally; only the detection outcome (a structured signal) crosses the privacy boundary.**

### Alternatives Considered

| Alternative | Description | Reason Not Selected |
|-------------|-------------|---------------------|
| **SaaS-Only Detection** | Transmit raw or richly detailed telemetry to SaaS for anomaly detection | Requires transmitting sensitive data; violates the privacy boundary established in ADR-001 |
| **No Local Detection** | Omit anomaly detection entirely from Tier 1; rely only on metadata patterns in Tier 2 | Severely limits detection capability; compound anomalies involving sensitive fields would be undetectable |
| **Edge ML Models (Pre-trained, Static)** | Deploy pre-trained ML models to Tier 1 for detection | Static models cannot adapt to customer-specific baseline behavior; retraining requires data export, which risks privacy |

### Advantages

- Enables anomaly detection capability without transmitting raw telemetry to Tier 2
- Local detection has access to the full telemetry context, including sensitive fields that would not pass the allow-list
- Anomaly signals are compact and schema-controlled, making allow-list validation straightforward
- Reduces the volume of data transmitted to Tier 2 (only anomaly signals, not raw metric streams)

### Trade-offs

- Local anomaly detection consumes computational resources on customer infrastructure (CPU for metric analysis, memory for time-series windows)
- Local models operating on a single customer's data may have lower accuracy than centralized models that could observe cross-tenant patterns, particularly for novel or low-frequency anomaly types
- Detection model updates must be delivered to and applied within the customer environment, requiring a secure update mechanism
- The design and selection of appropriate lightweight detection algorithms for Tier 1 is an open engineering challenge at this stage

### Consequences

- The Local Anomaly Detection Engine is a first-class Tier 1 component with its own lifecycle, configuration, and resource requirements
- The schema for anomaly signals must be defined and version-controlled; it is part of the allow-list vocabulary
- Cross-layer correlation (see [ADR-006](#adr-006-cross-layer-correlation-over-single-layer-detection)) in Tier 2 operates on anomaly signals received from Tier 1, not on raw telemetry
- See [ARCHITECTURE.md](ARCHITECTURE.md) for the Local Anomaly Detection Engine component design

---

## ADR-004: Local Buffering with Retry During Network Outages

**Date:** 2026-11-20
**Status:** \[Proposed\]

### Context

The two-tier architecture requires network connectivity between the Tier 1 Local Collector and the Tier 2 SaaS platform for transmission of approved metadata. In production enterprise environments, network connectivity is not always reliable. Causes of outage include scheduled maintenance windows, ISP disruptions, DNS failures, firewall policy changes, certificate expiry, and infrastructure incidents that the Local Collector is itself monitoring.

If approved metadata is discarded during a connectivity outage, the following consequences occur:

- Gaps appear in the Tier 2 timeline, creating blind spots in incident analysis
- Anomaly signals generated during an outage (which may correlate directly with the cause of the outage) are permanently lost
- Egress Proof records (see [ADR-005](#adr-005-egress-proof-as-a-first-class-feature)) cannot be completed for lost events
- Operators investigating a post-outage incident may be missing the most relevant data

> [!IMPORTANT]
> Network outages in an observability platform are highest-risk moments: an infrastructure incident may simultaneously degrade connectivity. The Local Collector must continue operating and preserving data during exactly these scenarios.

### Decision

Implement **local persistent buffering** in the Local Buffer and Retry Manager (Tier 1). The buffer:

- Stores approved, privacy-filtered metadata events to durable local storage (disk-backed queue) before and during transmission attempts
- Applies **exponential backoff with jitter** for reconnection and retry after a detected outage, to avoid thundering-herd effects on Tier 2 during recovery
- Applies **deduplication** at the point of delivery to Tier 2, using event IDs, to prevent duplicate event ingestion if a transmission was partially acknowledged before an interruption
- Enforces a configurable maximum buffer size and retention period; events exceeding retention limits are dropped with an audit log entry
- The buffer does not store raw telemetry — only events that have already passed through the full privacy pipeline

Events in the local buffer are already privacy-filtered and approved; the buffer does not re-execute the privacy pipeline on retry.

### Alternatives Considered

| Alternative | Description | Reason Not Selected |
|-------------|-------------|---------------------|
| **Drop-on-Disconnect** | Discard events that cannot be transmitted immediately | Permanent data loss during outages; loses anomaly signals generated during incidents; unacceptable for observability |
| **In-Memory-Only Buffer** | Buffer events in process memory without persistence | Events are lost if the Local Collector process restarts (e.g., during the incident causing the outage); does not solve the durability requirement |
| **Synchronous-Only Transmission** | Block telemetry pipeline until transmission succeeds | Causes backpressure to propagate upstream and may overwhelm the normalization and filtering pipeline; still risks data loss if blocking is bounded |

### Advantages

- Preserves observability continuity across network outages; no data loss for events within the buffer retention window
- Anomaly signals from the outage period are available for post-incident analysis in Tier 2 when connectivity is restored
- Deduplication ensures that Tier 2 event counts remain accurate even after retry storms
- Decouples the upstream telemetry pipeline from downstream transmission availability

### Trade-offs

- Requires allocation of local disk storage for the persistent buffer; the amount depends on event volume and expected outage duration
- Deduplication logic (event ID tracking, idempotency checks at Tier 2) adds implementation complexity
- Very long outages may exhaust the buffer; older events will be dropped to make room for newer ones — the most relevant events (most recent) are preserved at the cost of older ones
- Persistent storage on the Local Collector host must be secured against unauthorized access, as the buffer contains privacy-approved metadata

### Consequences

- The Local Buffer and Retry Manager is a critical Tier 1 reliability component; its failure modes must be addressed in the reliability architecture
- Tier 2 Metadata Ingestion API must support idempotent event ingestion (using event IDs) to correctly handle retried events
- Event timestamps must be preserved from the original capture time, not the retry transmission time, to ensure accurate Tier 2 timeline reconstruction
- See [ARCHITECTURE.md](ARCHITECTURE.md) for the Local Buffer and Retry Manager component design

---

## ADR-005: Egress Proof as a First-Class Feature

**Date:** 2026-11-20
**Status:** \[Proposed\]

### Context

A privacy-preserving architecture makes a claim: *only approved metadata left the customer's environment.* Without a mechanism to verify this claim, it rests entirely on trust in the system's implementation. For enterprise customers operating under compliance and audit obligations — and for security-conscious operators who want to verify vendor behavior — an unverifiable privacy claim is insufficient.

The question an auditor or security engineer must be able to answer is: *"What exactly was transmitted to the SaaS platform, and can I prove it matches what the system says was transmitted?"*

A logging-only approach — recording that an event was transmitted — does not provide integrity assurance. A log entry can be edited, deleted, or forged. A cryptographic hash of the transmitted payload provides integrity verification, but a hash alone is not sufficient: it proves that a particular payload was transmitted, but only allows verification *if the original payload is also available for comparison*. Without access to the payload itself alongside the hash, verification is impossible.

> [!WARNING]
> A hash without the corresponding payload is not a verifiable egress proof. Meaningful egress verification requires that the approved serialized payload, the hash, and the SaaS receipt are all recorded and can be compared. NEXORA's egress proof design must record all three.

### Decision

Implement **Egress Proof** as a first-class, non-optional feature of the Local Collector. At the moment of transmission, the following are recorded as an Egress Proof record:

| Egress Proof Field | Description | Purpose |
|---|---|---|
| **Event ID** | Unique identifier for the event | Correlation between local record and SaaS receipt |
| **Timestamp** | ISO 8601 transmission timestamp | Timeline reconstruction and audit |
| **Policy Version** | The allow-list policy version applied | Auditability of which policy governed the event |
| **Approved Serialized Payload** | The exact bytes of the payload transmitted | Enables payload-level comparison with SaaS receipt |
| **SHA-256 Integrity Hash** | Hash of the approved serialized payload | Detects payload tampering or corruption in transit |
| **Transmission Status** | Success, failure, retry count | Operational record of delivery outcome |
| **SaaS Receipt** | Acknowledgement from Tier 2, including received payload hash | Closes the verification loop: compares what was sent with what was received |

The **Egress Proof Panel** in the Tier 2 dashboard displays these records to operators, allowing side-by-side comparison of the approved payload and the SaaS-received payload. A mismatch between the two indicates a potential integrity failure.

> [!IMPORTANT]
> The integrity hash alone does not constitute proof of what was transmitted. The approved serialized payload must be stored alongside the hash to allow meaningful verification. The Egress Proof Panel must display both and explicitly highlight any payload mismatch.

### Alternatives Considered

| Alternative | Description | Reason Not Selected |
|-------------|-------------|---------------------|
| **Logging Only** | Record a log entry when each transmission occurs | No integrity guarantee; logs can be edited or deleted; cannot detect tampering in transit |
| **Trust-Based Claims** | Publish a policy document stating what the system transmits | Provides no event-level verification; offers no recourse if the system behaves differently from its documented policy |
| **External Audit Service** | Route all transmissions through a third-party audit intermediary | Introduces a third party into the trust model; audit service itself becomes a privacy risk; adds latency and operational dependency |

### Advantages

- Provides auditable, event-level evidence of what left the customer environment
- Enables operators and auditors to verify privacy compliance without trusting the system's implementation alone
- Payload comparison between approved payload and SaaS receipt detects both transmission corruption and potential man-in-the-middle modification
- Policy version recording allows auditors to determine which privacy policy was applied to each event
- Strengthens customer trust in the NEXORA privacy model with verifiable, not merely claimed, evidence

### Trade-offs

- Storing approved serialized payloads locally increases disk usage on the Tier 1 host
- Egress Proof records must themselves be protected from unauthorized access and tampering; they describe what was transmitted
- Retention management for Egress Proof records must be defined and enforced
- The SaaS receipt mechanism requires Tier 2 to actively participate in the proof process (not just acknowledge receipt, but return received-payload metadata)

### Consequences

- Egress Proof record schema must be defined as part of the Tier 1–Tier 2 protocol; it is not optional metadata
- The Egress Proof Panel is a required Tier 2 UI component, not an enhancement
- The design of ADR-004 (buffering) must account for Egress Proof state: a buffered event's proof record is not complete until the SaaS receipt is received
- See [ARCHITECTURE.md](ARCHITECTURE.md) for the Egress Proof architecture (Stages A–E) and [THREAT_MODEL.md](THREAT_MODEL.md) for the threat analysis of egress proof bypass

---

## ADR-006: Cross-Layer Correlation over Single-Layer Detection

**Date:** 2026-11-20
**Status:** \[Proposed\]

### Context

Modern infrastructure incidents rarely manifest as isolated events within a single observability layer. A storage I/O degradation may cause application timeouts, which triggers elevated error rates, which in turn causes autoscaler activity, which produces network traffic anomalies — all within seconds of each other. Each event in this chain may be detectable individually by a layer-specific monitoring tool, but the causal relationship between them is invisible if each layer is monitored in isolation.

Operational teams managing complex distributed systems frequently face alert storms during incidents: dozens of independent alerts fire simultaneously across infrastructure, application, network, and security dashboards. Without a mechanism to identify which alerts are related, operators must manually correlate events across multiple tools and dashboards, increasing mean time to detection (MTTD) and mean time to resolution (MTTR).

Single-layer monitoring — the dominant model in most existing observability tools — optimizes for detection within a layer but provides no structural mechanism for cross-layer reasoning.

### Decision

Implement a **Cross-Layer Correlation Engine** as a core Tier 2 analytical component. This engine:

- Receives approved metadata and anomaly signals from multiple observability layers (infrastructure, application, network, security) sourced from Tier 1
- Uses **temporal proximity** (events occurring within a configurable correlation window) as a primary correlation signal
- Uses **shared identifiers** (host ID, service name, namespace, cluster ID) to associate events across layers that share a common subject
- Groups correlated events into **incident candidates** — structured collections of potentially related events presented to operators for review
- Explicitly labels correlation output as **"potentially related"** — NEXORA does not claim to establish confirmed causal links; correlation is a signal for human investigation, not a determination of causation

> [!IMPORTANT]
> Cross-layer correlation identifies events that co-occur and share identifiers. It does not prove causal relationships. All correlation outputs must be clearly labeled as "potentially correlated" and operators must be explicitly informed that manual investigation is required to confirm causation.

### Alternatives Considered

| Alternative | Description | Reason Not Selected |
|-------------|-------------|---------------------|
| **Single-Layer Monitoring** | Each layer monitored independently with no cross-layer linkage | Misses compound failures; produces alert storms without context; does not reduce MTTD for cascading incidents |
| **Manual Correlation by Operators** | Operators manually investigate and correlate alerts across tools | Does not scale with alert volume or system complexity; increases operator cognitive load during incidents |
| **ML-Based Root-Cause Analysis Only** | Use machine learning to automatically determine root cause without structural correlation | Root-cause determination from approved metadata alone is methodologically unreliable at this design stage; structured correlation is a more defensible foundation |

### Advantages

- Reduces alert fatigue by grouping related events into incident candidates rather than presenting each independently
- Surfaces compound failure patterns that single-layer monitoring cannot detect
- Provides operators with cross-layer context during incident investigation, reducing manual correlation effort
- Provides a structured foundation for the Explainable Recommendation Engine (see [ADR-007](#adr-007-explainability-as-a-requirement-for-recommendations))

### Trade-offs

- Cross-layer correlation requires **consistent event schemas** across all telemetry sources; schema inconsistencies (resolved by [ADR-010](#adr-010-telemetry-normalization-before-privacy-filtering)) will degrade correlation quality
- **False correlation is possible:** unrelated events that co-occur within the correlation window and share identifiers may be grouped together; this risk must be managed through tunable window parameters and clear labeling
- The correlation engine must clearly and consistently distinguish "correlated" from "causally linked" — failure to do so will mislead operators and erode trust
- Correlation window tuning (too narrow misses related events; too wide produces false associations) requires operational experience with specific customer environments

### Consequences

- The Cross-Layer Correlation Engine depends on consistent normalized event schemas from [ADR-010](#adr-010-telemetry-normalization-before-privacy-filtering); schema drift will cause correlation failures
- Output of the correlation engine feeds the Explainable Recommendation Engine ([ADR-007](#adr-007-explainability-as-a-requirement-for-recommendations)); correlation quality directly affects recommendation quality
- Dashboard design must present correlated event groups in a way that clearly communicates the probabilistic, investigative nature of the correlation
- See [ARCHITECTURE.md](ARCHITECTURE.md) for the Cross-Layer Correlation Engine component design

---

## ADR-007: Explainability as a Requirement for Recommendations

**Date:** 2026-11-20
**Status:** \[Proposed\]

### Context

The Explainable Recommendation Engine is a Tier 2 component responsible for translating anomaly signals and correlated incident candidates into actionable operational recommendations. The fundamental question this component must answer is: *"What should the operator do next, and why?"*

A system that produces a recommendation without explaining its basis creates two failure modes:

1. **Blind acceptance:** The operator acts on the recommendation without understanding it, potentially executing an inappropriate action in a context the system did not fully understand.
2. **Blanket rejection:** The operator distrusts the recommendation because they cannot evaluate its validity, rendering the system's analytical output useless.

Both failure modes are common consequences of black-box AI systems in operational contexts. In production environments, where remediation actions carry real risk, unexplained recommendations are a liability rather than an asset. Operators have institutional knowledge of their specific environment that the system cannot have; they need enough information to evaluate whether a recommendation is applicable to their situation.

Additionally, in audit and compliance contexts, being able to demonstrate *why* a system made a recommendation — and what evidence supported it — is often a regulatory or organizational requirement.

### Decision

**Every recommendation generated by the Explainable Recommendation Engine must include the following structured components:**

| Recommendation Component | Description |
|---|---|
| **Affected Component** | The specific service, host, container, or resource the recommendation concerns |
| **Supporting Evidence** | The anomaly signals and correlated events that triggered the recommendation |
| **Related Telemetry References** | References (not raw values) to the telemetry types and time window that contributed to the analysis |
| **Plain-Language Explanation** | A human-readable description of why this recommendation was generated, in operational terms |
| **Confidence Indicator** | A qualitative signal (not a falsely precise probability) of recommendation confidence |
| **Recommended Action** | The specific proposed action, clearly identified as a proposal requiring human approval (see [ADR-008](#adr-008-human-approval-required-for-high-risk-remediation-actions)) |

No recommendation may be surfaced to operators without all required components. A recommendation that cannot be explained must not be presented.

### Alternatives Considered

| Alternative | Description | Reason Not Selected |
|-------------|-------------|---------------------|
| **Anomaly Score Only** | Surface a numerical anomaly score without explanation | Score alone provides no actionable guidance; operators cannot evaluate applicability to their specific context |
| **Probabilistic Output Without Explanation** | Present recommendations with probability estimates but no supporting evidence | Probabilities without evidence context are not interpretable by operators; false precision in probability estimates erodes trust |
| **Trust ML Output Directly** | Present recommendations as authoritative without requiring explanation | Removes human judgment from operational decisions; unacceptable in production environments; contradicts human-approval requirement in ADR-008 |

### Advantages

- **Increases operator trust:** Operators can evaluate the basis of each recommendation rather than accepting or rejecting it blindly
- **Enables human review before action:** Transparent recommendations support the human-approval model required by [ADR-008](#adr-008-human-approval-required-for-high-risk-remediation-actions)
- **Supports audit and compliance:** Recommendation explanations are auditable records of system reasoning
- **Surfaces knowledge gaps:** When a recommendation's explanation reveals a gap in the system's understanding, operators can identify it and act accordingly
- **Reduces alert fatigue by increasing signal quality:** Explained recommendations carry more information per alert, enabling operators to triage more effectively

### Trade-offs

- Generating high-quality plain-language explanations is a significant engineering challenge; poorly written explanations may confuse operators rather than assist them
- Requiring all explanation components for every recommendation may slow the recommendation pipeline if any component is unavailable
- The structured explanation schema must be maintained as the recommendation engine evolves; schema changes require documentation updates

### Consequences

- The Explainable Recommendation Engine must include an explanation generation subsystem as a first-class concern, not a post-hoc feature
- The structured recommendation schema must be defined and versioned as part of the Tier 2 data model
- Dashboard design must present all required recommendation components without overwhelming operators; UX design is a first-class concern
- See [ARCHITECTURE.md](ARCHITECTURE.md) for the Explainable Recommendation Engine component design

---

## ADR-008: Human Approval Required for High-Risk Remediation Actions

**Date:** 2026-11-20
**Status:** \[Proposed\] — The human-approval UI and action execution engine are **not implemented** in the current design phase. This ADR records the intended policy for future implementation.

### Context

An AIOps observability platform that can detect anomalies and generate recommendations naturally raises the question of autonomous remediation: should the system be able to act on its recommendations automatically without operator intervention? Autonomous execution could reduce MTTD/MTTR and alleviate operational burden in high-volume environments.

However, autonomous execution of remediation actions in production environments carries substantial risk:

- **Incorrect diagnosis:** If the correlation engine identifies the wrong root cause, an autonomous action may worsen the incident (e.g., restarting a healthy service that depends on a failing one)
- **Environment-specific context:** The system may lack knowledge of maintenance windows, deployment freeze periods, change management gates, or business-specific constraints that an operator would know
- **Cascade risk:** Actions like scaling, restarting, or reconfiguring resources can trigger cascading effects that the system cannot fully predict from metadata alone
- **Auditability:** Autonomous actions in regulated environments require a human authorization record

> [!WARNING]
> Autonomous remediation in production systems without human oversight is a high-risk operational pattern. NEXORA operates on approved metadata, not raw telemetry; its view of the environment is intentionally limited. This limitation makes autonomous action particularly dangerous — the system cannot have full situational awareness.

### Decision

For the current design phase, **all remediation actions proposed by NEXORA require explicit human approval before execution.** The system's role in the remediation workflow is:

1. **Detect** an anomaly or incident (Tier 1 and Tier 2)
2. **Correlate** related events (Cross-Layer Correlation Engine)
3. **Recommend** an action with full explanation (Explainable Recommendation Engine, see [ADR-007](#adr-007-explainability-as-a-requirement-for-recommendations))
4. **Present** the recommendation to an authorized operator via the dashboard
5. **Wait** for explicit operator approval before any action is executed
6. **Record** the approval decision (and approving identity) in the audit log

NEXORA **proposes; the operator decides.**

### Alternatives Considered

| Alternative | Description | Reason Not Selected |
|-------------|-------------|---------------------|
| **Fully Autonomous Execution** | System executes approved recommendation classes without human review | Unacceptable risk in production given the system's limited view from approved metadata; audit and compliance concerns |
| **Automated Low-Risk Actions** | Classify certain actions (e.g., log rotation, alert silence) as low-risk and execute autonomously | Classification of "low-risk" is environment-dependent; what is low-risk in a development environment may be high-risk in production; deferred to future design phase |
| **Canary-Based Autonomous Actions** | Execute actions on a subset of infrastructure first; expand if no negative signals | Requires reliable rollback mechanisms and sophisticated progressive deployment infrastructure not in scope for this design phase |

### Advantages

- Maintains operator situational awareness and control during incidents
- Prevents automated cascade failures caused by incorrect system diagnosis
- Provides a clear, auditable human authorization record for every action taken
- Aligns with change management and compliance requirements in regulated industries
- Preserves operator trust in the system by keeping humans in the decision loop

### Trade-offs

- Human approval introduces latency into the remediation loop; autonomous systems would respond faster
- In high-volume alert environments, operator attention may be a bottleneck
- The benefit of AI-powered recommendations is partially offset if operators routinely approve recommendations without review (the "rubber-stamp" failure mode); UX design must actively support thoughtful review

### Consequences

- A human-approval workflow UI is a required component of the Tier 2 dashboard (not yet designed)
- An action execution engine that waits for and processes approval decisions must be designed in a future phase
- The audit log must record: recommendation ID, proposed action, operator identity, approval/rejection decision, and timestamp
- Escalation workflows (e.g., what happens if no operator approves within a time window) must be designed
- See [ARCHITECTURE.md](ARCHITECTURE.md) for the Dashboard and Alert Manager component design

---

## ADR-009: Fail-Closed Privacy Behavior on Policy Errors

**Date:** 2026-11-20
**Status:** \[Proposed\]

### Context

The privacy filtering pipeline is software, and software has failure modes. Policy files may be malformed. The allow-list evaluation engine may encounter an unexpected field type, a null value where a string was expected, or an internal exception during evaluation. Configuration may be inconsistent across component versions. In each of these cases, the privacy pipeline must decide what to do with the event that triggered the failure.

There are two possible defaults:

- **Fail-open:** On error, transmit the event anyway (prioritize observability continuity over privacy)
- **Fail-closed:** On error, block the event and quarantine it for review (prioritize privacy over observability continuity)

Fail-open behavior is tempting from an operational continuity perspective — it means anomaly signals are never lost due to policy engine errors. However, fail-open behavior in a privacy system means that a policy error silently creates a window in which unvalidated data is transmitted to Tier 2. If the error is caused by a malformed or corrupted policy, the contents of the outgoing payload during the error window are undefined and potentially include sensitive raw telemetry.

> [!IMPORTANT]
> Fail-open privacy behavior is architecturally incompatible with a privacy-by-design system. An error in the privacy pipeline is not an anomaly to be tolerated silently — it is a potential privacy breach in progress. The correct response is to block, not to pass.

> [!WARNING]
> Any error state in the privacy pipeline that results in data transmission without completed allow-list validation is a privacy violation, regardless of the probability that sensitive data was present in the event.

### Decision

The Privacy Filtering Engine and Allow-List Policy Engine operate with **fail-closed behavior as the unconditional default.** The following conditions cause an event to be blocked and quarantined for operator review:

| Failure Condition | Consequence |
|---|---|
| Unhandled exception in the privacy pipeline | Event blocked and quarantined |
| Unknown or unrecognized field (not in allow-list) | Event blocked (this is normal allow-list behavior per ADR-002) |
| Policy file parse failure | All events blocked until policy is restored; alert raised |
| Policy version mismatch or integrity failure | Event blocked; alert raised |
| Validation error on normalized event schema | Event blocked and quarantined |
| Allow-list evaluation timeout | Event blocked; timeout alert raised |

A **quarantine queue** holds blocked events for operator inspection. Quarantined events do not contain the full raw telemetry but do contain sufficient diagnostic metadata to identify the failure cause. The raw telemetry that triggered the failure remains in Tier 1 and is not accessible via the quarantine queue.

### Alternatives Considered

| Alternative | Description | Reason Not Selected |
|-------------|-------------|---------------------|
| **Fail-Open (Transmit on Error)** | Transmit the event when the privacy pipeline encounters an error | Creates an undefined privacy window; unvalidated data may reach Tier 2; incompatible with privacy-by-design |
| **Partial Transmission** | Transmit only the fields that passed validation before the error; drop the rest | Partial allow-list validation is not equivalent to full validation; the partial payload's safety cannot be guaranteed |
| **Alert and Pass** | Raise an alert about the error but transmit the event anyway | Same as fail-open with a notification; does not prevent the privacy exposure; alert may not be seen before the event reaches Tier 2 |

### Advantages

- **Strongest available privacy guarantee:** No unvalidated data can reach Tier 2 regardless of the failure mode
- **Aligns with privacy-by-design principles:** Privacy is preserved by default without requiring explicit operator action during a failure
- **Forces visibility of policy errors:** Fail-closed behavior makes policy failures immediately visible through blocked events, preventing silent policy degradation
- **Auditability:** Quarantined events create an audit trail of policy failures that can be reviewed and remediated

### Trade-offs

- Policy errors cause observability gaps: anomaly signals generated during a policy failure window are lost until the policy is restored
- A malformed policy update can inadvertently block all telemetry transmission; rigorous policy change management is required
- The quarantine queue adds storage and operational complexity; it must be monitored and periodically drained

### Consequences

- Policy deployment processes must include validation and rollback capabilities; a bad policy update must be detectable and reversible
- Operators must be alerted when events are quarantined so they can investigate and restore the policy
- The Local Buffer (see [ADR-004](#adr-004-local-buffering-with-retry-during-network-outages)) does not store quarantined events; they are held separately pending policy resolution
- See [THREAT_MODEL.md](THREAT_MODEL.md) for the threat analysis of policy manipulation as an attack vector

---

## ADR-010: Telemetry Normalization Before Privacy Filtering

**Date:** 2026-11-20
**Status:** \[Proposed\]

### Context

NEXORA is designed to monitor heterogeneous enterprise environments, ingesting telemetry from a wide variety of sources simultaneously. These sources include but are not limited to:

- **Infrastructure metrics:** Prometheus, StatsD, SNMP
- **System logs:** syslog (RFC 3164, RFC 5424), Windows Event Log
- **Application telemetry:** OpenTelemetry traces and metrics, custom JSON logs
- **Network telemetry:** NetFlow, sFlow, firewall logs
- **Security telemetry:** IDS/IPS alerts, audit logs, authentication events

Each of these sources uses a different schema, field naming convention, severity scale, timestamp format, and encoding. A field called `src_ip` in a firewall log may carry the same semantic meaning as `source.address` in an OpenTelemetry event and `RemoteAddr` in a Windows Event Log — but they are lexically distinct and cannot be treated as equivalent without normalization.

If the Privacy Filtering Engine and Allow-List Policy Engine are applied to raw, un-normalized telemetry, the allow-list must enumerate every source-specific field name for every telemetry source — a combinatorially large and continuously growing maintenance burden. Worse, source-specific allow-lists may be inconsistently maintained, creating gaps where semantically identical data is permitted from one source and blocked from another, or vice versa.

> [!IMPORTANT]
> A privacy filtering engine that operates on a non-uniform data structure cannot apply policy consistently. Normalization is a prerequisite for reliable, auditable privacy enforcement.

### Decision

Implement a **Telemetry Normalization Engine** as a Tier 1 component that processes all incoming telemetry *before* it enters the Privacy Filtering Engine or Allow-List Policy Engine. The normalization engine:

- Applies **source-specific adapters** (one per supported telemetry source type) that translate raw source events into a **Common Internal Event Schema (CIES)**
- The CIES is a fixed, versioned schema with well-defined field names, types, and semantics
- After normalization, the Privacy Filtering Engine operates exclusively on CIES-formatted events; it has no knowledge of source-specific schemas
- The allow-list (see [ADR-002](#adr-002-strict-allow-list-as-primary-privacy-mechanism)) is defined in terms of CIES field names, not source-specific field names; a single allow-list entry covers all sources that map to the same CIES field
- Schema evolution (adding or modifying CIES fields) is a governed process requiring allow-list review

The normalization pipeline processes events in the following order: **Collection → Normalization (CIES) → Privacy Filtering → Allow-List Validation → Local Anomaly Detection → Buffer → Transmission.**

### Alternatives Considered

| Alternative | Description | Reason Not Selected |
|-------------|-------------|---------------------|
| **Source-Specific Privacy Filters** | Maintain a separate privacy filter and allow-list for each telemetry source | Combinatorial maintenance burden; inconsistent coverage; high risk of gaps in privacy enforcement across sources |
| **Raw Format Transmission** | Pass raw telemetry directly to Tier 2 and normalize there | Raw telemetry must not cross the privacy boundary; this is a direct violation of ADR-001 and ADR-002 |
| **Schema-on-Read** | Store raw events locally and normalize only at query time | Privacy filtering cannot be applied without normalization; schema-on-read requires raw storage which delays privacy enforcement |

### Advantages

- **Consistent privacy enforcement:** A single allow-list defined in CIES terms covers all telemetry sources uniformly
- **Reduced allow-list maintenance:** New telemetry sources require only a new adapter; the allow-list itself does not need to change if the new source maps to existing CIES fields
- **Predictable privacy pipeline:** The Privacy Filtering Engine operates on a fixed, known schema, simplifying testing and validation
- **Foundation for cross-layer correlation:** Normalized events share common identifier fields, enabling the Cross-Layer Correlation Engine (see [ADR-006](#adr-006-cross-layer-correlation-over-single-layer-detection)) to operate on consistent data

### Trade-offs

- Each new telemetry source requires a dedicated source adapter; adapter development is a recurring engineering cost as the platform supports additional sources
- Schema evolution of the CIES must be carefully managed: adding a new field requires allow-list review; renaming a field is a breaking change that requires adapter updates
- Normalization introduces a processing step in the Tier 1 pipeline, consuming additional CPU and memory resources
- Adapter bugs or schema mapping errors may cause data to be incorrectly normalized, potentially misclassifying fields — a normalization error that promotes a sensitive field to an approved CIES field could create a privacy exposure

### Consequences

- The Common Internal Event Schema (CIES) is a core design artifact that must be formally specified, versioned, and published as part of the NEXORA design
- CIES evolution is a security-sensitive change process; schema changes must be reviewed alongside allow-list changes
- All Tier 1 components downstream of normalization (Privacy Filtering, Allow-List, Local Anomaly Detection, Buffering, Egress Proof) operate on CIES exclusively
- Adapter quality and correctness are a first-class concern; adapter testing must verify correct field mapping for all source types
- See [ARCHITECTURE.md](ARCHITECTURE.md) for the Telemetry Normalization Engine component design and [METHODOLOGY.md](METHODOLOGY.md) for the schema governance process

---

*Document end. All decisions in this document are proposed design choices for the NEXORA AIOps observability platform. No ADR in this document reflects implemented behavior or experimentally validated results.*

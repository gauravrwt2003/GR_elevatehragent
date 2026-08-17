# Business Requirements Document (BRD) - HR Agentic Solution (MVP 1 - Revised)

## Executive Summary

The **HR Agentic Solution** is a secure, enterprise AI-driven virtual assistant designed to provide employees with immediate, conversational access to HR services. By orchestrating workflows across core enterprise systems (WorkWeek HCM, ServiceImmediately ITSM/HRSD) and retrieving answers from approved HR policies, the assistant automates routine inquiries and facilitates self-service transactions.

To ensure enterprise governance and zero-trust security during the MVP 1 phase, the solution deploys an **API Gateway Mock Identity Assertion Layer** (signed JWT session claims) that binds conversational turns to authenticated employee profiles. The system enforces traceably bounded execution, authenticates automation origins for all backend operations, and dynamically validates interactions with probabilistic safety benchmarks to protect against prompt injection and data leaks.

---

## 1. Project Objectives

* **Deflect Tier 1 Inquiries (40% Target):** Reduce routine HR and IT helpdesk ticket volume by at least 40% within the first six months.
  * *Baseline Assumption:* 5,000 Tier-1 inquiries/month (60% IT Helpdesk, 40% HR Inquiries).
  * *Deflection Definition:* A session is counted as deflected if the user resolves their inquiry via grounded policy Q&A (without opening a ticket within 24 hours) or successfully completes a self-service transaction without human escalation.
  * *Target Volume:* Deflect at least 2,000 inquiries/month by Month 6.
* **Streamline HR & IT Transactions:** Enable employees to perform core self-service actions (leave submission in work days, incident ticket status checks, and ticket creation) conversationally with mandatory Human-in-the-Loop (HITL) confirmation cards.
* **Validate Cross-System Orchestration:** Demonstrate resilient chaining across HR policies, WorkWeek, and ServiceImmediately with forward error recovery and Dead-Letter Queue (DLQ) tracking before wider enterprise rollout.
* **Ensure Enterprise AI Governance:** Maintain 100% audit visibility over tool invocations, deployment versions, and authorized capability boundaries.
* **Mitigate AI Risks:** Achieve high-precision safety enforcement (&gt;= 99.5% precision/recall on safety benchmarks) and 0% ungrounded hallucinations on approved benchmark test suites via deterministic retrieval guardrails and strict PII redaction.

---

## 2. Project Scope (MVP 1)

### 2.1. Functional Scope (In-Scope for MVP 1)

* **Conversational User Interface (UI):** Web-based chat client equipped with a test persona switcher injecting signed mock identity assertions (`X-Authenticated-User`), streaming response tokens, and interactive confirmation cards.
* **Policy & Informational Queries (RAG):** Answering employee inquiries derived strictly from ingested, approved static documents with exact deep-link citations.
* **Employee Self-Service Transactions (WorkWeek HCM):**
  * *Read Actions:* Real-time retrieval of employee profile details and accrued/used/remaining leave balances (in work days).
  * *Write Actions:* Updating personal home address and phone number; submitting new leave requests with business calendar validation.
* **Support Desk Management (ServiceImmediately ITSM/HRSD):**
  * *Read Actions:* Querying incident ticket status, priority, category, assignee, and public comment history.
  * *Write Actions:* Creating new incident tickets (P1-P4), posting follow-up comments to active tickets, and canceling tickets in 'New' status.
* **Resilient Cross-System Orchestration:** Multi-step chaining (e.g., verifying remote work eligibility, checking WorkWeek profile, and generating hardware procurement tickets in ServiceImmediately) with explicit user confirmation and DLQ failure handling.

### 2.2. Data Scope (In-Scope for MVP 1)

* **Static HR Policies:** Curated repository of approved documents (PDF/Text) covering Leave Policies, Expense Guidelines, Remote Work Guidelines, and Code of Conduct.
* **WorkWeek Data Elements:**
  * *Employee Profile:* Employee ID, Full Name, Email, Department, Role, Manager ID, Hire Date.
  * *Contact Info:* Personal Address (street, city, state, postal code), Personal Phone Number (E.164).
  * *Leave Data:* Accrued, Used, and Remaining balances for "Vacation" and "Sick" categories (tracked in work days).
* **ServiceImmediately Data Elements:**
  * *Incident Record:* Ticket ID, Short Description, Detailed Description, Category, Priority (1-Critical, 2-High, 3-Moderate, 4-Low), State (New, In Progress, On Hold, Resolved, Closed, Canceled), Assignee, Comment Timeline.

### 2.3. Out of Scope for MVP 1

* Integration with production Identity Providers (Okta/Azure AD/Ping) — simulated via signed Gateway assertions in MVP 1.
* Direct processing of core payroll adjustments, formal compensation planning, or performance appraisal records.
* Automated badge/facility physical access provisioning and email routing automation (UC-2.3 Relocation is deferred to Phase 2).
* Multi-lingual natural language translation.
* Voice/telephony IVR integrations.

---

## 3. MVP 1 Use Cases and Interaction Specifications

| Use Case ID | Category | Triggering User Prompt (Example) | Systems Involved | System Actions & Expected Behavior |
| :--- | :--- | :--- | :--- | :--- |
| **UC-1.1** | Policy Q&A | *"What is the company's bereavement leave policy?"* or *"Can remote employees expense monitors?"* | Policy Repository | Retrieve relevant policy chunks; generate grounded response with clickable citation deep links. Refuse out-of-domain queries. |
| **UC-1.2** | HR Self-Service | *"How many days of PTO do I currently have accrued?"* or *"Submit a vacation request for this Thursday and Friday."* | WorkWeek (HCM) | 1. Query real-time leave balance in work days.<br>2. Calculate business days (excluding holidays/weekends).<br>3. Present User Confirmation Card.<br>4. Submit request to WorkWeek in 'Submitted for Approval' state. |
| **UC-1.3** | IT Incident Mgmt | *"What is the status of ticket INC102938?"* or *"Create an IT ticket for intermittent VPN disconnections."* | ServiceImmediately (ITSM) | 1. Query ticket status/timeline.<br>2. Check for duplicate tickets submitted within 15 minutes.<br>3. Create incident ticket with mapped category/priority. |
| **UC-2.1** | Cross-System: Equipment Procurement | *"I am a remote employee. Can you check my eligibility and request a home office monitor for me?"* | Policy Docs, WorkWeek, ServiceImmediately | 1. Verify remote policy stipend.<br>2. Check employee location/status in WorkWeek.<br>3. Render Confirmation Card with item specs and shipping address.<br>4. Upon user confirmation, create procurement request ticket in ServiceImmediately. |
| **UC-2.2** | Cross-System: Medical Leave | *"I need to take short-term medical leave starting next Monday. What is the process and can you submit it?"* | Policy Docs, WorkWeek, ServiceImmediately | 1. Quote medical leave procedure and required documentation.<br>2. Present confirmation card and submit Leave of Absence in WorkWeek.<br>3. Attempt to open notification ticket in ServiceImmediately.<br>4. *Resilience:* If ticket creation fails, retain WorkWeek leave, emit an alert to the HR Ops DLQ, and provide user with the WorkWeek confirmation reference ID. |
| **UC-2.3** | Cross-System: Relocation | *Deferred to Phase 2* | — | *Deferred to Phase 2 due to physical access badge and payroll relocation dependencies.* |

---

## 4. Functional Requirements

### 4.1. AI Governance & Security Infrastructure

| Requirement ID | Requirement Name | Description & Specification |
| :--- | :--- | :--- |
| **FR-1.1** | **Capability & Tool Governance** | Enforce strict registry boundaries on authorized tools (WorkWeek adapter, ServiceImmediately adapter, Vector Search). Block unauthorized system or network invocations. |
| **FR-1.2** | **Verification of Request Origin** | Audit records must capture the verified caller identity (`employee_id`, `role`) alongside the automation execution ID. |
| **FR-1.3** | **Conversation Safety & Guardrails** | **Input Guardrails:** Intercept prompt injections, jailbreaks, and off-domain queries before execution (&lt; 300ms overhead).<br>**Output Guardrails:** Filter hallucinated policy statements, toxic text, and unredacted sensitive data. |
| **FR-1.4** | **PII / SPII Redaction** | Automatically detect and mask Sensitive Personally Identifiable Information (SSNs, home addresses, phone numbers, banking details) from persistent logs. |
| **FR-1.5** | **Orchestration-Level RBAC** | Enforce RBAC at the Agent Orchestration layer using verified Gateway identity claims. Ensure users can only query/mutate their own profile and ticket records. |

### 4.2. Core Capabilities & Dialog Management

| Requirement ID | Requirement Name | Description & Specification |
| :--- | :--- | :--- |
| **FR-2.1** | **Natural Language Understanding** | Accurately parse intents across synonyms, typos, and conversational phrasing with contextual disambiguation. |
| **FR-2.2** | **Session State & Memory** | Maintain multi-turn conversational context within an active session. Automatically purge session context upon timeout (30 minutes of inactivity) or explicit user `/reset`. |
| **FR-2.3** | **Human-in-the-Loop Confirmation** | Present an interactive summary confirmation card before executing any state-mutating transaction (leave booking, address edit, ticket creation). |

### 4.3. WorkWeek Integration (HCM)

| Requirement ID | Requirement Name | Description & Operations |
| :--- | :--- | :--- |
| **FR-3.1** | **Gateway Delegated Authorization** | Chat gateway injects a signed JWT with `employee_id` and `role`. WorkWeek adapter scopes all read/write operations strictly to the asserted caller. |
| **FR-3.2** | **Core HCM Operations** | • **Profile Query:** Retrieve name, email, department, role, manager, hire date, home address, and phone number.<br>• **Contact Update:** Update personal address and phone number following schema validation.<br>• **Leave Balance Query:** Retrieve accrued, used, and remaining balances in **work days** for Vacation and Sick categories.<br>• **Submit Leave Request:** Submit time off specifying start date, end date, leave category, and calculated work days. Default status is `Submitted for Manager Approval`. |
| **FR-3.3** | **Validation Guardrails** | • **Balance Constraint:** Reject requests exceeding remaining accrued days.<br>• **Business Calendar Validation:** Exclude weekend days and declared company holidays from requested durations.<br>• **Temporal Consistency:** Reject past dates or start dates after end dates.<br>• **Format Validation:** Validate E.164 phone formats and postal codes. |
| **FR-3.4** | **Real-Time Fetch & Circuit Breaker** | Fetch dynamic balances in real time on every query (no AI-layer caching). Apply a 5-second timeout with exponential backoff (max 2 retries). If WorkWeek is offline, return a graceful degradation notice while keeping Policy Q&A operational. |

### 4.4. ServiceImmediately Integration (ITSM/HRSD)

| Requirement ID | Requirement Name | Description & Operations |
| :--- | :--- | :--- |
| **FR-4.1** | **Auditable Ticket Management** | Log all ticket creations with caller identity, automation agent ID, and idempotency keys to prevent duplicate creation. |
| **FR-4.2** | **Ticket Lifecycle Operations** | • **Query Ticket:** Retrieve status, category, priority, assignee, short description, and comment timeline.<br>• **Create Incident:** Open new incident specifying category, short description, detailed description, and priority (P1-P4).<br>• **Add Comment:** Append user update notes to the ticket activity timeline.<br>• **Cancel Ticket:** Allow end-users to transition tickets to 'Canceled' only if ticket is currently in 'New' state. Status updates to 'Resolved' or 'Closed' are restricted to technician roles. |
| **FR-4.3** | **Deduplication & Quality Guardrails** | • **Deduplication:** Scan tickets created within the past 15 minutes by the same caller. If semantic similarity on short description &gt; 0.85, prompt user to confirm or append to existing ticket.<br>• **Priority Validation:** Verify P1/Critical criteria before assigning top-tier priority tags. |

### 4.5. Policy Document Q&A (RAG)

| Requirement ID | Requirement Name | Description & Specification |
| :--- | :--- | :--- |
| **FR-5.1** | **Document Ingestion & Indexing** | Ingest, chunk, and generate embeddings for approved HR policy PDFs/documents. |
| **FR-5.2** | **Strict Grounding** | Generate answers derived strictly from retrieved context chunks. If document evidence is insufficient, explicitly state inability to answer. |
| **FR-5.3** | **Clickable Source Citations** | Return metadata with every answer, including document title, section heading, and verified deep link URL. |
| **FR-5.4** | **Domain Containment** | Intercept and refuse non-HR/non-IT questions (e.g., coding, general knowledge, personal advice). |
| **FR-5.5** | **Knowledge Sync SLA** | Reflect document updates in the Knowledge Base within **&lt;= 1 hour** via webhook/CDC triggers, supported by a manual admin sync endpoint and a daily 24-hour batch refresh. |

---

## 5. Non-Functional Requirements (NFR)

### 5.1. Security, Privacy & Compliance

| Requirement ID | Requirement Name | Specification |
| :--- | :--- | :--- |
| **NFR-1.1** | **Interaction Safety** | Filter malicious prompt injections, indirect RAG injections, and harmful outputs (&gt;= 99.5% benchmark accuracy). |
| **NFR-1.2** | **Immutable Audit Logging** | Log all system interactions, tool calls, and blocked safety events with timestamp, caller ID, and action metadata. |
| **NFR-1.3** | **Data Protection & Compliance** | Mask all SPII/PII in log stores; enforce TLS 1.3 in transit and AES-256 at rest. Support GDPR Right to be Forgotten purge scripts. |

### 5.2. Performance & Scalability

| Requirement ID | Requirement Name | Target Latency & Throughput Targets |
| :--- | :--- | :--- |
| **NFR-2.1** | **Conversational Latency SLAs** | • **Policy Q&A (RAG):** p50 &lt; 1.5s, p95 &lt; 3.0s.<br>• **Single-System Transactions:** p50 &lt; 2.0s, p95 &lt; 4.0s.<br>• **Chained Orchestration (UC-2.x):** p50 &lt; 4.0s, p95 &lt; 7.0s with token streaming.<br>• **Safety Scanning Overhead:** &lt; 300ms per conversational turn. |
| **NFR-2.2** | **Availability & Capacity** | 99.9% uptime SLA; designed to support 50 peak concurrent user sessions and 5,000 monthly transactions in MVP 1. |
| **NFR-2.3** | **Asynchronous Tool Execution** | Backend adapter calls and external lookups execute asynchronously without blocking conversational streaming. |

### 5.3. Quality & Grounding

| Requirement ID | Requirement Name | Benchmark Target |
| :--- | :--- | :--- |
| **NFR-3.1** | **Answer Accuracy & Grounding** | Achieve &gt;= 95% accuracy on predefined HR policy evaluation sets with 0% ungrounded hallucinations. |

### 5.4. Resilience & Error Handling

| Requirement ID | Requirement Name | Specification |
| :--- | :--- | :--- |
| **NFR-4.1** | **Graceful Degradation** | Display friendly, actionable user messages on backend failures without exposing internal stack traces or API keys. |
| **NFR-4.2** | **Idempotent Retries** | Implement exponential backoff with jitter on network timeouts. Pass `Idempotency-Key` headers on all state-mutating requests. |
| **NFR-4.3** | **Forward SAGA Recovery (DLQ)** | In multi-system flows, if a subsequent step fails, retain prior successful steps, emit an event to the HR Operations Dead-Letter Queue (DLQ), and provide the user with the successful transaction reference ID. |

---

## 6. MVP 1 Implementation Architecture & Constraints

* **Gateway Mock Identity Assertion Layer:** To satisfy zero-trust RBAC and origin verification without blocking on enterprise Okta/SSO integration, the Chat UI includes a test persona selector that issues signed JWT assertion headers (`X-Authenticated-User`) to the API Gateway. The Gateway verifies the signature, and the Agent Orchestrator enforces caller data boundaries.
* **Backend Credentials:** Backend adapters connect to WorkWeek and ServiceImmediately sandbox instances using dedicated service accounts scoped by the API Gateway's asserted caller context.
* **Tenancy Scope:** Deployed as a single-tenant instance for MVP 1 evaluation.

---

## 7. Success and Evaluation Criteria

| Evaluation Category | Success Metric / Criterion | Target Benchmark |
| :--- | :--- | :--- |
| **Deflection Rate** | Tier-1 tickets deflected via self-service and policy answers | 40% reduction (2,000 tickets/month deflected by Month 6) |
| **Policy Q&A Quality** | Grounded precision, recall, and citation validity | &gt;= 95% Accuracy on test benchmark; 0% Hallucination |
| **Transaction Integrity** | Correct execution of leave submissions and ticket operations | 100% Correctness; 0 duplicate transactions |
| **Cross-System Chaining** | Successful execution of UC-2.1 and UC-2.2 with DLQ fallback | 100% pass on test suite scenarios |
| **Safety & Guardrails** | Interception of jailbreaks, injections, and toxic prompts | &gt;= 99.5% Detection rate; &lt; 1% False Positive rate |
| **Response Latency** | End-to-end turnaround across RAG and tool calls | Policy Q&A p95 &lt; 3.0s; Chained workflows p95 &lt; 7.0s |
| **Audit Traceability** | Log coverage of all system actions and blocked requests | 100% Log Coverage with caller identity binding |

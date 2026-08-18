# HR Agentic Solution: Architecture & Workflow Diagrams (MVP 1)

This document contains the complete visual architectural specifications, sequence flows, state machines, and security pipelines for the **HR Agentic Solution (MVP 1)** using Mermaid diagrams.

---

## 1. High-Level System Architecture

This diagram details the end-to-end component topology, including the **Gateway Mock Identity Assertion Layer**, **Interaction Safety Interceptors**, **Agent Orchestration Engine**, **System Adapters**, and the **Dead-Letter Queue (DLQ)**.

```mermaid
flowchart TD
    subgraph Client_Layer ["1. Client Layer"]
        User(["Employee / Client Browser"])
        PersonaSelector["Test Persona Selector\n(Jane Doe / John Smith / Alice Brown)"]
        ChatUI["Web Chat UI\n(Token Streaming & HITL Cards)"]
        User <--> PersonaSelector
        User <--> ChatUI
    end

    subgraph Gateway_Layer ["2. API Gateway & Auth Proxy"]
        APIGateway["API Gateway\n(Rate Limiting & Session Router)"]
        AuthProxy["Mock Identity Assertion Service\n(Issues Signed JWT)"]
        ChatUI -->|HTTPS Request| APIGateway
        PersonaSelector -.->|Selects Identity| AuthProxy
        AuthProxy -->|Injects X-Authenticated-User| APIGateway
    end

    subgraph Security_Layer ["3. Interaction Safety & Governance"]
        InputGuard["Input Safety Guardrail\n(Prompt Injection & Jailbreak Filter)"]
        OutputGuard["Output Safety Guardrail\n(Hallucination Filter & PII Redactor)"]
        APIGateway -->|1. Raw Prompt + JWT| InputGuard
    end

    subgraph Core_Agent_Platform ["4. Agent Orchestration Platform"]
        SessionMgr["Session & Context Manager\n(Redis: 30m TTL / Reset)"]
        Orchestrator["Agent Orchestration Engine\n(Intent Parser & Tool Router)"]
        RBACFilter["Orchestration-Level RBAC\n(Enforces caller_id == target_id)"]
        
        InputGuard -->|2. Clean Prompt| Orchestrator
        Orchestrator <--> SessionMgr
        Orchestrator --> RBACFilter
    end

    subgraph Integration_Adapters ["5. System Integration Adapters"]
        WW_Adapter["WorkWeek HCM Adapter\n(Idempotent REST / Circuit Breaker)"]
        SI_Adapter["ServiceImmediately Adapter\n(Deduplication & Lifecycle Engine)"]
        VectorEngine["Policy RAG Engine\n(Semantic Search & Citations)"]
        
        RBACFilter -->|Authorized Tool Call| WW_Adapter
        RBACFilter -->|Authorized Tool Call| SI_Adapter
        RBACFilter -->|Authorized Tool Call| VectorEngine
    end

    subgraph Enterprise_Backends ["6. External Enterprise Services"]
        WorkWeek[("WorkWeek Sandbox\n(HCM & Leave Balances)")]
        ServiceImmediately[("ServiceImmediately Sandbox\n(ITSM & HRSD Incidents)")]
        VectorDB[("Policy Vector Database\n(Embeddings & Metadata)")]
        DocRepo[("HR Policy Document Repo\n(PDF / DOCX Guidelines)")]
        
        WW_Adapter <-->|Service Credential + Scoped Token| WorkWeek
        SI_Adapter <-->|Service Credential + Caller Context| ServiceImmediately
        VectorEngine <-->|Cosine Search (k=4)| VectorDB
        DocRepo -.->|Ingestion Webhook <= 1hr| VectorDB
    end

    subgraph Governance_Observability ["7. Governance, Audit & Dead-Letter Queue"]
        AuditStore[("Immutable Audit Database\n(PII-Masked Access Logs)")]
        DLQ[("HR Ops Dead-Letter Queue\n(Partial Failure Event Bus)")]
        
        Orchestrator -->|Async Audit Log| AuditStore
        Orchestrator -.->|On Chained Step Failure| DLQ
        Orchestrator -->|3. Generated Response| OutputGuard
        OutputGuard -->|4. Safe Stream| ChatUI
    end

    style InputGuard fill:#fff3cd,stroke:#ffc107,stroke-width:2px;
    style OutputGuard fill:#fff3cd,stroke:#ffc107,stroke-width:2px;
    style RBACFilter fill:#e8f5e9,stroke:#4caf50,stroke-width:2px;
    style DLQ fill:#ffebee,stroke:#f44336,stroke-width:2px;
    style WorkWeek fill:#e1f5fe,stroke:#03a9f4,stroke-width:2px;
    style ServiceImmediately fill:#e1f5fe,stroke:#03a9f4,stroke-width:2px;
```

---

## 2. Sequence Diagrams: Core User Workflows

### 2.1. Grounded Policy Q&A Flow (RAG with Citations)

Illustrates strict domain containment, similarity score gating ($\ge 0.70$), and deep-link citation generation.

```mermaid
sequenceDiagram
    autonumber
    actor Employee as Employee (Jane Doe)
    participant UI as Chat UI
    participant Gateway as API Gateway
    participant Safety as Safety Interceptor
    participant Agent as Agent Orchestrator
    participant RAG as Policy Vector DB

    Employee->>UI: "What is the bereavement leave policy?"
    UI->>Gateway: POST /chat/message (Headers: X-Authenticated-User JWT)
    Gateway->>Safety: Scan Input Prompt (<300ms)
    Safety-->>Gateway: Prompt Status: Clean & In-Domain
    Gateway->>Agent: Dispatch Query(Prompt, Context)
    Agent->>RAG: Vector Search (Query: "bereavement leave policy", k=4)
    RAG-->>Agent: Retrieved Chunks (Score: 0.88) + Metadata (Doc: Leave_Policy.pdf, Sec: 3.4)
    Agent->>Agent: Ground Response strictly on Chunks
    Agent->>Safety: Scan Output Response
    Safety-->>Agent: Output Clean (0% Hallucination, PII Scubbed)
    Agent-->>Gateway: Stream Response Tokens + Citation Metadata
    Gateway-->>UI: Forward Stream
    UI-->>Employee: Display Answer with Clickable Citation Chip: "[Leave Policy 2026, Sec 3.4]"
```

---

### 2.2. Transactional Leave Booking with HITL Confirmation Flow

Illustrates real-time balance queries, business calendar holiday/weekend exclusion, interactive **Human-in-the-Loop (HITL)** confirmation, and idempotent submission to WorkWeek.

```mermaid
sequenceDiagram
    autonumber
    actor Employee as Employee (Jane Doe - EMP-1042)
    participant UI as Chat UI
    participant Agent as Agent Orchestrator
    participant Calendar as Business Calendar Service
    participant WW as WorkWeek Adapter
    participant CoreWW as WorkWeek Sandbox

    Employee->>UI: "Book vacation from Friday Sep 4 to Tuesday Sep 8"
    UI->>Agent: Dispatch Turn (JWT: EMP-1042)
    Agent->>WW: getLeaveBalances(employee_id="EMP-1042")
    WW->>CoreWW: GET /balances (Real-Time)
    CoreWW-->>WW: { vacation_remaining: 10.0 days }
    WW-->>Agent: 10.0 Work Days Available
    Agent->>Calendar: calculateWorkDays(2026-09-04, 2026-09-08)
    Note over Calendar: Excludes Sat (09-05), Sun (09-06), and Labor Day (09-07)
    Calendar-->>Agent: 2.0 Work Days
    Agent->>Agent: Validate (2.0 <= 10.0 Remaining)
    Agent-->>UI: Render Confirmation Card (Dates: Sep 4-8, Total: 2.0 Work Days, Type: Vacation)
    Employee->>UI: Clicks [Confirm & Submit]
    UI->>Agent: POST /actions/confirm (ActionID: act-8921)
    Agent->>WW: submitLeave(Idempotency-Key, EMP-1042, 2.0 Days, Vacation)
    WW->>CoreWW: POST /leave_requests
    CoreWW-->>WW: HTTP 201 Created (Ref: LV-4819, Status: SUBMITTED_FOR_APPROVAL)
    WW-->>Agent: Success (Ref: LV-4819)
    Agent-->>UI: "Your vacation request for 2.0 days has been submitted for approval (Ref: LV-4819)."
    UI-->>Employee: Render Success Banner with Reference ID
```

---

### 2.3. Resilient Medical Leave Chained Flow with DLQ Forward Recovery (UC-2.2)

Illustrates a multi-system workflow where WorkWeek succeeds, ServiceImmediately fails, and the system performs **Forward Escalation with Dead-Letter Queue (DLQ)** without rolling back the valid HCM state.

```mermaid
sequenceDiagram
    autonumber
    actor Employee as Employee (Jane Doe)
    participant UI as Chat UI
    participant Agent as Agent Orchestrator
    participant WW as WorkWeek Adapter
    participant CoreWW as WorkWeek Sandbox
    participant SI as ServiceImmediately Adapter
    participant CoreSI as ServiceImmediately Sandbox
    participant DLQ as HR Ops DLQ / Event Bus
    participant Audit as Audit Store

    Employee->>UI: "Confirm 5-Day Medical Leave Submission"
    UI->>Agent: Dispatch Execution (ActionID: act-9901)
    
    rect rgb(232, 245, 233)
        Note over Agent,CoreWW: Step 1: WorkWeek Leave of Absence Submission
        Agent->>WW: submitLeaveRequest(Idempotency-Key, Type: LOA, Days: 5.0)
        WW->>CoreWW: POST /leave_requests
        CoreWW-->>WW: HTTP 201 Created (Ref: LOA-8821)
        WW-->>Agent: Step 1 SUCCESS (Ref: LOA-8821)
    end

    rect rgb(255, 235, 238)
        Note over Agent,CoreSI: Step 2: ServiceImmediately IT Notification Ticket Creation
        Agent->>SI: createTicket(Category: AccessRouting, Ref: LOA-8821)
        SI->>CoreSI: POST /incidents (Attempt 1)
        CoreSI--xSI: HTTP 503 Gateway Timeout
        SI->>SI: Exponential Backoff (1.5s)
        SI->>CoreSI: POST /incidents (Attempt 2)
        CoreSI--xSI: HTTP 503 Gateway Timeout
        SI-->>Agent: Step 2 FAILED (HTTP 503 Service Unavailable)
    end

    rect rgb(255, 243, 224)
        Note over Agent,DLQ: Step 3: Forward Resilience & DLQ Alerting
        Agent->>Agent: Preserve WorkWeek State (Do NOT Rollback LOA-8821)
        Agent->>DLQ: Publish Event (PARTIAL_ORCHESTRATION_FAILURE, LOA-8821, EMP-1042)
        Agent->>Audit: Log Transaction Audit Record (State: PARTIAL_SUCCESS)
        Agent-->>UI: "Your Medical Leave has been recorded in WorkWeek (Ref: LOA-8821). However, the IT routing ticket timed out. HR Operations has been automatically alerted."
        UI-->>Employee: Display Transparent Partial-Success Summary
    end
```

---

## 3. State Machines & Lifecycle Diagrams

### 3.1. Incident Ticket Lifecycle State Machine (ServiceImmediately)

Defines allowed state transitions, guardrails, and role boundaries (End-User vs. Technician/Fulfiller).

```mermaid
stateDiagram-v2
    [*] --> New : User creates ticket via Chat (P1-P4)
    
    state New {
        [*] --> Unassigned
        Unassigned --> Assigned : Helpdesk routes ticket
    }

    New --> Canceled : User executes /cancel (Only allowed when state is New)
    New --> In_Progress : Fulfiller starts work
    
    state In_Progress {
        [*] --> Active_Investigation
        Active_Investigation --> Awaiting_User_Info : Technician adds comment
        Awaiting_User_Info --> Active_Investigation : User adds comment
    }

    In_Progress --> On_Hold : Awaiting vendor/hardware
    On_Hold --> In_Progress : Dependency unblocked
    
    In_Progress --> Resolved : Fulfiller solves issue (Restricted to Fulfillers)
    
    state Resolved {
        [*] --> Pending_Closure
        Pending_Closure --> Reopened : User reports issue persists (Within 5 days)
    }

    Reopened --> In_Progress : Fulfiller re-engages
    Resolved --> Closed : Auto-closed after 5 days (Terminal)
    Canceled --> [*]
    Closed --> [*]

    note right of New
        End-User Permissions:
        - Query Details
        - Add Comments
        - Cancel Ticket (Only in 'New' state)
    end note

    note right of Resolved
        Fulfiller Permissions:
        - Transition to Resolved
        - Transition to Closed
        - Change Priority/Category
    end note
```

---

### 3.2. Leave Request Lifecycle State Machine (WorkWeek)

Illustrates the conversational booking flow from balance validation to manager approval.

```mermaid
stateDiagram-v2
    [*] --> Initial_Prompt : User requests time off
    
    Initial_Prompt --> Balance_Validation : Check accrued days in WorkWeek
    
    state Balance_Validation {
        [*] --> Query_WorkWeek
        Query_WorkWeek --> Balance_Sufficient : Balance >= Requested
        Query_WorkWeek --> Balance_Insufficient : Balance < Requested
    }

    Balance_Insufficient --> Rejected_By_Guardrail : Display overdraft error
    Rejected_By_Guardrail --> [*]

    Balance_Sufficient --> Calendar_Validation : Calculate business days
    
    state Calendar_Validation {
        [*] --> Exclude_Weekends
        Exclude_Weekends --> Exclude_Company_Holidays
        Exclude_Company_Holidays --> Net_Work_Days_Calculated
    }

    Calendar_Validation --> Pending_User_Confirmation : Render HITL Confirmation Card
    
    state Pending_User_Confirmation {
        [*] --> Card_Rendered
        Card_Rendered --> User_Confirmed : Clicks [Confirm & Submit]
        Card_Rendered --> User_Canceled : Clicks [Cancel]
    }

    User_Canceled --> Canceled_By_User : Purge pending action
    Canceled_By_User --> [*]

    User_Confirmed --> Submitted_For_Approval : Idempotent POST to WorkWeek
    
    state Submitted_For_Approval {
        [*] --> Manager_Review_Queue
        Manager_Review_Queue --> Approved : Manager signs off
        Manager_Review_Queue --> Denied : Manager rejects
    }

    Approved --> Deducted_From_Balance : Balance officially updated
    Denied --> Balance_Restored : Balance holds released
    Deducted_From_Balance --> [*]
    Balance_Restored --> [*]
```

---

## 4. Security & Safety Processing Pipeline

Illustrates how an incoming conversational turn is sanitized, authorized, and scrubbed for Sensitive Personally Identifiable Information (SPII).

```mermaid
flowchart LR
    subgraph Step1 ["1. Input Ingestion"]
        InPrompt["User Prompt\n& Identity Header"]
    end

    subgraph Step2 ["2. Input Guardrails (<300ms)"]
        direction TB
        InjCheck{"Prompt Injection\nor Jailbreak?"}
        DomainCheck{"In-Domain HR/IT\nIntent?"}
        BlockInj["Block & Log\nSAFETY_INTERCEPT_INPUT"]
        BlockDomain["Refuse Non-HR/IT\nQuery"]
        
        InPrompt --> InjCheck
        InjCheck -- Yes --> BlockInj
        InjCheck -- No --> DomainCheck
        DomainCheck -- No --> BlockDomain
    end

    subgraph Step3 ["3. Orchestration RBAC"]
        direction TB
        RBACCheck{"caller_id ==\ntarget_resource_id?"}
        BlockRBAC["Block Unauthorized\nCross-User Access"]
        ExecuteTool["Dispatch Tool Execution\nto Adapter Layer"]
        
        DomainCheck -- Yes --> RBACCheck
        RBACCheck -- No --> BlockRBAC
        RBACCheck -- Yes --> ExecuteTool
    end

    subgraph Step4 ["4. Output Guardrails"]
        direction TB
        HallucCheck{"Grounded in\nRetrieved Context?"}
        PIIScrub["Scrub SPII:\n- SSNs\n- Addresses\n- Phone numbers\n- Health notes"]
        RefuseHalluc["Refuse to Guess\nState Insufficient Info"]
        
        ExecuteTool --> HallucCheck
        HallucCheck -- No --> RefuseHalluc
        HallucCheck -- Yes --> PIIScrub
    end

    subgraph Step5 ["5. Delivery & Logging"]
        direction TB
        StreamUI["Stream Clean Tokens\nto Chat UI"]
        AuditLog[("Write Immutable Log\n(PII Redacted)")]
        
        PIIScrub --> StreamUI
        PIIScrub --> AuditLog
    end

    style BlockInj fill:#ffebee,stroke:#f44336;
    style BlockDomain fill:#fff3e0,stroke:#ff9800;
    style BlockRBAC fill:#ffebee,stroke:#f44336;
    style RefuseHalluc fill:#fff3e0,stroke:#ff9800;
    style StreamUI fill:#e8f5e9,stroke:#4caf50;
```

---

## 5. Knowledge Base Sync & Ingestion Pipeline (RAG)

Illustrates how static HR documents are chunked, embedded, and kept synchronized within the $\le 1\text{ hour}$ SLA.

```mermaid
flowchart TD
    subgraph Source_Repo ["1. HR Policy Document Source"]
        Repo[("Central Document Store\n(PDF / DOCX / Markdown)")]
        Admin["HR Admin / Policy Owner"]
        Admin -->|Uploads / Edits Document| Repo
    end

    subgraph Trigger_Layer ["2. Ingestion Triggers"]
        Webhook["CDC Event / Webhook\n(Instant Trigger)"]
        ManualSync["Admin Endpoint\nPOST /rag/sync"]
        CronSync["Nightly Batch Cron\n(02:00 UTC)"]
        
        Repo -->|File Change Detected| Webhook
        Admin -.->|Manual Override| ManualSync
    end

    subgraph Processing_Pipeline ["3. Chunking & Embedding Engine"]
        Parser["Document Parser\n(Extracts Headings, Tables & Text)"]
        Chunker["Semantic Chunker\n(512 Tokens, 50-Token Overlap)"]
        Embedder["Embedding Generator\n(Generates Dense Vectors)"]
        
        Webhook --> Parser
        ManualSync --> Parser
        CronSync --> Parser
        Parser --> Chunker
        Chunker --> Embedder
    end

    subgraph Vector_Storage ["4. Policy Knowledge Base"]
        VectorIndex[("Vector DB Index\n(Embeddings & Metadata)")]
        OldIndex["Stale Chunks Invalidation\n(Hash-Based Purge)"]
        
        Embedder --> OldIndex
        OldIndex --> VectorIndex
    end

    subgraph Query_Service ["5. Retrieval Layer"]
        SearchAPI["Similarity Search API\n(Cosine Distance, k=4, Threshold >= 0.70)"]
        VectorIndex <--> SearchAPI
    end

    style Webhook fill:#e1f5fe,stroke:#03a9f4;
    style VectorIndex fill:#e8f5e9,stroke:#4caf50;
    style OldIndex fill:#fff3e0,stroke:#ff9800;
```

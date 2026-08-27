# Lead Passing Flow Diagram

A single end-to-end view of how a Lead moves from HubSpot through to a Qualified Engagement
Record (or a manual save). This is a Mermaid diagram — GitHub renders it natively in both
regular markdown files and the wiki, no export needed.

For the detail behind each node, see the linked pages: `Legacy-HS2SF-Function-App`,
`Celigo-HubSpot-to-Salesforce-Sync`, `LPA-Trigger-Flows`, `LPA-Overview`,
`PassInboundLead-Flow`, `Manual-Lead-Passing-Process`.

```mermaid
flowchart TD
    A[HubSpot: form submission] --> B[HubSpot to Salesforce sync<br/>legacy Azure Function, migrating to Celigo]
    B --> C[Salesforce: Lead created]
    C --> D{Owned by<br/>Referral queue?}
    D -->|No| E[LeadPasserLauncher flow<br/>fires on Lead Create]
    D -->|Yes| D2[Excluded from<br/>auto-enrollment]
    E --> F[Apex action:<br/>N8nNotificationServiceLeadPasser]
    F --> G[n8n webhook: lpa]

    subgraph LPA["LPA Production (n8n)"]
        G --> H[LPA Production - RR Rollout<br/>live workflow]
        H -. fire-and-forget shadow .-> I[LPA Production - Refactor<br/>shadow/test only, no SF writes]
        H --> J[Multi-agent qualify, relationship,<br/>and routing decision]
        J --> K[Round-robin assignment<br/>Google Sheet: Lead Passing - Round Robin]
    end

    K --> L[Calls Salesforce flow<br/>PassInboundLead, with Action + IDs]
    L --> M[PassInboundLead flow<br/>some owner routing hardcoded to named reps]
    M --> N{Outcome}
    N -->|Success| O[Lead qualified<br/>Engagement Record owned]
    N -->|Fails or skipped| P[Sits in Inbound / Automation<br/>Review queue]
    P --> Q[Human, e.g. Nickole,<br/>fixes underlying data]
    Q --> R{Retry path}
    R -->|Toggle checkbox| S[LeadPasserLauncherCheckbox flow<br/>fires on checkbox change]
    S --> F
    R -->|Still stuck| T[Manually invoke<br/>PassInboundLead directly]
    T --> M

    classDef sf fill:#EEEDFE,stroke:#534AB7,color:#26215C;
    classDef n8n fill:#E1F5EE,stroke:#0F6E56,color:#04342C;
    classDef manual fill:#FAEEDA,stroke:#854F0B,color:#412402;
    classDef success fill:#EAF3DE,stroke:#3B6D11,color:#173404;
    classDef neutral fill:#F1EFE8,stroke:#5F5E5A,color:#2C2C2A;

    class A,D,D2,N,R neutral;
    class B,G,H,I,J,K,L n8n;
    class C,E,F,M,S sf;
    class P,Q,T manual;
    class O success;
```

**Reading the colors:** purple = Salesforce-native (Leads, Flows, the Apex action); teal =
the automation/integration layer (the sync app and n8n); amber = a human doing manual work;
green = the successful outcome; gray = decision points and the start/excluded states.

**Two things this diagram simplifies, spelled out in prose:**
- The "still stuck" manual retry path assumes the person working the queue has already done
  the account/contact cleanup described in `Manual-Lead-Passing-Process` — the diagram shows
  the retry mechanics, not that underlying data-fixing work.
- `PassInboundLead`'s hardcoded named-rep routing (see `PassInboundLead-Flow`) isn't broken out
  as its own branch here — it's a property of the single "PassInboundLead flow" node, not a
  separate path a Lead takes.

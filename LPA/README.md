# Inbound Lead Passing (HubSpot → Salesforce → LPA)

> Status: **In takeover / active documentation.** This README is the map. Deep-dive detail for
> each component lives in the project Wiki (see links below). If something here looks stale,
> check the Wiki page's own status line before trusting it.

## What this system does

An interested party submits a form (currently: **HubSpot** — this is the path in scope for now;
other intake channels exist but are out of scope for this documentation pass). From there, the
lead is created in Salesforce and routed to the right owner — either automatically or with human
help — with as little manual touch as possible.

```
Prospect fills out a HubSpot form
        │
        ▼
HubSpot → Salesforce Lead sync   ("the function app" — see Legacy HS2SF Function App)
  • TODAY: legacy Azure Function ("hs2sf" / cp-hubspot-to-salesforce v3.1.0) — source code
    obtained and documented (see Wiki), though the copy we have is confirmed out of date
    vs. whatever's actually deployed. Read-only reference; we don't own/edit this code.
  • REPLACING IT: Celigo middleware flow (documented — see Wiki). Not yet at full field
    parity with the legacy app — see Open Questions.
        │
        ▼
Salesforce Lead is created
        │
        ▼
LeadPasserLauncher (Flow, Lead Create, excludes Referral queue)  — CONFIRMED, see
LPA Trigger Flows — calls Apex action N8nNotificationServiceLeadPasser
  (also: LeadPasserLauncherCheckbox fires the same path on Update, when the "Launch Lead
  Passing Automation" checkbox changes — this is the manual retry lever)
        │
        ▼
n8n webhook "lpa" → LPA Production - RR Rollout (CONFIRMED live workflow — its sibling
"LPA Production - Refactor" is a shadow/test copy fired in parallel for comparison only,
never writes to Salesforce)
  • Multi-agent LLM qualification/relationship/routing calls decide an "Action"
  • Round-robin owner assignment via a Google Sheet-backed roster (Hot/Warm pools,
    OOO-aware) when routing calls for it
  • (see Wiki — LPA Overview)
        │
        ▼
LPA calls the PassInboundLead Salesforce Flow directly with that Action — CONFIRMED in the
n8n export itself (apiName: "PassInboundLead", matching input variables)
  • This Flow is LPA's only way to write to Salesforce, by design (see Wiki — LPA Overview)
  • Qualifies / disqualifies / pauses the Lead
  • Creates or updates the Engagement (Opportunity), OCRs, Deal, routes to an owner
  • (see Wiki — PassInboundLead Flow)
        │
        ├── Succeeds → Lead is Qualified, Engagement Record owned by the right SDR/AM/EAM
        │   (Lead History will show this under Holden Ruch's user — Holden is a human
        │   teammate, but LPA authenticates as him, so this is LPA acting automatically,
        │   not Holden manually touching the record — see Wiki, to be moved to a
        │   dedicated service user post-handover)
        │
        └── Fails, or never ran → Lead sits in a queue, and a human (currently Nickole)
            finds it and passes it manually using the same PassInboundLead flow
            (see Wiki — Manual Lead-Passing Process)
```

## Components and ownership

| Component | What it does | Current owner | Status |
|---|---|---|---|
| Legacy HubSpot→SF integration ("hs2sf" Azure Function) | Creates/updates SF Leads from HubSpot | Unknown/unowned by us — read-only reference | Being retired. Source code obtained and documented in Wiki, but confirmed **out of date** vs. production |
| Celigo middleware flow | Replacement for the legacy function app | Kiersten | In progress — documented in Wiki. Not yet at full field parity with the legacy app (see Open Questions) |
| `LeadPasserLauncher` / `LeadPasserLauncherCheckbox` (Salesforce Flows) | Hand a Lead to LPA on Create, or on-demand via the checkbox | — | **Documented in Wiki.** Both call an undocumented Apex action (`N8nNotificationServiceLeadPasser`) |
| LPA (n8n) — `LPA Production - RR Rollout` | Decides how a Lead should be routed/qualified via multi-agent LLM calls + round-robin assignment | Holden's team | **Documented in Wiki** (first-pass architecture). Confirmed live/production workflow. A sibling shadow/test workflow (`LPA Production - Refactor`) runs in parallel for comparison only |
| `PassInboundLead` Salesforce Flow (v55) | LPA's only write path into Salesforce, by design; executes the actual routing/qualify/Engagement-creation logic; called by LPA **or** manually | — | Documented in Wiki (structure + confirmed architecture rationale, confirmed direct call from n8n); decision-by-decision logic pending deeper pass |
| Manual lead passing / Automation Review queue | Human fallback when LPA fails or skips a Lead | Nickole (transitioning off her) | Documented in Wiki |

## Where things live

- **Wiki** (deep-dive pages — one per component):
  - `Lead-Passing-Flow-Diagram` — the whole process as one Mermaid diagram, start to finish
  - `Legacy-HS2SF-Function-App` — the legacy Azure Function's actual source code: field
    mapping, product mapping, auth, and known failure/error behavior (out-of-date snapshot,
    read-only)
  - `Celigo-HubSpot-to-Salesforce-Sync` — field mappings, scripts, connections, exports/imports
  - `LPA-Trigger-Flows` — the two Salesforce Flows that hand a Lead to LPA (on Create, and via
    the manual retrigger checkbox)
  - `LPA-Overview` — what LPA does, why, and how it's triggered/monitored, per its own team,
    plus first-pass architecture of the actual n8n workflow now that we have it
  - `PassInboundLead-Flow` — the Salesforce Flow that LPA and humans both call to pass a Lead
  - `Manual-Lead-Passing-Process` — Nickole's Automation Review / Inbound queue process
  - `Open-Questions-and-Gaps` — everything we don't yet know, with owners to chase
- **Repo** (this README + source artifacts as they're collected):
  - Source files referenced by the wiki pages should be added under a `sources/` folder in this
    repo as they're gathered (Celigo export zip, Salesforce Flow metadata, mapping spreadsheets)
    so the wiki pages can link back to the exact file a claim came from.

## Key terms

- **LPA** — Lead Passing Automation. The decision-making layer, in n8n, that decides whether a
  Lead is Qualified/Unqualified/Paused and who it should go to.
- **Engagement Record (ER)** — an Opportunity created to represent an SDR/AM/EAM's active work
  with a lead/contact. "Passing a lead" generally means qualifying it and creating or attaching
  it to an ER.
- **OCR** — OpportunityContactRole; links a Contact to an Opportunity/Engagement Record.
- **Automation Review queue** — Leads LPA identified as needing a human decision (e.g., can't
  tell if it should be Qualified, or needs a Contact/Account created first).
- **Inbound / Inbound Compliance queues** — new Leads waiting for LPA to launch; if a Lead is
  sitting here from before today, LPA is failing to pick it up.

## Known limitations of this documentation

See `Open-Questions-and-Gaps` in the Wiki for the full list. Highlights:
- We now have the actual n8n workflow (`LPA-Overview`) and the Salesforce trigger Flows
  (`LPA-Trigger-Flows`), but the Apex class that actually calls n8n
  (`N8nNotificationServiceLeadPasser`) and two n8n sub-workflows it references (`LPA Production -
  Eval` and a shared error-handling workflow) are still not obtained.
- The n8n architecture write-up is a **first pass** — high-level flow and the round-robin
  mechanism are confirmed, but the LLM agent prompts and routing code nodes haven't been traced
  condition-by-condition (this is likely where the EAM-account routing gap Nickole/Holden flagged
  actually lives).
- We now have the legacy function app's source code, but it's confirmed to be **out of date**
  relative to production — treat its specifics as "true of this snapshot," not "true today."
- Field-parity between the legacy function app and its Celigo replacement hasn't been fully
  reconciled — a handful of fields (GCLID, MSCLKID, device, etc.) appear in the legacy mapping
  but not yet in Celigo's.
- `PassInboundLead`'s internal decision logic (15 decision elements, ~60 total elements) is
  documented structurally (what it's called with, what it can do) but not traced
  branch-by-branch yet.
- ~~It's unclear whether "Holden Ruch"... is a human manually working leads or LPA~~ —
  **resolved**: Holden Ruch is a human teammate, but LPA currently authenticates using his
  Salesforce credentials rather than a dedicated service account, so those timeline entries are
  LPA acting automatically under his identity, not Holden manually touching the record. This is
  a known interim state, expected to change to a dedicated service user post-handover. See
  `Open-Questions-and-Gaps` → Resolved.

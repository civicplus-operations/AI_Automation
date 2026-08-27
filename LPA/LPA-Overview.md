# LPA (Lead Passing Automation) — Overview

**Status:** Conceptual/behavioral documentation only, sourced from a Slack Q&A between Meg
Gunther and Holden Ruch (LPA team). **We still do not have the actual n8n workflow/steps** — this
page documents what LPA does and why, as described by its owner, not its internal
implementation. Treat any specific claim here as "per Holden Ruch, unverified against the actual
n8n build" until we get direct access to it.

## What LPA is

LPA is the decision-making layer for inbound Leads, built and run in **n8n**. It reads a
Salesforce Lead, decides how it should be routed/qualified, and hands that decision off to the
**`PassInboundLead`** Salesforce Flow to actually execute (see `PassInboundLead-Flow`). LPA
itself does not write to Salesforce directly — see "Why PassInboundLead exists" below.

## What triggers LPA

A Lead is enrolled in LPA **most frequently upon Lead creation in Salesforce**, and
**occasionally via the "Trigger LPA" / "Launch Lead Passing Automation" checkbox** on the Lead
(the same checkbox referenced in `Manual-Lead-Passing-Process` as the first retry step).

Notably: **HubSpot does not trigger LPA directly.** There's a deliberate order of operations:
1. A **function app** creates the Lead in Salesforce first.
2. That Lead creation is what enrolls the Lead into LPA.

Per Holden Ruch, this is intentional, for two reasons:
- **Attribution** — the Lead data needs to exist in Salesforce for attribution purposes
  regardless of LPA.
- **Order of operations** — LPA-generated assets (Engagement Record / Sales Opp / Contact) need
  an existing Lead to associate back to. LPA can't create those *and* have them point at
  something that doesn't exist yet.

**Confirmed trigger mechanism:** two small Salesforce Flows — `LeadPasserLauncher` (fires on
Lead Create, excluding the Referral queue) and `LeadPasserLauncherCheckbox` (fires on Lead
Update when the `Launch_Lead_Passing_Automation__c` checkbox changes) — both call a single Apex
action, `N8nNotificationServiceLeadPasser`, which is what actually reaches out to n8n. See
`LPA-Trigger-Flows` for full detail. The Apex class's own source is still not obtained.

> Note this is a separate question from *what creates the Lead in the first place* — that's
> still most likely the legacy `hs2sf` Azure Function or its Celigo replacement (see
> `Legacy-HS2SF-Function-App` / `Celigo-HubSpot-to-Salesforce-Sync`), not explicitly confirmed
> by name in any conversation we have. `LeadPasserLauncher` fires on Lead Create regardless of
> which of those two created it.

## The n8n workflows (obtained, not yet fully documented)

We now have two n8n workflow exports, and have determined which is actually live:

| Workflow | `active` flag | Webhook path | Role |
|---|---|---|---|
| **`LPA Production - RR Rollout`** | `true` | `lpa` (clean, human-chosen path) | **Confirmed production** — both by our own analysis of the export and directly confirmed by the project owner: "this is the workflow that is still running/producing actual assignments." |
| **`LPA Production - Refactor`** | `true` | random UUID path (n8n's auto-generated default) | **Confirmed shadow/test only** — project owner: "the new version that is just writing to a test sheet for testing." RR Rollout fires a "fire-and-forget" HTTP call to Refactor's webhook on every real Lead (node `Fire Refactor Shadow`, 3-second timeout, errors ignored) purely so Refactor's decisions can be logged and compared — Refactor's own `Invoke...Flow` nodes (the ones that would write to Salesforce) are all disabled, so it can never affect a real Lead. It has to stay `active` only so the shadow call succeeds. |

The `Fire Refactor Shadow` node's own inline note confirms this directly: *"Fire-and-forget
shadow trigger: GETs the refactor's webhook with the same lead id. continue-on-error + 3s timeout
so it can never affect live routing. Requires the refactor workflow to be ACTIVE."*

### The Google Sheet backing all of this: "Lead Passing - Round Robin"

Confirmed directly by the project owner: the Google Sheet referenced throughout both n8n
workflows (`Get LPA RR (Hot)`, `Get LPA RR (Warm)`, `Update Hot`, `Update Warm`, `Log Prod
Result`, `Log Refactor Result`, `Audit`, etc.) is a single sheet named **"Lead Passing - Round
Robin"**, organized into tabs. The one tab we have a name for: **"Parallel Testing"** — this is
specifically where `LPA Production - Refactor`'s shadow-test outcomes get written (i.e., what
the code nodes `Build Refactor Row` / `Log Refactor Result` are writing to). The Hot/Warm
roster tabs and any others are not yet named/confirmed — see `Open-Questions-and-Gaps`.

**Confirmed link back to `PassInboundLead`:** RR Rollout's three Salesforce "invoke flow" nodes
(`Invoke a flow`, `Invoke a Flow (Streamline Route)`, `Invoke Diligent Flow`) all target
`apiName: "PassInboundLead"` directly, passing exactly the input variables documented in
`PassInboundLead-Flow` (`Action`, `leadID`, `contactID`, `parentID`, `targetagencyID`,
`productinstanceId`, `opportunityID`, `engagementrecordID`, `engagementrecordtocloseID`,
`Reason`). This is no longer an inference — it's directly confirmed in the workflow JSON.

### RR Rollout architecture, at a glance (first pass — not a full node-by-node trace)

```
Webhook (path: lpa, receives a Lead Id)
  → Get Lead (Salesforce)
  → Fire Refactor Shadow (fire-and-forget, non-blocking)
  → Contact/Account resolution: ShouldCreateContact / ContactFieldExists / Create a contact /
    Update ACR / Get an account / Get Parent Account / Get Segment (EGM-Named check)
  → Multiple LangChain agent calls make the actual judgment calls:
      LPA Agent - Qualification, LPA Agent - Relationships, LPA Agent - Simple Existing,
      LPA Agent - Existing, LPA Agent - Customer, LPA Agent - Routing
    (backed by Claude Haiku, z.ai GLM 4.6, and a dedicated "Haiku 4.5 - Routing" model)
  → Output parsed into a structured object: { action, reasoning[], reason, ids{...},
    contact_updates{...} } (Structured Output Parser node — `action` is a free-text string,
    not a hard enum, matched by convention to the Action values PassInboundLead expects)
  → Branch by pool: CP (CivicPlus/standard) vs Streamline (via StreamlineRoutingRules code node,
    itself branching EDU vs Other, each with its own Google-Sheet-backed routing table) vs
    Diligent (hardcoded Action: "DILIGENT", Reason: "Diligent Partner - DocAccess partner lead
    (cannot pass)" — this looks like a fixed carve-out for a specific partner scenario, not a
    generally-computed outcome)
  → Round-robin owner assignment (Warm vs Hot pools — see below) when the outcome calls for it
  → Invoke a flow / Invoke a Flow (Streamline Route) / Invoke Diligent Flow → PassInboundLead
  → Build Prod Row (CP/SL/Diligent) → Log Prod Result (Google Sheets audit trail)
  → Debounce mechanism (n8n Data Table: Debounce / Get Debounce / Delete Debounce) — appears to
    prevent duplicate/rapid-fire re-processing of the same Lead within a short window
```

### Round-robin assignment mechanism (confirmed from code)

The `Identify Owner` code node implements the actual round-robin logic against a Google
Sheet-backed roster (separate "Hot" and "Warm" pools, via `Get LPA RR (Hot)` /
`Get LPA RR (Warm)`):
- Each rep row tracks a running `Leads` count and a `SkippedOOOLeads` count.
- Candidates are sorted by **total leads (Leads + SkippedOOOLeads), ascending**, with ties
  broken by sheet row order.
- Going down that sorted list, any rep currently marked **OOO** (an `OOO` field is non-blank) is
  skipped — but their `SkippedOOOLeads` counter still increments, so **they don't lose their
  place in the rotation** while out; they just don't receive a Lead until they're back.
- The first non-OOO rep found gets the Lead; their `Leads` count increments and the roster is
  written back (`Update Hot`/`Update Warm`).

This is the "ASSIGN_TO_INBOUND_SDR_ROUND_ROBIN" mechanism referenced in `PassInboundLead-Flow`'s
Action table, and likely the actual place to look first if round-robin routing "feels wrong" in
practice (stale OOO flags, a rep stuck at the bottom of rotation, a sheet write that didn't take,
etc.).

> ⚠️ **This is not the only owner-assignment mechanism in the system.** `PassInboundLead` itself
> also has **hardcoded named-rep assignments** on some paths, confirmed directly by the project
> owner (see `PassInboundLead-Flow` → "Hardcoded named reps"). Updating the Google Sheet above
> only affects the n8n-side round robin; it does **not** touch whatever's hardcoded inside the
> Flow. Don't assume a sheet edit alone reroutes everything — check both places.

### Two workflows referenced but still not obtained

Both n8n exports reference, via a disabled `executeWorkflow` node, a third workflow:
**`Prod — LPA Production - Eval"`** (id `zLNHAhcVhaoaqGS1`) — currently wired in but disabled in
both Refactor and RR Rollout, so not part of live execution today, but clearly intended to be
called eventually.

Both workflows also share a configured **error workflow** (n8n `settings.errorWorkflow`, id
`rygASdZeqSLngIMs`) — i.e. whatever n8n workflow handles failures for LPA runs, we don't have
that either. This is likely relevant to "AI CoE" monitoring mentioned elsewhere on this page.

## What data LPA uses

Per Holden Ruch directly: **no data is pulled from HubSpot or any other system at LPA
run-time.** The only inputs are:
- The Salesforce **Lead record's own fields**, queried by LPA within milliseconds of enrollment.
- LPA's own reasoning, which gets logged back onto the Lead — this is technically
  n8n-generated data, but it's only used for routing/traceability, not as an input to further
  decisions.

### Timing / staleness risk (important)

Enrollment → LPA querying the Lead's data happens in **milliseconds**. This means: if a Lead's
fields are updated even ~1 minute after Lead creation (e.g. a UTM or Lead Source field arriving
late from a downstream HubSpot workflow), **LPA has almost certainly already run and passed the
Lead by the time that update lands** — that later data is not re-processed by LPA. It's simply
not part of the decision, because the decision already happened.

This directly explains behavior seen in `Lead_Timelines.xlsx` — e.g., Andrew Carter's Formatted
Campaign syncing ~3 hours after Lead creation, or John Foley-Murphy having no Formatted Campaign
entry at all. Those late/missing updates aren't LPA malfunctioning; they're simply arriving after
LPA already made its call. Per Holden Ruch, this was flagged as a hard requirement back to
Kiersten: **there cannot be delay between Lead creation in Salesforce and accurate data
populating on the Lead** — because LPA won't wait for it or re-check.

## What causes Leads to "fail"

In practice, roughly in order of frequency:
1. **Data issues in Salesforce** — most common. The Lead is missing data LPA needs to route it.
2. **n8n or "openrouter"-side issues** — second most common (errors, downtime).
3. Less common: a RevOps-side data model change conflicting with what `PassInboundLead` expects.

Failures overall are described as uncommon, and the **Trigger LPA / Launch Lead Passing
Automation checkbox retry mechanism covers the vast majority of failure recoveries** — this is
exactly the mechanism Nickole's manual process uses (see `Manual-Lead-Passing-Process`).

## Error handling / monitoring by failure type

| Failure type | Who monitors it | What happens |
|---|---|---|
| n8n-side or `PassInboundLead`-side failures | **AI CoE** (per Holden Ruch — likely an internal "AI Center of Excellence" team; not yet confirmed as a formal name) | Monitored directly by that team |
| Salesforce data issues | **Kenna / Sumre** | Lead is re-enrolled in LPA once the underlying data is fixed — this is the same mechanic as toggling the "Launch Lead Passing Automation" checkbox in the manual process |

## PDF Accessibility and EAM-account handling

PDF Accessibility Leads are **not** enrolled or triggered any differently — enrollment is
identical for every Lead. What differs is **routing logic only**, in exactly the same sense that
an EAM-named account gets different routing logic (see `IsPDFAccessibility` and
`EGM_Named_Check` decisions in `PassInboundLead-Flow`). If PDF Accessibility Leads appear stuck,
the retry/enrollment story is the same as any other Lead — look at routing, not triggering.

## Execution identity

**Holden Ruch is a human teammate on the LPA team** — but LPA currently authenticates to
Salesforce using **his personal Salesforce user's credentials**, not a dedicated
service/integration user with its own OAuth credentials. Per Holden: *"switching it to a
dedicated API user would be preferred, but seats are limited,"* and *"I basically never touch SF
data — if it's my user, it's almost certainly LPA."*

**This resolves the open question from `Lead_Timelines.xlsx`:** the Lead Status changes
attributed to "Holden Ruch" in that timeline audit are LPA acting automatically under Holden's
identity, not Holden manually completing LPA's job by hand. Any Lead History entry made under
Holden Ruch's user should be read as LPA, not a manual action — as long as this interim
authentication setup is in place.

**Action item for handover:** this is flagged as a temporary/interim state, not the intended
end state. Once handover of this project is complete, LPA's Salesforce authentication should be
migrated off Holden's personal user and onto a **dedicated service user with its own OAuth
credentials**. Until that migration happens, remember that "Holden Ruch" in any Salesforce audit
trail is ambiguous by construction — it could technically be Holden himself in a rare manual
edit, even though in practice he says this essentially never happens. Confirm this migration
gets a ticket/owner rather than falling through the cracks during the transition.

## Rules of Engagement (ROE)

ROE gets updated "pretty regularly (more or less weekly)" per Holden Ruch, but these updates do
**not** change LPA's overall architecture — they tune routing rules within the existing
structure. *(Where ROE itself is documented/tracked is not yet known — see
`Open-Questions-and-Gaps`.)*

## Why `PassInboundLead` exists as a separate Flow

Per Holden Ruch: `PassInboundLead` was purpose-built when LPA was constructed, specifically to
act as **LPA's "write action" back into Salesforce** — deliberately, because the team didn't
want to give the AI/n8n side full write access to production Salesforce. LPA's routing logic
produces a "result," which is passed into the Flow as its inputs (this is exactly the `Action`
input variable and the other IDs documented in `PassInboundLead-Flow`); the Flow then applies
deterministic logic based on those inputs to actually make the Salesforce changes.

There is a separate, **fully retired and unrelated** flow called something like *"Inbound Lead
Passing Flow W/..."* — don't confuse this with `PassInboundLead` if you find it while browsing
Salesforce Setup; it predates LPA and isn't part of the current architecture.

## Outstanding / requested changes (not yet built)

- **Nickole Boloix (Slack, undated in this thread):** requested that when a Contact is created
  through LPA and already has a single `Persona__c` value, that value also be pushed into
  `Primary_Persona__c` on creation. Status: raised, no confirmed owner/process for requesting
  LPA changes yet identified — see `Open-Questions-and-Gaps`.

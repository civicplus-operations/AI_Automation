# PassInboundLead (Salesforce Flow, v55)

**Status:** Structure and input contract documented directly from the exported Flow metadata
(`PassInboundLead_v55.json`, description on the flow itself: *"8.26.26 - Diligent routing logic
added"*). **Not yet documented:** full branch-by-branch decision logic — see
Open-Questions-and-Gaps. Treat this page as "what it's called with and what it's capable of
doing," not yet "exactly how it decides."

## What it is

An **Autolaunched Flow** (`processType: AutoLaunchedFlow`, API version 65) that does the actual
work of passing/qualifying an inbound Lead: creating or updating Engagement Records
(Opportunities), OpportunityContactRoles, Deals, and routing ownership. It is invoked two ways:

1. **By LPA** (n8n) once it's decided what should happen to a Lead.
2. **Manually**, by a human working the Automation Review/Inbound queues, via the "Pass Inbound
   Lead" action referenced in the manual process (see `Manual-Lead-Passing-Process`).

This is the single shared mechanism behind both the automated and manual paths — anything true
about how a Lead gets passed is true here, regardless of who/what triggered it.

**Confirmed architecture (per Holden Ruch, LPA team, Slack):** this Flow was purpose-built
alongside LPA specifically to be LPA's "write action" back into Salesforce — deliberately, so
the n8n/AI side never has direct write access to production Salesforce. LPA's routing logic
produces a "result," which becomes this Flow's inputs (the `Action` value and supporting IDs
below); the Flow then applies deterministic logic to make the actual Salesforce changes. See
`LPA-Overview` for the full n8n-side context.

There is a separate, **fully retired and unrelated** flow with a similar-sounding name
("Inbound Lead Passing Flow W/...") — it predates LPA and isn't part of the current
architecture. Don't confuse it with this one if it turns up in Salesforce Setup.

## ⚠️ Hardcoded named reps — operational risk, confirmed

**Some owner/routing assignment inside this Flow is hardcoded to specific named account
reps, not pulled dynamically from a roster.** Confirmed directly by the project owner today:
*"today I switched out inbounds sent to Kevin Gallagher to Abraham Amidon."* That's a manual
edit made directly inside this Flow's Owner-lookup logic (most likely one of the `Get*Owner`
lookups documented in the element inventory below — e.g. `GetInboundQueueOwner`,
`GetEGMQueueOwner`, `GetDiligentOwner`, `GetStreamlineOwner`, `GetSalesOppOwner`,
`GetParentAccountOwner`, `GetProductInstanceOwner`, `GetEAMUserID` — we don't yet know exactly
which one(s), since we only have element names, not their filter criteria).

This is a **second, independent owner-assignment mechanism**, distinct from the n8n-side
round-robin described in `LPA-Overview` (which pulls from a Google Sheet, "Lead Passing -
Round Robin," and is updated by editing that sheet — no Flow edit required). Whatever paths
route through these hardcoded lookups instead require someone to **manually edit this
Salesforce Flow itself** whenever a named rep changes — leaves, changes territory, goes on
leave, etc.

**Why this matters for handover:**
- If a rep referenced by name here leaves or changes role and nobody remembers to update the
  Flow, Leads on that path will likely misroute or silently keep going to someone who
  shouldn't have them anymore — with no error, since the lookup will still succeed.
- We don't yet have a full inventory of *which* lookups/paths use hardcoded names vs. dynamic
  queries vs. the n8n round-robin sheet. This needs a deliberate pass through the Flow (or a
  conversation with whoever's been maintaining it) before this can be handed over safely.
- See `Open-Questions-and-Gaps` — this has been flagged there as a blocking item, not just a
  documentation gap, since it's an active operational risk.

## Input contract

The flow is called with these input variables:

| Variable | Purpose |
|---|---|
| `Action` | **The routing switch.** Tells the flow what to do (see Action values below). |
| `leadID` | The Lead being passed. |
| `contactID` | Contact to attach/use, if known. |
| `parentID` | Parent Account. |
| `targetagencyID` | Target Agency Account. |
| `opportunityID` | An existing Engagement Record to use, if applicable. |
| `engagementrecordID` | Engagement Record reference (used in some branches alongside/instead of `opportunityID`). |
| `engagementrecordtocloseID` | An Engagement Record that should be closed as part of this pass (e.g. superseded by a new one). |
| `productinstanceId` | Existing Product Instance, if the Lead relates to one. |
| `secondaryEmailAddress` | Used to decide whether the Contact record needs updating (see `UpdateContact` decision). |
| `Reason` | Free-text/coded reason — used for Unqualified/Pause outcomes. |
| `OwnerEmail` | **Output.** The email of whoever the Lead/Engagement ends up routed to. |

## `Action` values (top-level router)

The first decision in the flow, `ActionRouter`, switches on the `Action` input. These are the
values it recognizes:

| Action value | What it broadly does |
|---|---|
| `UNQUALIFIED` | Marks the Lead Unqualified (`UpdateLeadUnqual`). |
| `PAUSE` | Pauses the Lead (`UpdateLeadPause`) — presumably for "not now, but don't disqualify." |
| `CONVERT_TO_EXISTING_SALES_OPP` / `CONVERT_TO_EXISTING_CUSTOMER_OPP` | Attaches the Lead to an existing Sales Opportunity rather than creating a new Engagement Record. |
| `CONVERT_TO_EXISTING_ER_WITH_ACTIVITY` / `ASSIGN_TO_SAME_INBOUND_SDR` | Routes to an existing Engagement Record, keeping the same Inbound SDR who already owns related activity. |
| `NOTIFY_CSM_EAM_CONVERT_TO_CONTACT` | Converts to a Contact and notifies the CSM/AM/EAM rather than creating a new SDR-owned Engagement. |
| `CLOSE_INACTIVE_ER_PASS_TO_INBOUND` / `ASSIGN_TO_INBOUND_SDR_ROUND_ROBIN` | Closes an inactive Engagement Record and round-robins the Lead to Inbound SDRs. |
| `Streamline` | Creates/updates a "Streamline" Engagement (`CreateStreamlineEngagement`, `UpdateNewlyCreatedStreamlineDeal`, `CreateOCRStreamline`). |
| `DILIGENT` | Creates/updates a "Diligent" Engagement (`CreateDiligentEngagement`, `UpdateNewlyCreatedDiligentDeal`, `CreateOCRDiligent`, `DiligentEmailAssignment`) — this is the routing logic the flow description says was added most recently (8/26/26). |
| *(no match)* | Falls to `Default Outcome`. |

## What happens after routing (by theme)

Regardless of which `Action` fired, the flow generally works through some combination of:

- **Finding/creating the right Account structure** — `GetParent`, `GetTargetAgency`,
  `EGM_Named_Check` (checks if the Parent Account's `Segment__c` is `"EGM-Named Account"`, which
  changes email/owner routing vs. "Standard Inbound").
- **Finding/creating the Contact and its OCR** — `GetContact`, `Does_OCR_Exist`/`CreateOCR`,
  `UpdateContact` (only if `secondaryEmailAddress` isn't null).
- **Creating or updating the Engagement Record (Opportunity)** and its OCR —
  `Create_Inbound_Engagement`/`CreateDiligentEngagement`/`CreateStreamlineEngagement`, each with
  a matching `CreateOCR*` and `UpdateNewlyCreated*Deal`.
- **Closing a prior Engagement Record if superseded** — `ER_To_Close?`, `GetEngagementToClose`,
  `CloseER`, plus its own OCR handling (`OCR_Present?`, `getOCRstoclose`).
- **Merging Products of Interest** onto the Deal/Engagement — a loop-driven merge
  (`Products_of_Interest_Assignment`, `Selected_Products_Assignment`,
  `Single_Selected_Product_Assignment`, `Product_Keep_Assigning` loop-continuation check,
  `POI_and_SPOI_Merge`, `Create_Combined_SPOI`) that appears to combine primary and secondary
  product-of-interest values before deciding whether the existing Deal needs updating
  (`Update_SPOI?`).
- **Determining ownership/routing** — a family of `Get*Owner` lookups
  (`GetInboundQueueOwner`, `GetEGMQueueOwner`, `GetDiligentOwner`, `GetStreamlineOwner`,
  `GetSalesOppOwner`, `GetParentAccountOwner`, `GetProductInstanceOwner`, `GetEAMUserID`) paired
  with matching `*EmailAssignment` steps (`InboundEmailAssignment`, `EGMEmailAssignment`,
  `EGMEmailAssignment_PDFAccessibility`, `DiligentEmailAssignment`, `StreamlineEmailAssignment`,
  `SalesEmailAssignment`, `ParentAccountEmailAssignment`, `ProductInstanceEmailAssignment`,
  `EngagementEmailAssignment`) — this is almost certainly where the "LPA is missing routing
  logic on EAM accounts" failure mode (flagged by Nickole/Holden) actually lives, since EAM-named
  accounts get a distinct branch (`EGM_Named_Check`) from standard inbound routing.
- **Special-casing PDF Accessibility** — `IsPDFAccessibility` checks whether
  `Primary_Product_Interest__c == "PDF Accessibility"` and routes email assignment differently
  in that case.
- **Setting the final Lead fields** — one of `UpdateLeadDiligent`, `UpdateLeadExistingER`,
  `UpdateLeadNewER`, `UpdateLeadPause`, `UpdateLeadProductInstance`, `UpdateLeadSales`,
  `UpdateLeadStreamline`, or `UpdateLeadUnqual`, depending on the path taken.
- **Logging inbound activity** — `CreateEngagementInboundTask`, `CreateSalesInboundTask`,
  `CreateProductInstanceTask`.
- **Case creation for PI/PA ownership disputes** — `CreatePICase`/`CreatePACase`, gated by the
  `PI_or_PA_Owner?` decision, which checks whether `Formatted_Campaign__c` contains `"Migration"`
  or `"Contact My Account"` to decide whether the case should go to a PA (Partner/Account) owner
  vs. PI (Product Interest) owner.

## Element inventory (for reference / future deep-dive)

| Category | Count | Notes |
|---|---|---|
| Decisions | 15 | `ActionRouter`, `Does_OCR_Exist`, `DoesEROCRExist`, `EGM_Named_Check`, `ER_To_Close`, `ERStageCheck`, `InboundSDR`, `IsPDFAccessibility`, `OCR_Present`, `PI_or_PA_Owner`, `Product_Keep_Assigning`, `Update_Inbound_Deal`, `Update_SPOI`, `UpdateContact`, `UpdateEngagementDeal` |
| Record Lookups | 28 | Accounts, Contacts, Leads, Opportunities, Deals, OCRs, Users, Campaigns, RecordTypes |
| Record Updates | 18 | 9 Lead updates, 6 Opportunity/Deal updates, 1 Contact update |
| Record Creates | 13 | 4 Engagement (Opportunity) creates, 5 OCR creates, 3 Task creates, 1 RecordType-driven, 2 Case creates |
| Assignments | 22 | Owner/email routing + product-of-interest merge logic |

If this flow ever needs to be modified or debugged in detail, the next step is walking each
decision's actual formula/condition tree against a real test Lead — this page gives you the map,
not yet the turn-by-turn directions.

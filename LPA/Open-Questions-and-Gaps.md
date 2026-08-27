# Open Questions & Gaps

Living list. Move items to "Resolved" with the answer and date once closed — don't delete them,
so there's a record of what was unclear and why.

## Blocking / high priority

- **`PassInboundLead` contains hardcoded named-rep owner assignments on at least some paths —
  needs a full inventory before handover.** Confirmed directly by the project owner, with a
  real example: a rep swap made today (Kevin Gallagher → Abraham Amidon) required editing the
  Flow itself, not a spreadsheet. This is separate from the n8n-side round-robin (Google Sheet
  "Lead Passing - Round Robin") and creates real risk: if a hardcoded rep leaves/changes role and
  nobody remembers to edit the Flow, Leads will keep routing to the wrong person with no error.
  **We don't yet know which specific lookups/paths are hardcoded vs. dynamic** — needs someone
  to walk the Flow's `Get*Owner` lookups (see `PassInboundLead-Flow`) and document exactly which
  ones hardcode a name, so this is a known, documented maintenance task rather than a landmine
  for whoever inherits the Flow. *Owner: whoever currently maintains `PassInboundLead` (Holden's
  team, or wherever the RevOps/Salesforce admin side of this sits) — needs to walk through this
  with us directly.*
- **Full tab inventory of the "Lead Passing - Round Robin" Google Sheet.** We know it exists and
  that one tab is named "Parallel Testing" (confirmed by the project owner); we don't have the
  full list of tabs (presumably at least separate Hot/Warm roster tabs, an audit/log tab, and
  the production result-logging tab), who owns/edits it, or how a rep is added/removed/marked
  OOO in practice. *Owner: whoever manages that sheet — likely Holden's team.*

- **What exactly "the function app" is that creates Leads and kicks off LPA — LIKELY RESOLVED,
  not fully confirmed.** We now have the actual source code (`hs2sf.zip`, package
  `cp-hubspot-to-salesforce` v3.1.0 — see `Legacy-HS2SF-Function-App`), which is almost certainly
  this function app: it's the Azure Function that creates/updates Salesforce Leads from HubSpot.
  **Not yet nailed down:** (a) explicit confirmation from Holden/Kiersten that this is the same
  app referenced in the Slack thread, since the thread didn't name it directly; (b) whether the
  code we have matches what's actually running in production — **we've been told this copy is
  out of date**, so treat specifics (field lists, product maps, error behavior) as "true of this
  snapshot," not "true of prod today." *Owner: confirm naming with Holden/Kiersten; get a current
  export if/when this app is finally retired or before, if feasible.*
- **Whether `SALESFORCE_USERNAME` (this app's JWT auth identity) corresponds to the "Marketing
  API" user seen creating/assigning Leads in `Lead_Timelines.xlsx`.** Plausible given the timing
  and behavior match, not confirmed. *Owner: Kiersten or whoever manages the Salesforce
  Connected App / JWT credentials for this integration.*
- **Field-parity gap between the legacy function app and the Celigo replacement.** The legacy
  app maps `gclid`, `msclkid`, `paid_search_keyword_matchtype`, `device`, and a tradeshow
  "submitted by" field that Celigo's current mapping doesn't appear to include (see
  `Legacy-HS2SF-Function-App` for the full comparison table). Also unconfirmed whether Celigo
  normalizes `state_picklist` to a full state name the way this app does. **Needs a decision
  before the legacy app is fully retired** — either confirm these fields are genuinely
  deprecated/unused, or add them to the Celigo mapping. *Owner: Kiersten.*
- **LPA's own decision logic lives in n8n — LARGELY RESOLVED.** We now have both n8n workflow
  exports and have determined `LPA Production - RR Rollout` is the live one (`LPA Production -
  Refactor` is a shadow/test workflow that RR Rollout fire-and-forgets into for comparison
  logging, confirmed by an inline note on the `Fire Refactor Shadow` node). Architecture is
  documented at a first-pass level in `LPA-Overview` — multi-agent LLM qualification/routing,
  round-robin assignment logic, confirmed direct calls into `PassInboundLead` with matching input
  variables. **Still not done:** a full node-by-node trace of the LLM agent prompts and the
  `StreamlineRoutingRules`/`Routing Tree` code nodes' exact conditions — the EAM-account routing
  failure mode Nickole/Holden flagged likely lives inside these agent calls or the `EGM-Named`
  branch, but hasn't been traced end to end. *Owner: internal follow-up, no external owner
  needed now that we have the export.*
- **Two n8n workflows referenced but not obtained:** `Prod — LPA Production - Eval` (id
  `zLNHAhcVhaoaqGS1`, wired in via a currently-disabled node in both exports we have) and
  whatever workflow is configured as the shared **error workflow** (id `rygASdZeqSLngIMs`) for
  both RR Rollout and Refactor. *Owner: Holden's team — ask if these can be exported too.*
- **The Apex class `N8nNotificationServiceLeadPasser` — source not obtained.** We now know
  which two Salesforce Flows call it and when (see `LPA-Trigger-Flows`), but not what it does
  internally or which n8n webhook URL it's actually configured to hit. *Owner: whoever owns Apex
  deployments — possibly Holden's team, possibly a separate RevOps/dev owner.*
- **Referral-queue Leads and the checkbox path.** `LeadPasserLauncher` explicitly excludes Leads
  owned by the Referral queue from automatic LPA enrollment on create. `LeadPasserLauncherCheckbox`
  has no such exclusion — so checking the box on a Referral-queue Lead would, on its face, still
  enroll it in LPA. Not confirmed whether that's intentional. *Owner: whoever owns Referral queue
  process — confirm with Holden's team or Nickole.*
- **Description/configuration mismatch in `LeadPasserLauncher` (v4).** Its description text says
  "Launches n8n Lead Passer workflow upon lead update," but it's actually configured to trigger
  on Lead **Create**. Likely stale documentation from an earlier version, not a functional bug —
  but worth a quick confirmation next time this flow is touched, in case it's actually a sign the
  trigger type changed without an intended side effect being caught. *Owner: whoever next edits
  this flow.*
- **Where/how Rules of Engagement (ROE) are documented or tracked** — Holden says ROE updates
  roughly weekly but this thread doesn't say where those updates live or how they're proposed.
  *Owner: Holden's team.*
- **No confirmed process for requesting LPA changes.** Nickole raised a specific feature request
  (see `LPA-Overview` → Outstanding/requested changes) and asked where such requests should go —
  unanswered in the thread we have. *Owner: TBD — likely Holden's team or the "AI CoE," but not
  confirmed.*
- **"AI CoE"** is referenced by Holden Ruch as the team monitoring n8n/`PassInboundLead`-side
  failures — full name/scope/contact point not yet confirmed. (Possibly related to the
  not-yet-obtained n8n error workflow above.)
- **LPA's Salesforce authentication needs to move off Holden Ruch's personal user onto a
  dedicated service user with its own OAuth credentials, once this handover is complete.** This
  is currently an interim/known-debt state (see `LPA-Overview` → Execution identity), not yet
  ticketed anywhere we're aware of. *Owner: needs to be assigned — likely Holden's team plus
  whoever owns Salesforce user/license administration, given the stated seat constraint.*

## Documentation gaps (not blocking, but incomplete)

- `PassInboundLead` (v55) has 15 decision elements and roughly 60 total elements. This pass
  documents its input contract (the `Action` values it accepts and what each broadly does) but
  not every decision's full branch-by-branch logic. A deeper pass should walk each decision's
  formulas/assignments if this flow needs to be modified or debugged.
- Two Celigo lookups reference asynchronous Salesforce API calls (`Find Lead`, `Get Contact`)
  marked `asynchronous: true` — worth confirming expected latency/retry behavior with Kiersten,
  since that affects the hs2sf timing numbers in `Lead_Timelines.xlsx`.
- The legacy black-box HubSpot↔Salesforce integration being replaced is, by definition,
  undocumented. If it's still running in parallel with Celigo during cutover, worth capturing
  *when* it gets fully decommissioned so this README's "TODAY" framing doesn't go stale.
- `Update SPOI?` decision and `MergedPOIandSPOI` variable in `PassInboundLead` reference a
  formula/loop (`Selected_Product_Loop`) whose purpose (merging primary + secondary product of
  interest) is inferred from naming, not confirmed.

## Resolved

- **"Holden Ruch" in the Lead_Timelines.xlsx audit — RESOLVED.** Confirmed via Slack (Holden
  Ruch, responding to Meg Gunther): Holden Ruch is a human teammate on the LPA team, but LPA
  currently *authenticates to Salesforce using his personal user's credentials* rather than a
  dedicated service/integration user with its own OAuth credentials ("switching it to a
  dedicated API user would be preferred, but seats are limited"). Holden says he essentially
  never touches Salesforce data manually — "if it's my user, it's almost certainly LPA."
  **Conclusion: the Lead Status changes attributed to Holden Ruch in the timeline audit are LPA
  acting automatically under his identity, not Holden doing manual work.** The
  `Lead_Timelines.xlsx` "avg 2.0 min" LPA timing can be treated as real automated LPA
  performance. *Resolved via Slack thread, Meg Gunther / Holden Ruch.*
- **Whether HubSpot triggers LPA directly, and what the intermediate hand-off is — PARTIALLY
  RESOLVED.** Confirmed it is *not* direct: a Lead is created in Salesforce first (by "the
  function app" — see still-open item above on what that actually is), and Lead creation is what
  enrolls the Lead in LPA. Order-of-operations reason confirmed: LPA needs the Lead to exist so
  it has something to associate generated Engagement/Sales Opp/Contact records back to; also an
  attribution requirement independent of LPA. *Resolved (conceptually) via Slack thread; the
  concrete "what creates the Lead" identity is still open above.*
- **Whether LPA pulls live data from HubSpot at decision time — RESOLVED.** It does not. LPA
  only reads the Salesforce Lead record's own fields, queried within milliseconds of enrollment.
  *Resolved via Slack thread, Holden Ruch.*
- **Why data (e.g. Formatted Campaign) sometimes syncs oddly late relative to LPA, per
  `Lead_Timelines.xlsx` — RESOLVED.** LPA queries the Lead almost immediately after enrollment;
  any field updates arriving after that point are not re-processed by LPA — the decision has
  already been made. This is a known/flagged architecture constraint, not a bug. *Resolved via
  Slack thread, Holden Ruch.*
- **Relationship between `PassInboundLead` and LPA — RESOLVED.** `PassInboundLead` was
  purpose-built as LPA's Salesforce "write action," specifically so n8n/AI never has direct
  production Salesforce write access. LPA produces a routing "result" that becomes this Flow's
  inputs. A separate, similarly-named legacy flow ("Inbound Lead Passing Flow W/...") is fully
  retired and unrelated. *Resolved via Slack thread, Holden Ruch.*
- **The Salesforce → LPA hand-off mechanism — RESOLVED.** Two small Flows,
  `LeadPasserLauncher` (Lead Create, excludes Referral queue) and `LeadPasserLauncherCheckbox`
  (Lead Update, on `Launch_Lead_Passing_Automation__c` change), both call one Apex action,
  `N8nNotificationServiceLeadPasser`. See `LPA-Trigger-Flows`. *Resolved via Flow metadata
  provided directly (`LeadPasserLauncher_v4.json`, `LeadPasserLauncherCheckbox_v1.json`).*
- **Which n8n workflow is actually live — RESOLVED, and directly confirmed.** `LPA Production -
  RR Rollout` is production; `LPA Production - Refactor` is a shadow/test workflow fired in
  parallel for comparison logging only. Confirmed two ways: (1) an inline node note on `Fire
  Refactor Shadow` plus the disabled Salesforce-writing nodes in Refactor, and (2) direct
  confirmation from the project owner: RR Rollout "is the workflow that is still
  running/producing actual assignments," Refactor "is just writing to a test sheet for testing."
  See `LPA-Overview`. *Resolved via n8n workflow exports + project owner's notes.*

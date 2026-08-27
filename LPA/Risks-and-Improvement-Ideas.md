# Risks & Improvement Ideas — LPA / n8n

**Status:** Analysis findings from reading both n8n workflow exports (`LPA Production - RR
Rollout`, live; `LPA Production - Refactor`, shadow) directly — node configs, credentials,
error-handling settings, and embedded code. These are conclusions, not open questions — for
things we still need someone else to answer, see `Open-Questions-and-Gaps`. Where a risk here
depends on an answer from that page, it's noted.

Ordered roughly by "how bad it'd hurt if it went wrong," not by how likely it is.

## 1. Read-modify-write race condition in round-robin assignment

`Identify Owner` reads the roster (`Get LPA RR Hot/Warm`), computes the next rep in a JavaScript
code node, then writes the result back (`Update Hot`/`Update Warm`) — three separate steps, with
no locking in between and no check that the sheet hasn't changed since the read.

**Why it matters:** if two Leads reach the webhook close together — a form-submission burst, a
bulk import, two people submitting around the same time — both executions can read the same
starting counts, both independently decide "give it to rep X," and both write back. One rep
silently gets two Leads in a row while someone else gets skipped, and round-robin fairness
drifts with nothing to point at afterward. Google Sheets has no transactional guarantees to
catch this, and nothing in the workflow currently checks for it.

**Idea:** move the roster to something with real locking (a proper database row, or n8n's own
Data Table with an atomic increment/claim operation) — or at minimum add a version/timestamp
column and have `Update Hot`/`Update Warm` fail (and retry the read) if the row changed since it
was read.

## 2. Two single points of failure: one Salesforce credential, one Google credential

Every Salesforce node in both workflows — every lookup, every create, all three `Invoke...Flow`
calls — uses the exact same OAuth credential, named `Salesforce PROD`. Every Google Sheets
operation — both round-robin rosters, both Streamline tables, both result-logging tabs — uses
one shared `Google Sheets` credential.

**Why it matters:** this is the same shape of risk twice. If whoever owns either credential
loses access, changes their password, offboards, or has their token revoked, the *entire*
system stops — not degrades, stops. This is the n8n-side mirror of the already-known issue that
LPA authenticates to Salesforce as Holden Ruch's personal user (see `LPA-Overview` → Execution
identity); the Google credential likely has the same "whose personal account is this" question
and hasn't been asked yet.

**Idea:** move both to dedicated service accounts with their own credentials, tracked
explicitly as infrastructure (not tied to any one person's login) — same fix, applied twice.

## 3. Diligent path fails silently; the main path doesn't

`Invoke a flow` and `Invoke a Flow (Streamline Route)` (the CP/Streamline calls into
`PassInboundLead`) have no error override — a failure stops the workflow and should route to
the configured error workflow (id `rygASdZeqSLngIMs`, still not obtained). But `Diligent Owner
Check` and `Invoke Diligent Flow` are both explicitly set to `onError: continueRegularOutput` —
if the actual Salesforce write fails on the Diligent path, the workflow keeps going anyway and
logs the row as if it succeeded.

**Why it matters:** a failed Diligent-path Lead pass can look identical to a successful one in
the audit sheet. There's no reason visible in the export for this path to be handled
differently from the other two — it looks like an inconsistency rather than a deliberate choice.

**Idea:** remove the `continueRegularOutput` override on these two nodes, or if there's a real
reason for it, document that reason somewhere so it doesn't get "fixed" into a regression later.

## 4. Debounce/concurrency limiter can leak until it fails permanently "open" or "closed"

There's a genuine backpressure mechanism here — an n8n Data Table (`LPA Debounce`) tracks
in-flight Lead IDs; if the count exceeds a hardcoded ceiling of **40**, new Leads route to an
external proxy (`prxy.civicplusai.solutions/debounce`) to be delayed. A row gets added when
processing starts and removed (`Delete Debounce`) when it finishes.

**Why it matters:** if any execution errors or crashes before it reaches its `Delete Debounce`
step, that Lead's row is never removed. There's no visible TTL or expiry on these rows. Over
time, orphaned rows can only accumulate — eventually every new Lead looks "too busy" even when
the system is idle, and nothing about this would be obvious without someone thinking to check
the table's row count directly.

**Idea:** add a TTL/expiry to debounce rows (or a scheduled cleanup workflow), so a crashed
execution can't permanently reduce available throughput.

## 5. Business rules hardcoded into n8n code, not a system of record

`StreamlineRoutingRules` hardcodes roughly 25 Account segmentation codes directly in a
JavaScript code node. Its own comments show it's already been patched once — it used to key off
`Industry_Segmentation__c`, now keys off `Account.Industry`, with a carve-out exception layered
on top for Local Government + Website/Accessibility leads.

**Why it matters:** this is the n8n-side twin of the hardcoded named reps already found inside
`PassInboundLead` (see `PassInboundLead-Flow`). Any new segmentation code added in Salesforce
won't route to Streamline correctly until someone remembers this specific code block exists and
edits it. Two completely different systems (a Flow and an n8n workflow) both have this same
"business logic quietly embedded in code, no single place to look" problem.

**Idea:** same fix as the hardcoded reps — pull segmentation-to-pool mapping into a Custom
Metadata Type or a config sheet that both systems can read, rather than duplicating hardcoded
logic in two different places that can drift from each other.

## 6. Diligent `Reason` is a fixed string, not computed per Lead

Every Diligent-routed Lead gets the exact same hardcoded reason —
`"Diligent Partner - DocAccess partner lead (cannot pass)"` — regardless of the specific
circumstance that triggered the routing.

**Why it matters:** anyone reading Lead history later to understand why a specific Lead landed
in Diligent is reading a canned explanation, not the real one. Low severity on its own, but
compounds with #3 above — a Diligent Lead that failed silently *and* has a generic reason string
gives almost no signal that anything went wrong.

## 7. The webhook has no visible authentication

Neither `Webhook` trigger node (production `lpa` path or the shadow UUID path) has an
authentication method configured in the export — no header secret, no basic auth. Production's
path is the short, guessable `lpa`.

**Why it matters:** if this is genuinely unauthenticated at the application layer, anyone who
finds the URL could POST an arbitrary Lead ID and trigger real Salesforce writes — Engagement
Record creation, ownership changes — with no credential check. **This may already be handled at
an infrastructure layer we can't see from the export** (a reverse proxy, an IP allowlist, a WAF
rule) — worth confirming rather than assuming either way.

**Idea:** if it isn't already covered elsewhere, add a shared-secret header check as the first
step after the webhook fires, and reject anything that doesn't match.

## 8. The LLM's routing decision is a loosely-validated free string

The `action` field the qualification/routing agents produce is plain text, matched only by
convention against the values `PassInboundLead`'s `ActionRouter` expects. The Structured Output
Parser has `autoFix: true`, which silently repairs malformed JSON rather than surfacing an
error.

**Why it matters:** a slightly-off or hallucinated Action value from the model wouldn't raise an
error anywhere in the pipeline — it would just fail to match any branch in `ActionRouter` and
fall through to the Default Outcome, with nothing connecting the bad output to the miss.

**Idea:** validate the model's `action` output against the literal known-good list *in n8n*,
immediately after parsing — fail loud there, where it's cheap to catch, rather than letting a
bad value travel all the way to Salesforce before quietly doing nothing.

## One thing that's working as intended

The LLM agents' only direct Salesforce access is through an MCP tool scoped to **read-only**
operations (`run_soql`, `run_sosl`, `get_salesforce_object_schema`) — no create/update/delete
tools are exposed to the model. This matches what Holden Ruch described in Slack: the AI
genuinely cannot write to Salesforce directly, and every real write does funnel through
`PassInboundLead`. Worth knowing this part of the design held up under inspection.

## Open questions for whoever maintains `PassInboundLead`

These came out of the same review but need a person to answer, not more file-reading:
- Exactly which `Get*Owner` lookups have hardcoded rep names, and which are dynamic? (Still
  unresolved from the Kevin Gallagher → Abraham Amidon example — see `PassInboundLead-Flow`.)
- Is this Flow's transaction model fully atomic? If it fails partway through — after creating
  the Engagement Record but before updating the Lead, say — does everything roll back, or can a
  Lead end up half-processed?
- Where do this Flow's error emails go? If it's Holden's personal inbox — consistent with the
  personal-user Salesforce auth — that's a monitoring single point of failure stacked on top of
  the authentication one.

## Improvement ideas for `PassInboundLead`

- Move hardcoded rep names into a Custom Metadata Type or Custom Setting — turns a reassignment
  into a data edit instead of a Flow edit, and could in principle be read by n8n too, so both
  systems draw from one source instead of two independently-maintained ones.
- Add an explicit "unrecognized Action" branch in `ActionRouter` that does something visible (a
  Task, a distinct Lead status) instead of silently falling to the default outcome — turns a
  silent miss into a debuggable one.

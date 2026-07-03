# Headless360 Workshop — Approach Comparison

Running the **same 15-prompt sales workflow** against one live Salesforce org, three ways, to see how the experience and results differ:

- **Iteration 1 — Default Claude tools:** org access via the connected `sf` CLI (SOQL, `apex run`) through Bash. No MCP. *(Assumes a developer workstation — see the [README](README.md) for why that matters.)*
- **Iteration 2 — Standard Salesforce MCP server:** the same prompts run through the Salesforce-hosted `sobject-all` MCP server (schema, SOQL, aggregates).
- **Custom MCP server (Prompts 14–15 only):** an org-built server exposing this org's own logic (Account Health Check, Sync Meeting) as callable tools.

**Org:** `epic.ee57ef6782f0@orgfarm.salesforce.com`. Iteration 1 started 2026-07-01; Iteration 2 run 2026-07-02 (**blind** — no access to Iteration 1's findings during the prompts, for a fair comparison).

This document has two parts:
- **Part A — Per-prompt comparison (side by side):** each prompt with both iterations next to each other + a verdict. *(This merges what was previously two files — the Iteration-1 log and `iteration-2-results.md`.)*
- **Part B — Comparison summary:** the overall analysis, scorecard, scored dimensions, recommendations, the sales story, the MCP benefits, and the business-user POV.

> The short, leadership-facing version lives in the **[README](README.md)**.

---

# Part A — Per-prompt comparison (side by side)

_For each prompt: Iteration 1 (default `sf` CLI tools) vs. Iteration 2 (standard MCP). Prompts 14–15 also cover the custom MCP server. "Steps" counts tool calls. Both runs reached the same domain conclusions independently, using their own invented definitions where the prompt was open-ended — differences in numbers below are definitional, not correctness gaps._

### Prompt 1 — Most relevant objects & fields

> What Salesforce objects and fields are most relevant for understanding Sales Cloud activity in this org? Focus on Account, Lead, Opportunity, Campaign, CampaignMember, Task, and Contact.

| | Iteration 1 — Default tools (`sf` CLI) | Iteration 2 — Standard MCP |
|---|---|---|
| **Tools & steps** | 2 calls: `sf sobject describe` (probe field counts) + a combined describe piped through Python to filter fields | 2 calls: `getObjectSchema` (object index) + `getObjectSchema` for the 7 objects |
| **Result** | Field counts per object (Account 49, Lead 50, Opp 39, Campaign 33, CampaignMember 36, Task 39, Contact 66); themed field lists; surfaced 3 custom fields (`Partner_Tier__c`, `Quarterly_Quota__c`, `Partner_Referral__c`) | LLM-ready field lists with types + picklist values; same partner custom fields surfaced automatically; identified the partner/channel motion |
| **Effort / notes** | Had to know the CLI describe command, write a filter to reduce hundreds of fields, and dump a huge polymorphic `WhatId` list; relevance judgment entirely manual | Low effort — schema tool returns relevance-shaped lists directly. One wrinkle: 132 KB payload truncated, needed a follow-up for Contact |

**Verdict:** *Slight edge — standard MCP.* Same picture (incl. the same custom fields), but the schema tool returns fields already shaped for relevance, so less manual filtering.

### Prompt 2 — Key fields by theme

> Describe the key fields on Account, Lead, Opportunity, Campaign, CampaignMember, and Task. Call out fields that help explain partner attribution, pipeline health, campaign follow-up, or sales activity.

| | Iteration 1 — Default tools | Iteration 2 — Standard MCP |
|---|---|---|
| **Tools & steps** | 1 call: Python pulling live picklist values for theme-relevant fields; reused P1 schema | 0 new calls — reused the P1 schema detail |
| **Result** | Four themed groups with live picklist values (e.g. Campaign Type includes `Partners`/`Referral Program`); partner fields form a referral→tier→deal-credit chain | Same four themes mapped to concrete fields; three of four themes lean on custom fields |
| **Effort / notes** | Needed to know which fields carry picklists worth pulling; theme grouping is analytical | Zero extra calls — one describe answered both "what fields exist" and "which serve which question" |

**Verdict:** *Standard MCP.* One schema call served both prompts; Iteration 1 still needed a query for picklists.

### Prompt 3 — Accounts sample by state/industry/type/employees

> Show me a sample of Accounts by state, industry, type, and employee count.

| | Iteration 1 — Default tools | Iteration 2 — Standard MCP |
|---|---|---|
| **Tools & steps** | 1 query, `LIMIT 30`; distribution eyeballed | 3 queries incl. `GROUP BY` for state and industry×type |
| **Result** | Returned 30 rows and **mistook them for the whole org**; approximate distribution | True distribution over **all 152 accounts** (NV 33, CA 30, CO 22…); partner ~10% |
| **Effort / notes** | Mapped NL field names to API names; `LIMIT 30` masqueraded as "full org" — corrected two prompts later | `GROUP BY` did the counting server-side; self-consistent (sums to 152) |

**Verdict:** *Standard MCP.* Server-side aggregation gave a more accurate first answer; the default run undercounted the org.

### Prompt 4 — Nevada priority accounts

> Which Nevada accounts have the largest employee counts and active open opportunities? Summarize why each account may be a good sales priority.

| | Iteration 1 — Default tools | Iteration 2 — Standard MCP |
|---|---|---|
| **Tools & steps** | 1 parent-child SOQL (NV accounts + open opps) | 1 relationship subquery (33 NV accounts + nested open opps) |
| **Result** | Tiered priorities; top picks Xavier Road, Cactus Valley, Everline, Laurel Terrace; deprioritized Partner-type (no opps) | Same top picks and same reasoning; Partner-type accounts correctly excluded; identified the physical-security product theme |
| **Effort / notes** | Hand-wrote the parent-child SOQL, knew the child relationship name; also revealed the org has 34+ NV accounts (contradicting P3's "30") | One subquery returned accounts + open opps together, no manual joining |

**Verdict:** *Tie.* Same query pattern, same conclusions.

### Prompt 5 — Security-equipment buyers (CA/NV/AZ/CO)

> Find accounts in California, Nevada, Arizona, and Colorado that look like security equipment buyers. Use account fields, descriptions, and related opportunity context to explain your reasoning.

| | Iteration 1 — Default tools | Iteration 2 — Standard MCP |
|---|---|---|
| **Tools & steps** | 2 calls (+1 retry from a Python typo); 101 accounts + Description keyword scan | 1 call; 88 accounts (those with opps) + Description + opp context, parsed offline |
| **Result** | **Whole org = security buyers**; the buying signal lives in the structured `Description` ("current footprint… next motion…"); tiered strongest buyers | Same core finding; footprint mix quantified (15 "legacy keys" = greenfield); top pick **Falcon Gate Manufacturing** |
| **Effort / notes** | Needed a keyword regex; had to read free-text to realize the signal lived there; one code error cost a retry | One query returned everything; the free-text `Description` carried the richest signal |

**Verdict:** *Tie / slight standard MCP* (no retry). Both discovered the whole book is security buyers and the signal is in the Description.

### Prompt 6 — At-risk open opportunities

> Find open opportunities closing soon that have weak next steps, no recent activity, or overdue related tasks. Summarize why each deal may be at risk.

| | Iteration 1 — Default tools | Iteration 2 — Standard MCP |
|---|---|---|
| **Tools & steps** | 4 calls (aggregate + 2 pulls + Python scoring); anchored "present" to ~Dec-15 | 1 call over 229 open opps, analyzed offline |
| **Result** | 27/64 scored at-risk; **templated `NextStep`** across all deals; ~30 zero-activity; top: Juniper Hill $250K | 41-deal at-risk cohort; **44% have zero logged activity**; same templated-NextStep + `HasOverdueTask`/`PushCount`/`LastStageChangeDate` all unpopulated; same top deals |
| **Effort / notes** | Heavy manual assembly; **detected the offset demo dates** (a naive 90-day filter returns zero) | One query; key move was verifying which risk fields are even populated (most aren't) and pivoting to activity-absence |

**Verdict:** *Tie.* Iteration 1 was more rigorous on the date-offset trap; MCP reached the same findings in far fewer calls.

### Prompt 7 — Underworked hot/warm leads

> Which hot or warm leads look underworked? Group them by lead source, campaign, state, and partner referral.

| | Iteration 1 — Default tools | Iteration 2 — Standard MCP |
|---|---|---|
| **Tools & steps** | 2 calls (profile + pull/score/group) | 2 calls (funnel shape + 263-lead pull), grouped offline |
| **Result** | 166/263 underworked (looser definition), 93 hot; grouped by source/campaign/state/partner; 29 partner-referred | 94/263 underworked (tighter definition), 50 hot; **0 of 94 partner-referred — all 69 partner leads are worked** |
| **Effort / notes** | Had to invent "underworked"; counted each lead's first campaign only (a silent simplification) | Relationship traversal returned all four groupings from one result set; verified the partner-referral finding |

**Verdict:** *Tie.* Different (defensible) definitions; MCP's "the gap is entirely non-partner leads" cut is a sharper insight.

### Prompt 8 — Campaign follow-up gap

> Find campaigns with strong engagement but limited sales follow-up. Include the campaign name, related leads or campaign members, and the follow-up gap.

| | Iteration 1 — Default tools | Iteration 2 — Standard MCP |
|---|---|---|
| **Tools & steps** | 3 calls (+1 retry on a nested-field JSON gotcha) | 3 calls (rollups + two filtered join-aggregates) |
| **Result** | Engagement uniform across 18 campaigns; **0/336 responders ever converted**; worst gaps named + individual dropped leads listed | **9 campaigns at a 100% follow-up gap**; every campaign ≥47%; gap computed cleanly via join-aggregate |
| **Effort / notes** | Relationship-field JSON trap (`Campaign.Name` nested) caused a retry; invented both "engagement" and "gap" | Metric had to be *built* from the CampaignMember→Lead join server-side; ratios exact |

**Verdict:** *Tie.* Same systemic finding; Iteration 1 named names, MCP quantified more cleanly.

### Prompt 9 — Top risk: 360° records + next question

> For the highest-priority risk you found, show the related records and recommend the next question I should ask before taking action.

| | Iteration 1 — Default tools | Iteration 2 — Standard MCP |
|---|---|---|
| **Tools & steps** | ~3 calls (+1 error on a bad relationship path) stitched in Python | 4 calls (Account, opps, contacts, tasks) |
| **Result** | Caught contradictions: created *yesterday* yet at 90%/Negotiation, no activity, no primary contact → "is the stage even real?"; found a likely buyer + two blocking cases | Caught the Task allocation: account looks active (8 tasks) but **all tasks are on the small deals; the $250K flagship is untouched** |
| **Effort / notes** | Assembling a 360 view took ~6 hand-written queries; hit a relationship-path error needing a debug round-trip | Drill-down via `WhatId` exposed a risk hidden at the account-rollup level |

**Verdict:** *Tie.* Complementary discoveries; both landed on "is the 90% real?"

### Prompt 10 — Bar chart: open pipeline by stage

> Create a bar chart that compares open pipeline by opportunity stage. Use the live opportunity data you can access through Salesforce MCP tools.

| | Iteration 1 — Default tools | Iteration 2 — Standard MCP |
|---|---|---|
| **Tools & steps** | 1 aggregate query → ASCII chart, then `pip install matplotlib` + script + render (~4 extra calls) | 1 aggregate query (1 timeout, retried) + matplotlib render |
| **Result** | $24.98M across 229 opps; Proposal/Price Quote the largest ($6.27M) | Identical numbers; chart at `workshop-assets/iteration-2/p10-open-pipeline-by-stage.png` |
| **Effort / notes** | Real PNG required installing a plotting lib first | Data cleaner to get; **neither MCP server draws charts — Claude's matplotlib does** |

**Verdict:** *Slight standard MCP.* Same result, cleaner data path. Key myth-buster: visualization is the model's tooling, not the MCP server.

### Prompt 11 — Pie chart: hot/warm open leads by source

> Create a pie chart that shows hot and warm open leads by lead source.

| | Iteration 1 — Default tools | Iteration 2 — Standard MCP |
|---|---|---|
| **Tools & steps** | 1 aggregate + matplotlib (already installed) | 1 aggregate + matplotlib |
| **Result** | 263 hot/warm open leads (Partner 26% largest) | 73 leads with `Status='Open'` (Partner 30% largest) |
| **Effort / notes** | Different denominator (all hot/warm open) | Different denominator (`Status='Open'` only); flagged `LeadSource` vs `Partner_Referral__c` |

**Verdict:** *Tie.* Both rendered a PNG; the count differs only because each chose a different definition of "open."

### Prompt 12 — Timeline: campaign response activity

> Create a line chart or timeline that shows campaign response activity over time. Explain any spike or drop you see in the data.

| | Iteration 1 — Default tools | Iteration 2 — Standard MCP |
|---|---|---|
| **Tools & steps** | 3 calls (probed several date fields) + matplotlib | 4 calls (incl. one SOQL error) + matplotlib |
| **Result** | Detected all responses stamped one date (seed); built 2 proxies incl. a **real weekly Task-activity trend** with a Dec-15 seeding artifact | Detected the same single-date seed; pivoted to `Campaign.StartDate`; reported flatness as the finding |
| **Effort / notes** | Avoided the trap of charting the seed-date field and inventing a fake spike | Same trap avoided; Iteration 1 additionally surfaced a genuine activity trend |

**Verdict:** *Tie / slight Iteration 1.* Both dodged the fake-spike trap; the default run found one extra real trend.

### Prompt 13 — One-page sales-leader PDF

> Create a one-page PDF summary for a sales leader. Include the strongest account signals, the riskiest open opportunities, missed lead or campaign follow-up, and the next questions a seller should ask.

| | Iteration 1 — Default tools | Iteration 2 — Standard MCP |
|---|---|---|
| **Tools & steps** | matplotlib PDF backend + 1 confirming aggregate (+1 bug retry) | 1 confirming aggregate + matplotlib PDF |
| **Result** | `workshop-assets/sales-leader-summary.pdf`; confirmed **152 accounts** | `workshop-assets/iteration-2/p13-sales-leader-summary.pdf`; verified 1 page via pypdf |
| **Effort / notes** | Hand-built layout coordinate-by-coordinate; couldn't preview (no poppler) | Same tooling constraints; pure synthesis of prior prompts |

**Verdict:** *Tie.* Both synthesize prior findings with the same tooling limits; the MCP server plays no role in PDF generation.

### Prompt 14 — Account health (Arroyo Medical Group)

> Find the Account named Arroyo Medical Group and check the account health.

| | Iteration 1 — Default tools | Iteration 2 — Custom MCP |
|---|---|---|
| **Tools & steps** | **14:** hand-wired anonymous Apex invoking the 3 scoring classes. **14b (fair baseline):** raw-data pull + self-devised rubric (no Apex) | 2 calls: find the account + `Account_Health_Check_Calculator` |
| **Result** | **14:** 56.67/100 (Activity 0/30). **14b:** qualitative "healthy-but-watch," **opposite** engagement read — spotted real Dec-2026 activity the Apex's `Date.today()` misses | **56.67/100** with a 30/30/40 breakdown — reproduces the Apex's date-blind Activity 0/30 |
| **Effort / notes** | Needs deep Apex knowledge *or* an invented rubric; not repeatable | One governed call returns the org's own health definition — **but inherits the date-blind bug that 14b caught** |

**Verdict:** *Custom MCP* (for convenience & standardization). One call vs. Apex expertise or a from-scratch rubric — with the caveat that a custom tool is only as correct as its internal logic.

### Prompt 15 — Create a sync meeting *(production write)*

> Create a sync meeting for the account.

| | Iteration 1 — Default tools | Iteration 2 — Custom MCP |
|---|---|---|
| **Tools & steps** | 3 calls + a confirmation gate; gathered owner/cases/contact, hand-built the `Task` | 1 call `CreateAccountSyncMeetingTask` + 1 read-back to verify (user-confirmed) |
| **Result** | Task `00Tbm00000DIuqfEAD`, contact Alex Garcia (chosen deterministically), open cases referenced | Task `00Tbm00000DKQWjEAP`, **also Alex Garcia**, open cases auto-embedded, due +7, owner-assigned |
| **Effort / notes** | Had to know the org's "sync meeting" convention and build the record field-by-field | One call encapsulated contact selection, case lookup, description, and defaults |

**Verdict:** *Custom MCP.* One governed call vs. multi-step assembly; notably both runs independently picked the same contact and cases.

---

# Part B — Comparison summary

> **This is the deliverable** — the overall analysis distilled from Part A above. Iteration 2 was run **blind** (no access to Iteration 1 during the prompts), so some numbers differ by design. Prompts 14–15 were run against the **custom MCP server only** in Iteration 2 (the manual/default-tools path for these is covered by Iteration 1's 14b/15).

### 1. Executive summary

For **read/analysis prompts (1–13), standard MCP and default tools reach essentially the same answers** — both need Claude to supply all the *interpretation* ("at-risk," "underworked," "buyer") — but **standard MCP gets there with far fewer steps and workarounds**: the server handled the querying, grouping, and joining of data that Iteration 1 had to build by hand (hand-written queries, data-format errors that caused retries, passing record IDs between steps, and extra scripting to combine results). The **single biggest differentiator is the custom MCP server (Prompts 14–15)**: it collapses work that took Iteration 1 either deep Apex knowledge (hand-wired anonymous Apex) or a from-scratch rubric into **one governed tool call** that returns the org's *own* definition of "account health" and its *own* "sync meeting" action. The trade-off to teach: that convenience is **opaque and can hide data-quality problems** — the custom health tool faithfully reproduced the Apex's date-blind `Activity 0/30` (scoring the account 56.67/100) even though raw data shows the account was actively engaged, a trap Iteration 1's raw-data baseline (14b) actually caught.

### 2. Per-prompt scorecard

| # | Prompt (short) | Iter 1: default tools | Iter 2: standard MCP | Iter 2: custom MCP (14–15) | Winner | Why |
|---|----------------|-----------------------|----------------------|----------------------------|--------|-----|
| 1 | Relevant objects/fields | 2 `sf describe` + Python filter; field counts; 3 custom fields | 2 `getObjectSchema`; LLM-ready fields+picklists; same 3 custom fields; 132KB truncation → 1 follow-up | — | **Std MCP (slight)** | Schema tool returns relevance-shaped field lists; less manual filtering |
| 2 | Key fields by theme | 1 Bash picklist extract | 0 new calls (reused P1 schema) | — | **Std MCP** | One describe served both prompts; 0 extra calls |
| 3 | Accounts sample | 1 query; **LIMIT 30 mistaken for full org** | 3 queries incl `GROUP BY`; true 152-acct distribution | — | **Std MCP** | Server-side aggregation gave real distribution; Iter 1 undercounted the org |
| 4 | NV priority accounts | 1 relationship query | 1 relationship subquery | — | **Tie** | Same parent-child pattern, same picks (Xavier Rd, Everline, Cactus Valley, Laurel Terrace) |
| 5 | Security-buyer accounts | 2 calls (+1 typo retry); 101 accts | 1 call; 88 accts (filtered to opp-owners) | — | **Tie / slight Std MCP** | Both found "whole org = security buyers, signal lives in Description"; MCP no retry |
| 6 | At-risk opps | 4 calls; anchored PRESENT=Dec-15; 27/64 scored; templated NextStep | 1 call; today=Jul-02; 100/229 no-activity; 41-deal cohort; same templated-NextStep + unpopulated risk fields | — | **Tie** | Iter 1 more rigorous on date-offset; MCP far fewer calls; same core findings |
| 7 | Underworked leads | 2 calls; 166/263 (looser def); 93 hot | 2 calls; 94/263 (tighter def); 50 hot; **0/94 partner-referred (all 69 partner leads worked)** | — | **Tie** | Different "underworked" definitions; MCP's partner-referral cut is a sharper insight |
| 8 | Campaign follow-up gap | 3 calls; 0/336 converted; named individual dropped leads | 3 calls; 9 campaigns @100% gap; clean gap% via filtered join-aggregate | — | **Tie** | Both: uniform engagement + systemic gap; Iter 1 named names, MCP quantified cleaner |
| 9 | Top risk 360 + next Q | 3 calls (+1 error); caught "created-yesterday+90%" hygiene contradiction | 4 calls; caught Task→Opp allocation (8 tasks, flagship untouched) | — | **Tie** | Complementary discoveries; both landed on "is the 90% real?" |
| 10 | Pipeline bar chart | SOQL→ASCII, then pip-install matplotlib + render (~4 extra calls) | 1 aggregate (1 timeout) + matplotlib render | — | **Std MCP (slight)** | Same $24.98M/229; MCP data cleaner. **Neither MCP renders charts natively — Claude/matplotlib does** |
| 11 | Leads pie chart | 1 aggregate + matplotlib; 263 (all hot/warm open) | 1 aggregate + matplotlib; 73 (`Status=Open` only) | — | **Tie** | Different "open" denominator; both rendered PNG |
| 12 | Response timeline | detected single-date seed; 2-panel incl. real Task-activity trend | detected single-date seed; pivoted to Campaign.StartDate | — | **Tie / slight Iter 1** | Both avoided fake-spike trap; Iter 1 found an extra real trend (Task activity) |
| 13 | Sales-leader PDF | matplotlib PDF; couldn't preview (no poppler) | matplotlib PDF; verified via pypdf | — | **Tie** | Both synthesize prior prompts; same tooling constraints |
| 14 | Account health (Arroyo) | 14: hand-wired Apex → 56.67. **14b (fair): raw-data rubric, no 0–100, reached _opposite_ engagement read** | — | **1 tool call → 56.67/100**, reproduces Apex `Activity 0/30` | **Custom MCP** | One governed call vs. Apex expertise or invented rubric — **but inherits the date-blind bug 14b caught** |
| 15 | Create sync meeting | 3 calls + confirm; hand-built Task; deterministically picked Alex Garcia | — | **1 tool call**; Task w/ Alex Garcia, open cases embedded, due+7 | **Custom MCP** | One call vs. multi-step assembly; both runs independently chose the same contact & cases |

### 3. Scored dimensions (1 = poor, 5 = excellent)

| Dimension | Default tools | Standard MCP | Custom MCP | Notes |
|-----------|:---:|:---:|:---:|-------|
| Setup / auth cost | 4 | 3 | 2 | CLI "just works"; MCP needs per-server OAuth (we hit token/auth setup); custom MCP also needs someone to **build** the server |
| Steps / tool-calls per answer | 2 | 4 | 5 | Iter 1 = query + Python assembly every time; std MCP = direct SOQL/aggregates; custom MCP = one call for a whole task |
| Answer accuracy & completeness | 4 | 4 | 4 | All high on reads; custom MCP is governed but **inherited** the Apex date bug (would be 5 if date-correct) |
| Handling data-quality traps | 5 | 4 | 2 | Iter 1 dug hardest (offset dates, LIMIT cap, templated fields); std MCP caught seeded data; **custom MCP hides the trap** (returned 56.67 without flagging Activity=0 is a date artifact) |
| Interpretation burden (who defines terms) | 2 | 2 | 5 | Default & std MCP return rows — analyst invents every definition; custom MCP **encodes the org's definition** in the tool |
| Visualization / artifact generation | 3 | 3 | 3 | **Charts/PDF are a Claude-harness capability (matplotlib), not an MCP feature** — neither server returns native visuals |
| Write actions (safety, correctness) | 3 | 3 | 5 | Hand-built insert vs. one encapsulated, owner-aware, case-referencing action; still gated by user confirmation |
| Standardization / repeatability | 2 | 2 | 5 | Raw-data rubric varies run-to-run; custom tool returns the same score/record every time |
| Transparency (can you see how it got there) | 5 | 5 | 2 | You read every SOQL in Iter 1/std MCP; custom MCP is a black box (can't see the weighting or why Alex Garcia) |

### 4. Standard MCP vs. default tools — key findings

**Where standard MCP clearly helped:**
- **Much less manual work.** Iteration 1's recurring overhead — hand-written queries, passing record IDs between separate queries, extra scripting to aggregate/score results, and data-format errors (nested and invalid field paths) that cost several retries — **largely disappeared.** The schema tool returned ready-to-use field lists; queries with `GROUP BY` did the counting on the server; and relationship queries returned an account and its child records in one call.
- **Avoided the LIMIT-30 accuracy error.** Iteration 1's Prompt 3 mistook `LIMIT 30` for the full org and had to self-correct two prompts later. Iteration 2 used `GROUP BY` and saw all **152 accounts** immediately — a cleaner, more accurate first answer.
- **Schema tool is genuinely better than raw describe.** One `getObjectSchema` call answered both Prompt 1 and Prompt 2; it's built for LLM consumption (labels + types + picklists), not the verbose `sf sobject describe` dump Iteration 1 had to filter.

**Where standard MCP added no value / hit the same wall:**
- **Interpretation is still 100% Claude's job.** "At-risk," "underworked," "strong engagement," "buyer," "sales priority" — the MCP returns rows, not judgment. Every definition was invented in-agent, exactly as in Iteration 1. Standard MCP is a *better data pipe*, not an analyst.
- **Same data-quality traps still require vigilance.** Seeded/uniform campaign data, the single-date `FirstRespondedDate`, templated `NextStep`, unpopulated hygiene fields (`HasOverdueTask`/`PushCount`/`LastStageChangeDate`) — all still had to be *detected*. (Iteration 1 arguably probed the offset-date "present" more aggressively, anchoring to Dec-15; Iteration 2 reasoned against the real system date. Neither was wrong, but it shows the trap doesn't go away.)
- **No native visualization.** The prompts asking for charts/PDF (10–13) were served by Claude's matplotlib in *both* iterations. The standard MCP server does not emit charts — a common misconception worth correcting for the workshop audience.
- **New overhead of its own:** OAuth setup per server + occasional timeouts (one on Prompt 10) + a 132 KB schema response that was too large in one call and needed a follow-up.

**Net:** for read/analysis work, **standard MCP is an ease-of-use and accuracy upgrade over default tools, not a capability leap.** Same answers, fewer mistakes, less manual assembly.

### 5. Custom MCP server — the differentiator (Prompts 14–15)

This is the "why a custom server" story, and it's the strongest part of the comparison.

- **It collapses multi-step expertise into one call.** *Account health* in Iteration 1 required either (14) knowing the three scoring Apex classes exist and hand-wiring anonymous Apex to invoke them, or (14b) inventing a health rubric from raw records. The custom MCP did it in **one tool call → `56.67/100` with a component breakdown.** *Sync meeting* in Iteration 1 took SOQL for owner/cases/contact + a hand-built `Task` insert; the custom MCP did it in **one call** that auto-selected the contact, embedded the open cases in the description, set owner + due-date defaults, and returned a task ID.
- **It encodes the org's own definitions.** The value isn't data access (standard MCP already has that) — it's that the *business logic* (the 30/30/40 health weighting; what a "sync meeting" task should contain) lives server-side and is applied consistently. This is the one place the interpretation burden shifts **off** Claude and onto a governed, repeatable tool.
- **It reproduced the Apex's date-blind behavior — exactly as Iteration 1 predicted.** The health tool returned `Activity & Engagement 0.00/30` because the underlying Apex uses `Date.today()` while the seeded activity sits in Oct–Dec 2026. Iteration 1's raw-data baseline (14b) reached the *opposite* engagement conclusion (🟢 actively worked) precisely because it could see and reason around the offset. **The convenience of the custom tool hid a data-quality issue that the "harder" manual path caught.** This is the essential teaching nuance: a custom tool is only as correct as the logic inside it.
- **Standardized vs. contextual — which is more useful?** Depends on the consumer. A **sales leader comparing 200 accounts** wants the *standardized, repeatable* 56.67 (custom MCP) so accounts are ranked on one consistent yardstick. A **rep working one account** is better served by 14b's *contextual* read that caught the account is actually engaged. Best practice: use the custom score to triage at scale, then a contextual pull to validate before acting.
- **Write safety held.** The production write (Prompt 15) was gated by explicit user confirmation in both iterations; the custom tool's one-shot convenience did not bypass the confirmation step. Notably, both runs independently picked **Alex Garcia** and the same two open cases — so the Apex's `Math.random()` contact-selection concern from Iteration 1 didn't produce a visible divergence here (Alex Garcia is the contact on the open cases).

### 6. Recommendation & when-to-use guidance

**Reach for default tools (sf CLI + SOQL) when:** you're doing ad-hoc, one-off exploration; you need full transparency and control over every query; you're debugging data quality itself; or no MCP server is set up and the task is quick. Cost: you write all the SOQL and invent all the analysis.

**Reach for the standard MCP server when:** you want the same analytical power with far less manual work — repeated schema/relationship/aggregate work, multi-object joins, or anything you'd otherwise hand-assemble in Python. It's the **default for day-to-day analysis you drive by asking in plain language** once auth is set up. It won't invent business definitions for you, and it won't draw charts by itself.

**Invest in a custom MCP server when:** a task has (a) **org-specific business logic** that should be consistent across users (scoring, eligibility, health, pricing), (b) **a repeatable action** with embedded rules (create X the way we always create X), or (c) a workflow you don't want every analyst re-deriving. The payoff is one-call, governed, standardized outcomes. **What to wrap first:** your highest-value "everyone computes this slightly differently" metric (here: Account Health) and your most common structured write (here: the sync-meeting task). **Guardrails to build in:** make the logic date-aware/parameterized (the offset bug), and return enough detail (component sub-scores, chosen contact, referenced cases) that the black box stays auditable.

**Rule of thumb for the workshop:** *Standard MCP makes Claude a faster analyst. A custom MCP makes Claude act with your organization's judgment.*

### 7. Caveats

- **Iteration 2 was run blind** (no Iteration 1 access during the prompts) and reached some different numbers by design — different but defensible definitions of "underworked" (94 vs. 166) and "open" leads (73 vs. 263), and a different CA/NV/AZ/CO account count (88 with-opps vs. 101 all). These are definitional, not correctness, gaps — and they *illustrate* the recurring theme that interpretation lives in the agent, not the tool.
- **Prompts 14–15 are not a clean 3-way in Iteration 2.** By decision, only the custom MCP was run there; the default-tools comparison for these draws on Iteration 1's 14b/15. The standard-MCP column for 14–15 is intentionally blank (a standard server has no health/sync-meeting tool — you'd fall back to raw-data assessment, i.e. 14b).
- **Synthetic / offset demo data** underlies everything: tiered employee counts, uniform ~44-member campaigns, all responses stamped 2026-12-15, activity dated Oct–Dec 2026 vs. a Jul system clock, "Workshop Support:" case titles. Conclusions about *this org's* health are illustrative, not real.
- **Production-org constraints:** Prompt 15 wrote a real `Task` in both iterations (IDs `00Tbm00000DIuqfEAD` in Iter 1, `00Tbm00000DKQWjEAP` in Iter 2). Both were user-confirmed.
- **Repo Apex was treated as a black box in Iteration 2** (per instruction) — the custom-MCP findings are based only on tool inputs/outputs, not on reading the scoring classes.
- **Environment limits, not MCP limits:** no poppler (couldn't preview PDFs), reportlab absent (used matplotlib), one MCP timeout, one schema truncation. Worth separating from the tool comparison itself.

---

## 8. The sales story behind the prompts (workshop narrative intro)

_For the hands-on storybook. This is the human story the 15 prompts tell — use it to frame the session before diving into tools._

**The company.** Our org is a **physical-security vendor** — badge readers, access-control systems, security cameras, visitor-management kiosks, smart locks, and maintenance plans — selling B2B across the **US West and Southwest** (Nevada, California, Colorado, Washington, Arizona, Utah lead the book). About **152 accounts**, a **$24.98M open pipeline** across 229 opportunities, 18 marketing campaigns, and a **partner/channel motion** layered on top (partner tiers, partner referrals, partner-sourced deals).

**The protagonist.** A **sales leader (and their reps)** starting a territory review. They don't yet know the data, the priorities, or where revenue is leaking. Over 15 questions they go from *"what's even in here?"* to *"here's the one action I'm taking today."* That arc is the story:

| Act | Prompts | The rep's real question | What they're doing |
|---|---|---|---|
| **I. Orient** | 1–2 | "What can I even ask about? What do these fields mean?" | Learn the data model & the signals that matter (partner, pipeline, campaign, activity) |
| **II. Survey the territory** | 3–5 | "What's my book of business, and who should I care about?" | Lay of the land → top priorities in NV → which accounts are real buyers |
| **III. Find the leaks** | 6–8 | "Where am I about to lose money or waste demand?" | At-risk deals → underworked hot leads → campaigns marketing ran but sales never followed up |
| **IV. Diagnose** | 9 | "What's my #1 problem, and what should I ask before I act?" | 360° drill-down on the single riskiest deal + the smart next question |
| **V. Communicate** | 10–13 | "How do I show this to my team and my boss?" | Pipeline chart, lead-source chart, campaign trend, one-page exec PDF |
| **VI. Act** | 14–15 | "Is this specific account healthy, and let me set up the follow-up." | Check account health → create the sync-meeting task |

**The dramatic tension the data reveals.** Independently, both iterations surfaced the same story: **the pipeline looks busy, but it's leaking at the seams.** There's real money in play ($24.98M), but **~$4M sits in late-stage deals with zero logged activity**, **~half of hot/warm leads are never worked**, and **every campaign leaks responders sales never called** — while the *partner channel* is the one motion being worked reliably. It's a classic **"marketing generates, sales doesn't follow through, and the forecast is quietly inflated"** narrative — exactly the kind of thing a leader wants an AI copilot to surface fast.

**Why this arc is the right teaching vehicle.** The first 13 prompts show Claude as an **analyst** — and let you contrast *default tools vs. standard MCP* (same insight, less manual work). The last 2 prompts show Claude as an **operator** — and reveal *why a custom MCP server matters*: turning "check this account's health" and "set up the meeting" into one governed call using the company's **own** logic. The story climaxes exactly where the custom server earns its keep: **the moment insight becomes action.**

---

## 9. The benefits of the MCP approach (discussion)

_Requested explicitly — a standalone case for MCP, distilled from the run for the exec summary and storybook._

**A. Standard MCP — a better pipe between Claude and Salesforce.**
1. **Less plumbing, fewer footguns.** Server-side SOQL, aggregation, and relationship traversal replaced Iteration 1's hand-written queries, `/tmp` ID-passing, and inline-Python assembly — and eliminated the JSON-shape and relationship-path errors that cost Iteration 1 several retries.
2. **Purpose-built schema discovery.** `getObjectSchema` returns LLM-shaped, picklist-annotated field lists — one call served two prompts, versus filtering verbose `sf sobject describe` output.
3. **More accurate first answers.** `GROUP BY` gave true distributions immediately (all 152 accounts), avoiding Iteration 1's `LIMIT 30`-as-"full org" mistake.
4. **Governed access.** OAuth + hosted server means org permissions/sharing are enforced by Salesforce, not by whatever the CLI session can reach — better for scaled, multi-user rollouts.
5. **Honest limits:** it doesn't invent analytical definitions, doesn't dodge data-quality traps, doesn't draw charts, and adds its own setup (per-server OAuth) and occasional timeouts.

**B. Custom MCP — the real leap: organizational judgment as a tool.**
1. **Encapsulated business logic.** The org's *own* health formula and sync-meeting convention run server-side and consistently — the one place interpretation moves off the model and onto a governed tool.
2. **One call replaces expertise.** No need to know the scoring Apex exists or how to wire it; "check the account health" and "create a sync meeting" each became a single tool call producing a standardized, decision-ready result.
3. **Standardized & repeatable at scale.** Every account is scored on the same yardstick — essential when a leader ranks hundreds of accounts, where an ad-hoc rubric would drift run-to-run.
4. **Safe, structured actions.** Writes carry embedded rules (owner assignment, case-referencing description, sensible defaults) and still respect a human confirmation gate.
5. **The caveat that makes it credible:** a custom tool is only as good as the logic inside it. Ours faithfully reproduced a **date-blind** score (Activity 0/30) that the manual baseline caught — so custom servers must be **date-aware/parameterized and auditable** (return the sub-scores and the choices they made), or convenience becomes a silent accuracy risk.

**One-line takeaways for the deck:**
- *Default tools:* maximum control, maximum manual work.
- *Standard MCP:* same analysis, much less manual work — Claude as a faster analyst.
- *Custom MCP:* your organization's judgment, one call away — Claude as an operator. **Build it for the metrics and actions everyone re-derives today.**

---

## 10. POV: the business user on Claude web (not Claude Code on a dev workstation)

_This reframes the whole comparison for the audience that actually matters most — non-technical business users._

**The hidden assumption in Iteration 1.** Iteration 1's entire "default tools" baseline exists **only because the run happened on a pre-configured developer workstation** that had the `sf` CLI authenticated, plus Bash, Python, and (after a `pip install`) matplotlib. That is a *developer's* environment. It is **not** what a sales manager, RevOps analyst, or sales leader has.

**What the business user actually has.** A **web chat frontend** (e.g. Claude on the web / a desktop app) with an AI agent, and the ability to **connect MCP servers** that an admin has approved. No terminal. No `sf` CLI. No `apex run`. No Bash. No Python. No `pip install matplotlib`. For this person, **the "default tools" column doesn't exist** — the honest baseline isn't "hand-write SOQL," it's **copy-paste from reports, export CSVs, or ask someone in IT.**

### What changes when there's no CLI

| | Developer on Claude Code (Iteration 1 premise) | **Business user on Claude web (real world)** |
|---|---|---|
| Live org data without MCP | Yes — via `sf` CLI/SOQL | **No** — no CLI, no query tool at all |
| Hand-write SOQL / debug data formats | Possible (it's the manual work we measured) | **Not realistically** — not their skill or their surface |
| Anonymous Apex (Iter-1 Prompt 14) | Possible (hand-wired the scoring classes) | **Impossible** |
| Build a Task record field-by-field (Prompt 15) | Possible via `data create` | **Impossible** without a tool |
| Charts / PDF | Yes — but needed `pip install` + scripting | Depends on the web surface's own artifact features (still the *harness*, not MCP) |
| Get to an answer at all | Effortful but doable | **MCP is the enabler, not the optimizer** |

### The reframed conclusion for business users

1. **For a business user, standard MCP doesn't just save steps — it's the way in.** On a dev workstation, MCP *saved steps* over the CLI. On the web, there is no CLI to save steps over: **MCP is the only practical way to reach live Salesforce data at all.** The comparison shifts from "MCP vs. CLI" to "**MCP vs. nothing / manual exports**."

2. **Custom MCP goes from "nice differentiator" to "the only way they can act."** Iteration 1 reached the account-health score by *hand-wiring anonymous Apex* and created the sync-meeting by *hand-building a Task* — both flatly impossible for a non-technical user. The custom server turns those into a sentence: *"check this account's health," "set up the sync meeting."* For this audience the custom tool isn't a shortcut past SOQL — it's the **difference between "can do it" and "cannot do it at all."**

3. **The interpretation burden matters even more — and cuts both ways.** Claude still supplies the analytical judgment ("at-risk," "underworked"), which is *more* valuable when the user can't fall back on writing their own query. But the flip side is real: a business user **cannot sanity-check the tool the way Iteration 1 did.** They won't notice a `LIMIT`-capped result, won't catch the offset-date bug that zeroed the health score, won't spot templated `NextStep` fields. **This raises the bar on tool quality and transparency** — the custom health tool must expose its sub-scores and be date-correct, precisely because the business user can't audit it by hand.

4. **The web + hosted-MCP model is actually the *better-governed* one for this audience.** A CLI session inherits whatever broad access the workstation was configured with. A business user on web connecting an **admin-approved, OAuth-scoped, Salesforce-hosted MCP server** gets access that IT controls centrally, with sharing rules enforced server-side. For scaled, non-technical rollouts, **hosted MCP is more secure and more manageable than handing people a CLI.**

### Deployment nuances to flag for the workshop
- **Not every MCP tool works on every surface.** Some servers require interactive/browser auth and may not run in headless or automated contexts — fine for an interactive web user, a caveat for background jobs.
- **Admin gating is a feature, not a bug.** On web, connecting a server is an approve-and-configure step, not a `pip install` — this is what makes it safe for non-technical users, but it means **IT/admin enablement is the real prerequisite**, replacing the "workstation setup cost" from the dev scenario.
- **Charts/PDF still come from the model's harness**, whose artifact capabilities vary by surface — set expectations accordingly (the MCP server never draws the chart).

### Bottom line
On a developer's Claude Code workstation, the story is *"MCP makes Claude a faster analyst."* For the **business user on Claude web — the real target audience — the story is stronger: without MCP there is no live-data workflow at all, and without a custom MCP server there is no way for a non-technical user to execute the org's own logic safely.** MCP is what makes Claude usable by the people who most need it and can least script around its absence — provided the tools are governed, transparent, and correct, because those users can't check the work by hand.

---

## 11. Anticipated objections & responses (workshop prep)

_Likely pushback from a technical or skeptical audience, with honest responses. The pattern that works: **concede the real part, locate the concern correctly, then connect it back to the recommendation.** Don't over-claim — several of these rest on a single un-benchmarked run._

### Objection 1 — "MCP floods the context window; code-use (sf/bash/Python) keeps big data out of the LLM"

**The objection (steel-manned):** A code-capable agent runs `sf data query > result.json`, previews a few rows, and processes the file with Python — so large result sets never enter the context window, and the *processing* is deterministic code, not the LLM reading tokens. A naive MCP client instead injects the entire tool result into context ("context pollution") and "processes" it by having the LLM read it. This is a real, documented concern (it's why Anthropic has published guidance on code execution with MCP).

**Response (three layers):**
1. **Concede the real part.** Dumping large tool results into context *is* a genuine problem worth designing against. Not FUD.
2. **Locate it correctly — it's a property of the *client/harness*, not of MCP.** A code-capable client truncates + offloads + processes large results with code — whether they came from the CLI *or* an MCP tool. A minimal client would pollute context by blindly injecting a full result — but equally from a full `sf` CLI stdout. **Both our iterations ran in Claude Code, which truncated large results to a ~2 KB preview, wrote the full payload to a file, and let me process it with Python/grep — including the MCP results.** So the harness converged the two approaches; our run does *not* isolate this effect (an honest confound to state openly).
3. **Push-down usually makes the result small anyway.** Well-formed SOQL sends aggregation to Salesforce (`GROUP BY`, `COUNT`, `SUM`, `LIMIT`, selective fields), so most results come back tiny regardless of transport — in our run, 152 accounts became ~11 rows *in Salesforce*, not in context. That's the reason the MCP path used fewer calls and a smaller window overall here.

**Turn it into the point:** The strongest antidote to context pollution is a **custom MCP server that returns a computed *summary*, not raw rows** — exactly like the Account Health tool returning one score + a breakdown instead of hundreds of activity/opportunity/case records. The heavy data never touches the context window at all. **So this objection is actually an argument *for* custom, summary-returning MCP tools — not against MCP.**

**Honest caveats to keep credibility:**
- Not every MCP result in our run was small: several `soqlQuery` calls returned large blobs (P5 ~90 KB, P6 ~120 KB, P7 ~248 KB, P1 schema 132 KB and truncated). Those are exactly the "big query result" cases the objection warns about — handled cleanly here *only because* the harness offloaded them to files.
- Neither MCP nor CLI "intelligently filters" on its own; both depend on the model writing a good query. The difference is what the client does with the *result*, not smart filtering.
- **Where the objection bites hardest: the business-user-on-web persona.** If that client lacks code execution (no Python, no file offload), a large raw-data tool result *does* pollute context. That's a further reason to invest in tools that return summaries rather than rows — and to prefer aggregate/paged queries.

### Objection 2 — "This isn't a fair or empirical benchmark"

**Response:** Correct, and we say so throughout. Iteration 2 was run blind (good — independent), but the two runs used different-but-defensible definitions (e.g. "underworked" 94 vs. 166), both ran in the same code-capable harness (confounds the context question above), the data is synthetic/offset, and it's a single pass, not a repeated trial. **Frame the deliverable as a *qualitative capability comparison and teaching tool*, not a benchmark.** The robust conclusions are structural (who can use it, one call vs. many, standardized vs. invented definitions), not numeric.

### Objection 3 — "MCP is another dependency/attack surface — why not just use the API or CLI we already have?"

**Response:** For a *developer* who already has the CLI authenticated, the marginal win is convenience, not capability (that's literally our Iteration 1 vs. 2 finding). The case for MCP isn't "developers should drop the CLI" — it's **(a) the non-technical business user who has no CLI at all, and (b) governance**: an admin-approved, OAuth-scoped, hosted MCP server enforces sharing rules server-side and is centrally managed, versus distributing CLI access and long-lived org credentials to end users. MCP is the *more* controlled option for scaled, non-developer rollouts, not an extra liability.

### Objection 4 — "The custom health score was wrong (Activity 0/30) — so custom tools are risky"

**Response:** Agreed, and this is the most important teaching point, not a weakness to hide. The tool faithfully executed the org's Apex, which uses `Date.today()` against future-dated demo data — the tool was *consistent*, just built on date-blind logic. The lesson is **a custom tool is only as correct as the logic inside it**, so build them to be date-aware/parameterized and to **expose their sub-scores** (ours did — the 0/30 was visible), precisely because business users can't audit the black box by hand. This raises the bar on tool quality; it doesn't argue against custom tools.

### Objection 5 — "The LLM still invents the definitions ('at-risk', 'underworked') — can we trust that?"

**Response:** Yes — and that's true of *every* approach here (CLI, standard MCP, and a human analyst writing SOQL all impose a definition). The point is to make the definition **explicit and reviewable**, which the agent did each time ("underworked = Hot/Warm + no activity + not yet qualified"). Where an organization wants a *fixed, governed* definition, that's the argument for encoding it in a **custom MCP tool** (or an Apex/flow the tool wraps) so it's consistent across users and runs — again pointing at custom servers.

### Objection 6 — "Standard MCP gave the same answers — so it added little value"

**Response:** True for a *developer with the CLI already set up* — for reads, standard MCP is convenience, not new capability, and we say so plainly. But (a) it produced a **more accurate first answer** once (the 152-account distribution vs. the CLI's LIMIT-30 mistake), (b) it removed a class of manual errors (relationship/JSON gotchas that cost the CLI run several retries), and (c) for the **business user it's not "same answer, less effort" — it's the only way to get the answer at all.** The value depends entirely on who's asking.

### Objection 7 — "Charts and the PDF came from the MCP server, right?"

**Response:** No — a common misconception worth correcting proactively. Visualization/PDF generation is the **model's own tooling** (matplotlib in the harness), identical in both iterations. No MCP server in this comparison draws a chart or renders a document; the server only supplies data. Set that expectation so nobody over-attributes capabilities to MCP.

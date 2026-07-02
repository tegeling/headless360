# Iteration 2 Results — Salesforce MCP servers

Same 15 workshop prompts as Iteration 1, run through the Salesforce-hosted MCP servers.
**This file is written blind** — the session running it has not seen Iteration 1's findings (see `iteration-2-runbook.md` for the fairness protocol). Do not cross-reference `workshop-comparison.md` while filling this in.

- **Prompts 1–13:** standard MCP server (`salesforce-sobject-all`).
- **Prompts 14–15:** run twice each — standard MCP server AND custom MCP server (`CustomSalesTools`) — for the three-way comparison.

For each prompt: verbatim prompt, steps & MCP tools used, full response, effort & accuracy notes.

---

<!-- Record Prompt 1..15 below, one section per prompt, as they are executed. -->

## Prompt 1 — Most relevant objects & fields for Sales Cloud activity

**Verbatim prompt:** What Salesforce objects and fields are most relevant for understanding Sales Cloud activity in this org? Focus on Account, Lead, Opportunity, Campaign, CampaignMember, Task, and Contact.

**Steps & MCP tools used (standard server `salesforce-sobject-all`):**
1. `getObjectSchema` (no args) → index of 31 objects.
2. `getObjectSchema` with `objects=Account,Lead,Opportunity,Campaign,CampaignMember,Task,Contact` → field-level detail (payload ~132 KB, truncated mid-stream; Contact fields extracted in a follow-up step).

Total: 2 MCP tool calls.

**Full response:**

Most relevant objects & fields:

- **Account** (firmographic anchor): `Industry`, `NumberOfEmployees`, `AnnualRevenue`, `Type`, `BillingState`/`BillingCity`, `Website`, `Description`, `LastActivityDate`. Custom: **`Partner_Tier__c`** (Bronze/Silver/Gold/Platinum), **`Quarterly_Quota__c`**.
- **Lead** (top-of-funnel): `Status`, `Rating` (Hot/Warm/Cold), `LeadSource`, `Industry`, `IsConverted`, `State`, `Company`, `LastActivityDate`, `OwnerId`. Custom: **`Partner_Referral__c`** (lookup → referring Account).
- **Opportunity** (pipeline health): `StageName`, `Amount`, `Probability`, `CloseDate`, `ForecastCategory`, `IsClosed`/`IsWon`, `Type`, `NextStep`, `HasOpenActivity`, `HasOverdueTask`, `LastActivityDate`, `LastStageChangeDate`, `PushCount`, `CampaignId`. Custom: **`Partner_Account__c`**.
- **Campaign** (engagement/ROI rollups): `NumberOfLeads`, `NumberOfContacts`, `NumberOfResponses`, `NumberOfOpportunities`, `NumberOfWonOpportunities`, `AmountWonOpportunities`, `BudgetedCost`, `ActualCost`, `Type`, `Status`, `IsActive`, `StartDate`/`EndDate`.
- **CampaignMember** (join): `CampaignId`, `LeadId`/`ContactId`, `Status` (Sent/Responded), `HasResponded`, `FirstRespondedDate`, `Type`.
- **Task** (activity): `WhatId` (→Account/Opp/Campaign), `WhoId` (→Contact/Lead), `Subject`, `Status`, `Priority`, `ActivityDate`, `IsClosed`, `CallType`, `CallDurationInSeconds`. (Events are a separate object for meetings.)
- **Contact** (people): `AccountId`, `Name`/`Title`/`Department`, `Email`/`Phone`, `LeadSource`, `ReportsToId`, `TitleType` (seniority), `LastActivityDate`.

Connection path: `Campaign → CampaignMember → Lead/Contact → (convert) → Account → Opportunity`, with `Task`/`Event` on Accounts/Contacts/Opps via `WhatId`/`WhoId`. The four custom fields indicate a **partner/channel sales motion** layered on standard Sales Cloud.

**Effort & accuracy notes:** 2 tool calls, low friction. `getObjectSchema` returns LLM-optimized field lists (labels, types, picklist values) directly — no raw describe parsing. One wrinkle: the 7-object payload (132 KB) truncated, so Contact required a follow-up extraction. Accuracy high; picklist values that later prompts depend on (Lead Rating, Opp stages) came through. The four custom fields were surfaced automatically.

---

## Prompt 2 — Key fields, with partner / pipeline / campaign / activity callouts

**Verbatim prompt:** Describe the key fields on Account, Lead, Opportunity, Campaign, CampaignMember, and Task. Call out fields that help explain partner attribution, pipeline health, campaign follow-up, or sales activity.

**Steps & MCP tools used:** None new — reused the field-level `getObjectSchema` detail already retrieved in Prompt 1 (0 additional MCP calls).

**Full response:**

The four callout dimensions map to concrete fields:

- 🤝 **Partner attribution:** `Account.Partner_Tier__c` (Bronze→Platinum), `Lead.Partner_Referral__c` (→ referring Account), `Opportunity.Partner_Account__c` (partner-sourced deal). A coherent channel-attribution chain across all three objects.
- 📈 **Pipeline health:** `Opportunity.StageName` / `Amount` / `CloseDate` / `Probability` / `ForecastCategory`, plus the diagnostic trio `NextStep`, `HasOverdueTask`, `HasOpenActivity`, and stall signals `PushCount` / `LastStageChangeDate` / `LastActivityDate`.
- 📣 **Campaign follow-up:** `Campaign` rollups (`NumberOfResponses`, `NumberOfOpportunities`, `NumberOfWonOpportunities`, `AmountWonOpportunities`), `CampaignMember.HasResponded` / `Status` / `FirstRespondedDate`, `Lead.LeadSource`, and `Opportunity.CampaignId` to close response→deal loop.
- 📞 **Sales activity:** `Task.WhatId` / `WhoId` / `Status` / `ActivityDate` / `Priority` / `CallType`, plus `LastActivityDate` rollups on Account/Lead/Opportunity/Contact.

Per-object key fields captured in full detail (Account firmographics + partner customs; Lead qualification/rating/source + partner referral; Opportunity pipeline + at-risk booleans + partner/campaign links; Campaign engagement/ROI rollups; CampaignMember join/response fields; Task activity fields).

**Effort & accuracy notes:** Zero additional tool calls — the Prompt 1 describe already carried everything, so this was pure interpretation. Efficiency win: one describe answers both "what fields exist" and "which fields serve which business question." Three of the four callout dimensions map partly onto *custom* fields the tool surfaced automatically.

---

## Prompt 3 — Sample of Accounts by state, industry, type, employee count

**Verbatim prompt:** Show me a sample of Accounts by state, industry, type, and employee count.

**Steps & MCP tools used (standard server):**
1. `soqlQuery`: `SELECT Name, BillingState, Industry, Type, NumberOfEmployees, AnnualRevenue, Partner_Tier__c FROM Account ORDER BY NumberOfEmployees DESC NULLS LAST LIMIT 40`
2. `soqlQuery`: `SELECT COUNT(Id) cnt, BillingState st FROM Account GROUP BY BillingState ORDER BY COUNT(Id) DESC`
3. `soqlQuery`: `SELECT COUNT(Id) cnt, Industry ind, Type typ FROM Account GROUP BY Industry, Type ORDER BY COUNT(Id) DESC`

Total: 3 MCP tool calls.

**Full response:**

Sample (largest by headcount): Global Media (Ontario, Media, Prospect, 14,668) is an outlier; the rest cluster at discrete tiers — Juniper Mesa Schools (CA, Manufacturing, Prospect, 2,500), Laurel Terrace Housing (NV, Manufacturing, Customer, 2,500), North Basin Warehouses (CA, Transportation, Customer, 2,500), Doorway Systems Alliance (CO, Manufacturing, **Partner/Bronze**, 1,200), VisitorWorks Channel (AZ, Real Estate, **Partner/Silver**, 1,200).

By state (152 total): NV 33, CA 30, CO 22, WA 19, AZ 16, UT 16, OR 7, NM 4, ID 3, Ontario 1, blank 1. → West/Southwest concentration, Nevada leads; one non-US outlier.

By industry × type (top): Customer — Transportation 21, Education 15, Consulting 15, Real Estate 14, Retail 12, Hospitality 12, Healthcare 8, Manufacturing 8. Prospect — Real Estate 6, Transportation/Manufacturing/Healthcare 5 each. Partner — 15 total (Manufacturing 4, Real Estate/Transportation/Retail 3 each, Hospitality 2).

Observations: ~152 accounts (small enough to analyze exhaustively); regional West/Southwest B2B book with no dominant industry; partner channel ~10% and only Partner-type accounts carry `Partner_Tier__c`; data-quality flags (revenue often null, employee counts tiered/synthetic-looking, "Ontario" as a state value).

**Effort & accuracy notes:** 3 tool calls, very low friction — `GROUP BY` aggregates returned directly, no client-side counting. Self-consistent (state counts sum to 152). SOQL aggregation is a clear advantage over row-by-row default tooling.

---

## Prompt 4 — Nevada accounts: largest headcount + active open opportunities

**Verbatim prompt:** Which Nevada accounts have the largest employee counts and active open opportunities? Summarize why each account may be a good sales priority.

**Steps & MCP tools used (standard server):**
1. `soqlQuery`: `SELECT Id, Name, Industry, Type, NumberOfEmployees, AnnualRevenue, Partner_Tier__c, (SELECT Id, Name, StageName, Amount, CloseDate, NextStep FROM Opportunities WHERE IsClosed = false) FROM Account WHERE BillingState = 'NV' ORDER BY NumberOfEmployees DESC NULLS LAST` → 33 NV accounts with nested open opps.

Total: 1 MCP tool call.

**My definitions:** "largest employee counts" = top headcount tier; "active open opportunities" = ≥1 non-closed opp; priority = big AND live pipeline. (As of 2026-07-02 all open opps close Dec 2026–Apr 2027, none overdue.)

**Full response — top NV priorities:**
1. **Xavier Road Logistics** (2,500 emp, Real Estate/Customer) — 2 open, **$425K** ($250K Kiosk Pilot Prospecting + $175K Badge Rollout Negotiation). Largest tier, biggest NV pipeline.
2. **Cactus Valley Storage** (2,500, Transportation/Customer) — 3 open, **$175K** (anchored by $125K Multi-Site Access). Most open deals; land-and-expand underway.
3. **Laurel Terrace Housing** (2,500, Manufacturing/Customer) — 1 open, $32K; highest NV revenue ($60M). Whitespace/expansion target.
4. **Everline Property Trust** (800, Manufacturing/Customer) — 2 open, **$375K** ($250K + $125K, both early-stage). High upside.
5. **Yucca Grove Healthcare** (1,200, Consulting/Customer) — 1 open, $175K at Proposal/Price Quote. Near decision.
6. **Xenia Market Group** (800, Hospitality/Customer) — 1 open, $250K Badge Rollout in Needs Analysis.

Notes: Partner-type NV accounts (Turnstile Tech Services, Quartz City Low Voltage/Platinum, AccessPoint Integrators, FacilityShield Partners) have **no opportunities** — correctly excluded. Product theme = **physical security** (badge readers, access control, cameras, visitor kiosks, smart locks) — first domain signal. Headcounts sit on discrete tiers. Recommended focus: Xavier Road + Everline (size × pipeline); Laurel Terrace (expansion).

**Effort & accuracy notes:** 1 tool call — parent-child subquery returned accounts + open opps together, no manual joining. Pipeline sums computed client-side from returned rows. Low friction, high accuracy. Relationship subquery is a genuine SOQL/MCP advantage over flat default tooling.

---

## Prompt 5 — Security-equipment buyers in CA/NV/AZ/CO

**Verbatim prompt:** Find accounts in California, Nevada, Arizona, and Colorado that look like security equipment buyers. Use account fields, descriptions, and related opportunity context to explain your reasoning.

**Steps & MCP tools used (standard server):**
1. `soqlQuery`: `SELECT Name, BillingState, Industry, Type, NumberOfEmployees, Description, (SELECT Name, StageName, Amount, IsWon, IsClosed FROM Opportunities) FROM Account WHERE BillingState IN ('CA','NV','AZ','CO') AND Id IN (SELECT AccountId FROM Opportunity) ORDER BY BillingState, NumberOfEmployees DESC NULLS LAST` → 88 accounts w/ descriptions + all opp context. Parsed offline.

Total: 1 MCP tool call.

**My definition:** best-fit buyer = footprint gap (legacy/single-system) AND active open pipeline AND Hot/Warm rating.

**Full response — key finding:** ALL 88 accounts are security-equipment buyers; `Account.Description` is structured and encodes a "current footprint" for each. Footprint mix: entry systems+visitor kiosks 23, badge readers only 15, **legacy keys+manual logs 15 (greenfield, highest intent)**, smart locks+badge 12, multi-site access 12, cameras w/o monitoring 11. Opp names corroborate the physical-security domain.

Strongest greenfield candidates (legacy keys + open pipeline): **Falcon Gate Manufacturing** (CA, 1,200, Warm, $425K/2 open) — single strongest; **Juniper Mesa Schools** (CA, 2,500, Warm, $315K); **Mariposa Campus Services** (CO, 1,200, Hot, $250K + 1 won, 16 locations); **Orchard Park Offices** (CO, 450, Hot, $228K/3).

Highest open pipeline overall: **Alta Canyon Clinics** (CA, 2,500, $550K), **Nevada Horizon Hotels** (CO, 220, $510K), **Xavier Road Logistics** (NV, 2,500, $425K, Hot), **Foothill Retail Collective** (NV, 450, $398K, Hot).

Reasoning combined (1) Description footprint, (2) embedded "hidden insight" next-motion hint, (3) opp context (open pipeline + won/lost). Pick: Falcon Gate Manufacturing.

**Effort & accuracy notes:** 1 tool call returned fields + descriptions + nested opp context; ranking was offline parsing of structured `Description`. Very low friction. Accuracy win: `soqlQuery` surfaced free-text `Description` verbatim, which carried the richest buying signal (an aggregate-only approach would miss it). Caveat: descriptions read as seeded/synthetic ("hidden insight:" labels) — treat "next motion" as artifact, not a real field.

---

## Prompt 6 — At-risk open opportunities closing soon

**Verbatim prompt:** Find open opportunities closing soon that have weak next steps, no recent activity, or overdue related tasks. Summarize why each deal may be at risk.

**Steps & MCP tools used (standard server):**
1. `soqlQuery`: `SELECT Name, Account.Name, StageName, Amount, CloseDate, NextStep, LastActivityDate, HasOpenActivity, HasOverdueTask, LastStageChangeDate, PushCount FROM Opportunity WHERE IsClosed = false ORDER BY CloseDate ASC` → 229 open opps. Parsed offline.

Total: 1 MCP tool call.

**Data reality check (which risk signals exist):** `HasOverdueTask=true` on 0/229 (unusable); `LastStageChangeDate` & `PushCount` unpopulated (unusable); **100/229 (44%) have no `LastActivityDate` and `HasOpenActivity=false`** (usable = zero touch); `NextStep` always filled but templated (6 canned values; vaguest = "Send follow-up", 8 opps). "Closing soon" is relative — soonest close is 2026-12-16 (~5.5mo out), nothing imminent. At-risk cohort defined = closing by 2027-01-31 with zero logged activity → 41 deals.

**Full response — highest-risk (late-stage + neglected + soonest):**
- **Juniper Hill Academy** — Entry System Upgrade, $250K, Negotiation/Review, close 2026-12-19, zero activity.
- **Lakepoint Distribution** — Warehouse Camera Expansion, $250K, Negotiation/Review, 2026-12-22; next step "confirm decision maker" this late = DM unidentified.
- **Laguna Property Trust** — Maintenance Plan Attach, $250K, Proposal/Price Quote, 2027-01-15, no activity.
- **Fairview Education Center** — $175K, Proposal/Price Quote, 2027-01-03, next step "Send follow-up" (weakest hygiene).
- **Foothill Retail Collective** — Maintenance Attach, $175K, Negotiation/Review, 2026-12-29 (Hot NV account — worst to lose).
- **Desert Star Resorts** — Visitor Kiosk Pilot, $85K, Negotiation/Review, 2026-12-16 (soonest close of all), zero touch.
- **Wildflower Schools** — $175K, Proposal/Price Quote, 2026-12-30, no activity.

Risk patterns: (1) late-stage + zero engagement; (2) next step contradicts stage (Negotiation but still "confirm DM" → inflated stage); (3) generic "Send follow-up".

**Effort & accuracy notes:** 1 tool call. Key analytical step was verifying which risk signals the data supports — `HasOverdueTask`/`PushCount`/`LastStageChangeDate` all empty, so a naive "find overdue tasks" query returns nothing and falsely reads as "no risk." Pivoted to the populated signal (activity absence) + stage/amount. Honest caveat: with templated next steps and no imminent closes, "at-risk" here is a hygiene-based judgment, directional not definitive.

---

## Prompt 7 — Underworked hot/warm leads

**Verbatim prompt:** Which hot or warm leads look underworked? Group them by lead source, campaign, state, and partner referral.

**Steps & MCP tools used (standard server):**
1. `soqlQuery`: `SELECT COUNT(Id) cnt, Rating, Status, IsConverted FROM Lead GROUP BY Rating, Status, IsConverted ORDER BY Rating, Status` → funnel shape.
2. `soqlQuery`: `SELECT Id, Name, Company, Rating, Status, LeadSource, State, LastActivityDate, CreatedDate, Partner_Referral__r.Name, (SELECT Campaign.Name FROM CampaignMembers) FROM Lead WHERE Rating IN ('Hot','Warm') AND IsConverted = false ORDER BY Rating, LastActivityDate ASC NULLS FIRST` → 263 leads. Parsed offline (+1 validation parse for partner-referral coverage).

Total: 2 MCP tool calls (+1 offline validation).

**My definition of "underworked":** unconverted Hot/Warm lead with no `LastActivityDate` AND Status = Open/Contacted (not yet Qualified). → 94 of 263. (194/263 have no activity at all; tightened to Open/Contacted so it means genuinely not-progressed.)

**Full response — 94 underworked leads grouped:**
- **By Rating:** Hot 50, Warm 44 (half are untouched Hot leads).
- **By Lead Source:** Web 28, External Referral 28, Advertisement 23, Trade Show 15.
- **By State:** NV 29, CA 21, WA 12, AZ 9, CO 8, NM 8, UT 4, OR 3.
- **By Campaign (top):** Multi-Site Access Control Webinar 18, Education Campus Access Review 17, Colorado Facility Safety Roadshow 15, Western Region Maintenance Attach 14, Denver Access Control Clinic 14, California Retail Security Push 13. Every underworked lead belongs to ≥1 campaign.
- **By Partner Referral:** 0 of 94 underworked leads are partner-referred. Of all 263 Hot/Warm leads, 69 have a partner referral and ALL 69 have logged activity (0 underworked). Partner-referred leads are worked reliably; the gap is entirely marketing/inbound leads.

Takeaways: (1) 50 Hot leads untouched = priority worklist; (2) gap is source-specific (Web/External Referral/Advertisement, not partner); (3) all underworked are campaign responders → marketing→sales handoff breakdown, not lead quality.

**Effort & accuracy notes:** 2 tool calls (+1 validation). Relationship traversal did the heavy lifting — `Partner_Referral__r.Name` + `CampaignMembers` subquery returned in one query, so all four groupings came from a single result set. Standout finding (0/94 underworked are partner-referred) required cross-checking that partner leads exist and are all worked — prevented a "no partner data" misread. High accuracy; groupings are exact counts.

---

## Prompt 8 — Campaigns with strong engagement but limited follow-up

**Verbatim prompt:** Find campaigns with strong engagement but limited sales follow-up. Include the campaign name, related leads or campaign members, and the follow-up gap.

**Steps & MCP tools used (standard server):**
1. `soqlQuery`: 18 Campaigns with rollups (`NumberOfLeads/Responses/Opportunities/WonOpportunities/AmountWonOpportunities/BudgetedCost/ActualCost`).
2. `soqlQuery`: `SELECT Campaign.Name cname, COUNT(Id) responded FROM CampaignMember WHERE HasResponded = true AND Lead.IsConverted = false AND Lead.LastActivityDate = null GROUP BY Campaign.Name ORDER BY COUNT(Id) DESC` → unworked responders.
3. `soqlQuery`: same but total responded (no lead filter). Gap ratio computed offline.

Total: 3 MCP tool calls.

**My definition:** strong engagement = many responders (`HasResponded=true`); limited follow-up = high share of responders that are unconverted leads with no activity. Gap% = unworked ÷ total responders. (Rollup fields are seeded-uniform ~44 leads/~18–20 responses/16–17 opps, so gap discriminates, not headline engagement.)

**Full response — worst follow-up gaps (100%):** Warehouse Security Assessment (20/20), Multi-Site Access Control Webinar (20/20), Education Campus Access Review (19/19), Dormant Customer Winback (19/19), Western Region Maintenance Attach (18/18), Healthcare Lobby Modernization (18/18), Denver Access Control Clinic (18/18), Colorado Facility Safety Roadshow (18/18), California Retail Security Push (18/18). Nine campaigns leak 100% of engaged leads; the two highest-engagement campaigns overall are among them. Remaining 9 campaigns sit at 47–53% gap.

Takeaways: (1) every campaign has ≥47% gap — systemic; (2) worst 9 leak 100% despite spend (Warehouse Security Assessment $42K actual); (3) corroborates Prompt 7 — same webinars/roadshows/reviews generated the 94 underworked leads. One marketing→sales handoff breakdown, two angles.

**Effort & accuracy notes:** 3 tool calls. Engagement rollups alone were a dead end (seeded-uniform); the move was to construct the gap via filtered aggregate across CampaignMember→Lead join server-side (one call per metric). Gap ratios exact. Good example where the metric had to be built from relationships, not read from a field — aggregate SOQL made it 3 calls vs. manual member-by-member counting.

---

## Prompt 9 — Highest-priority risk: related records + next question

**Verbatim prompt:** For the highest-priority risk you found, show the related records and recommend the next question I should ask before taking action.

**Steps & MCP tools used (standard server):** 4 `soqlQuery` calls for the selected account — Account, its Opportunities, Contacts, Tasks (Tasks selected with `WhatId`/`WhoId`).

**Risk chosen (from my Prompts 4–8):** intersection of Prompt 6 (large late-stage deal, no engagement), Prompts 4–5 (Hot account), Prompts 7–8 (systemic follow-up gap) → **Juniper Hill Academy "Entry System Upgrade," $250K, Negotiation/Review, close 2026-12-19, Probability 90%, LastActivityDate = null.**

**Full response — related records:**
- Account: Juniper Hill Academy (CA, Retail, Customer, 45 emp, $25M; footprint multi-site access control, 16 locations, rating=Hot; owner OrgFarm EPIC; account LastActivityDate 2026-12-09).
- Opportunities (3 open, $353K): Entry System Upgrade $250K/Negotiation/90%/2026-12-19/**no activity** ⚠️; Warehouse Camera Expansion $85K/Prospecting/10%/2027-02-10/activity 2026-12-09; Multi-Site Access Control $18K/Proposal/75%/2027-03-17/activity 2026-11-07.
- Contacts (3): Emerson Khan (Ops Mgr), Cameron Kim (Ops Mgr), Morgan Ross (Campus Safety Lead).
- Tasks (8 on account) — **all 8 attached to the $85K and $18K deals; ZERO reference the $250K deal.** Small deals actively worked; flagship untouched.

Risk stated: reps are busy on the wrong deals. $250K/90% deal has no owner activity, no contact touchpoints, no open task; 90% looks unsupported; next step references a different workstream (likely stale).

**Recommended next question:** "Who is the economic buyer/decision-maker on the $250K Entry System Upgrade, and when did we last actually speak with them — is 90% backed by a verbal commit or a stale forecast?" (None of the 3 known contacts is a budget-holder — 2 Ops Mgrs + a Safety Lead = influencers.)

**Effort & accuracy notes:** 4 tool calls, one per related object. Payoff was the Task→Opportunity linkage via `WhatId`: account looks active (8 tasks) but the at-risk deal is untouched — invisible at account-rollup level (account LastActivityDate is recent, which would hide the risk). Relational drill-down beat the summary signal and refined Prompt 6: the account isn't neglected, the deal is.

---

## Prompt 10 — Bar chart: open pipeline by stage

**Verbatim prompt:** Create a bar chart that compares open pipeline by opportunity stage. Use the live opportunity data you can access through Salesforce MCP tools.

**Steps & MCP tools used (standard server):**
1. `soqlQuery`: `SELECT StageName, COUNT(Id) cnt, SUM(Amount) total FROM Opportunity WHERE IsClosed = false GROUP BY StageName ORDER BY SUM(Amount) DESC` (timed out once, succeeded on retry).
2. matplotlib render → `workshop-assets/iteration-2/p10-open-pipeline-by-stage.png`.

Total: 1 MCP tool call (2 attempts) + local chart render.

**Full response — open pipeline by stage (live, 2026-07-02):**
- Proposal/Price Quote — 54 opps, **$6.27M**
- Qualification — 39 opps, $4.59M
- Needs Analysis — 33 opps, $3.80M
- Prospecting — 35 opps, $3.68M
- Negotiation/Review — 34 opps, $3.50M
- Value Proposition — 34 opps, $3.14M
- **Total: 229 opps, $24.98M**

Chart artifact: `workshop-assets/iteration-2/p10-open-pipeline-by-stage.png`.

Shape: Proposal/Price Quote is the clear bulge (~25% of pipeline, 54 deals); other five stages remarkably even ($3.1–4.6M, consistent with seeded data). Ties to Prompt 6 — the Proposal bulge is where several no-activity at-risk deals sit.

**Effort & accuracy notes:** 1 aggregate query gave chart-ready data directly (stage→count+sum), no row math. One transient timeout on first attempt (only friction so far) — resolved on retry; worth noting hosted MCP occasionally times out. Chart written to separate `iteration-2/` subfolder to avoid overwriting pre-existing `workshop-assets/*.png`. Totals cross-check with Prompt 6's 229 open opps.

---

## Prompt 11 — Pie chart: hot & warm open leads by source

**Verbatim prompt:** Create a pie chart that shows hot and warm open leads by lead source.

**Steps & MCP tools used (standard server):**
1. `soqlQuery`: `SELECT LeadSource, COUNT(Id) cnt FROM Lead WHERE Rating IN ('Hot','Warm') AND IsConverted = false AND Status = 'Open' GROUP BY LeadSource ORDER BY COUNT(Id) DESC`.
2. matplotlib render → `workshop-assets/iteration-2/p11-hotwarm-open-leads-by-source.png`.

Total: 1 MCP tool call + local chart render.

**My interpretation of "open":** `Status='Open'` (not yet worked) — tighter/more actionable than "unconverted" (which would be the 263 from Prompt 7). → 73 leads.

**Full response — hot & warm open leads by source:** Partner 22 (30%), Web 17 (23%), External Referral 14 (19%), Advertisement 13 (18%), Trade Show 7 (10%). Total 73. Chart: `workshop-assets/iteration-2/p11-hotwarm-open-leads-by-source.png`.

Reading: Partner is the single largest source (30%). Contrast w/ Prompt 7 — there partner-*referred* (`Partner_Referral__c`) leads were all worked; here `LeadSource=Partner` is the biggest Open slice (different field, flagged). Inbound (Web+Advertisement) = 41%, most likely to decay.

**Effort & accuracy notes:** 1 aggregate query, chart-ready, no friction (no timeout). Judgment call = defining "open" (`Status='Open'`, documented). Slices sum to 73. Flagged `LeadSource` vs `Partner_Referral__c` distinction so Partner slice isn't misread against Prompt 7.

---

## Prompt 12 — Line chart/timeline: campaign response activity over time

**Verbatim prompt:** Create a line chart or timeline that shows campaign response activity over time. Explain any spike or drop you see in the data.

**Steps & MCP tools used (standard server):**
1. `soqlQuery`: responses by `CALENDAR_YEAR/CALENDAR_MONTH(FirstRespondedDate)` → all 336 in 2026-12.
2. `soqlQuery`: `DAY_ONLY(FirstRespondedDate)` → SOQL error (DAY_ONLY needs datetime; field is Date).
3. `soqlQuery`: `GROUP BY FirstRespondedDate` → all 336 responses on ONE date (2026-12-15).
4. `soqlQuery`: 18 Campaigns by `StartDate` + `NumberOfResponses` → real weekly spread Jun–Oct 2026.
5. matplotlib render → `workshop-assets/iteration-2/p12-campaign-response-timeline.png`.

Total: 4 MCP tool calls (1 errored) + local chart render.

**Data finding:** `FirstRespondedDate` is fully seeded to a single day (2026-12-15) → a response-date timeline is one meaningless spike. Pivoted to `Campaign.StartDate` (18 campaigns, weekly cadence 2026-06-16 → 2026-10-13) as the genuine "over time" axis.

**Full response:** Charted responses-per-campaign (flat 18–20/wk) + cumulative (linear to 336) against launch date. Spike/drop explanation: **none — no meaningful variation.** Per-campaign responses flat, cumulative near-perfectly linear; wobble (20 vs 18) is noise. Cause: seeded/synthetic data (uniform ~44 members, ~18–20 responses each, all stamped same response date). Only real temporal structure = even weekly launch schedule. The flatness itself is the finding.

**Effort & accuracy notes:** 4 tool calls incl. one SOQL error (DAY_ONLY on a Date field) and the single-date discovery. Clearest case where the data fights the question — naive `FirstRespondedDate` line would be one dot. Value = diagnosing that, pivoting to `StartDate`, and honestly reporting flat-by-construction. Skeptical interpretation appropriate.

---

## Prompt 13 — One-page PDF summary for a sales leader

**Verbatim prompt:** Create a one-page PDF summary for a sales leader. Include the strongest account signals, the riskiest open opportunities, missed lead or campaign follow-up, and the next questions a seller should ask.

**Steps & MCP tools used (standard server):**
1. `soqlQuery`: `SELECT COUNT(Id) cnt, SUM(Amount) amt FROM Opportunity WHERE IsClosed = false AND StageName IN ('Negotiation/Review','Proposal/Price Quote') AND LastActivityDate = null` → 41 deals / $4.05M (confirming figure).
2. PDF build via matplotlib `PdfPages` (reportlab unavailable) → `workshop-assets/iteration-2/p13-sales-leader-summary.pdf`. Verified 1 page + extractable text via pypdf.

Total: 1 MCP tool call + local PDF render.

**Full response — PDF sections:** Executive snapshot (152 accounts, $24.98M/229 opps, 18 campaigns, West/SW, physical-security motion, ~10% partner); (1) Strongest account signals (Alta Canyon $550K, Nevada Horizon $510K, Xavier Road $425K/Hot, Everline $375K, 15 legacy-keys greenfield led by Falcon Gate, Laurel Terrace whitespace); (2) Riskiest opps ($4.05M in 41 late-stage no-activity deals; Juniper Hill $250K/90%, Lakepoint $250K, Laguna $250K; unpopulated risk fields caveat); (3) Missed follow-up (94 underworked leads/50 Hot, source-specific gap, 9 campaigns at 100% gap); (4) Four next questions (economic buyer; marketing-vs-partner routing; campaign re-spend; greenfield blockers); Data-quality caveats (seeded/synthetic signals).

Tooling note: reportlab unavailable → matplotlib `PdfPages`; genuine 1-page 8.5x11 PDF; written to `iteration-2/` subfolder to preserve pre-existing `workshop-assets/sales-leader-summary.pdf`.

**Effort & accuracy notes:** 1 confirming query + local PDF build. Pure synthesis — assembles prior 12 prompts; accuracy inherits from them. Every headline number traces to a specific earlier query ($24.98M/229 from P10, 94 from P7, 9 campaigns from P8, $4.05M/41 confirmed here). Couldn't visually preview PDF (no poppler); verified structure via pypdf — minor environment friction, not MCP.

---

## END OF STANDARD-SERVER RUN (Prompts 1–13)

**Scope adjustment (per user, mid-run):** Prompts 14–15 run **custom MCP server ONLY**. The standard/manual-tools path is already covered by Iteration 1 (which used the local `sf` CLI), so re-doing a standard-MCP raw-data assessment adds nothing to the comparison. Repo Apex classes are treated as black boxes — their logic is NOT read or reused (also per runbook rule #4).

---

## Prompt 14 — Custom-MCP run: Arroyo Medical Group account health

**Verbatim prompt:** Find the Account named Arroyo Medical Group and check the account health. (Custom-MCP run: use the custom server's account-health tool.)

**Steps & MCP tools used (custom server `CustomSalesTools`):**
1. `soqlQuery`: `SELECT Id, Name, BillingState, Industry, Type, NumberOfEmployees FROM Account WHERE Name = 'Arroyo Medical Group' LIMIT 5` → Id `001bm00002HgMUGAA3` (WA, Education, Customer, 125 emp).
2. `Account_Health_Check_Calculator` with `inputs=[{AccountId: 001bm00002HgMUGAA3}]` → scored breakdown.

Total: 2 custom-server tool calls.

**Full response — Health Score 56.67 / 100:**
- Activity & Engagement: **0.00 / 30** — Past Tasks (60d) 0, Past Events (60d) 0, Future Events 0.
- Opportunity Health: **30.00 / 30** — Closed-Lost (6mo) 0, Won 1, Open 2.
- Support Signals: **26.67 / 40** — Recent Cases (30d) 2, Baseline (30–90d) 0, Escalated (30d) 2, Unresolved Critical (>7d) 0.

Read: moderately healthy — perfect opportunity health (active pipeline, no losses), but zero engagement (the score drag, matches the org-wide no-activity pattern from P6/P9) and a mild support flare-up (2 escalations, nothing critical unresolved). Worth a proactive touch → sets up Prompt 15.

**Comparison significance:** one tool call returned a decision-ready score. No schema discovery, no self-defined "health", no multi-object weighting — the custom tool encapsulates the org's OWN health definition (30/30/40 across activity/opportunity/support) pre-computed with a readable breakdown. vs. Iteration 1's manual path where health must be invented and assembled from raw Tasks/Events/Opps/Cases. Apex behind it was NOT inspected (per instruction).

**Effort & accuracy notes:** 2 calls, ~zero friction. Standout differentiator: the tool applies proprietary business logic and returns a governed score + explanation, not just data. Caveat: weighting/penalty math is opaque from the tool alone (components + sub-scores visible, internal math not) — but that's the point: the seller trusts the org's encoded definition rather than reinventing it.

---

## Prompt 15 — Custom-MCP run: create sync meeting (PRODUCTION WRITE — user confirmed)

**Verbatim prompt:** Create a sync meeting for the account. (Custom-MCP run: use the custom server's sync-meeting tool.)

**Steps & MCP tools used (custom server `CustomSalesTools`):**
1. `CreateAccountSyncMeetingTask` with `inputs=[{accountId: 001bm00002HgMUGAA3}]` → success, Task ID `00Tbm00000DKQWjEAP`, message "Successfully created task for meeting with Alex Garcia".
2. `soqlQuery` read-back of the Task to verify what was written.

Total: 2 custom-server tool calls (1 write + 1 verify). User gave explicit confirmation before the write.

**What was created in production (verified):**
- Subject: "Prepare for Arroyo Medical Group meeting with Alex Garcia"
- Related To (WhatId): Arroyo Medical Group (`001bm00002HgMUGAA3`)
- Contact (WhoId): Alex Garcia (`003bm00001RdMpgAAF`)
- Assigned To: OrgFarm EPIC; Status Not Started; Priority Normal; Due 2026-07-09 (one week out).
- Description: "Prepare for upcoming Arroyo Medical Group meeting with: Alex Garcia — Review open cases: 00001074 Badge reader intermittent failure; 00001104 Badge reader intermittent failure".
- Created 2026-07-02 10:38 UTC.

**What the custom tool did (from a single `accountId`):** (1) auto-selected a relevant contact; (2) pulled the account's open cases and embedded them in the description (context-aware); (3) set sensible defaults (1-week due, Not Started/Normal, owner-assigned); (4) returned success message + task ID. All business logic encapsulated server-side. Manual/standard path (Iter 1) would require: query contacts, choose one, query open cases, compose description, pick due date, construct Task insert — many steps/judgment calls. Here = one call, one governed record. The task also reflects Arroyo's Prompt-14 health profile (the 2 escalated badge-reader cases are what the prep task says to review) — the two custom tools compose into a diagnose→act workflow.

**Effort & accuracy notes:** 2 calls, zero friction; created record correct and genuinely useful. Only `accountId` needed; tool supplied contact/cases/description/due-date via internal logic. Strongest illustration of *why a custom MCP server* — exposes an org-specific ACTION with embedded business rules, not generic data access. Verified the write rather than trusting the success message alone.

---

## END OF ITERATION 2 — all 15 prompts recorded.

Prompts 1–13: standard MCP server (`salesforce-sobject-all`). Prompts 14–15: custom MCP server (`CustomSalesTools`) only, per mid-run scope adjustment. Ready for the comparison-analysis phase (open `workshop-comparison.md`).

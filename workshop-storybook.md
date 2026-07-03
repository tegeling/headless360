# Workshop Storybook — "The Territory Review"

_A hands-on story for running the 15-prompt Salesforce workshop with customers. The prompts aren't a feature demo — they're a **sales narrative**. Follow the story; the tools reveal themselves along the way._

---

## Set the scene (facilitator intro — read this first)

> "Imagine you've just inherited a sales territory. You have a Salesforce org full of accounts, pipeline, leads, and campaigns — but you don't yet know what's in it, where the money is, or where it's leaking. Today, you and an AI copilot are going to go from *'what's even in here?'* to *'here's the one action I'm taking this afternoon'* — in fifteen questions."

**The company (our demo org).** A **physical-security vendor** — badge readers, access control, security cameras, visitor-management kiosks, smart locks, maintenance plans. B2B, selling across the **US West & Southwest** (NV, CA, CO, WA, AZ, UT). Roughly **152 accounts**, **$24.98M open pipeline** across 229 opportunities, 18 marketing campaigns, plus a **partner/channel motion** on top (partner tiers, referrals, partner-sourced deals).

**The protagonist.** A **sales leader** (and their reps) doing a territory review. They start blind and end with a prioritized action.

**The tension the data reveals.** The pipeline *looks* busy — but it's leaking at the seams: **~$4M sits in late-stage deals with zero logged activity**, **~half of hot/warm leads are never worked**, and **every campaign leaks responders sales never called** — while the partner channel is the one motion worked reliably. It's the classic **"marketing generates, sales doesn't follow through, and the forecast is quietly inflated"** story. That's the problem our copilot surfaces fast.

---

## The six acts

### Act I — Orient (Prompts 1–2)
**Rep's question:** *"What can I even ask about? What do these fields mean?"*
- **P1:** Most relevant objects & fields for Sales Cloud activity.
- **P2:** Key fields by theme — partner attribution, pipeline health, campaign follow-up, sales activity.

**Story beat:** The copilot learns the data model *and* spots that this org runs a partner motion (custom fields for partner tier, referral, partner-account). We're oriented.
**Tool lens:** Schema discovery. Standard MCP's schema tool vs. default `describe` — same picture, less noise.

### Act II — Survey the territory (Prompts 3–5)
**Rep's question:** *"What's my book of business, and who should I care about?"*
- **P3:** Accounts by state, industry, type, employee count.
- **P4:** Nevada accounts with the biggest headcount + active pipeline → sales priorities.
- **P5:** Which accounts are real security-equipment buyers, and why.

**Story beat:** The territory takes shape (West/SW, no single dominant industry), NV priorities emerge (Xavier Road, Everline, Cactus Valley), and the copilot discovers the whole book is security buyers — the buying signal lives in the account **Description**.
**Tool lens:** Aggregation & relationship queries. This is where `GROUP BY` beats "eyeball 30 rows" — and where default tools once mistook `LIMIT 30` for the whole org.

### Act III — Find the leaks (Prompts 6–8)
**Rep's question:** *"Where am I about to lose money or waste demand?"*
- **P6:** At-risk open opportunities (weak next steps, no activity, overdue tasks).
- **P7:** Underworked hot/warm leads, grouped by source, campaign, state, partner referral.
- **P8:** Campaigns with strong engagement but weak sales follow-up.

**Story beat:** The leaks appear — untouched late-stage deals, ~half of hot leads never worked, campaigns leaking every responder. And a twist: **partner-referred leads *are* worked; the gap is marketing/inbound.**
**Tool lens:** The data fights back — templated next-steps, unpopulated risk fields, seeded uniformity. Great teaching moment: the tool returns rows; **the analyst must detect the traps.**

### Act IV — Diagnose (Prompt 9)
**Rep's question:** *"What's my #1 problem, and what should I ask before I act?"*
- **P9:** For the top risk, show related records + the smart next question.

**Story beat:** Drill into the single riskiest deal (a $250K deal at 90%, zero activity). The 360° view reveals the account *looks* active — but every task is on the *small* deals; the flagship is untouched. The right move isn't "chase it," it's **"is that 90% even real?"**
**Tool lens:** Multi-object 360. Relationship drill-down turns a summary number into a diagnosis.

### Act V — Communicate (Prompts 10–13)
**Rep's question:** *"How do I show this to my team and my boss?"*
- **P10:** Bar chart — open pipeline by stage.
- **P11:** Pie chart — hot/warm open leads by source.
- **P12:** Timeline — campaign response activity (and explain the spike/drop).
- **P13:** One-page PDF summary for a sales leader.

**Story beat:** Insight becomes shareable — charts and an exec one-pager.
**Tool lens:** Important myth-buster: **charts/PDFs come from Claude's own tooling (matplotlib), not from any MCP server.** P12 is the adversarial test — the "obvious" date field is a seeded dead-end; a naive agent would chart a fake spike.

### Act VI — Act (Prompts 14–15) ⭐ the climax
**Rep's question:** *"Is this specific account healthy — and set up my follow-up."*
- **P14:** Check the health of Arroyo Medical Group.
- **P15:** Create a sync meeting for the account. *(Writes to the org — confirm first.)*

**Story beat:** From insight to action. This is where the **custom MCP server** earns its keep: "check account health" and "create the sync meeting" each become **one governed call** using the company's *own* logic — instead of hand-wiring Apex or building the task field-by-field.
**Tool lens:** The payoff of the whole session. Standard MCP has no such tool; the custom server encapsulates the org's judgment. (And the teachable catch: the health tool inherited a date-blind bug — convenience can hide a data problem.)

---

## Running it live — facilitator tips

- **Pause between acts, not prompts.** Ask the room "what would *you* ask next?" before revealing the next prompt — the arc is intuitive.
- **Let the traps happen.** When the data fights back (P6 templated fields, P12 seeded dates), that's the lesson — the tool doesn't think, the copilot does.
- **Save the reveal for Act VI.** Run 14–15 first *without* the custom tool (raw data), then *with* it. The contrast — many manual steps vs. one call — is the "aha."
- **Name the myth.** Someone will assume MCP "makes the charts." Correct it at Act V: visualization is the model's harness, not the server.
- **Close on the one-liner:** *Standard MCP makes Claude a faster analyst; a custom MCP makes Claude act with your organization's judgment.*

## Optional discussion prompts for the room
- Which of *your* metrics does everyone compute slightly differently? That's your first custom-MCP tool.
- Which repeated structured action (create X the way we always create X) would you wrap?
- Where would a black-box score be dangerous in your org — and what would you require it to show to stay auditable?

---

_Goals for the specific session (timing, audience level, which acts to emphasize) — to be finalized with the facilitator._

# Executive Summary — Claude + Salesforce: Default Tools vs. Standard MCP vs. Custom MCP

**TL;DR.** We ran the same 15-prompt sales workflow against a live Salesforce org three ways. **For analysis, a standard Salesforce MCP server gives the same answers as default tools with much less manual effort — Claude as a faster analyst. The real leap is a custom MCP server: it turns org-specific logic ("check account health," "create the sync meeting") into one governed call — Claude as an operator.** The catch worth teaching: a custom tool is only as correct as the logic inside it.

---

## The three approaches

| Approach | What it is | One-line verdict |
|---|---|---|
| **Default tools** | Claude drives the `sf` CLI (SOQL, anonymous Apex) via Bash | Maximum control, maximum manual work |
| **Standard MCP** | Salesforce-hosted MCP server (`sobject-all`): schema, SOQL, aggregates | Same analysis, much less manual work |
| **Custom MCP** | Org-built server wrapping our own Apex (Account Health, Sync Meeting) | Our organization's judgment, one call away |

## What we found

**Reads & analysis (Prompts 1–13):** Standard MCP and default tools reach essentially the **same conclusions** — both rely on Claude to supply every business definition ("at-risk," "underworked," "buyer"). But standard MCP gets there with **far fewer steps and workarounds**: the server did the data querying, grouping, and joining that the default run had to build by hand (hand-written queries, data-format errors that caused retries, passing IDs between steps, and extra scripting to combine results). It also produced a **more accurate first answer** in one case (saw all 152 accounts via `GROUP BY`, avoiding the default run's "LIMIT 30 = whole org" mistake).

**Actions (Prompts 14–15):** The **custom MCP server is the differentiator.** Checking account health took the default path either deep Apex knowledge (hand-wired anonymous Apex) or a from-scratch rubric; the custom server did it in **one call → 56.67/100 with a full breakdown.** Creating a sync meeting took the default path multi-step SOQL + a hand-built record; the custom server did it in **one call** that auto-picked the contact, embedded the account's open cases, and set sensible defaults.

## The nuance that makes it credible

The custom health tool returned **`Activity & Engagement 0/30`** because the underlying Apex uses `Date.today()` while the demo data is future-dated — so it scored a genuinely-engaged account as disengaged. The "harder" manual baseline **caught** this; the convenient tool **hid** it. **Lesson:** custom MCP tools must be date-aware/parameterized and return their sub-scores so the black box stays auditable.

## The audience that matters most: business users on Claude web

Iteration 1's "default tools" baseline only exists because it ran on a **pre-configured developer workstation** (authenticated `sf` CLI, Bash, Python, matplotlib). **A sales manager, RevOps analyst, or sales leader has none of that** — they have a **web chat with an AI agent and the ability to connect admin-approved MCP servers.** No terminal, no CLI, no `apex run`, no scripting.

That reframes the whole comparison for the real target audience:

- **For a business user, standard MCP doesn't just *save steps* — it's the *way in*.** There's no CLI to save steps over; MCP is the only practical path to live Salesforce data. The comparison becomes **"MCP vs. nothing / manual CSV exports,"** not "MCP vs. hand-written SOQL."
- **Custom MCP goes from "nice differentiator" to "the only way they can act."** Iteration 1 got the health score by hand-wiring anonymous Apex and created the meeting by hand-building a Task record — **both impossible for a non-technical user.** The custom server turns them into a sentence.
- **Tool quality & transparency matter more, because business users can't check the work.** They won't notice a `LIMIT`-capped result or the offset-date bug that zeroed the health score. So custom tools must be **date-correct and expose their sub-scores** — exactly because the user can't audit them by hand.
- **Web + hosted MCP is the better-governed model for this audience:** admin-approved, OAuth-scoped, sharing rules enforced server-side — safer and more manageable at scale than handing everyone a CLI. The prerequisite shifts from "workstation setup" to **"IT/admin enablement."**

> **Bottom line for business users:** without MCP there is no live-data workflow at all, and without a *custom* MCP server there's no safe way for a non-technical user to run the org's own logic. MCP is what makes Claude usable by the people who most need it and can least script around its absence.

## Recommendation

- **Default tools** → ad-hoc exploration, full transparency, debugging the data itself.
- **Standard MCP** → the day-to-day default for NL-driven analysis once auth is set up. Won't invent definitions or draw charts for you.
- **Custom MCP** → build it for **org-specific logic that should be consistent** (scoring, eligibility, health, pricing) and **repeatable structured actions**. Wrap your highest-value "everyone computes this differently" metric and your most common structured write first.

> **Rule of thumb:** *Standard MCP makes Claude a faster analyst. A custom MCP makes Claude act with your organization's judgment.*

## Caveats

Synthetic/offset demo data underlies the specifics (a physical-security vendor, US West/Southwest, seeded dates) — the *approach* comparison is the takeaway, not this org's numbers. The two analysis runs used some different-but-defensible definitions (e.g. "underworked" leads 94 vs. 166), which itself proves the point: **interpretation lives in the agent, not the tool.** Both production writes were user-confirmed.

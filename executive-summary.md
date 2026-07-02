# Executive Summary — Claude + Salesforce: Default Tools vs. Standard MCP vs. Custom MCP

**TL;DR.** We ran the same 15-prompt sales workflow against a live Salesforce org three ways. **For analysis, a standard Salesforce MCP server gives the same answers as default tools with far less friction — Claude as a faster analyst. The real leap is a custom MCP server: it turns org-specific logic ("check account health," "create the sync meeting") into one governed call — Claude as an operator.** The catch worth teaching: a custom tool is only as correct as the logic inside it.

---

## The three approaches

| Approach | What it is | One-line verdict |
|---|---|---|
| **Default tools** | Claude drives the `sf` CLI (SOQL, anonymous Apex) via Bash | Maximum control, maximum manual work |
| **Standard MCP** | Salesforce-hosted MCP server (`sobject-all`): schema, SOQL, aggregates | Same analysis, far less friction |
| **Custom MCP** | Org-built server wrapping our own Apex (Account Health, Sync Meeting) | Our organization's judgment, one call away |

## What we found

**Reads & analysis (Prompts 1–13):** Standard MCP and default tools reach essentially the **same conclusions** — both rely on Claude to supply every business definition ("at-risk," "underworked," "buyer"). But standard MCP gets there with **markedly less friction**: server-side aggregation and one-call relationship queries replaced hand-written SOQL, JSON-shape gotchas, temp-file ID plumbing, and inline-Python assembly. It also produced a **more accurate first answer** in one case (saw all 152 accounts via `GROUP BY`, avoiding the default run's "LIMIT 30 = whole org" mistake).

**Actions (Prompts 14–15):** The **custom MCP server is the differentiator.** Checking account health took the default path either deep Apex knowledge (hand-wired anonymous Apex) or a from-scratch rubric; the custom server did it in **one call → 56.67/100 with a full breakdown.** Creating a sync meeting took the default path multi-step SOQL + a hand-built record; the custom server did it in **one call** that auto-picked the contact, embedded the account's open cases, and set sensible defaults.

## The nuance that makes it credible

The custom health tool returned **`Activity & Engagement 0/30`** because the underlying Apex uses `Date.today()` while the demo data is future-dated — so it scored a genuinely-engaged account as disengaged. The "harder" manual baseline **caught** this; the convenient tool **hid** it. **Lesson:** custom MCP tools must be date-aware/parameterized and return their sub-scores so the black box stays auditable.

## Recommendation

- **Default tools** → ad-hoc exploration, full transparency, debugging the data itself.
- **Standard MCP** → the day-to-day default for NL-driven analysis once auth is set up. Won't invent definitions or draw charts for you.
- **Custom MCP** → build it for **org-specific logic that should be consistent** (scoring, eligibility, health, pricing) and **repeatable structured actions**. Wrap your highest-value "everyone computes this differently" metric and your most common structured write first.

> **Rule of thumb:** *Standard MCP makes Claude a faster analyst. A custom MCP makes Claude act with your organization's judgment.*

## Caveats

Synthetic/offset demo data underlies the specifics (a physical-security vendor, US West/Southwest, seeded dates) — the *approach* comparison is the takeaway, not this org's numbers. The two analysis runs used some different-but-defensible definitions (e.g. "underworked" leads 94 vs. 166), which itself proves the point: **interpretation lives in the agent, not the tool.** Both production writes were user-confirmed.

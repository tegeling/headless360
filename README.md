# Headless360 — Claude + Salesforce MCP Workshop

A hands-on comparison of **how a business user gets work done in Salesforce with Claude**, depending on what tools Claude can reach. We ran the **same 15-prompt sales workflow** against one live Salesforce org three ways, and captured what changed.

## Why this project exists

A sales leader (or RevOps analyst) wants to ask an AI copilot plain-language questions about their Salesforce data — *"which deals are at risk?", "check this account's health", "set up the follow-up"* — and get useful, trustworthy answers and actions. **What Claude can actually do depends entirely on which tools it's connected to.** This project measures that difference, prompt by prompt, and turns it into workshop material.

## The three approaches compared

| Approach | What it is | One-line verdict |
|---|---|---|
| **Default tools** | Claude drives the `sf` CLI (SOQL, anonymous Apex) via Bash — a developer's setup | Maximum control, maximum manual work |
| **Standard MCP** | Salesforce-hosted MCP server (`sobject-all`): schema, SOQL, aggregates | Same analysis, much less manual work |
| **Custom MCP** | Org-built server wrapping our own Apex (Account Health, Sync Meeting) | Our organization's judgment, one call away |

## What we found

**Reads & analysis (Prompts 1–13):** Standard MCP and default tools reach essentially the **same conclusions** — both rely on Claude to supply every business definition ("at-risk", "underworked", "buyer"). But standard MCP gets there with **far fewer steps and workarounds**: the server did the querying, grouping, and joining the default run had to build by hand. It also produced a **more accurate first answer** in one case (saw all 152 accounts via `GROUP BY`, avoiding the default run's "LIMIT 30 = whole org" mistake).

**Actions (Prompts 14–15):** The **custom MCP server is the differentiator.** Checking account health took the default path either deep Apex knowledge (hand-wired anonymous Apex) or a from-scratch rubric; the custom server did it in **one call → 56.67/100 with a full breakdown**. Creating a sync meeting took the default path multi-step SOQL + a hand-built record; the custom server did it in **one call** that auto-picked the contact, embedded the account's open cases, and set sensible defaults.

→ *Prompt-by-prompt evidence: **[Part A of the comparison](workshop-comparison.md#part-a--per-prompt-comparison-side-by-side)**. Full analysis, scorecard, and scored dimensions: **[Part B](workshop-comparison.md#part-b--comparison-summary)**.*

## The nuance that makes it credible

The custom health tool returned **`Activity & Engagement 0/30`** because the underlying Apex uses `Date.today()` while the demo data is future-dated — so it scored a genuinely-engaged account as disengaged. The "harder" manual baseline **caught** this; the convenient tool **hid** it. **Lesson:** custom MCP tools must be date-aware/parameterized and return their sub-scores so the black box stays auditable.

## The audience that matters most: business users on Claude web

The default-tools baseline only exists because it ran on a **pre-configured developer workstation** (authenticated `sf` CLI, Bash, Python, matplotlib). **A sales manager, RevOps analyst, or sales leader has none of that** — they have a **web chat with an AI agent and the ability to connect admin-approved MCP servers.** No terminal, no CLI, no scripting. That reframes the whole comparison:

- **For a business user, standard MCP doesn't just *save steps* — it's the *way in*.** The comparison becomes **"MCP vs. nothing / manual CSV exports,"** not "MCP vs. hand-written SOQL."
- **Custom MCP goes from "nice differentiator" to "the only way they can act."** Hand-wiring Apex or building a Task record by hand is impossible for a non-technical user.
- **Tool quality & transparency matter more, because business users can't check the work** — they'd never spot the offset-date bug that zeroed the health score.
- **Web + admin-approved, OAuth-scoped hosted MCP is the better-governed model at scale** than handing everyone a CLI. The prerequisite shifts from "workstation setup" to **"IT/admin enablement."**

→ *Full argument: **[Part B, §10 — the business-user POV](workshop-comparison.md#10-pov-the-business-user-on-claude-web-not-claude-code-on-a-dev-workstation)**.*

## Recommendation

- **Default tools** → ad-hoc exploration, full transparency, debugging the data itself.
- **Standard MCP** → the day-to-day default for analysis you drive by asking in plain language, once auth is set up. Won't invent definitions or draw charts for you.
- **Custom MCP** → build it for **org-specific logic that should be consistent** (scoring, eligibility, health, pricing) and **repeatable structured actions**. Wrap your highest-value "everyone computes this differently" metric and your most common structured write first.

> **Rule of thumb:** *Standard MCP makes Claude a faster analyst. A custom MCP makes Claude act with your organization's judgment.*

## What's in this repo

| File | What it is |
|---|---|
| **[workshop-comparison.md](workshop-comparison.md)** | The full comparison — **Part A** (per-prompt, side by side) + **Part B** (analysis, scorecard, scored dimensions, sales story, MCP benefits, business-user POV) |
| **[workshop-storybook.md](workshop-storybook.md)** | "The Territory Review" — the 6-act sales narrative for running the workshop live, with facilitator tips |
| **[slack-message-en.md](slack-message-en.md)** / **[slack-message-de.md](slack-message-de.md)** | Ready-to-post internal announcements (English / German) |
| **[Headless.md](Headless.md)** | How to connect Claude to the Salesforce-hosted MCP servers (setup/auth) |
| **[iteration-2-runbook.md](iteration-2-runbook.md)** | The protocol used to run Iteration 2 blind (fairness rules) |
| `workshop-assets/` | Generated charts + PDF (Iteration 1 at root, Iteration 2 under `iteration-2/`) |
| `force-app/` | The Salesforce DX project: the Account Health Check Apex + Flow the custom MCP server wraps |

## The sales story (for context)

The demo org is a **physical-security vendor** (badge readers, access control, cameras, visitor kiosks, smart locks) selling across the **US West/Southwest**: ~152 accounts, **$24.98M open pipeline**, 18 campaigns, plus a partner/channel motion. The 15 prompts walk a sales leader from *"what's even in here?"* to *"here's the one action I'm taking today"* — and reveal a pipeline that looks busy but leaks at the seams (~$4M in untouched late-stage deals, ~half of hot leads never worked, every campaign leaking responders). See the **[storybook](workshop-storybook.md)** for the full arc.

## Caveats

Synthetic/offset demo data underlies the specifics (seeded dates, tiered employee counts) — the *approach* comparison is the takeaway, not this org's numbers. Iteration 2 was run **blind** (no access to Iteration 1 during the prompts), so the two runs used some different-but-defensible definitions (e.g. "underworked" leads 94 vs. 166) — which itself proves the point: **interpretation lives in the agent, not the tool.** Both production writes were user-confirmed.

---

_This is a Salesforce DX project (`sfdx-project.json`, `force-app/`). For CLI/deploy specifics see the [Salesforce DX Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_intro.htm)._

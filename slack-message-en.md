# Slack Message (internal, English)

_To paste into Slack. Emojis/formatting are Slack-flavored._

---

:mag: *Claude + Salesforce: Default Tools vs. Standard MCP vs. Custom MCP — my findings*

Hey team :wave: — I ran the same 15-prompt sales workflow against a live Salesforce org *three* ways and compared the approaches. Short version:

*TL;DR:* For analysis, a *standard Salesforce MCP server* gives the same answers as the default tools — but with much less manual work (Claude as a faster analyst). The real leap is a *custom MCP server*: it turns org-specific logic ("check account health", "create a sync meeting") into *one governed tool call* (Claude as an operator).

*The three approaches:*
• *Default tools* – Claude drives the `sf` CLI (SOQL, Apex) via Bash → maximum control, maximum manual work
• *Standard MCP* – Salesforce-hosted server (schema, SOQL, aggregation) → same analysis, much less manual work
• *Custom MCP* – self-built server wrapping our own Apex (Account Health, Sync Meeting) → our organization's judgment, one call away

*What we found:*
• *Analysis (Prompts 1–13):* Standard MCP and default tools reach essentially the same conclusions — both need Claude to supply every business definition ("at-risk", "underworked", "buyer"). But MCP removes the mechanics: server-side aggregation & relationship queries instead of hand-written SOQL, JSON gotchas and Python glue. In one case it was even *more accurate* (saw all 152 accounts via `GROUP BY` — the default run mistook "LIMIT 30" for the whole org).
• *Actions (Prompts 14–15):* This is where *custom MCP* shines. Account health via the default path would need deep Apex knowledge or a self-invented scoring rubric — the custom server does it in *one call → 56.67/100 with a full breakdown*. Sync meeting: instead of multi-step SOQL + a hand-built record, *one call* that auto-picks the contact, embeds open cases, and sets sensible defaults.

*The honest catch (important!):* The custom health tool returned `Activity 0/30` because the underlying Apex uses `Date.today()` while the demo data is future-dated — so a genuinely active account was scored as inactive. The "harder" manual path *caught* the bug; the convenient tool *hid* it. :warning: Lesson: a custom tool is only as good as its internal logic — it must be date-aware/parameterized and expose its sub-scores so the black box stays auditable.

*Recommendation / when to use what:*
• *Default tools* → ad-hoc exploration, full transparency, debugging the data itself
• *Standard MCP* → the day-to-day default for analysis you drive by asking in plain language (once auth is set up). Won't invent definitions or draw charts, though.
• *Custom MCP* → build it for org-specific logic that should be *consistent* (scoring, eligibility, health, pricing) and for repeatable structured actions. Wrap your highest-value "everyone computes this differently" metric and your most common structured write first.

*Reality check for business users (important):* Iteration 1's "default tools" baseline only worked because it ran on a *developer workstation* with the `sf` CLI, Bash, Python, etc. A real *business user* (sales manager, RevOps, sales leader) has a *web chat + connectable MCP servers* — no terminal, no CLI, no scripting. That reframes everything:
• For them, standard MCP doesn't just *save steps*, it's the *way in* — the comparison is "MCP vs. nothing / manual CSV exports," not "MCP vs. hand-written SOQL."
• Custom MCP goes from "nice differentiator" to "the *only* way they can act" — hand-wiring Apex or building a Task record by hand is simply impossible for a non-technical user.
• Because they *can't* sanity-check the tool (they'd never spot the offset-date bug), tool quality & transparency matter even more: custom tools must be date-correct and show their sub-scores.
• Web + admin-approved, OAuth-scoped hosted MCP is also the *better-governed* model at scale than handing everyone a CLI. Prerequisite shifts from "workstation setup" to "IT/admin enablement."

:bulb: *Rule of thumb:* Standard MCP makes Claude a faster analyst. A custom MCP makes Claude act with our organization's judgment. And for non-technical business users on the web, MCP isn't the optimization — *it's what makes the workflow possible at all.*

*Context/caveats:* Runs on synthetic demo data (a security vendor, US West) — the *approach comparison* is the message, not the specific numbers. Both write actions were user-confirmed.

Full details, per-prompt scorecard and the workshop story are in the docs — ping me if you want a look. :salesforce: :robot_face:

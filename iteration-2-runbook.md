# Iteration 2 Runbook — READ THIS FIRST (fresh session)

You are running **Iteration 2** of a workshop comparison. A prior session ran Iteration 1; **you must NOT benefit from its findings**. This runbook contains the protocol only — no findings, definitions, or discoveries.

## Fairness rules (critical)
1. **DO NOT open `workshop-comparison.md`.** It contains all of Iteration 1's answers. Reading it invalidates the comparison. Only open it if the user explicitly asks, and never during a prompt.
2. **Record into `iteration-2-results.md`** (a fresh, separate file), NOT into `workshop-comparison.md`.
3. **Start every prompt from zero.** Discover the data, invent your own definitions ("at-risk", "underworked", "relevant", etc.), and reach conclusions independently — exactly as a first-time analyst would. Do not assume anything about record counts, date ranges, data quality, or the org's domain.
4. **Tool constraint — use Salesforce MCP tools ONLY.** Do NOT use the `sf` CLI, Bash SOQL, or `apex run` for org data. Do NOT read the repo's Apex source (`force-app/main/default/classes/*.cls`) to understand scoring/logic — the whole point is to use the MCP tools. `CLAUDE.md` (architecture) is allowed; it existed before Iteration 1 started.

## Tool mapping
- **Standard MCP server** (`salesforce-sobject-all`): use for **Prompts 1–13** (generic sObject query/describe/CRUD).
- **Custom MCP server** (`CustomSalesTools`): exposes this org's custom logic (account health scoring, sync-meeting creation) as tools. Used for the custom-server runs of **Prompts 14–15**.
- Run `/mcp` first to confirm both servers are connected and authenticated. If a call fails with auth errors, stop and tell the user to authenticate.

## Recording format (per prompt, in `iteration-2-results.md`)
For each prompt capture: **verbatim prompt**, **steps & MCP tools used** (which tool calls, in order), **full response**, and **effort & accuracy notes** (tool-call count, whether the tool returned data directly vs. required assembly, friction, accuracy). Mirror the structure the user expects — one section per prompt.

## The prompts (run in order; paste-confirm each with the user as they did in Iteration 1)

**Prompts 1–13 — run with the STANDARD MCP server:**
1. What Salesforce objects and fields are most relevant for understanding Sales Cloud activity in this org? Focus on Account, Lead, Opportunity, Campaign, CampaignMember, Task, and Contact.
2. Describe the key fields on Account, Lead, Opportunity, Campaign, CampaignMember, and Task. Call out fields that help explain partner attribution, pipeline health, campaign follow-up, or sales activity.
3. Show me a sample of Accounts by state, industry, type, and employee count.
4. Which Nevada accounts have the largest employee counts and active open opportunities? Summarize why each account may be a good sales priority.
5. Find accounts in California, Nevada, Arizona, and Colorado that look like security equipment buyers. Use account fields, descriptions, and related opportunity context to explain your reasoning.
6. Find open opportunities closing soon that have weak next steps, no recent activity, or overdue related tasks. Summarize why each deal may be at risk.
7. Which hot or warm leads look underworked? Group them by lead source, campaign, state, and partner referral.
8. Find campaigns with strong engagement but limited sales follow-up. Include the campaign name, related leads or campaign members, and the follow-up gap.
9. For the highest-priority risk you found, show the related records and recommend the next question I should ask before taking action. _(References YOUR OWN Iteration-2 findings from prompts 1–8 — not Iteration 1's.)_
10. Create a bar chart that compares open pipeline by opportunity stage. Use the live opportunity data you can access through Salesforce MCP tools.
11. Create a pie chart that shows hot and warm open leads by lead source.
12. Create a line chart or timeline that shows campaign response activity over time. Explain any spike or drop you see in the data.
13. Create a one-page PDF summary for a sales leader. Include the strongest account signals, the riskiest open opportunities, missed lead or campaign follow-up, and the next questions a seller should ask.

**Prompts 14–15 — three-way comparison. Run EACH twice: once with the STANDARD MCP server, once with the CUSTOM MCP server. Record both runs.**

14. Find the Account named Arroyo Medical Group and check the account health.
   - _Standard-MCP run:_ no health tool exists → assess health from raw data with your own framework.
   - _Custom-MCP run:_ use the custom server's account-health tool.
15. Create a sync meeting for the account. _(This is a WRITE to a production org — confirm with the user before creating, and show exactly what will be created.)_
   - _Standard-MCP run:_ build the meeting/task manually from primitives.
   - _Custom-MCP run:_ use the custom server's sync-meeting tool.

## THE GOAL: a comparison analysis (this is the real deliverable)
The two iterations are inputs. The end goal is the **Comparison summary** in `workshop-comparison.md` — a structured analysis of default tools vs. standard MCP vs. custom MCP. That section already has a **pre-defined template** (per-prompt scorecard, scored dimensions, standard-MCP findings, custom-MCP differentiator, recommendation, caveats) — it just needs filling in. Do not design it from scratch; populate the existing structure.

**Strict sequence (protects fairness AND guarantees the deliverable):**
1. Run all 15 prompts (14–15 twice: standard + custom) and record each into `iteration-2-results.md` **blind** — do NOT open `workshop-comparison.md` during this phase.
2. Only AFTER every prompt is recorded, tell the user Iteration 2 is complete and ask to proceed to the analysis.
3. THEN (and only then) you may open `workshop-comparison.md` to read Iteration 1 and fill in the Comparison summary template, comparing against `iteration-2-results.md`.
4. The analysis must cover, per prompt: same result or not, which approach was faster / more accurate / lower friction — plus the cross-cutting dimensions and the "why a custom server" story (Prompts 14–15). Keep it honest: if MCP reached the same wall as default tools, say so.

## Environment facts (safe, non-findings)
- Org: production Salesforce org, user `epic.ee57ef6782f0@orgfarm.salesforce.com`. **Prompt 15 writes to production** — get explicit confirmation.

## How the user resumes you
The user will say something like: _"Read `iteration-2-runbook.md` and start iteration 2."_ Follow this file; do not reconstruct Iteration 1 from memory. Remember the goal is the comparison analysis — Iteration 2 recording is a means to it, not the end.

# Metadata Creation & Deployment — Approach Comparison Runbook

**READ THIS FIRST.** You are running one of **two iterations** of a comparison. Both iterations build the **exact same Salesforce metadata** against the same org and deploy it — only the **tooling differs**. The goal is a fair side-by-side of *how the metadata gets created and deployed* under each toolchain, and a final comparison analysis (mirroring the earlier `workshop-comparison.md`).

- **Iteration A — Installed MCP servers.** Branch `metadata-iteration-mcp`. Use the **Salesforce DX MCP** server (and the org's other MCP servers) as the *only* Salesforce tooling. The DX MCP has **no per-type "generate" tools**, so you author the metadata source yourself (Edit/Write) and use MCP tools for **deploy / retrieve / code-analysis / SOQL / permission-set assignment**. Do **not** install or use sf-skills on this branch.
- **Iteration B — sf-skills.** Branch `metadata-iteration-skills`. Use the skills installed from [`forcedotcom/sf-skills`](https://github.com/forcedotcom/sf-skills) (`platform-custom-field-generate`, `platform-validation-rule-generate`, `platform-apex-generate`, `platform-apex-test-generate`, `platform-permission-set-generate`, `platform-metadata-deploy`, `platform-metadata-retrieve`, `dx-code-analyzer-run`, `platform-soql-query`). Prefer a matching skill for each step. Do **not** use the Salesforce DX MCP generate/deploy tools on this branch (SOQL/verify via skills where possible).

  **Setup (run once on this branch, from the repo root):**
  ```bash
  npx skills add forcedotcom/sf-skills   # installs into .agents/skills/ + symlinks .claude/skills/
  ```
  The skills are **not vendored into git** (`.agents/`, `.claude/skills/`, `skills-lock.json` are gitignored) — reinstall to get the latest. After installing, restart the session (or run `/mcp`) so Claude Code discovers the new skills, then confirm they appear in the available-skills list before step 1.

**Org:** `epic.ee57ef6782f0@orgfarm.salesforce.com` (alias `headless360`, default target org). **Steps 6–7 write metadata to this org — confirm with the user before deploying.**

---

## Fairness rules (critical)

1. **Build the exact spec below — identical field names, types, logic, API names.** No embellishing, no "improving" the design differently per iteration. If a tool wants to add extra, trim it back to the spec.
2. **Same order, same 8 steps.** Paste-confirm each step with the user before running it (as in the earlier workshops).
3. **Tooling constraint per iteration** (above). If a needed capability is missing in your toolchain, do it the closest legitimate way *within that toolchain* and **record the friction** — that gap is the finding, not something to route around with the other toolchain.
4. **Record into the iteration's own results file**, NOT into the comparison analysis:
   - Iteration A → `metadata-iteration-mcp-results.md`
   - Iteration B → `metadata-iteration-skills-results.md`
5. **Run blind.** Do not open the *other* iteration's results file during your run.
6. All Apex is `with sharing`; API version 66.0; no namespace; source lives under `force-app/main/default/`.

---

## The fixed metadata spec — "Account Health Snapshot"

A small, coherent feature: persist the latest computed health score on the Account, derive a tier from it via Apex, guard the data with a validation rule, and grant access via a permission set. Self-contained; does not modify the existing scoring classes or Flow.

**M1 — Custom field:** `Account.Health_Score__c`
- Type: Number, length 3, decimal places 0 (range 0–100). Label: `Health Score`. Description: "Latest computed Account Health Check score (0–100)."

**M2 — Custom field:** `Account.Health_Tier__c`
- Type: Picklist. Label: `Health Tier`. Values (exact, in order): `Platinum`, `Gold`, `Silver`, `Bronze`. Not restricted-to-values-required beyond the four. Description: "Tier derived from Health Score."

**M3 — Validation rule:** `Account.Health_Score_In_Range`
- Error if `Health_Score__c` is populated and outside 0–100: `NOT(ISBLANK(Health_Score__c)) && (Health_Score__c < 0 || Health_Score__c > 100)`
- Error message: "Health Score must be between 0 and 100." Error location: the `Health_Score__c` field.

**M4 — Apex class:** `AccountHealthTier` (`with sharing`)
- Public static method `String tierForScore(Integer score)` returning: `>= 85` → `Platinum`; `>= 70` → `Gold`; `>= 50` → `Silver`; else `Bronze`. Null score → `Bronze`.
- Public static `void applyTier(List<Account> accounts)` that sets `Health_Tier__c` from `Health_Score__c` on each account (does not DML — caller persists).

**M5 — Apex test class:** `AccountHealthTierTest` (`@isTest`)
- Cover each tier boundary (85, 70, 50, below), the null case, and `applyTier` over a small list. Target ≥ 90% coverage of `AccountHealthTier`.

**M6 — Permission set:** `Account_Health_Snapshot`
- Label: `Account Health Snapshot`. Grants: field read/edit on `Health_Score__c` and `Health_Tier__c`; Apex class access to `AccountHealthTier`.

---

## The 8 steps (run in order)

1. **M1 + M2** — create the two Account custom fields.
2. **M3** — create the validation rule.
3. **M4** — create the `AccountHealthTier` Apex class.
4. **M5** — create the `AccountHealthTierTest` Apex test class.
5. **M6** — create the `Account_Health_Snapshot` permission set.
6. **Deploy** all of M1–M6 to the org. *(Production write — confirm first.)*
7. **Retrieve / verify** the deployed metadata came back clean; run the Apex test in the org and report coverage.
8. **Run the code analyzer** over the new Apex and report findings.

---

## Recording format (per step, in the iteration's results file)

For each step capture: **step + spec item**, **tool(s) used** (which MCP tool or which skill, in order — count them), **what you had to author by hand vs. what the tool generated**, **outcome** (deployed? test coverage? analyzer findings?), and **effort/friction notes** (retries, missing capabilities, manual assembly).

---

## THE GOAL: the comparison analysis (real deliverable)

After **both** iterations are recorded, produce `metadata-comparison.md` (mirroring `workshop-comparison.md`): Part A per-step side-by-side + verdict, Part B scored dimensions (authoring effort, deploy ergonomics, correctness of first attempt, transparency/auditability, setup cost) and a recommendation on **when to reach for installed MCP servers vs. task-specific skills** for metadata work. Keep it honest — if both reached the same result with similar effort on a step, say so.

## How the user resumes you
"Read `metadata-comparison-runbook.md` and start iteration A (or B)." Follow this file; confirm the org and both MCP servers via `/mcp` (iter A) or that skills are installed (iter B) before step 1.

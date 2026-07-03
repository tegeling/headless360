# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Salesforce DX project (`headless360`). The domain logic is an **Account Health Check** scoring system: three Apex invocable classes each compute a sub-score, and an autolaunched Flow orchestrates them into a single 0–100 health score for an Account.

## Commands

```bash
npm run lint                 # ESLint over aura/lwc JS
npm run test:unit            # Jest (sfdx-lwc-jest) for LWC — no LWC exists yet, so passes empty
npm run test:unit:watch      # Jest watch mode
npm run test:unit:coverage   # Jest with coverage
npm run prettier             # Format cls/html/js/xml/etc. in place
npm run prettier:verify      # Check formatting without writing
```

Deploy / test against an org (Apex has no JS test runner — use the CLI):

```bash
sf project deploy start -d force-app          # deploy source to the target org
sf project retrieve start -d force-app        # pull org changes back
sf apex run test -c -r human                  # run Apex tests with coverage
sf apex run test -t OpportunityHealthScore    # run a single Apex test class
sf apex run -f scripts/apex/hello.apex        # execute anonymous Apex
sf data query -f scripts/soql/account.soql    # run a stored SOQL query
```

A pre-commit hook (Husky + lint-staged) runs prettier, eslint, and `sfdx-lwc-jest --findRelatedTests` on staged files.

## Architecture

**Scoring pipeline.** `Account_Health_Check_Calculator` (autolaunched Flow, `force-app/main/default/flows/`) takes an `AccountId` input and calls three invocable Apex actions, then sums their scores into `FinalHealthScore` (0–100) and concatenates their `details` strings into `ScoreDetails`:

- `ActivityEngagementScore` — 0–30, based on past activities (Tasks/Events in last 60 days) and future scheduled Events.
- `OpportunityHealthScore` — 0–30, based on Closed-Lost trend (last 6 months) and pipeline dry-up (won but no open opps).
- `SupportSignalsScore` — 0–40, based on case-volume spike (last 30 days vs. a 30–90-day baseline), escalated cases, and unresolved critical cases.

Each scoring class follows the same shape: an `@InvocableMethod calculateScore(List<Request>)` that loops requests, a private per-account calculator, and nested `Request` (input `accountId`) / `Response` (output `score` + `details`) inner classes. Scores start at max and deduct fixed penalties per negative signal, floored at 0 via `Math.max`. Detail strings are built with Apex `String.template(...)` interpolation.

`CreateAccountSyncMeetingTask` is a separate global invocable (not part of the score flow) that creates a follow-up `Task` for an account's open cases, selecting a random case contact. It returns a `Result` (success/taskId/message) and swallows exceptions into the message rather than throwing.

**Adding a new sub-score:** create an Apex invocable class mirroring the existing three (Request/Response inner classes, `@InvocableMethod`), then wire it into the Flow as an Action element and add its score to the `Calculate_Final_Score` formula. Keep the total across all sub-scores at 100.

## Conventions

- API version 66.0; no namespace. Only `force-app` is a package directory.
- Apex classes are `with sharing`; use `global` only when the invocable must be exposed across packages (see `CreateAccountSyncMeetingTask`).
- The `.forceignore` excludes `package.xml`, LWC config/test files, and `node_modules` from source sync.

## Salesforce MCP servers

`Headless.md` documents connecting Claude Code to Salesforce-hosted MCP servers (e.g. `sobject-all`) via `claude mcp add --transport http ...` with an ECA consumer key/secret. Authenticate through `/mcp` after adding.

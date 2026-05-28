---
name: ghl-followup-report
description: >-
  Generate a follow-up report from GoHighLevel (GHL): surfaces (1) new leads
  that have not been contacted yet and (2) open leads with no activity in the
  last 3+ days, formatted as a concise morning briefing for Telegram. Use this
  skill whenever the user asks for a leads report, a follow-up report, a morning
  CRM digest, "who do I need to follow up with", GoHighLevel / GHL / High Level
  lead status, or sets up a daily/scheduled leads briefing. Trigger it even if
  the user only says "run my leads report" or "morning report" without naming GHL.
version: 1.0.0
license: MIT
metadata:
  author: boreal_leads
  category: crm
  env:
    - GHL_TOKEN
    - GHL_LOCATION_ID
  permissions:
    - terminal
---

# GoHighLevel Follow-Up Report

Pulls leads out of GoHighLevel that need attention and produces a short,
scannable report. Designed to be run on a schedule (e.g. every morning) and
delivered to Telegram, but also works on demand.

## What it reports

Two buckets:

1. **New leads — not contacted yet.** Open opportunities sitting in the *first*
   stage of a pipeline (the entry stage), i.e. leads that have come in but not
   yet been worked.
2. **Going cold — no activity in 3+ days.** Open opportunities whose last
   activity/update was more than 3 days ago. These are at risk of being lost.

A lead can appear in only one bucket; if it qualifies for both, put it in
"New leads" (it's the more urgent label for an untouched lead).

## Required configuration

These come from environment variables — never hard-code them, never print the
token in output or logs:

- `GHL_TOKEN` — a GoHighLevel **Private Integration Token** with read access to
  contacts and opportunities/pipelines. Used as a Bearer token.
- `GHL_LOCATION_ID` — the GHL sub-account (location) ID the leads live in.

If either variable is missing, stop and report that clearly to the user rather
than guessing.

### Loading credentials in terminal

The `.env` file is at `~/.hermes/.env` but is **not auto-exported** into
subprocesses. Always source it explicitly in shell commands:

```bash
source ~/.hermes/.env && curl ...
```

When using `execute_code` (Python), the vars are not available via `os.environ`
even though they exist in `.env`. **Prefer `terminal()` with `source ~/.hermes/.env &&`
prepended**, or use a subprocess that sources the file first. Do not assume
env vars are set just because they appear in `.env`.

## API basics

- Base URL: `https://services.leadconnectorhq.com`
- Every request needs these headers:
  - `Authorization: Bearer $GHL_TOKEN`
  - `Version: 2021-07-28`
  - `Content-Type: application/json`
- GHL is on API v2. The legacy `GET /contacts/` endpoint is deprecated — use the
  **search** endpoints described below.
- Rate limits are generous (about 100 requests / 10s); a daily report stays well
  within them. Still, paginate politely.

> The exact query-parameter and request-body shapes occasionally change. Treat
> the calls below as the starting point: make the call, inspect the JSON that
> actually comes back, and adapt field names to what you see. If a call returns
> 4xx, read the error body and consult https://marketplace.gohighlevel.com/docs
> before retrying. Do not invent fields.

## Step 1 — Discover pipelines and stages

Fetch the pipelines so you can identify the entry (first) stage of each one and
map stage IDs to human-readable names:

```bash
source ~/.hermes/.env && curl -s --request GET \
  "https://services.leadconnectorhq.com/opportunities/pipelines?locationId=$GHL_LOCATION_ID" \
  -H "Authorization: Bearer $GHL_TOKEN" \
  -H "Version: 2021-07-28"
```

From the response, for each pipeline record the ordered list of stages. The
**first stage** in each pipeline is the "new / uncontacted" stage for Step 2.
Keep a lookup of `stageId -> stageName` and `pipelineId -> pipelineName` for
formatting later.

**⚠️ Pitfall — pipelines endpoint may return 401:** Private Integration Tokens
with limited scope can access `/opportunities/search` but not
`/opportunities/pipelines`. If the pipelines call returns 401, do **not** abort
the report. Instead, fall back to empirical entry-stage detection:

1. Fetch all open opportunities (Step 2).
2. Find the stage ID that appears most frequently and/or has the most recently
   created opportunities — that is almost always the entry/intake stage.
3. Treat that stage as the "new / uncontacted" bucket for the report.
4. Note in the report header that stage names are unavailable (token scope
   limited) so the user knows why stage labels are missing.

## Step 2 — Bucket A: new leads not contacted yet

Search opportunities that are **open** and sitting in a first/entry stage. Use
the opportunities search endpoint:

```bash
source ~/.hermes/.env && curl -s --request GET \
  "https://services.leadconnectorhq.com/opportunities/search?location_id=$GHL_LOCATION_ID&status=open&limit=100" \
  -H "Authorization: Bearer $GHL_TOKEN" \
  -H "Version: 2021-07-28"
```

From the returned opportunities, keep those whose `pipelineStageId` matches a
first-stage ID you found in Step 1. Paginate until all open opportunities are
retrieved (follow whatever pagination the response provides — e.g. an offset, a
`startAfter`/`startAfterId` cursor, or a `meta.nextPageUrl`).

Heuristic note: "not contacted yet" is approximated as *open + in the entry
stage*. This matches most simple agency pipelines. If the user has a more
specific definition (a tag like `contacted`, a custom field, or a particular
stage name), prefer that — see "Tuning" below.

## Step 3 — Bucket B: going cold (no activity 3+ days)

From the full set of **open** opportunities (reuse the Step 2 fetch — don't
re-query), compute how stale each one is. Look for a timestamp field on each
opportunity such as `lastStatusChangeAt`, `updatedAt`, or `dateUpdated`
(use whichever the API actually returns; prefer the most recent activity-like
field). Convert to days-since-now.

Keep opportunities where days-since-last-activity is **greater than 3** AND the
opportunity is still open AND it is *not* already in Bucket A.

## Step 4 — Enrich with contact details

For each opportunity you're going to report, you'll usually have a `contactId`
and often the contact name on the opportunity itself. If a phone/email is
missing and you have a `contactId`, look it up:

```bash
curl -s --request GET \
  "https://services.leadconnectorhq.com/contacts/$CONTACT_ID" \
  -H "Authorization: Bearer $GHL_TOKEN" \
  -H "Version: 2021-07-28"
```

Only fetch contact details for leads that made it into the report — don't
enrich the entire database.

## Step 5 — Format the report

Produce a clean, mobile-friendly message. Plain text with light formatting is
safest for Telegram (avoid heavy MarkdownV2, which requires escaping many
characters and breaks easily). Suggested layout:

```
📋 Lead Follow-Up — {today's date}

🆕 New leads — not contacted ({count})
• {Name} — {phone or email} — {pipeline} → {stage}
  {opportunity link}
• ...

🕒 Going cold — no activity 3+ days ({count})
• {Name} — {phone or email} — {N} days quiet — {pipeline} → {stage}
  {opportunity link}
• ...
```

Rules:
- Sort each bucket oldest-first (most urgent at top).
- Opportunity deep link format:
  `https://app.gohighlevel.com/v2/location/{GHL_LOCATION_ID}/opportunities/list/{opportunityId}`
- If a bucket is empty, show it with "(none)".
- If **both** buckets are empty, send a single line: `✅ All caught up — no leads need follow-up today.`
- If there are a lot of leads (say >15 in a bucket), show the top 15 oldest and
  add a line like `…and 9 more` so the message stays readable.

## Account-specific notes

See `references/account-setup.md` for this user's token scope limitations,
empirically determined stage/pipeline IDs, delivery target, and scale numbers.
Load it when starting any GHL report run to skip re-discovery.

## Resilience and safety

- Never print or echo `GHL_TOKEN` anywhere, including error messages.
- On any API error, report the HTTP status and a short human explanation
  (e.g. "GHL returned 401 — the token may be invalid or missing scopes") rather
  than failing silently or fabricating data.
- If the pipelines call or search call returns zero results, say so plainly —
  don't pretend there are leads.

## Tuning (tell the user these are adjustable)

- **Staleness threshold:** "3 days" is set in Step 3 — change the number if the
  user wants a tighter or looser window.
- **Definition of "new/uncontacted":** defaults to *open + entry stage*. If the
  user works leads via a tag, a specific stage name, or a custom field, adjust
  Step 2 to filter on that instead.
- **Which pipelines:** by default this scans all pipelines. To limit to one,
  filter Step 1's results to the relevant `pipelineId`.

Project Overview

A fully automated gym membership management system built for Hot Spurs Gym, handling the complete member lifecycle — registration, renewal reminders, and expiry/renewal processing — with zero manual intervention.

Stack: n8n (self-hosted on Railway with PostgreSQL) · Fillout · Airtable · Cloudinary · Gmail · Telegram


The Three Workflows

✅ Workflow 1 — Member Registration

Fillout registration form → webhook triggers conditional Cloudinary photo upload (IF/Merge nodes handle photo and no-photo paths) → dynamic Member ID generation → calendar-based renewal date calculation per plan → dynamic pricing → Airtable record creation → branded navy/gold HTML confirmation email via Gmail → Telegram notification to gym owner.

✅ Workflow 2 — 3-Day Renewal Reminder

Daily Schedule Trigger → searches Airtable for members whose Renewal Date falls exactly 3 days out → IF node match → branded reminder email → Telegram alert to owner.

✅ Workflow 3 — Expiry Notification & Renewal Processing

Two chains on one canvas:


Chain 1: Daily schedule auto-flips member Status to Expired on their renewal date → sends expiry email with bank transfer details and a "Renew Now" button linking to a Fillout renewal form.
Chain 2: Triggered by renewal form submission → finds the expired record by email → updates Status, Payment Status, Start Date, Renewal Date, Membership Type, and dynamically recalculated Amount → sends confirmation email and Telegram alert.


Pricing structure: Monthly ₦15,000 · Quarterly ₦40,500 (10% discount) · Annual ₦135,000 (25% discount)


Migration Story

The system was originally built on Render's free tier, which uses ephemeral storage — meaning workflow data lived inside the container and was wiped on every restart or redeploy. This caused a full data loss pathway through the build. All three workflows were rebuilt from scratch on Railway with a persistent PostgreSQL database, permanently solving the storage issue and making the system production-durable.


Case Study: Debugging a Silent Failure in a Production n8n Workflow

The problem: After rebuilding the registration workflow, the photo-upload path tested cleanly. The no-photo path — supposedly the simpler branch — showed green "Succeeded" checkmarks on every node, yet produced empty or broken Airtable records. Nothing threw an error. The workflow reported success while silently failing.

Root causes (three separate, unrelated bugs stacked together):


Test URL vs. Production URL — Fillout was still pointed at n8n's Test URL, which works for manual testing but stays silent on real submissions.
Fillout Advanced mode — the webhook integration had been accidentally switched to Advanced mode with nothing mapped, sending a completely empty payload on every live submission.
Merge node behavior — the node combining the photo/no-photo branches was set to "Choose Branch" mode, which was assumed to auto-detect the active input. It didn't — it deterministically pulled from Input 1 (the photo branch) every time, even when that branch never ran.
A related fourth issue: once data was flowing, the Airtable Photo URL field referenced the Cloudinary node directly by name. On the no-photo path, that node never executes, so the reference itself was invalid — not just empty — causing a hard error. Fixed with a conditional fallback: {{ $('Upload to Cloudinary').isExecuted ? $('Upload to Cloudinary').item.json.secure_url : '' }}


Lesson: A green "Succeeded" status confirms only that no error was thrown — not that the data was correct. The only reliable way to validate a multi-branch workflow is to inspect actual node outputs at each step, on every distinct path, not just the one tested first. Conditional branches that converge (via Merge, Switch, or similar) deserve particular scrutiny, since their failure modes are often silent rather than loud.


### A Fifth Fix — Handling Missed Scheduler Runs (Production Edge Case)

While testing Chain 1's expiry detection, a subtler reliability issue surfaced — one that wouldn't show up in a single test run, but would cause real problems over time in production.

**The original logic:**
```
{{ $json.fields['Renewal Date'] }}  is equal to  {{ $now.toFormat('yyyy-MM-dd') }}
```

This only matched members whose Renewal Date was **exactly today**. Two problems with that:

1. **Missed runs are unrecoverable.** If the Schedule Trigger ever failed to run on the exact expiry day (server downtime, a Railway restart, etc.), that member's one-day match window would pass silently. Their Airtable record would stay "Active" indefinitely, even though they'd never renewed — the workflow would have no way to catch them on a later run.

2. **No protection against duplicate notifications.** Once a Renewal Date is in the past, it stays in the past. On a strict `<=` match with no other filter, the *same* expired member would be re-matched on every single subsequent daily run — meaning they'd get repeat expiry emails, and the gym owner would get repeat Telegram alerts about a member who was already flagged the day before.

**The fix — two conditions, combined with AND:**
```
{{ $json.fields['Renewal Date'] }}  is before or equal to  {{ $now.toFormat('yyyy-MM-dd') }}
{{ $json.fields['Status'] }}        is equal to      Active
```

Switching the date comparison from "is equal to" to "is before or equal to" means any member whose renewal date has passed — whether caught exactly on time or a few days late — still gets flagged, closing the missed-run gap. Adding the Status = Active condition means that once a member has already been flipped to "Expired" by this workflow, they're excluded from matching again on future runs, preventing repeat notifications.

**Lesson:** A condition that works correctly in a single manual test can still be fragile in production. Testing against "does this match today's exact scenario" isn't the same as testing against "what happens if this runs unreliably, or runs on the same stale data twice." Building in idempotency (safe to re-run without duplicating side effects) and tolerance for missed executions are both easy to overlook when a workflow is only ever tested happy-path.

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

The system was originally built on Render's free tier, which uses ephemeral storage — meaning workflow data lived inside the container and was wiped on every restart or redeploy. This caused a full data loss partway through the build. All three workflows were rebuilt from scratch on Railway with a persistent PostgreSQL database, permanently solving the storage issue and making the system production-durable.


Case Study: Debugging a Silent Failure in a Production n8n Workflow

The problem: After rebuilding the registration workflow, the photo-upload path tested cleanly. The no-photo path — supposedly the simpler branch — showed green "Succeeded" checkmarks on every node, yet produced empty or broken Airtable records. Nothing threw an error. The workflow reported success while silently failing.

Root causes (three separate, unrelated bugs stacked together):


Test URL vs. Production URL — Fillout was still pointed at n8n's Test URL, which works for manual testing but stays silent on real submissions.
Fillout Advanced mode — the webhook integration had been accidentally switched to Advanced mode with nothing mapped, sending a completely empty payload on every live submission.
Merge node behavior — the node combining the photo/no-photo branches was set to "Choose Branch" mode, which was assumed to auto-detect the active input. It didn't — it deterministically pulled from Input 1 (the photo branch) every time, even when that branch never ran.
A related fourth issue: once data was flowing, the Airtable Photo URL field referenced the Cloudinary node directly by name. On the no-photo path, that node never executes, so the reference itself was invalid — not just empty — causing a hard error. Fixed with a conditional fallback: {{ $('Upload to Cloudinary').isExecuted ? $('Upload to Cloudinary').item.json.secure_url : '' }}


Lesson: A green "Succeeded" status confirms only that no error was thrown — not that the data was correct. The only reliable way to validate a multi-branch workflow is to inspect actual node outputs at each step, on every distinct path, not just the one tested first. Conditional branches that converge (via Merge, Switch, or similar) deserve particular scrutiny, since their failure modes are often silent rather than loud.

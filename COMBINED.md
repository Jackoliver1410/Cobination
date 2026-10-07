# Combined app — SDR Outreach + Rajeshcrm features

Base: the fixed SDR Outreach app (React + Supabase, see `CHANGES.md`).
Added: the features chosen from `Jackoliver1410/Rajeshcrm`, rebuilt as pages on this app.
Data: starts empty. Nothing is migrated from either live system.

## What was added

| Page | Menu | What it does |
|---|---|---|
| Leads | 🎯 Leads | Lead intake and demo tracking. New lead → pending → manager approves. After approval a rep can only propose changes; a manager approves or rejects them. Excel export. |
| Activity Report | 📝 Activity Report | Daily numbers per rep (companies, contacts, emails, LinkedIn, responses). Rep edits until a manager approves; totals; Excel export. |
| Leave & WFH | 🌴 Leave & WFH | Balances, requests, approvals, team view, company holidays, leave types and per-person allowances. |
| Tasks | ✅ Tasks | To-do list with due dates; managers can assign to their team. (The old CRM had the API but no screen.) |
| Sequences | 🔁 Sequences | Rebuilt: step editor, templates, enrol contacts or a whole list, pause / resume / stop, real sending, send log, one-off email. |

New files: `src/pages/{Leads,ActivityReports,Leave,Tasks,Sequences}.jsx`, `src/components/ui/kit.jsx`,
`src/utils/{team,xlsx}.js`, `supabase/migrations/20261007_combined_features.sql`, `supabase/functions/send-email/`.

## Setup (three steps)

1. **Database** — open the Supabase SQL editor and run `supabase/migrations/20261007_combined_features.sql`.
   It only adds tables and functions; it does not touch existing data. Safe to run twice.
   Until it is run, the new pages show a message asking for it.
2. **Functions** — `supabase functions deploy admin-users generate-email generate-research send-email`
3. **Email sending** — set secrets for `send-email` (one provider):
   - Resend: `RESEND_API_KEY`, `MAIL_FROM` (an address on a domain verified with Resend)
   - or SMTP: `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`, `MAIL_FROM`
   - optional `APP_URL` for the link in approval emails

   Supabase functions cannot connect on ports 25 or 587. Office 365 SMTP only uses 587, so the
   old CRM's Office 365 SMTP settings will not work here. Use Resend, or an SMTP relay on 465 / 2525.
   Without these secrets everything else works; the app says sending is not switched on.

Then build and deploy the site as before: `npm ci && npm run build`.

## How the two role models were merged

| Rajeshcrm | Here |
|---|---|
| admin | admin |
| manager, sub_admin | sub-admin (sees everyone below them in the reporting chain) |
| sales_rep | owner |
| hr | **no separate role — an admin acts as HR** for leave types that need the second approval |

Rajeshcrm's per-user "data grants" and per-feature access switches were not brought over;
visibility follows the reporting chain in Team Members.

## Deliberate differences from Rajeshcrm

- **Leave balances are calculated**, not stored: allowance − approved − pending for the year.
  The old app subtracted on approval and never gave days back when a request was later deleted.
- **Pending requests count against the balance**, so two pending requests can't overbook it.
- **Overlapping requests are refused**, and the day count is checked on the server (weekends and holidays excluded; halves allowed).
- **Nobody below admin can approve their own** lead change, activity report or leave request.
- **A sequence step that fails to send stays due** and is retried. The old app marked it done and moved on.
- **Sequences stop automatically** when the contact replies, bounces, unsubscribes or is closed.
- **Nothing is sent by opening a page.** The old app sent due emails whenever the Emailing tab loaded; here it is always the "Send due emails" button, and each rep sends only their own.
- **Reps can email only their own contacts.**
- Step timing uses this app's existing format: `day` = days after enrolment (0, 3, 7…), not a gap from the previous step.
- `{{company}}` in templates is now filled in (it was always blank in the old app). Email bodies are escaped before sending.
- Deleting a lead / leave request asks for confirmation instead of re-entering your password.
- Approval rules live in the database (row-level security + functions), so they hold even if someone bypasses the screens.

## Not brought over (not selected)

Salesforce sync and iframe, Outlook mailbox connect and inbox sync, LinkedIn / Unipile search,
Apollo prospecting screens, Accounts Gem, Quick Add, browser-extension import, themes,
field-level permissions, company profile.
The SDR app's own Apollo list import and AI research are unchanged.

## What was tested

- The migration was run twice in a local Postgres and 39 checks passed: visibility per role, lead approval
  and change requests, activity report locking, leave day counting / balance / overlap / approval chain,
  template sharing, duplicate enrolment, send-log ownership, and signed-out access.
- All 26 routes were opened in a browser as admin, sub-admin and owner with mocked data, and the main
  flows on each new page were clicked through (add, approve, request change, submit leave, reject with
  reason, enrol, send, Excel export).
- **Not tested:** anything against the live Supabase project, and real email delivery
  (the `send-email` function was syntax-checked only).

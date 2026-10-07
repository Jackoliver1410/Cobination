# Bug-fix pass — 7 Oct 2026

Every source file under `src/` (not the stale `src/src/` copy) and `supabase/functions/` was reviewed.
Fixed code carries a comment at the spot explaining what was wrong.

## Would not run at all
- `supabase/functions/admin-users` — escaped backticks (`` \` ``) made the file a syntax error; it could not be deployed.
- Contacts → any "Date Added" / "Last Reached" preset called an undefined `dayStart()`; the table stuck on "Loading…".
- Lists → "Add from platform" crashed the whole page (`STACE_META` typo) when a contact was won / lost / bounced / cooling off.

## Access and sign-in
- A signed-in user with no profile row was let into the app; only `approved: false` was blocked.
- Profile was loaded after routes rendered (approval gate skipped for a moment; dashboard stuck on "Loading dashboard…").
- Supabase calls were awaited inside `onAuthStateChange` (documented deadlock). Deferred.
- Signup page was unmounted mid-registration by the public-route redirect; "already registered" emails got a confusing error.
- Pending-approval page now finishes an interrupted registration and has "Check again".
- Forgot password swallowed every error; reset password hung forever on an expired link.
- `reject_user` could delete any approved user, including the caller. Now pending signups only.
- Sub-admins could not edit a sub-admin teammate (or themselves); reporting loops and self-demotion of the last admin are now refused.
- `generate-email` / `generate-research` accepted any caller holding the public anon key. Now require a signed-in user.

## Wrong data written
- Contacts → "Start N Fresh" reset every selected contact (F1–F5, won, bounced) to Fresh.
- Lists → Apollo / CSV import upserted on email with no owner: overwrote existing contacts (stage reset to Fresh, fields blanked) and created ownerless ones.
- Accounts → "Generate All Missing" kept only the last section (each save overwrote the previous).
- Accounts → "Overwrite" duplicate contact reset its stage to Fresh; "Keep Both" never saved; duplicate-account check never ran.
- Accounts → contacts added from an account had no company name; CSV phone column was dropped; edits turned "1,000-5,000" employees into 1 and "1,500" revenue into 1.
- Follow-ups → Snooze wrote a future date into `last_contacted`; F5 → cooling off wrote `sequence_step = -1`; a failed stage update still logged the email as sent.
- Contact page → inferred persona / pitch type was never saved (query not awaited); advancing a stage did not update "Last Reached Out".
- Contacts CSV import → "Skip — import contacts only" dropped links to existing accounts; "$1,200" revenue failed account creation silently.
- Account Discovery → adding an account could blank its industry.

## Wrong numbers shown
- All four dashboards, Analytics, Reports, Leaderboard, Teams counted retired statuses (`fresh`, `replied`, `meeting`): replies/meetings were 0 and "contacted" was everyone. Now shared in `src/utils/metrics.js`.
  **"Meetings" columns are relabelled "Positive replies" (warm / prospect / won) — the data has no meeting field.**
- Activity feeds were empty: the query embedded `contacts(full_name)` (no such column) and an `org_hierarchy` join.
- Lists over 1000 rows were silently cut off; large `.in()` filters overflowed the URL and returned nothing.
- Accounts list hid accounts you own that have no contacts yet; the Team/All toggle and Responses team toggle checked retired role names.
- Follow-ups → something due today showed as overdue; "N sent" history button did nothing unless a draft was open.

## Other
- CSV parsing (3 places) broke on quoted line breaks and `""`; CSV exports did not escape values. Shared in `src/utils/csv.js`.
- Search boxes broke on commas / parentheses; stale responses could overwrite newer ones.
- LinkedIn links without `https://` opened inside the app; `javascript:` links are now refused.
- Settings downloaded the org AI key into every user's browser to test if one existed.
- AI functions treated failed Anthropic calls as empty success; em-dash removal glued words together; the two research prompts ran one after the other.
- Save failures across Contacts, Contact, Accounts, Lists, Follow-ups are now shown instead of ignored.
- Account Discovery claimed "AI research running" after add; that call could never work and was removed.

## Deploy
1. `npm ci && npm run build` (the committed `dist/` is already rebuilt).
2. Redeploy the three edge functions: `supabase functions deploy admin-users generate-email generate-research`.

## Not changed — needs a decision
- `src/src/` is an older copy of the app that nothing imports; `ApolloImport.jsx` and `LinkedInImport.jsx` have no route.
- `discover-prospects`, `apollo-proxy`, `enrich-contact`, `owner-scorecard` are called by the app but are not in this repo, so they were not reviewed.
- Default cadence has `Fresh: 0`, so a contact is due for F1 the same day the first email is sent.
- `/accounts/:id` (`AccountDetail.jsx`) is an older design using `account_intents` and status values the rest of the app no longer writes.

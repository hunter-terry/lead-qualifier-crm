# Test results — lead-qualifier-crm

Real runs against the live workflow (n8n + Ollama + a real Airtable
base on this machine), not just read-throughs. 4 cases, all behaved as
expected — same discipline as `lead-qualifier`'s
`test-leads/test-results.md`.

## Setup
- Workflow: `workflow.json`, imported and run in n8n
- Model: `llama3.1:8b` via local Ollama
- Airtable: free-tier base "Lead qualifier", table "Lead Quality"
  (Name/Email/Message/Score/Date), Personal Access Token scoped to
  just this base
- Test leads dropped into `~/lead-qualifier-crm-data/inbox/`, run via
  n8n's manual "Execute workflow"

## Case 1 — clean hot lead (`lead-hot-01.json`)
Jordan Ellis, urgent message, budget approved, wants to start this
week.
**Result:** `Score: "hot"`, `Date` set, Airtable record created
(`recReAvZ67WcTdleN`). Original file archived from `inbox/` to
`processed/`.

## Case 2 — clean cold lead (`lead-cold-01.json`)
Priya Nair, vague curiosity, "might look into this sometime next
year."
**Result:** `Score: "cold"`, Airtable record created
(`rechB66ExigTapYHq`). Archived correctly.

## Case 3 — invalid: missing required field (`lead-invalid-missing-field.json`)
Valid JSON, but no `message` field at all.
**Result:** `Score: "invalid"`, `Message: ""` (empty, not a crash).
Airtable record created (`recEM2o7moFJoL07x`) so the invalid case is
still visible/reviewable in the CRM rather than silently dropped.
Archived correctly.

## Case 4 — invalid: malformed file, not JSON at all (`lead-invalid-malformed.json`)
File content has a missing comma — fails to parse as JSON entirely.
**Result:** `Score: "invalid"`, every other field empty (`Name`,
`Email`, `Message` all blank since there was no parseable lead data).
Airtable record created (`rec6GJhRsEYRbA3hR`). Workflow did not crash
or stop — kept running cleanly. Archived correctly.

## Model-down safety path — attempted, inconclusive

Tried to reproduce `lead-qualifier`'s "Ollama unreachable" test (stop
Ollama, drop a lead, confirm clean `invalid` routing) by
`taskkill`-ing the `ollama.exe` process, then running the workflow.
Ollama auto-restarted itself (new PID, fresh process) within the wait
window before the workflow's HTTP call fired, so the test actually
re-ran against a healthy Ollama, not a down one — result was another
clean `hot` score, not a useful data point for this specific path.
Not re-attempted with a tighter race, since the underlying Parse &
Validate logic that handles this case is byte-for-byte unchanged from
`lead-qualifier`, where this exact failure mode (`Could not reach
Ollama`) was already proven with a real down instance — see that
project's `test-leads/test-results.md`, Case 5. Noting this honestly
rather than claiming a re-test that didn't actually happen.

## Bugs found and fixed during this build

**1. Missing Airtable token scope.** The Base/Table picker in n8n
needs to *list* bases, which requires the `schema.bases:read` scope —
I only initially had the user add `data.records:read`/`write`
(sufficient for writing, not for browsing). Fixed by adding the third
scope to the existing token in Airtable's settings; no new token
needed, nothing to re-paste into n8n.

**2. Force-killing n8n mid-write corrupted one node in the saved
workflow.** While resetting the local n8n owner account (`npx n8n
user-management:reset`, needed because the login from an earlier
build session was lost), I stopped the running n8n process with a
forced `taskkill`. This appears to have interrupted a write to n8n's
SQLite database, silently dropping the "Format & Archive for Airtable"
node from the *saved* workflow — the connections skipped straight from
"Parse & Validate" to the Switch node. Symptom: every test lead landed
on the Switch's "Unrouted" fallback path and errored on the Airtable
API with "Your request is invalid or could not be processed by the
service," because the un-reshaped item still had a `route` field, not
the `Score` field the Switch was checking for.
**Fix:** re-ran `npx n8n import:workflow --input=workflow.json` — the
source file on disk was never corrupted, only the DB copy — which
restored the missing node. Re-import also reset the 3 Airtable nodes'
Base/Table pickers (credentials survived, since those are stored
separately), so those had to be re-selected once more.
**Lesson for the fixes-log:** don't force-kill a running n8n process;
stop it via its own shutdown path (Ctrl+C / graceful stop) if at all
possible, since SQLite writes mid-flight aren't safe to interrupt.

## Security audit fixes, verified live (2026-07-15)

**Webhook token rotation.** The header-auth token was rotated (old one
had appeared in a Claude Code session transcript, so treated as burned
per `standards/security.md`). Confirmed: a request with the old token
gets `403`; the new token works. `credentials.json`,
`lead-capture-form.html`, and n8n's own credential store were all
updated together.

**Rate limiting.** Added a global sliding-window cap (10 requests/min)
on the `Lead Webhook` entry point, ahead of the file-write step. Fired
12 rapid requests with the new token: the first 10 succeeded
(`received: true`, files written), requests 11 and 12 got a clean
`{"error":true,"message":"Too many requests..."}` instead of being
written to `inbox/` or reaching Ollama. Test files cleaned up after
(not real leads).

**Real Airtable base/table ID leak, caught before it was ever pushed.**
`workflow.json`'s 3 Airtable nodes had picked up the real base ID,
table ID, and credential id/name after being wired through the n8n UI
(expected, per this file's "mirror live-wired values back" note in
`automation-platforms.md`) — but that meant the *next* commit would
have published them. Stripped back to blank (same pattern
`inquiry-triage` already used) before anything was pushed; confirmed
via the GitHub API that the public repo was never actually exposed.
Re-wiring the Base/Table pickers by hand in the UI is now required
again after any fresh `import:workflow` — same manual step the README
already documented, just re-triggered by this fix.

## What this proves
- Real qualified leads score "hot" and land in Airtable correctly
- Real-but-unready leads score "cold" and land in Airtable correctly
- Bad or missing lead data never reaches Airtable un-flagged — every
  failure mode is still recorded, but marked `Score: invalid` so
  it's visible for human review, not silently dropped
- Malformed (non-parseable) input does not crash the workflow or stop
  it from processing later leads
- The archive step (`inbox/` -> `processed/`) ran correctly in all 4
  cases regardless of which branch the lead took

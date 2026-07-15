# CONTEXT - lead-qualifier-crm
(your living "where I'm at" note - keep updating this as you go)

## One-liner
n8n workflow that scores incoming leads with local AI and writes the result straight into an Airtable base instead of a local file. Now also has a real front door: a quote-request form that feeds the same proven pipeline.

## Where I'm at right now (update, 2026-07-15)
Full security audit (all 3 n8n portfolio pieces) found the real one:
`workflow.json`'s 3 Airtable nodes had picked up the real base ID,
table ID, and credential id/name after being mirrored back from the
live UI (the documented practice from 2026-07-14, below) — meaning the
*next* commit would have published them. Confirmed via the GitHub API
this never actually happened (public repo still had blank
Base/Table). Stripped back to blank, matching `inquiry-triage`'s
pattern; the Base/Table pickers had to be re-selected by hand in the UI
afterward (same account, "Lead qualifier" base / "Lead Quality" table).
Also fixed in the same pass: rotated the webhook token (old one shown
in a session transcript, treated as burned), added a 10 req/min rate
limiter ahead of the file write, and added the same prompt-injection
defense as `lead-qualifier`'s SCORE/URGENCY check (see that project's
CONTEXT.md — identical Parse & Validate logic, same fix applied to
both). Everything verified against a real Airtable write, checked via
`execution_entity.status='success'`, not just "the webhook didn't
error." One operational wrinkle worth remembering: after re-wiring the
pickers through the UI, the fix didn't actually take effect until also
running `npx n8n publish:workflow --id=lead-qualifier-crm-001` and
restarting — a UI save alone wasn't enough, matching the exact
draft/publish trap already documented in the vault's
`playbooks/fixes-log.md` top entry (this is now a second confirmed
instance of it). Full findings across all 3 projects: vault root
`playbooks/fixes-log.md`.

## Where I'm at right now (update, 2026-07-14)
Gave this a real front door instead of the scrapped video demo (see
root CONTEXT.md and `playbooks/demo-recording.md`'s "Strategy update"):
- `lead-capture-form.html` (gitignored, real token;
  `lead-capture-form.example.html` is the committed template) — a
  "Fieldstone Automation" quote-request form (Name/Company/Email/
  Source/Project details), deliberately different branding from
  `inquiry-triage`'s contact form since it's a different kind of
  intake (sales lead vs support inquiry).
- Architecture choice: added a new "Lead Webhook" + "Write Lead to
  Inbox" Code node pair that just drops the submitted lead into the
  *same* inbox folder the existing Schedule Trigger already polls,
  rather than rebuilding the proven Ollama/Validate/Airtable chain
  around a webhook trigger. Zero changes to the already-shipped scoring
  logic; submissions show up in Airtable within ~60s (one schedule
  tick), not instantly.
- Tested for real twice, both via `execution_entity.status='success'`
  and confirmed archived: an urgent budget-approved lead (via curl) and
  a "just browsing, not urgent" lead (through the actual browser form).
- Real mistake made and fixed live: editing `workflow.json` locally and
  re-importing wiped the existing Airtable credential/base/table wiring
  on the 3 existing Airtable nodes, because the local source file had
  never been updated to match what got wired through the n8n UI back on
  2026-07-12/13 — only the live database had it. Fixed by re-exporting
  the live (correct) values and folding them back into the source file.
  This exact trap is now documented in
  `playbooks/automation-platforms.md` so it doesn't happen again on any
  project.
- Also (separately) republished with the correct `publish:workflow`
  command after being found inactive from an earlier session - see
  `playbooks/fixes-log.md`.

## Where I'm at right now
- Airtable account created, base built (Leads table: Name/Email/
  Message/Score(single-select hot,cold,invalid)/Date), Personal Access
  Token created (scoped to just this base, records read/write only).
- `workflow.json` hand-authored, grounded against the installed
  `n8n-nodes-base` package source (found at
  `%LOCALAPPDATA%\npm-cache\_npx\a8a7eec953f1f314\node_modules\n8n-nodes-base`)
  so node types/params aren't guessed:
  - Airtable node: `n8n-nodes-base.airtable`, typeVersion 2.2,
    credential type `airtableTokenApi`, `columns.mappingMode:
    autoMapInputData` (maps by field name, so upstream JSON keys must
    exactly match Airtable column names).
  - Reused `lead-qualifier`'s Schedule Trigger -> Read/Extract ->
    Ollama scoring -> Parse & Validate chain unchanged.
  - New "Format & Archive for Airtable" Code node reshapes the
    validated lead into the 5 Airtable columns AND archives the raw
    file (inbox/ -> processed/) in the same step, before the Airtable
    write - documented tradeoff in README (no data loss, no
    auto-retry on Airtable failure).
  - Switch now branches on `$json.Score` (hot/cold/invalid) instead of
    the old `route` field, into 3 separate Airtable Create nodes (same
    base/table).
- Synthetic `test-leads/` written fresh for this project (lead-qualifier's
  didn't have an email field, needed one for a CRM demo): 1 hot, 1 cold,
  2 invalid (missing field, malformed JSON).
- DONE (2026-07-12): imported into n8n, credential + base/table wired
  on all 3 Airtable nodes, all 4 test cases run for real against a
  live Airtable base. See `test-leads/test-results.md` for full
  results, including two real bugs hit and fixed along the way (a
  missing Airtable token scope, and a corrupted node in n8n's saved
  workflow after a force-kill — both logged in the vault's
  `playbooks/fixes-log.md`).
- DONE (2026-07-12): walkthrough doc written (docs/walkthrough.html,
  same design system as lead-qualifier's, real captured run data).
  QA checklist run: unused 1-in/2-work/3-done scaffold folders removed,
  .gitignore added, scanned clean for secrets/personal data, no .py
  files so compile-check is N/A. Model-down safety path attempted but
  inconclusive (Ollama auto-restarted before the test could hit it) -
  documented honestly in test-leads/test-results.md rather than
  claiming a pass; same code path already proven in lead-qualifier.
  One real limitation found and documented (not fixed, since it's not
  fixable at this node level): the Airtable node has no configurable
  timeout option, unlike the Ollama HTTP node.
- DONE (2026-07-12): pushed public —
  github.com/hunter-terry/lead-qualifier-crm. Third n8n portfolio
  piece shipped, alongside inquiry-triage and lead-qualifier.

## Steps / plan
- [x] Airtable base + token
- [x] workflow.json authored
- [x] Synthetic test leads written
- [x] Import into n8n, wire up credential + base/table pickers, run all
      4 test cases, capture real results (test-leads/test-results.md,
      same pattern as lead-qualifier)
- [x] Plain-language walkthrough doc
- [x] QA checklist
- [x] git init, push public to GitHub — done

## Where I'm at right now (update, 2026-07-13)
Pulled into a Terry Studio demo-recording session (Goal 1) that was
ultimately scrapped — see vault root CONTEXT.md and
`playbooks/demo-recording.md`. The `demo-leads/` folder created for it
has been deleted (scratch work). **What's still real and NOT rolled
back: 10 test leads were pushed into the live Airtable base during
that session and are still there.** Claude attempted to build a
cleanup workflow to remove them but was blocked by a safety check
(bulk list/delete against a live external data store needs Hunter's
direct action, not an inference from context) — **Hunter needs to
delete those 10 rows himself in the Airtable app** (Leads table, most
recent entries from 2026-07-13) before this base is clean again. Two
things fixed along the way,
both now required at every n8n startup on this machine:
`N8N_RESTRICT_FILE_ACCESS_TO` must include this project's
`lead-qualifier-crm-data` path (was missing, caused a clean "Access to
the file is not allowed" failure); and the workflow was found
unpublished/inactive mid-session for unclear reasons — republished, no
data lost, but worth a glance if executions ever silently stop working
here again.

## Notes
- Runtime data folder is `~/lead-qualifier-crm-data/` (deliberately
  different from lead-qualifier's `~/lead-qualifier-data/` so the two
  don't collide if both are ever run on the same machine).
- MODE: PERSONAL (portfolio proof) - see intake brief in the
  conversation this was built from; sanitize before publishing per the
  vault's QA checklist.

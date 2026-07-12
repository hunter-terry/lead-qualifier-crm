# CONTEXT - lead-qualifier-crm
(your living "where I'm at" note - keep updating this as you go)

## One-liner
n8n workflow that scores incoming leads with local AI and writes the result straight into an Airtable base instead of a local file.

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

## Notes
- Runtime data folder is `~/lead-qualifier-crm-data/` (deliberately
  different from lead-qualifier's `~/lead-qualifier-data/` so the two
  don't collide if both are ever run on the same machine).
- MODE: PERSONAL (portfolio proof) - see intake brief in the
  conversation this was built from; sanitize before publishing per the
  vault's QA checklist.

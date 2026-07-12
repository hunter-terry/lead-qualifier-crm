# lead-qualifier-crm

A self-hosted n8n workflow that scores incoming sales leads with local
AI (intent, urgency, a 1-10 qualification score), checks that the AI
actually returned usable structured data, and writes the result
straight into an Airtable base instead of a local file — the CRM
follow-up to [`lead-qualifier`](../lead-qualifier).

Same "never trust the raw AI reply" pattern as `lead-qualifier` and
[`inquiry-triage`](https://github.com/hunter-terry/inquiry-triage), one
new piece: instead of demoing with a stand-in file-based inbox all the
way through, this one lands the final record in a real tool a small
business would actually use to track leads — Airtable, free tier.

Ollama still runs locally, on-device. No lead data reaches a cloud AI
provider for scoring. The only data that leaves the machine is the
final validated record, written to your own Airtable base over its
API — the same trust boundary a real CRM integration would have.

## What it does

1. **Schedule Trigger** polls on an interval (demoed at 1 minute).
2. **Read/Write Files from Disk** + **Extract from File** (native n8n
   nodes) read and parse every lead file waiting in the inbox folder.
3. **Ollama Score Lead** — a local model (`llama3.1:8b`) reads the
   lead's name/company/source/message and returns an intent summary,
   an urgency level, and a 1-10 qualification score.
4. **Parse & Validate** — the model's reply is checked before it goes
   anywhere: SCORE must be a real 1-10 whole number, URGENCY must be
   low/medium/high, INTENT and SUMMARY must be present, the lead
   itself must have a name and a message. Fails clean (not a crash) if
   Ollama is unreachable or the model isn't pulled.
5. **Format & Archive for Airtable** — reshapes the validated lead into
   Airtable's exact columns (`Name`, `Email`, `Message`, `Score`,
   `Date`) and moves the original inbox file to `processed/` so the
   next poll doesn't reprocess it.
6. **Switch: Route Lead** (native n8n Switch node) routes on `Score`:
   `hot`, `cold`, or `invalid`.
7. **Airtable Create record** (one per branch) writes the record into
   your Airtable base — same base and table for all three, `Score`
   tells you which lane it came from.

```
Schedule Trigger --> Read Lead Files --> Extract Lead Data --> Ollama Score Lead
     (poll)          (native, disk)      (native, JSON)         (local AI call)
                                                                        |
                                                                        v
                                                              Parse & Validate
                                                               (rule checks)
                                                                        |
                                                                        v
                                                    Format & Archive for Airtable
                                                    (reshape record + move file)
                                                                        |
                                                                        v
                                                            Switch: Route Lead
                                                          (native, 3-way + fallback)
                                                          /        |         \
                                                 Airtable:Hot  Airtable:Cold  Airtable:Invalid
```

The full workflow is in [`workflow.json`](workflow.json) — importable
directly into n8n.

## Proven, not assumed

Tested against 4 real cases with a live Airtable base — see
[`test-leads/test-results.md`](test-leads/test-results.md) for the
full record, including two real bugs found and fixed while building
this (a missing Airtable token scope, and a corrupted workflow node
after an n8n process was force-killed). A plain-language walkthrough
with two real captured runs is at
[`docs/walkthrough.html`](docs/walkthrough.html).

Same standout pattern as `lead-qualifier`: the two deliberately bad
lead files (one missing a required field, one that wasn't even valid
JSON) both still produced an Airtable record — marked `Score: invalid`
rather than silently dropped, so a human can review what came in and
why it was rejected, without bad data ever being mistaken for a real
lead.

## A known tradeoff, stated plainly

The file gets archived to `processed/` at the same step that formats
the record for Airtable — *before* the Airtable write actually
happens, not after it succeeds. That means if the Airtable API call
fails (bad credential, rate limit, network blip), the lead data isn't
lost — the original file is sitting safely in `processed/`, not
deleted — but it also won't automatically retry on the next poll.
Recovering it means moving the file back to `inbox/` by hand. For a
personal/portfolio build this is an acceptable tradeoff; a client build
handling real revenue-bearing leads would get a proper failure branch
that retries or alerts instead.

## How to run it

Requirements: [n8n](https://n8n.io) (self-hosted, no account needed),
[Ollama](https://ollama.com) with a model pulled, and a free
[Airtable](https://airtable.com) account.

### 1. Airtable setup

1. Create a base (any name, e.g. "Lead Qualifier").
2. Inside it, a table called `Leads` with these fields:
   - `Name` — single line text (Airtable's default primary field)
   - `Email` — single line text
   - `Message` — long text
   - `Score` — single select, options: `hot`, `cold`, `invalid`
   - `Date` — date
3. Get a Personal Access Token at
   [airtable.com/create/tokens](https://airtable.com/create/tokens):
   scopes `data.records:read` and `data.records:write`, access limited
   to just this one base. Save the token somewhere safe — Airtable
   only shows it once.

No credential file is used or needed for this — the token lives
entirely inside n8n's own credential store (added directly in the n8n
UI when you open one of the three Airtable nodes), never in this repo,
never in a chat, never hardcoded.

### 2. Choosing a model for your machine

Same rule of thumb as `lead-qualifier`:

| Total RAM | Suggested model |
|---|---|
| ~8GB | `llama3.2:3b` |
| ~16GB | `llama3.1:8b` (default) |
| 32GB+ | a larger model, e.g. `llama3.1:70b` or `mixtral` |

To use a different model, edit the `model` field inside the "Ollama
Score Lead" node's JSON body.

```
ollama pull llama3.1:8b
```

### 3. Start n8n with the right permissions

The Code nodes need Node's `fs`, `path`, and `os` built-ins, and this
n8n version restricts native file-node access to `~/.n8n-files` by
default — the leads-data folder needs to be explicitly allowlisted.

Windows (PowerShell):
```
$env:NODE_FUNCTION_ALLOW_BUILTIN = "fs,path,os"
$env:N8N_RESTRICT_FILE_ACCESS_TO = "~/.n8n-files;~/lead-qualifier-crm-data"
npx n8n
```

macOS/Linux:
```
NODE_FUNCTION_ALLOW_BUILTIN=fs,path,os N8N_RESTRICT_FILE_ACCESS_TO="~/.n8n-files;~/lead-qualifier-crm-data" npx n8n
```

### 4. Import and finish wiring it up

```
npx n8n import:workflow --input=workflow.json
```

Open the workflow in the n8n UI (CLI import doesn't activate it) and
finish the three things only you can do, since they involve your own
account:
1. On each of the 3 Airtable nodes, create/select the Airtable
   credential and paste in your Personal Access Token.
2. On each Airtable node, use the Base and Table pickers (they're
   blank on import) to select your base and the `Leads` table.
3. Save and activate the workflow.

### 5. Try it

All data lives under `~/lead-qualifier-crm-data/`, resolved from the
home directory at runtime:

```
cp test-leads/lead-hot-01.json ~/lead-qualifier-crm-data/inbox/
```

Wait for the next poll (or trigger a manual execution in the n8n UI),
then check your Airtable base for the new record. `test-leads/` has
one hot, one cold, and two invalid cases (a missing required field,
and a deliberately malformed JSON file) to exercise every branch.

## Safety guarantees

- Local AI only — no lead data reaches a cloud AI provider for scoring.
- Every AI output is checked against hard rules before it moves on —
  the model's compliance with the requested format is never assumed.
- Failures (bad input, AI unreachable, rule violations) are caught and
  routed to the `invalid` branch with a specific, plain-language
  reason — never silently dropped, never crash the workflow.
- Airtable access is scoped to one token, one base, read/write records
  only — no broader account access requested.
- The original lead file is never deleted, only moved (see the
  tradeoff noted above).
- **Known limitation:** the Ollama call has an explicit 120s timeout
  (see `workflow.json`'s "Ollama Score Lead" node); n8n's Airtable
  node does not expose a configurable timeout option at all (checked
  against the installed node's source) — it relies on n8n's own
  default request timeout instead. A slow/hung Airtable API call would
  delay that one execution, not leak data or corrupt anything, but
  it's a real gap worth knowing about rather than assuming it's
  covered.

## Folders

- `workflow.json` — the exported n8n workflow (the actual deliverable)
- `test-leads/` — synthetic test cases and real test results (no real
  client data)
- `docs/walkthrough.html` — plain-language, client-facing demo page
  with two real captured runs
- Runtime output (`inbox/`, `processed/`) lives under
  `~/lead-qualifier-crm-data/`, outside this repo.

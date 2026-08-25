---
name: extend-cli
description: Use when driving Extend from a shell with the `extend` CLI — extracting structured data from PDFs or images, parsing documents to text or markdown, classifying or identifying the type of a document (e.g. telling MSA from SOW from NDA), splitting multi-document bundles into segments, filling PDF forms via a values schema, running multi-step document AI workflows, inspecting, watching, or listing Extend runs by ID (exr_, pr_, clr_, splr_, edr_, workflow_run_), or uploading documents to an Extend workspace and managing the resulting file_xxx IDs — even if the user describes the task without naming Extend (e.g. "pull line items from these invoices", "OCR these receipts", "categorize this contract", "fill out this PDF form", or "split this combined PDF into individual statements"). For terminal, script, and CI work on local files; if Extend MCP tools are connected in this session, prefer them unless the user asks for the CLI.
---

# Extend CLI

## Authentication

`extend whoami` shows the workspace, environment, and credential in effect. Two credential sources:

    export EXTEND_API_KEY=sk_xxx              # API key: scripts, CI, agents
    extend login                              # browser OAuth: interactive use

A stored `extend login` session is used automatically when no API key resolves; an API key always takes precedence. If neither is configured, ask the user to run `extend login` or supply `EXTEND_API_KEY` — never invent a key.

    export EXTEND_REGION=us|eu                # optional, default us
    export EXTEND_WORKSPACE_ID=ws_xxx         # required only for org-scoped API keys

Per-call equivalents: `--region eu`, `--workspace ws_xxx`. For API-version pinning or `EXTEND_BASE_URL`, run `extend help auth`.

## Pick the right action

| Need | Command |
|---|---|
| Run extraction on a document | `extend extract <input>` |
| Parse a document into structured text | `extend parse <input>` |
| Classify a document into a configured category | `extend classify <input>` |
| Split a multi-document PDF into segments | `extend split <input>` |
| Start a workflow run on a document | `extend run <input>` |
| Fill a PDF form using a schema with values | `extend edit <input>` |

`<input>` is a local file path (auto-uploaded), a `file_xxx` ID, or an `https://` URL. For batches of up to 1,000 inputs, use `<verb> batch`.

Every action verb that needs a processor takes `--using <id>` — the ID prefix tells you the type: `ex_*` (extractors), `cl_*` (classifiers), `spl_*` (splitters), `workflow_*` (workflows). `parse` runs alone (no processor); `edit` takes `--instructions` (free-form prose). See `extend edit --help` for the full set.

## When this skill is active

- **Documents come from disk, not from messages.** When the user references a document ("this contract", "these invoices", "the PDF") without giving a path, glance at the current working directory for matching files (`*.pdf`, `*.png`, `*.jpg`, `*.tif`) before asking. Real users say "this PDF" when there's exactly one in cwd.
- **File uploads always go through `extend files upload`.** Never substitute a host-tool File API (e.g. an inline file upload tool that returns its own `file_xxx` ID). The skill's file IDs are only legitimate when produced by `extend files upload` or returned in another `extend` response.
- **Run IDs (`exr_`/`pr_`/`clr_`/`splr_`/`edr_`/`workflow_run_`) are Extend's, not the host's.** When the user mentions one, reach for `extend runs get|watch|cancel` — not a host-tool task tracker.
- **"OCR" alone is ambiguous; the user's intent disambiguates.** If they want specific values out (totals, line items, dates, names) → `extract` with a configured extractor. If they want raw text or markdown of the page → `parse`. "OCR this receipt and grab the total" is `extract`, not `parse`.

## Wait, async, watch

Action verbs (`extract`/`classify`/`parse`/`split`/`edit`) **wait by default** for terminal state and print the result. Pass `--wait=false` to return the run ID immediately.

`extend run` (workflow runs) is **async by default** because workflow runs can take minutes to hours. Pass `--wait` to block on it.

Follow a run by ID, regardless of type:

    extend runs watch <run-id>

The run type is auto-detected from the ID prefix (`exr_`, `pr_`, `clr_`, `splr_`, `workflow_run_`, `edr_`). Use `--exit-status` to gate downstream scripts on success:

    extend runs watch exr_xxx --exit-status && downstream-script.sh

To inspect current state without polling: `extend runs get <id>`.

**Run-type quirks** (the things that defy reasonable assumptions):

- **Edit runs** (`edr_*`) are not listable — the API has no `LIST /edit_runs`. Use `extend runs get edr_xxx` for individual edit runs.
- **Parse runs** (`pr_*`) cannot be cancelled; the API rejects the attempt. Other run types support best-effort cancel.
- **Workflow batches** (returned by `extend run batch`) have **no GET endpoint**. `extend batches get`/`watch` work only on processor batches (`bpr_*`). Track workflow batches with `extend runs list --type workflow --batch <id>`.

For the per-command wait/profile/failure-status table: `extend help lifecycle`.

## Pagination

List commands return one page by default. Pass `--max N` to fetch up to N total results — the CLI auto-paginates internally and never makes you handle page tokens:

    extend runs list --type extract --status FAILED --max 100

Use `--all` only when you genuinely want every result (scripts, not agents). Power users can still cursor explicitly with `--page-token`, but most callers should not need to see tokens at all.

`--jq <expr>` filters JSON output before rendering, but cannot combine with `-o markdown` (markdown is not JSON). Use `-o json --jq '...'` and select the markdown chunk paths instead.

## Common workflows

### Stand up an extractor and run it

1. Author the extractor config. Use a JSON Schema root object; make primitive
   fields nullable ("type": ["string", "null"]); use clear field
   names/descriptions; use arrays for repeated rows; use "extend:type"
   for date/currency/signature fields. Currency fields must be objects with
   amount and iso_4217_currency_code, not primitive numbers. If extraction
   misses a value, inspect parse output before over-tuning the schema.
2. Create the extractor draft from the config body:

       extend extractors create --from-file extractor.json --name "Q3 invoices"

   Returns a new `ex_xxx` ID. The draft is editable but not yet deployed.
3. Iterate on the draft as needed:

       extend extractors update ex_xxx --from-file patch.json

4. Publish version 1.0 once the draft is solid:

       extend extractors versions create ex_xxx --release-type major

5. Run extraction against a document:

       extend extract invoice.pdf --using ex_xxx

### Create, deploy, and run a workflow

1. Create a workflow draft and capture its ID:

       WORKFLOW=$(extend workflows create --from-file '{"name":"Invoice workflow"}' \
           --jq '.id' -o raw)

   The draft is editable. It is not runnable until you deploy a version.
2. Author the step graph in workflow-steps.json, then update the draft:

       extend workflows update "$WORKFLOW" --from-file workflow-steps.json

   Every graph starts TRIGGER -> PARSE. EXTRACT steps reference an extractor
   by id and version. CLASSIFY/SPLIT routes use classificationId values and
   cannot use version "latest"; pin semver or use "draft".
3. Deploy the draft as an immutable named version:

       extend workflows versions create "$WORKFLOW" --name v1

4. Run it asynchronously, or add --wait to block until terminal:

       RUN=$(extend run invoice.pdf --using "$WORKFLOW" --version v1 -o id)
       extend runs watch "$RUN"

### Process a folder of inputs and inspect failures

1. Submit all inputs in one batch and capture the batch ID:

       BATCH=$(extend extract batch *.pdf --using ex_xxx --jq '.id' -o raw)

2. Wait for the batch to finish; gate downstream work on success:

       extend batches watch "$BATCH" --exit-status || echo "batch failed"

3. List runs that failed (or any other status) for inspection:

       extend runs list --type extract --batch "$BATCH" --status FAILED -o json

4. Pull a specific failed run's full payload (auto-detects type from prefix):

       extend runs get exr_yyy -o json

### Configure a webhook for workflow completions

1. Create the receiving endpoint and capture the signing secret
   (returned only once — store it):

       extend webhooks endpoints create --url https://x.com/hook \
           --name prod \
           --events workflow_run.completed,workflow_run.failed -o json \
           | jq -r '.signingSecret' > webhook.secret

2. Bind the endpoint to a specific workflow:

       extend webhooks subscriptions create \
           --endpoint whe_xxx --resource workflow_yyy \
           --events workflow_run.completed,workflow_run.failed

3. In your receiver, verify each incoming payload before trusting it:

       extend webhooks verify \
           --signature "$X_EXTEND_REQUEST_SIGNATURE" \
           --timestamp "$X_EXTEND_REQUEST_TIMESTAMP" \
           --secret "$(cat webhook.secret)" \
           --body-file payload.json

### Fill a PDF form

**Simple fills**: pass values inline as `--instructions` and auto-download
the filled PDF. The server detects form fields and applies the prose:

    extend edit form.pdf \
        --instructions "name is Acme Corp; date is 2026-04-15" \
        --output-file filled.pdf

**Structured fills** (when you already have a populated schema, or want a
repeatable shape): scaffold the schema once, populate values on each
field per the generated shape (`extend_edit:value` for explicit values;
`extend_edit:image` for PNG/JPEG signature images), and then run
`edit --schema`:

    extend edit schema generate form.pdf > schema.json
    # populate values on each field per the generated shape, then:
    extend edit form.pdf --schema schema.json --output-file filled.pdf

Combine both for fills that need conditional or formatting guidance the
schema cannot express:

    extend edit form.pdf --schema schema.json \
        --instructions "format dates as MM/DD/YYYY; leave spouse blank if single"

Without `--output-file`, the filled PDF stays on the server; fetch later
with `extend files download <file-id>`. If you use the response's
`output.editedFile.presignedUrl` directly, download it promptly; it expires
after 15 minutes.

### Fill a PDF form from values in another document

When the values live in a source document (e.g. fill a 1040 from a W-2):

1. Extract or parse the source to surface the values you need:

       extend parse w2.pdf -o markdown > w2-content.md
       # or, with a configured extractor:
       extend extract w2.pdf --using ex_xxx -o json > w2-values.json

2. Fill the target form with those values via `--instructions`,
   `--schema`, or both — see "Fill a PDF form" above. Make sure the
   document you pass to `extend edit` is the *target* (the form), not
   the *source* (the document you read values from).

### Iterate an extractor against an evaluation set

1. Define an evaluation set scoped to the extractor:

       extend evaluations create \
           --from-file '{"name":"Q3 truth","entityId":"ex_xxx"}'

2. Add ground-truth items in bulk:

       extend evaluations items create evs_yyy --from-file items.json

   Each item is `{fileId, expectedOutput}`; the response wraps them in
   `{evaluationSetItems: [...]}`.
3. Iterate on the extractor draft, then publish a new version
   (`extend extractors versions create` as in workflow 1).
4. Trigger an evaluation run (e.g. against the new version) and capture
   its ID:

       extend evaluations runs create evs_yyy --entity ex_xxx --entity-version 2.0

5. Runs are async; poll for per-item accuracy and metrics once it finishes:

       extend evaluations runs get esr_zzz -o json

## Command reference

One line per command — invocation plus a summary. **Run `extend <command> --help` for flags, examples, and per-command gotchas.** The processor-resource block is parametric (the four families share an identical seven-command shape).

### Action verbs

- `extend extract <input>` — Run extraction on a document.
- `extend extract batch <input>...` — Run extraction on up to 1,000 files in one batch.
- `extend parse <input>` — Parse a document into structured text.
- `extend parse batch <input>...` — Parse up to 1,000 files in one batch.
- `extend classify <input>` — Classify a document into a configured category.
- `extend classify batch <input>...` — Run classification on up to 1,000 files in one batch.
- `extend split <input>` — Split a multi-document PDF into segments.
- `extend split batch <input>...` — Run splitting on up to 1,000 files in one batch.
- `extend run <input>` — Start a workflow run on a document.
- `extend run batch <input>...` — Run a workflow on up to 1,000 files in one batch.
- `extend edit <input>` — Fill a PDF form using a schema with values.
- `extend edit schema generate <input>` — Detect form fields and scaffold an edit schema (sync).
- `extend edit templates get <template-id>` — Fetch a saved edit template by ID.

### Inspection

- `extend runs get <run-id>` — Fetch a single run by ID.
- `extend runs watch <run-id>` — Poll a run until it reaches a terminal state.
- `extend runs list` — List runs of a given processor type.
- `extend runs cancel <run-id>` — Cancel a run by ID.
- `extend runs delete <run-id>` — Delete a run record (any run type).
- `extend runs update <workflow-run-id>` — Update workflow run name and metadata (workflow runs only).
- `extend batches get <batch-id>` — Show one batch run by ID.
- `extend batches watch <batch-id>` — Poll a batch run until it reaches a terminal state.
- `extend files upload <path>` — Upload a local file and print its file_id.
- `extend files list` — List uploaded files.
- `extend files get <file-id>` — Show metadata for a file (with presigned download URL).
- `extend files delete <file-id>` — Delete an uploaded file.
- `extend files download <file-id>` — Download a file to local disk (or stdout with -O -).
- `extend download <id>` — Download file artifacts produced by a run, or fetch a file by ID.

### Processor resources

**Extractors, classifiers, splitters, and workflows share an identical seven-command shape.** Substitute `<plural>` and the corresponding ID prefix (`ex_`, `cl_`, `spl_`, and `workflow_`):

- `extend <plural> list` — Page through processors of this type.
- `extend <plural> get <id>` — Show one processor.
- `extend <plural> create --from-file body.json` — New draft.
- `extend <plural> update <id> --from-file patch.json` — Edit the draft. Deployed versions are immutable; the draft is the only mutable surface.
- `extend <plural> versions list <id>` — List published versions.
- `extend <plural> versions get <id> <version|draft>` — Show one version (or the draft).
- `extend <plural> versions create <id> --release-type major|minor` — Publish the draft as a new version.

**Workflows differ:** `versions create` uses `--name <deploy-name>` instead of `--release-type`. The deployed name is what `extend run --version` references.

### Webhooks

- `extend webhooks endpoints list` — List webhook endpoints.
- `extend webhooks endpoints get <endpoint-id>` — Show one webhook endpoint.
- `extend webhooks endpoints create` — Create a webhook endpoint.
- `extend webhooks endpoints update <endpoint-id>` — Update mutable fields on a webhook endpoint.
- `extend webhooks endpoints delete <endpoint-id>` — Delete a webhook endpoint.
- `extend webhooks subscriptions list` — List webhook subscriptions.
- `extend webhooks subscriptions get <subscription-id>` — Show one webhook subscription.
- `extend webhooks subscriptions create` — Subscribe an endpoint to events for a specific resource.
- `extend webhooks subscriptions update <subscription-id>` — Replace the enabled events on a webhook subscription.
- `extend webhooks subscriptions delete <subscription-id>` — Delete a webhook subscription.
- `extend webhooks verify` — Verify the HMAC-SHA256 signature on a webhook payload.

### Evaluations

- `extend evaluations list` — List evaluation sets.
- `extend evaluations get <evaluation-set-id>` — Show one evaluation set.
- `extend evaluations create` — Create an evaluation set.
- `extend evaluations items list <evaluation-set-id>` — List items in an evaluation set.
- `extend evaluations items get <evaluation-set-id> <item-id>` — Show one evaluation item.
- `extend evaluations items create <evaluation-set-id>` — Add one or more items to an evaluation set (bulk create).
- `extend evaluations items update <evaluation-set-id> <item-id>` — Update an evaluation item.
- `extend evaluations items delete <evaluation-set-id> <item-id>` — Delete an evaluation item.
- `extend evaluations runs create <evaluation-set-id>` — Start an evaluation set run.
- `extend evaluations runs get <run-id>` — Show one evaluation run.

## When this skill isn't enough

The body above shows the CLI's *shape*. For depth, use the help system before guessing:

- `extend <command> --help` — every flag, multiple worked examples, and the full per-command gotcha list. Reach for this whenever a flag isn't obvious or the catalog example doesn't cover your case.
- `extend help auth` — Authentication: env vars, regions, workspace, API version. Use on auth errors, when working with org-scoped API keys, or when picking a region.
- `extend help output` — Output formats, --jq, color, pagination, per-command defaults. Use when an output format is unexpected or when writing a non-trivial pagination loop.
- `extend help lifecycle` — Run lifecycle: sync vs async, polling, exit codes, watching. Use when reasoning about run states, polling profiles, or when `--exit-status` should fail.
- `extend help errors` — Error envelope, request_id, retry/backoff, common codes. Use when interpreting an error envelope, picking up a `request_id`, or filing a support ticket.

These commands run offline and never contact the Extend API.

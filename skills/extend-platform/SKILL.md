---
name: extend-platform
description: Use when working with the Extend document processing platform through its MCP server or API — extracting structured data from PDFs or images, parsing documents to markdown, classifying documents by type, splitting multi-document bundles, filling PDF forms, running document workflows, building and publishing extractors/classifiers/splitters, managing files and webhooks, or measuring processor accuracy with evaluation sets. Applies whenever Extend MCP tools (extract_data, parse_document, get_me, ...) are available in the session, or the user asks how to do document AI work with Extend.
---

# Extend Platform

Extend (extend.ai) turns unstructured documents — PDFs, images, Office
files, spreadsheets — into structured data. This skill teaches the
interaction patterns for driving Extend through its MCP server. The
tools themselves carry their own schemas and per-tool rules; this file
covers what spans tools.

Surfaces, in preference order for an agent session:

- **MCP server** (`https://mcp.extend.ai/mcp`): the primary surface
  this skill targets. Tools mirror the public API.
- **API / SDKs** (`api.extend.ai`; TypeScript, Python, Java, Go): for
  code the user will keep. Docs: https://docs.extend.ai
- **`extend` CLI**: for shell-driven jobs — covered by the separate
  `extend-cli` skill; prefer it when the user works in a terminal
  against local files.
- **Dashboard** (`dashboard.extend.ai`): for humans — visual
  processor editing, run review, human-in-the-loop approvals.

## Orient first

Call `get_me` before anything else. It returns the granted
workspace × environment targets, and every other tool (except the
documentation tools) REQUIRES `workspaceId` and `environment`
(`"TEST"` or `"PRODUCTION"`) arguments on every call. There are no
silent defaults. If a tool returns UNAUTHORIZED or NOT_FOUND
unexpectedly, re-call `get_me` — the grant may have changed, and
resources live per-workspace: a processor created in one workspace
does not exist in another, and TEST and PRODUCTION are separate
universes for runs and resources.

Prefer TEST for experiments when the grant allows it; use PRODUCTION
only when the user means it.

## Pick the right capability

| The user wants | Use |
| --- | --- |
| The text/content of the pages ("OCR this", "what does it say") | `parse_document` |
| Specific values out (totals, dates, line items, tables) | `extract_data` |
| To know what kind of document it is (MSA vs SOW vs NDA) | `classify_document` |
| To divide a combined file into per-document segments | `split_document` |
| To fill in a PDF form | `detect_form_fields`, then `edit_pdf` |
| A saved multi-step pipeline run on a document | `run_workflow` |
| Accuracy numbers against ground truth | evaluation tools |

Extract vs parse is the most common confusion: "OCR" usually means
parse if they want text, extract if they want fields.

## Async runs: start, wait, resume

Action tools create asynchronous runs and block briefly. If a run
finishes in time you get the result in one call. Otherwise you get
`status: "running"` plus a `runId` — that is normal, not an error.
Resume with the matching per-type get tool (`get_extract_run`,
`get_workflow_run`, ...) with `wait: true`, repeatedly if needed.
Workflow runs routinely take minutes to hours. A workflow in
`NEEDS_REVIEW` is paused for human review: give the user its
`dashboardUrl` and stop; do not treat it as a failure.

Every tool result may carry an `llmContext` field with instructions
conditional on that response (poll choreography, recovery steps,
handoff links). Follow it — it is more current than anything here.

## Files

- Upload once, run many: a `file_...` id from any tool result is
  reusable across every tool. Only Extend-issued file ids are valid —
  never substitute ids from other file systems.
- Documents already at a public `https://` URL can be passed directly
  as `{ "url": ... }` without uploading.
- On the hosted server, local files reach Extend through the browser
  upload handoff: `request_file_upload` returns a dashboard link —
  put the link in your visible reply, end the turn, and only after
  the user says they uploaded call `get_file_upload` for the ids.
- Presigned download URLs expire after ~15 minutes; download promptly
  or re-fetch with `get_file`.

## Authoring processors

Extractors, classifiers, splitters, and workflows are created as
mutable drafts; published/deployed versions are immutable. The draft
is the only thing `update_*` touches.

Before hand-writing any config or schema, fetch the documentation
page named in that tool's description via `get_documentation` — the
config dialects have non-obvious rules (nullable-type arrays,
`extend:type` currency objects, a required `"other"` classification,
workflow version pinning) that the docs state authoritatively.
Validation errors name the exact failing field path: fix only that
field and resubmit; do not regenerate the whole config.

Iterate with evaluation sets: create a set scoped to the processor,
add ground-truth items, publish a version, `run_evaluation`, and read
the accuracy metrics before publishing further changes.

## Documentation lookup

`search_documentation` finds pages; `get_documentation` fetches one.
The docs index is https://docs.extend.ai/llms.txt and every page is
fetchable as markdown by appending `.md`. Prefer these over guessing
API or config details.

## Cost awareness

Runs consume credits. Batches multiply that: `run_*_batch` accepts up
to 1,000 inputs per submission. Confirm scope with the user before
submitting large batches or PRODUCTION runs they have not explicitly
asked for.

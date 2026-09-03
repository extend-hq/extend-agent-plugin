---
name: extend-api
description: Use when writing code against the Extend document processing API or SDKs - building extraction, classification, splitting, parsing, PDF form-filling, or workflow integrations, authoring extraction schemas, or handling Extend webhooks - in Python, TypeScript, Java, or Go.
---
> ## Documentation Index
> Fetch the complete documentation index at: https://docs.extend.ai/llms.txt
> Use this file to discover all available pages before exploring further.
>
> ## API version
> The current API version is `2026-02-09`, served at the site root (no version prefix in URLs).
> If this page URL contains `/2025-04-21/` or `/2024-12-23/`, you are reading an older API version.
> Prefer the current docs at https://docs.extend.ai/llms.txt unless the user explicitly needs that older version.
> Do not treat older-version pages as the source of truth for new integrations.

# Extend Agent Context

> Context file for AI coding assistants building on the [Extend](https://docs.extend.ai) document processing platform.

## What is Extend?

Extend is a platform for building, evaluating, and deploying AI-powered document processing. It provides APIs and SDKs for:

- **Extraction** -- Pull structured data from documents using a JSON Schema
- **Classification** -- Categorize documents by type
- **Splitting** -- Divide multi-page documents into sections
- **Parsing** -- Convert documents into clean, structured text (markdown, etc.)
- **Editing** -- Detect and fill PDF form fields
- **Workflows** -- Orchestrate multiple processors into pipelines with conditionals, human review, webhooks, and more

Full documentation: https://docs.extend.ai
Searchable docs index: https://docs.extend.ai/llms.txt

---

## Authentication

All API requests require Bearer token authentication and an API version header. **If using an SDK, authentication and versioning are handled automatically -- the details below apply to raw HTTP requests only.**

```bash
curl -X POST "https://api.extend.ai/extract" \
  -H "Authorization: Bearer sk_YOUR_API_KEY" \
  -H "x-extend-api-version: 2026-02-09" \
  -H "Content-Type: application/json" \
  -d '{ ... }'
```

| Header | Value | Required |
|--------|-------|----------|
| `Authorization` | `Bearer sk_...` | Yes |
| `x-extend-api-version` | `2026-02-09` (latest) | Yes |
| `Content-Type` | `application/json` | For POST/PUT |

Get your API key from the [Developers page](https://dashboard.extend.ai/developers) in the Extend dashboard.

**Omitting `x-extend-api-version` on raw HTTP requests returns an error.** SDKs set this automatically.

---

## API Versions

The API is versioned by date via the `x-extend-api-version` header. The latest version is `2026-02-09`. SDKs target the correct version automatically when kept up to date. For the full list of versions, their status, and per-version changes, see the [API versioning docs](https://docs.extend.ai/api-reference/api-versioning).

**If you are on an older version**, see the [migration guide](https://docs.extend.ai/api-reference/migrations/2026-02-09/overview) for breaking changes in `2026-02-09`. Key changes:

- **Dedicated endpoints** per resource type (`/extract`, `/classify`, `/split`) replace the generic `/processor_runs` endpoint
- **New ID prefixes**: extractors use `ex_`, extract runs use `exr_`, classifiers use `cl_`, splitters use `sp_`
- **Synchronous processing** support on all endpoints (new `/extract`, `/classify`, `/split` sync endpoints)
- **Simplified responses**: single objects no longer wrapped in containers; list responses standardized to `{ "object": "list", "data": [...] }`
- **Inline configuration**: pass extractor/classifier/splitter config inline without pre-creating a resource -- useful for managing schemas entirely in code
- **SDK polling helpers**: `createAndPoll` / `create_and_poll` methods with exponential backoff built into updated SDKs

**Migration path**: Update your SDK to the latest version (automatically targets `2026-02-09`), then migrate endpoint-by-endpoint. The old `/processor_runs` and `/processors` endpoints still work on older API versions but are now under Legacy in the docs.

Docs: https://docs.extend.ai/api-reference/api-versioning

---

## SDKs

**Official SDKs** are available for Python, TypeScript, Java, and Go. All are generated from the API spec and target the latest API version automatically.

**Python:**
```bash
pip install extend-ai
```

**TypeScript:**
```bash
npm install extend-ai
```

**Java (Gradle):**
```gradle
dependencies {
  implementation 'ai.extend:extend-java-sdk:1.13.0'
}
```

**Go:**
```bash
go get github.com/extend-hq/extend-go-sdk
```

**Community SDK:**
- **Haskell** -- maintained by Mercury Technologies: https://github.com/MercuryTechnologies/extend

The Python, TypeScript, and Java SDKs include polling helpers (`create_and_poll` / `createAndPoll`) for async operations, plus webhook signature verification utilities. The Go SDK ships neither -- call `Create` and poll the run yourself (or use webhooks), and verify webhook signatures manually (HMAC-SHA256).

Docs: https://docs.extend.ai/sdks

---

## CLI

The `extend` CLI runs every core operation from the shell, with no code to write. **Reach for it for one-off tasks, exploration, and agent-driven actions you'd otherwise script with raw HTTP** (creating extractors/classifiers/splitters/workflows, running a doc, kicking off an eval set). For durable application code, prefer an SDK above.

Install (any one):

```bash
curl -fsSL https://extend.ai/install.sh | sh                # install script (macOS, Linux)
brew install extend-hq/tap/extend                           # Homebrew
npm install -g @extend-ai/cli                               # npm (or: npx @extend-ai/cli --help)
go install github.com/extend-hq/extend-cli/cmd/extend@latest
```

Authenticate, then verify with a read-only command:

```bash
export EXTEND_API_KEY="sk_..."
extend extractors list
```

An `EXTEND_API_KEY` in the environment always wins. Interactive machines can instead run `extend setup` (browser sign-in via OAuth, or save an API key); check the credentials in effect with `extend whoami`.

Common commands (each `<input>` is a local path, a `file_...` ID, or an `https://` URL):

| Goal | Command |
|------|---------|
| Parse to markdown | `extend parse <input>` |
| Extract fields | `extend extract <input> --config <json>` or `--using <ex_id>` |
| Classify | `extend classify <input> --using <cl_id>` |
| Split | `extend split <input> --using <spl_id>` |
| Fill a PDF form | `extend edit <input> --instructions "..."` or `--schema <file>` |
| Scaffold an edit schema | `extend detect-form <input>` (schema under `output.schema`) |
| Run a workflow | `extend workflows run <input> --using <wf_id>` |
| Bulk (up to 1,000 files) | `extend <verb> batch <inputs>...` for extract, parse, classify, split; `extend workflows run batch` for workflows |

Inspect runs with the typed commands under each verb (`extend extract runs list`, `extend workflows runs watch <id>`, and so on). Manage resources with `extend <noun> ...` (`extractors`, `classifiers`, `splitters`, `workflows`, `files`, `evaluations`, `webhooks`). Filter output with `-o json --jq '<expr>'`, and run `extend <command> --help` for exact flags. The CLI is the source of truth. Teach your harness to use it with `extend skill install`, which writes the `extend-cli` skill to `~/.agents/skills/extend-cli/SKILL.md` (symlinked into `~/.claude/skills/extend-cli` for Claude Code).

Docs: https://docs.extend.ai/agent-quickstart

---

## MCP Server

If your harness connects to MCP (Model Context Protocol) servers, Extend ships one: the full platform (parse, extract, classify, split, edit, workflows, files, webhooks, evaluations) as tools with built-in run polling and structured errors. Pick it over raw HTTP when your agent works through tool calls rather than code.

Connect to `https://mcp.extend.ai/mcp` and complete the OAuth browser sign-in.

Call `get_me` first to discover your granted workspace and environment targets; every run-scoped tool requires `workspaceId` + `environment` arguments. Once connected, the tool descriptions and server instructions carry the full interaction contract.

Setup instructions per client (Cursor, Claude, ChatGPT, VS Code, and others): https://docs.extend.ai/mcp.md

---

## API Endpoints (2026-02-09)

> **Note on SDK method names vs REST paths:** This document describes the REST API. SDK method names follow language conventions and may differ (e.g., REST `POST /extract_runs` maps to Python `client.extract_runs.create()` and TypeScript `client.extractRuns.create()`). Always confirm exact method signatures against the SDK source or docs when writing code.

### Base URL

| Region | URL |
|--------|-----|
| US1 (default) | `https://api.extend.ai` |
| US2 | `https://api.us2.extend.app` |
| EU1 (EU data residency) | `https://api.eu1.extend.ai` |

SDKs accept a `baseUrl` (TypeScript) or `base_url` (Python) parameter to select the region.

### Files

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/files/upload` | Upload a file (multipart form data) |
| GET | `/files/{id}` | Get file metadata + presigned download URL |
| GET | `/files` | List files |
| DELETE | `/files/{id}` | Delete a file |

### Extract

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/extract` | Extract data (sync, 5-min timeout) |
| POST | `/extract_runs` | Extract data (async) |
| GET | `/extract_runs/{id}` | Get extract run status/output |
| GET | `/extract_runs` | List extract runs |
| DELETE | `/extract_runs/{id}` | Delete an extract run |
| POST | `/extract_runs/{id}/cancel` | Cancel an in-progress run |
| POST | `/extractors` | Create an extractor |
| GET | `/extractors/{id}` | Get extractor details |
| POST | `/extractors/{id}` | Update an extractor |
| GET | `/extractors` | List extractors |
| POST | `/extractors/{extractorId}/versions` | Publish a new version |
| GET | `/extractors/{extractorId}/versions/{versionId}` | Get a version |
| GET | `/extractors/{extractorId}/versions` | List versions |

### Classify

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/classify` | Classify a file (sync, 5-min timeout) |
| POST | `/classify_runs` | Classify a file (async) |
| GET | `/classify_runs/{id}` | Get classify run |
| GET | `/classify_runs` | List classify runs |
| DELETE | `/classify_runs/{id}` | Delete a classify run |
| POST | `/classify_runs/{id}/cancel` | Cancel an in-progress run |
| POST | `/classifiers` | Create a classifier |
| GET | `/classifiers/{id}` | Get classifier |
| POST | `/classifiers/{id}` | Update classifier |
| GET | `/classifiers` | List classifiers |
| POST | `/classifiers/{classifierId}/versions` | Publish a new version |
| GET | `/classifiers/{classifierId}/versions/{versionId}` | Get a version |
| GET | `/classifiers/{classifierId}/versions` | List versions |

### Split

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/split` | Split a file (sync, 5-min timeout) |
| POST | `/split_runs` | Split a file (async) |
| GET | `/split_runs/{id}` | Get split run |
| GET | `/split_runs` | List split runs |
| DELETE | `/split_runs/{id}` | Delete a split run |
| POST | `/split_runs/{id}/cancel` | Cancel an in-progress run |
| POST | `/splitters` | Create a splitter |
| GET | `/splitters/{id}` | Get splitter |
| POST | `/splitters/{id}` | Update splitter |
| GET | `/splitters` | List splitters |
| POST | `/splitters/{splitterId}/versions` | Publish a new version |
| GET | `/splitters/{splitterId}/versions/{versionId}` | Get a version |
| GET | `/splitters/{splitterId}/versions` | List versions |

### Parse

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/parse` | Parse a file (sync, 5-min timeout) |
| POST | `/parse_runs` | Parse a file (async) |
| GET | `/parse_runs/{id}` | Get parse run |
| GET | `/parse_runs` | List parse runs |
| DELETE | `/parse_runs/{id}` | Delete a parse run |
| POST | `/parse_runs/{id}/cancel` | Cancel an in-progress run |

### Edit

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/edit` | Edit a PDF (sync, 5-min timeout) |
| POST | `/edit_runs` | Edit a PDF (async) |
| GET | `/edit_runs/{id}` | Get edit run |
| DELETE | `/edit_runs/{id}` | Delete an edit run |
| GET | `/edit_templates/{id}` | Get an edit template (source file + default config) |
| POST | `/edit_schemas/generate` | Detect form fields and generate an edit schema (deprecated) |
| POST | `/detect_form` | Detect form fields and generate an edit schema (sync, 5-min timeout) |
| POST | `/form_detection_runs` | Start a form detection run (async) |
| GET | `/form_detection_runs/{id}` | Get a form detection run |

### Workflows

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/workflow_runs` | Run a workflow |
| POST | `/workflow_runs/batch` | Batch run a workflow |
| GET | `/workflow_runs/{id}` | Get workflow run |
| POST | `/workflow_runs/{id}` | Update workflow run metadata |
| POST | `/workflow_runs/{id}/cancel` | Cancel a workflow run |
| DELETE | `/workflow_runs/{id}` | Delete a workflow run |
| GET | `/workflow_runs` | List workflow runs |
| POST | `/workflows` | Create a workflow |
| GET | `/workflows/{id}` | Get a workflow |
| POST | `/workflows/{id}` | Update a workflow |
| GET | `/workflows` | List workflows |
| POST | `/workflows/{id}/versions` | Create (deploy) a workflow version |
| GET | `/workflows/{id}/versions/{versionId}` | Get a workflow version |
| GET | `/workflows/{id}/versions` | List workflow versions |

### Evaluation Sets

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/evaluation_sets` | Create an evaluation set |
| GET | `/evaluation_sets/{id}` | Get an evaluation set |
| GET | `/evaluation_sets` | List evaluation sets |
| POST | `/evaluation_sets/{id}/items` | Create items |
| GET | `/evaluation_sets/{id}/items/{itemId}` | Get an item |
| POST | `/evaluation_sets/{id}/items/{itemId}` | Update an item |
| DELETE | `/evaluation_sets/{id}/items/{itemId}` | Delete an item |
| GET | `/evaluation_sets/{id}/items` | List items |
| POST | `/evaluation_set_runs` | Create (start) an evaluation set run |
| GET | `/evaluation_set_runs/{id}` | Get an evaluation set run |

### Webhook Endpoints

Manage where Extend delivers events. The create response includes a `signingSecret` that is returned **only once**. Store it securely for [signature verification](#signature-verification).

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/webhook_endpoints` | Create a webhook endpoint |
| GET | `/webhook_endpoints` | List webhook endpoints |
| GET | `/webhook_endpoints/{id}` | Get a webhook endpoint |
| POST | `/webhook_endpoints/{id}` | Update a webhook endpoint |
| DELETE | `/webhook_endpoints/{id}` | Delete a webhook endpoint (and its subscriptions) |

### Webhook Subscriptions

Subscribe an endpoint to events for a specific resource (e.g. a single workflow) instead of the endpoint's global `enabledEvents`.

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/webhook_subscriptions` | Create a webhook subscription |
| GET | `/webhook_subscriptions` | List webhook subscriptions |
| GET | `/webhook_subscriptions/{id}` | Get a webhook subscription |
| POST | `/webhook_subscriptions/{id}` | Update a webhook subscription |
| DELETE | `/webhook_subscriptions/{id}` | Delete a webhook subscription |

---

## Common Patterns

### Extract (sync) -- Python

```python
from extend_ai import Extend

client = Extend(token="sk_...")

# Sync extract -- blocks until complete (5-min timeout)
result = client.extract(
    file={"url": "https://example.com/invoice.pdf"},
    extractor={"id": "ex_..."},
)
print(result.output)
```

### Extract (async with polling) -- Python

```python
result = client.extract_runs.create_and_poll(
    file={"url": "https://example.com/invoice.pdf"},
    extractor={"id": "ex_..."},
)
print(result.status)  # "PROCESSED"
print(result.output)
```

### Extract (sync) -- TypeScript

```typescript
import { ExtendClient } from "extend-ai";

const client = new ExtendClient({ token: "sk_..." });

const result = await client.extract({
  file: { url: "https://example.com/invoice.pdf" },
  extractor: { id: "ex_..." },
});
console.log(result.output);
```

### Extract (async with polling) -- TypeScript

```typescript
const result = await client.extractRuns.createAndPoll({
  file: { url: "https://example.com/invoice.pdf" },
  extractor: { id: "ex_..." },
});
console.log(result.status); // "PROCESSED"
console.log(result.output);
```

### Typed extraction with Zod -- TypeScript

The TypeScript SDK supports inline Zod schemas with full type inference:

```typescript
import { ExtendClient, extendDate, extendCurrency } from "extend-ai";
import { z } from "zod";

const client = new ExtendClient({ token: "sk_..." });

const result = await client.extract({
  file: { url: "https://example.com/invoice.pdf" },
  config: {
    schema: z.object({
      invoice_number: z.string().nullable().describe("The invoice number"),
      invoice_date: extendDate().describe("The invoice date"),
      line_items: z.array(z.object({
        description: z.string().nullable(),
        amount: extendCurrency(),
      })).describe("Line items on the invoice"),
      total: extendCurrency().describe("Total amount due"),
    }),
  },
});

console.log(result.output.value.invoice_number); // string | null
console.log(result.output.value.total.amount);   // number | null
```

### Extract (sync) -- Go

The Go SDK is fully typed. It has no polling helper, so call `Extract` (sync) or `ExtractRuns.Create` and poll the run yourself.

```go
package main

import (
	"context"
	"fmt"
	"log"

	extend "github.com/extend-hq/extend-go-sdk"
	client "github.com/extend-hq/extend-go-sdk/client"
)

func main() {
	c := client.NewClient()

	result, err := c.Extract(context.TODO(), &extend.ExtractRequest{
		File: &extend.ExtractRequestFile{
			FileFromURL: &extend.FileFromURL{URL: "https://example.com/invoice.pdf"},
		},
		Config: &extend.ExtractConfigJSON{
			Schema: map[string]any{
				"type": "object",
				"properties": map[string]any{
					"invoice_number": map[string]any{"type": []string{"string", "null"}, "description": "The invoice number"},
				},
			},
		},
	})
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(result)
}
```

### Parse a document -- Python

```python
result = client.parse(file={"url": "https://example.com/doc.pdf"})
for chunk in result.output.chunks:
    print(chunk.content)
```

### Parse (async with polling) -- Python

```python
result = client.parse_runs.create_and_poll(
    file={"url": "https://example.com/doc.pdf"},
)
for chunk in result.output.chunks:
    print(chunk.content)
```

### Run a workflow -- Python

```python
result = client.workflow_runs.create_and_poll(
    file={"url": "https://example.com/doc.pdf"},
    workflow={"id": "wf_..."},
)
for step_run in result.step_runs or []:
    print(step_run.step.type)
    print(step_run.result)
```

### Run a workflow -- TypeScript

```typescript
const result = await client.workflowRuns.createAndPoll({
  file: { url: "https://example.com/doc.pdf" },
  workflow: { id: "wf_..." },
});

for (const stepRun of result.stepRuns ?? []) {
  console.log(stepRun.step.type);
  console.log(stepRun.result);
}
```

---

## Sync vs Async Processing

All processing endpoints (extract, classify, split, parse, edit) support both sync and async modes. Workflows are async-only.

- **Sync** (`POST /extract` / SDK: `client.extract()`) -- Blocks until complete. Has a **5-minute timeout**. Best for testing and small files.
- **Async** (`POST /extract_runs` / SDK: `client.extractRuns.createAndPoll()` or `client.extract_runs.create_and_poll()`) -- Returns immediately with a run ID. Poll with `GET /extract_runs/{id}` or use webhooks. No timeout limit.

**Use async for production workloads.** Large documents can exceed the 5-minute sync timeout. SDK `createAndPoll` / `create_and_poll` methods are the recommended approach -- they handle polling automatically with built-in backoff.

SDK polling helpers use a hybrid strategy: fast polling for 30 seconds, then gradual backoff up to 30-second intervals.

Terminal statuses: `PROCESSED`, `FAILED`, `CANCELLED` (also `NEEDS_REVIEW`, `REJECTED` for workflows).

Docs: https://docs.extend.ai/general/async-processing

---

## Managing configs (recommended workflow)

Processor and workflow configs can get large and deeply nested, and create/update calls validate them strictly. **When building these, write the config to a file in the project and submit it from there. Don't hand-author it inline.** If a request fails validation, the API returns the specific field path that's invalid; edit just that part of the file and resubmit, instead of regenerating the entire config from scratch.

This applies to every config-bearing resource:

| Resource | What to keep in a file |
|----------|------------------------|
| **Extraction** | The extractor `config` (especially the JSON Schema under `config.schema`) |
| **Classification** | The classifier `config` (types and their descriptions) |
| **Splitting** | The splitter `config` (split classes / rules) |
| **Parsing** | The parse `config` (`blockOptions`, chunking strategy) |
| **Editing** | The edit `config` / generated edit schema |
| **Workflows** | The full `steps` array (step definitions, conditionals, `next` routing) |

Suggested loop:

1. Save the config as JSON, e.g. `extractor.config.json` (or save the whole request body if that's simpler for your call).
2. Create/update by loading the file, for example `--from-file extractor.config.json` with the CLI, or `json.load(open("extractor.config.json"))` / `JSON.parse(fs.readFileSync(...))` in the SDKs.
3. On a validation error, read the returned field path, edit only that part of the file, and resubmit. Keep the file as the source of truth so each fix is incremental.

This keeps iterations cheap and avoids reintroducing earlier mistakes when a single field is wrong. Note that Extend normalizes configs on submit (see the schema rules below), so the stored config the API echoes back may differ slightly from your file. Diff against your file intentionally rather than expecting a byte-for-byte match.

---

## Versioning: processors and workflows

Both processors and workflows have one editable **draft** and immutable **published/deployed versions**. Edits always land on the draft; publishing snapshots it. Nothing that references a specific version ever changes behaviour when you publish again.

| | Processors (extractor / classifier / splitter) | Workflows |
|---|---|---|
| Edit the draft | `extractors.update(id, config=...)` (same for classifiers/splitters) | Studio editor, or deploy `steps` directly |
| Publish / deploy | `extractorVersions.create(id, releaseType: "major"\|"minor", description?)` → `"1.0"`, `"1.1"`, `"2.0"` | `workflowVersions.create(id, { name?, steps? })` → `"1"`, `"2"`, ... |
| List versions | `extractorVersions.list(id)` (draft first, then newest) | `workflowVersions.list(id)` |
| Reference in a run | `extractor: { id, version }` on `extract` / `extractRuns.create` | `workflow: { id, version }` on `workflowRuns.create` |
| `version` omitted | `"latest"`: newest published, else draft | Newest deployed, else draft |
| `"draft"` | Current draft (development only) | Current draft (development only) |
| Pinned | `"1.1"` | `"3"` |

**Major vs minor** is only a numbering signal: minor for changes that keep the output shape (descriptions, rules, base processor); major when the schema, classification types, or split identifiers change so downstream code must be updated.

**Workflow steps reference processors with an explicit `version`** (`"latest"`, `"draft"`, or `"1.1"`). Deploying a workflow version freezes that string, but `"latest"` still resolves at run time, so a fully reproducible deployed workflow needs pinned processor versions in its steps.

**Recommended promotion loop:** update the draft → run the evaluation set against `"draft"` → publish → move the version string in your config/workflow step → deploy the workflow.

Docs: https://docs.extend.ai/evaluation/publishing-processors (processors) and https://docs.extend.ai/workflows/workflow-versioning (workflows), all four SDKs.

---

## Test environment

A **test API key** routes every request to an isolated test environment: same base URL, same code, no production data, no production webhooks. Workflow and processor definitions are shared between environments; runs and webhook deliveries are not. Create test keys and test webhook endpoints with **Test Environment** toggled on in Studio's Developers tab. All SDKs read `EXTEND_API_KEY`; switch environments by switching the key. Docs: https://docs.extend.ai/general/test-environment-guide

---

## Extraction Schema (JSON Schema)

Extractors use JSON Schema to define output structure. Key rules:

- **Root must be `"type": "object"`**
- **All primitive fields must be nullable**: use `"type": ["string", "null"]` not `"type": "string"`
- **Objects and arrays cannot be nullable**
- **Max nesting depth**: 5 levels
- **Property names**: letters, numbers, underscores, hyphens only
- `"required"` arrays and `"additionalProperties": false` are **optional in the schema you submit**. Extend normalizes your config on create/update by marking every listed property `required` and injecting `"additionalProperties": false` on all objects. The echoed-back config therefore won't match yours byte-for-byte; that's expected, not an error. Include them yourself if you want your stored schema to be explicit.

### Supported types

| JSON Schema Type | Notes |
|-----------------|-------|
| `["string", "null"]` | Nullable string |
| `["number", "null"]` | Nullable number |
| `["integer", "null"]` | Nullable integer |
| `["boolean", "null"]` | Nullable boolean |
| `"object"` | Nested object (not nullable) |
| `"array"` | Array of objects or scalars (not nullable) |

### Special Extend types

| Type | Usage | Output format |
|------|-------|---------------|
| `"extend:type": "date"` | Add to string fields | `yyyy-mm-dd` |
| `"extend:type": "currency"` | Object with `amount` + `iso_4217_currency_code` | Structured currency |
| `"extend:type": "signature"` | Object with `printed_name`, `signature_date`, `is_signed`, `title_or_role` | Signature detection |

### Enums

Enums must include `null` and only support string values. Use `"extend:descriptions"` for disambiguation:

```json
{
  "status": {
    "enum": ["active", "inactive", "pending", null],
    "extend:descriptions": ["Currently active", "No longer active", "Awaiting activation"]
  }
}
```

### Field descriptions

Use `"description"` to guide extraction. Use `"extend:name"` for display names without changing output keys.

### Unsupported

`anyOf`, `oneOf`, `allOf`, regex patterns, conditional schemas, `const`.

Docs: https://docs.extend.ai/extraction/schema

### Legacy: Fields Array schema

Extractors created before April 2025 may use the legacy "Fields Array" configuration instead of JSON Schema. Key differences:

- **Fields Array** used a `fields` array with `id`, `name`, `type`, `description` per field. Output mixed data and metadata together within each field object.
- **JSON Schema** uses a standard `schema` object. Output cleanly separates `value` (extracted data) from `metadata` (confidence, citations) using path-based keys.

**To migrate**: Open your processor in Studio, click the three-dot menu, select "Migrate to JSON Schema." This creates a new processor with the converted schema while preserving your original.

New extractors should always use JSON Schema. See the [migration guide](https://docs.extend.ai/migrating-to-json-schema) for full details.

---

## Webhooks

Webhooks deliver HTTP POST notifications when processing events complete.

### Setup

1. Create an endpoint in the Extend dashboard under Developers > Webhook Endpoints
2. Subscribe to events at global, workflow, or processor scope
3. Choose delivery format: JSON (default) or Signed Download URL (for large payloads)

### Key events

The table below lists common events. For the full list (including edit, lifecycle, and CRUD events for all resource types), see the [webhook events docs](https://docs.extend.ai/webhooks/events).

| Event | Fires when |
|-------|-----------|
| `extract_run.processed` | Extraction completes |
| `extract_run.failed` | Extraction fails |
| `classify_run.processed` | Classification completes |
| `classify_run.failed` | Classification fails |
| `split_run.processed` | Splitting completes |
| `split_run.failed` | Splitting fails |
| `parse_run.processed` | Parsing completes |
| `parse_run.failed` | Parsing fails |
| `edit_run.processed` | PDF editing completes |
| `edit_run.failed` | PDF editing fails |
| `workflow_run.completed` | Workflow completes |
| `workflow_run.failed` | Workflow fails |
| `workflow_run.needs_review` | Workflow requires human review |
| `workflow_run.step_run.processed` | Individual workflow step completes |

### Signature verification

Extend signs every webhook with HMAC-SHA256. Use the SDK's built-in verification:

**Python:**
```python
event = client.webhooks.verify_and_parse(body=body, headers=headers, signing_secret="wss_...")
```

**TypeScript:**
```typescript
const event = client.webhooks.verifyAndParse(body, headers, "wss_...");
```

**Java:**
```java
WebhookEvent event = client.webhooks().verifyAndParse(body, headers, "wss_...");
```

The Go SDK has no verification helper. Verify the signature manually (steps below).

For manual verification:
1. Extract `x-extend-request-timestamp` and `x-extend-request-signature` headers
2. Construct message: `v0:{timestamp}:{body}`
3. HMAC-SHA256 with your signing secret
4. Compare signatures; reject if timestamp > 5 minutes old

Docs: https://docs.extend.ai/webhooks/configuration

---

## Workflows

Workflows chain processors into pipelines. Build them visually in Extend Studio or manage them entirely via the API (create, configure, version/deploy, and run -- see the Workflows endpoints above).

### Capabilities

- Extraction, classification, splitting steps
- Conditional routing based on extracted values or classification results
- Human review steps (pauses workflow for manual review)
- External data validation (call your API mid-workflow)
- Webhook response steps
- Formula calculations
- Parse step configuration
- Validation rules

### Defining processor steps

`EXTRACT`, `CLASSIFY`, and `SPLIT` workflow steps accept either a saved processor reference or a full inline config. When `config` is present, provide exactly one source:

| Step | Saved processor | Inline config |
|------|-----------------|---------------|
| `EXTRACT` | `config.extractor: { id, version }` | `config.extractorConfig: { schema, ... }` |
| `CLASSIFY` | `config.classifier: { id, version }` | `config.classifierConfig: { classifications, ... }` |
| `SPLIT` | `config.splitter: { id, version }` | `config.splitterConfig: { splitClassifications, ... }` |

Inline extraction configs must include `schema`; schema-less extraction is not supported inside workflows. Inline classify/split routes use the IDs declared in their inline classifications. `CONDITIONAL_EXTRACT` rules remain saved-reference only. Full reference: https://docs.extend.ai/workflows/configuring-workflows

### Running a workflow via API

Via SDK, use `client.workflowRuns.createAndPoll()` (TypeScript) or `client.workflow_runs.create_and_poll()` (Python) -- see Common Patterns above. Raw HTTP example:

```bash
curl -X POST "https://api.extend.ai/workflow_runs" \
  -H "Authorization: Bearer sk_..." \
  -H "x-extend-api-version: 2026-02-09" \
  -H "Content-Type: application/json" \
  -d '{
    "workflow": { "id": "wf_..." },
    "file": { "url": "https://..." }
  }'
```

### Workflow run statuses

| Status | Meaning |
|--------|---------|
| `PENDING` | Queued, not yet started |
| `PROCESSING` | Currently executing |
| `PROCESSED` | Completed successfully |
| `FAILED` | Failed (check `failureReason`) |
| `NEEDS_REVIEW` | Paused for human review |
| `REJECTED` | Rejected during human review |
| `CANCELLED` | Cancelled via API |

### Retryable failure reasons

These failures are transient and safe to retry automatically:
- `INTERNAL_ERROR` -- Unexpected server error
- `DOCUMENT_PROCESSOR_ERROR` -- Extraction step failed after retries

Non-retryable:
- `INVALID_WORKFLOW` -- Workflow configuration error
- `FAILED_TO_PROCESS_FILE` -- File could not be downloaded (check your URL)

Docs: https://docs.extend.ai/workflows/overview

---

## Error Handling

| Error Code | Description | Retryable |
|------------|-------------|-----------|
| `INVALID_REQUEST` | Bad request body or parameters | No |
| `UNAUTHORIZED` | Missing or invalid API key | No |
| `NOT_FOUND` | Resource doesn't exist | No |
| `RATE_LIMIT_EXCEEDED` | Too many requests -- back off and retry | Yes |
| `USAGE_BLOCKED` | Out of credits | No |
| `ENDPOINT_REMOVED` | Deprecated endpoint -- check error message for replacement | No |
| `INTERNAL_ERROR` | Server error | Yes |

SDKs raise typed exceptions for these errors (e.g., `RateLimitError`, `UnauthorizedError`). Error responses include a `requestId` -- provide this when contacting support.

Docs: https://docs.extend.ai/api-reference/error-handling

---

## Rate Limits

All rate limits are per-organization. If you receive `429 Too Many Requests`, implement exponential backoff. SDK polling helpers handle backoff automatically; for other SDK calls, add your own retry logic.

Docs: https://docs.extend.ai/general/rate-limits (includes current limits by endpoint)

---

## Evaluation Sets

Evaluation sets benchmark a processor (extractor, classifier, or splitter) against reviewed ground truth; they can also be run against workflows, which this section does not cover. **Use the API for this rather than building a custom harness**: the loop is four calls and the metrics are computed server-side with the same scoring Studio uses.

| Step | Endpoint | SDK (TS / Python) |
|------|----------|-------------------|
| 1. Create a set for a processor | `POST /evaluation_sets` `{ name, entityId, description? }` | `evaluationSets.create` / `evaluation_sets.create` |
| 2. Upload each file | `POST /files/upload` | `files.upload` |
| 3. Add items (≤100 per call) | `POST /evaluation_sets/{id}/items` `{ items: [{ fileId, expectedOutput }] }` | `evaluationSetItems.create` / `evaluation_set_items.create` |
| 4. Start a run | `POST /evaluation_set_runs` `{ evaluationSetId, entity?: { id, version? }, evaluationSetItemIds? }` | `evaluationSetRuns.create` / `evaluation_set_runs.create` |
| 5. Poll until terminal | `GET /evaluation_set_runs/{id}` | `evaluationSetRuns.retrieve` / `evaluation_set_runs.retrieve` |
| 6. Fix wrong ground truth | `POST /evaluation_sets/{id}/items/{itemId}` `{ expectedOutput }` | `evaluationSetItems.update` / `evaluation_set_items.update` |

**`expectedOutput` shape** depends on the processor type:

- Extractor: `{ "value": { ...object conforming to the extractor schema... } }` - nest under `value`, include every schema property. Easiest source: run the extractor on the file, correct the output, save it.
- Classifier: `{ "id": "<classification id>", "type": "<classification name>" }`
- Splitter: `{ "splits": [{ "classificationId", "type", "startPage", "endPage" }] }`

**Run semantics:** `entity.version` accepts `"draft"`, `"latest"`, or a published version like `"1.0"`; omitting `entity` runs the set's default processor at `draft`. Status goes `PENDING` → `PROCESSING` → `PROCESSED` | `FAILED` | `CANCELLED`. No SDK has a polling helper for evaluation runs; poll every few seconds. Read `metrics` only when status is `PROCESSED`.

**`metrics`** is discriminated on `type`: `EXTRACT` has `accuracy` and `fieldMetrics` (field path → `{ countTotal, countPresent, countExpected, countAccurate, accuracy }`, nested fields dot-joined); `CLASSIFY` has `accuracy` and `classificationMetrics` (per-class precision/recall/F1); `SPLITTER` has `precision`, `recall`, `f1`. The run returns aggregates only: per-document diffs, CSV export, custom matchers (fuzzy, LLM judge, vector), and field exclusion are Studio features.

Docs: https://docs.extend.ai/evaluation/creating-evaluation-sets and https://docs.extend.ai/evaluation/running-evaluation-sets (all four SDKs)

---

## Key Documentation Links

| Topic | URL |
|-------|-----|
| Getting started | https://docs.extend.ai/overview |
| Extraction quick start | https://docs.extend.ai/extraction/overview |
| Parsing quick start | https://docs.extend.ai/parsing/overview |
| JSON Schema reference | https://docs.extend.ai/extraction/schema |
| Extraction best practices | https://docs.extend.ai/extraction/best-practices/field-names-and-prompt-crafting |
| Async processing | https://docs.extend.ai/general/async-processing |
| Webhook setup | https://docs.extend.ai/webhooks/configuration |
| Webhook events | https://docs.extend.ai/webhooks/events |
| Workflow creation | https://docs.extend.ai/workflows/overview |
| Processors (saved, versioned configs) | https://docs.extend.ai/evaluation/processors |
| Evaluation sets (create) | https://docs.extend.ai/evaluation/creating-evaluation-sets |
| Evaluation sets (run + metrics) | https://docs.extend.ai/evaluation/running-evaluation-sets |
| Workflow versioning (deploy, pin) | https://docs.extend.ai/workflows/workflow-versioning |
| Publishing processor versions | https://docs.extend.ai/evaluation/publishing-processors |
| Test environment | https://docs.extend.ai/general/test-environment-guide |
| API versioning | https://docs.extend.ai/api-reference/api-versioning |
| 2026-02-09 migration | https://docs.extend.ai/api-reference/migrations/2026-02-09/overview |
| JSON Schema migration | https://docs.extend.ai/migrating-to-json-schema |
| SDKs | https://docs.extend.ai/sdks |
| Error codes | https://docs.extend.ai/api-reference/error-handling |
| Rate limits | https://docs.extend.ai/general/rate-limits |
| Supported file types | https://docs.extend.ai/general/supported-file-types |
| Credits | https://docs.extend.ai/general/how-credits-work |
| Confidence scores | https://docs.extend.ai/extraction/confidence-scores |
| Citations | https://docs.extend.ai/extraction/response-format |
| API reference (full) | https://docs.extend.ai/api-reference |
| Searchable docs index | https://docs.extend.ai/llms.txt |
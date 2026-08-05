# ESS Payments Document Ingestion - API Reference
> **Status:** Final v7 — entity-only routing (aligned with [IDP_LLD.md §4](./IDP_LLD.md#4-entity-routing--template-config)); gateway façade for maker/checker
> **Created:** 2026-08-02 · **Corrected:** 2026-08-03 (v2) · 2026-08-03 (v3) · 2026-08-03 (v4) · 2026-08-04 (v6/v7)
> **Related:** [README.md](./README.md) · [IDP_LLD.md §9](./IDP_LLD.md#9-code-implementation-strategy) · [IDP_UX_Design.md §8](./IDP_UX_Design.md) · [ESS_Payments_Document_Ingestion.bpmn](./ESS_Payments_Document_Ingestion.bpmn) · [../51786-idp-extraction-service/extraction-main](../51786-idp-extraction-service/extraction-main) · [../51786-workflow-management/src/main/java/com/sc/fss/iap/workflow/controller/WorkflowServicesController.java](../51786-workflow-management/src/main/java/com/sc/fss/iap/workflow/controller/WorkflowServicesController.java)

This document lists **all APIs** in the document ingestion & extraction upload flow: what each endpoint does, who calls it, and how it fits the end-to-end journey. Extraction service endpoints are **already built**; gateway upload/action endpoints are **planned**; workflow-management task APIs are **existing** but invoked **server-side by the gateway** for maker/checker actions (portal must not call them directly — see [§3](#3-maker-checker-action-apis--gateway-facade-required)).

---
## 1. API landscape
| Service | Base path (incl. servlet context path) | Caller | Status |
|---|---|---|---|
| `51786-payment-gateway-service` | `/api/fss/payments/gateway/v1/extraction-uploads` | Portal UI (makers/checkers) — upload, list, detail, **and all maker/checker actions** | **Planned** (new) |
| `51786-payment-gateway-service` | `/api/fss/payments/gateway/v1/extraction-uploads/internal` | `51786-idp-extraction-service`, ops | **Planned** (new, internal network only) |
| `51786-workflow-management` | `/api/fss/payments/workflow/v1/set`, `/v1/complete`, `/v1/assign` | **payment-gateway-service only** (server-side `WorkflowService` client) — **not** portal UI for this feature | **Existing, reused server-side** |
| `51786-idp-extraction-service` | `/v1/extract` | `payment-gateway-service` | **Built** |

> **v7 correction — entity-only routing.** `country` and `entity` are the same concept; use **`entity` only** on upload APIs, DB (`fss_payment_upload_meta.entity`), Camunda process variable, and registry lookup. Phase 1: `entity=ID` (Indonesia). If the built extraction service metadata map still expects a `country` key, `ExtractionServiceClient` maps `entity` → `country` at the HTTP boundary only — see [LLD §4](./IDP_LLD.md#4-entity-routing--template-config).

> **v6 correction — gateway façade for maker/checker actions.** Submit, Cancel, Approve, and Reject must call `POST .../submit|cancel|approve|reject` on **payment-gateway-service**. The gateway updates `fss_payment_upload_meta`, `fss_payment_data_ingest_details`, and `fss_payment_upload_audit`, then calls the existing `setTaskDetails` + `completeCurrentTask` contract **server-side**. Direct UI → workflow-management leaves gateway tables stale even when Camunda advances.

> **v4 correction - BPMN redesign removes the need for a new message-correlation endpoint.** In `ESS_Payments_Document_Ingestion.bpmn` (process id `ESS_Payments_Document_Ingestion`), `Event_ExtractionCompleted`, `ExtractionFailedBoundary`, and the message form of `CancelExtractionUpload` were removed. Extraction result is now a plain gateway decision evaluated the instant `Trigger_Data_Extraction` completes, and Cancel is now a plain gateway decision (`MakerDecisionGateway`) evaluated the instant `Extraction_MakerReview` completes. **The `/v1/correlate/correlateMessage` endpoint proposed in v3 §3.4 is no longer needed** - Cancel now uses the same two-call WFM pattern, but **only from the gateway** after DB + audit updates. See [README §Naming change (v3)](./README.md#naming-change-v3).

> **v3 correction (still valid) - base paths and naming verified against real source.** `payment-gateway-service` has **no** global `server.servlet.context-path`; every controller hardcodes its own full path in `@RequestMapping` (confirmed in `PaymentController`, `PaymentsEnrichmentController`, `BulkUploadController`, `SummaryController`, `TransactionDetailController`), always following `/api/fss/payments/{module}/v{n}/...`. None of those controllers use REST path variables for ids - lookups are always `@RequestParam` query parameters, and sub-actions are camelCase verb paths (`getByDTO`, `getUploadsByDTO`, `retry`, `getMessage/inbound`); `@PatchMapping` is never used in this service. `workflow-management`, by contrast, **does** set `server.servlet.context-path=/api/fss/payments/workflow` in `application.properties`, on top of which `WorkflowServicesController` adds `@RequestMapping("v1")` - every call in §3 needs that `/api/fss/payments/workflow` prefix. `51786-idp-extraction-service` sets no context path; `ExtractionController` maps directly to `/v1/extract`.

```mermaid
sequenceDiagram
    participant UI as Portal
    participant GW as payment-gateway-service
    participant WFM as workflow-management
    participant EXT as 51786-idp-extraction-service
    UI->>GW: POST /api/fss/payments/gateway/v1/extraction-uploads
    GW->>WFM: POST /api/fss/payments/workflow/v1/start/startWorkflowProcess?processKey=ESS_Payments_Document_Ingestion
    WFM->>GW: External task Trigger_Data_Extraction
    GW->>EXT: POST /v1/extract (sync)
    EXT-->>GW: 200 structuredOutput OR 422 status=FAILED
    GW->>WFM: complete Trigger_Data_Extraction with extractionStatus (ExtractionResultGateway evaluates immediately)
    UI->>GW: GET /api/fss/payments/gateway/v1/extraction-uploads/getDetail?extractionUploadId={id}
    UI->>GW: POST /api/fss/payments/gateway/v1/extraction-uploads/fields?extractionUploadId={id}
    UI->>GW: POST .../submit|cancel|approve|reject?uploadId={id}
    GW->>GW: Validate role + status; INSERT audit; UPDATE meta/details status
    GW->>WFM: setTaskDetails + completeCurrentTask (server-side WorkflowService client)
    WFM->>GW: External task Trigger_Payment_From_Extraction (on approve only)
    GW->>WFM: startMessageCorrelation (registry-resolved message, cross-process handoff to IAP_ID_Payments only)
```

---

## 2. Portal upload APIs - `payment-gateway-service`

**Base path:** `/api/fss/payments/gateway/v1/extraction-uploads` (hardcoded in the new controller's `@RequestMapping`, matching every sibling controller in this service - there is no `server.servlet.context-path` here).
**Auth:** existing portal JWT. Role groups: `paymentMaker`, `paymentChecker` (same groups already configured for `IAP ID Payments`).

### 2.1 Summary table

| Method | Path (relative to `/api/fss/payments/gateway/v1/extraction-uploads`) | Purpose | UI use |
| --- | --- | --- | --- |
| `POST` | *(bare)* | Upload PDF or ZIP (+ metadata); persist row(s); start `ESS_Payments_Document_Ingestion` per PDF | Upload button |
| `GET` | `/getUploads` | List uploads (recent first, filters) | Upload tab table |
| `GET` | `/getDetail?extractionUploadId={id}` | Upload detail + latest extraction run + fields (no blob) | Detail modal |
| `POST` | `/fields?uploadId={id}&detailId={detailId}` | Maker saves draft field edits (one instruction) | Save draft |
| `POST` | `/re-extract?uploadId={id}` | Start a new extraction run | Re-extract button |
| `POST` | `/performAction?uploadId={id}` | Maker/checker action — body `{ "action": "SUBMIT"\|"CANCEL"\|"APPROVE"\|"REJECT", "remarks": "..." }` — **preferred** unified endpoint (see [LLD §9.4](./IDP_LLD.md#94-extractionuploadactionservice--single-entry-switch-inside-service)) |
| `POST` | `/submit?uploadId={id}` | Maker submit (alias of `performAction`) | Submit to checker |
| `POST` | `/cancel?uploadId={id}` | Maker cancel (alias) | Cancel upload |
| `POST` | `/approve?uploadId={id}` | Checker approve (alias) | Approve |
| `POST` | `/reject?uploadId={id}` | Checker reject (alias) | Reject |

This mirrors the exact style already used in this service: bare `POST` for the primary create action (as in `BulkUploadController`'s bare `@PostMapping`), camelCase `get*` sub-paths for lookups (as in `getUploadsByDTO`/`getByDTO`), ids passed as `@RequestParam` query parameters rather than `{id}` path variables (as in `retryTransaction(@RequestParam String transactionId, ...)`), and `POST` - never `@PatchMapping`, which is not used anywhere in this service - for the mutating "edit" call.

**Maker/checker actions** must go through the gateway (`POST .../performAction` preferred, or `/submit`, `/cancel`, `/approve`, `/reject`) — **not** `workflow-management` directly. See [§3](#3-maker-checker-action-apis--gateway-facade-required).

### 2.2 `POST /api/fss/payments/gateway/v1/extraction-uploads`

**Content-Type:** `multipart/form-data`

| Part | Type | Required | Description |
| --- | --- | --- | --- |
| `file` | file | Yes | Single **PDF** (`.pdf`) **or ZIP** (`.zip`) containing one or more PDFs. ZIP: only `.pdf` entries are processed; all other members are ignored. ZIP with zero PDFs → `400`. |
| `entity` | string | Yes | **Routing key** — selects payment BPMN + LLM template; phase 1: `ID` |
| `deptId` | string | Yes | Department metadata |
| `processId` | string | Yes | Process metadata |
| `subProcessId` | string | No | Sub-process metadata |
| `activityId` | string | No | Activity metadata |
| `subActivityId` | string | No | Sub-activity metadata |

**Server actions (single PDF):**

1. Insert `fss_payment_upload_content` + `fss_payment_upload_meta` (`status=UPLOADED` → `PROCESSING`).
2. `POST {workflow-management}/v1/start/startWorkflowProcess?processKey=ESS_Payments_Document_Ingestion&referenceId={uploadId}` with process variable `entity`.
3. On extraction complete: one `fss_payment_data_ingest_details` row per `initiationDetail` instruction.

**Server actions (ZIP bulk):**

1. Generate `batchId` (UUID).
2. Unpack ZIP; for each `.pdf` entry (case-insensitive): same steps as single PDF, with `batch_id=batchId`. Non-PDF entries collected in `skippedEntries` (not stored).
3. If no PDF entries found → `400` with `"No PDF files found in archive"`.
4. One `ESS_Payments_Document_Ingestion` instance **per extracted PDF**.

**Response `201 Created` (single PDF):**

```json
{
  "uploadId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "uploadStatus": "PROCESSING"
}
```

**Response `201 Created` (ZIP bulk):**

```json
{
  "batchId": "batch-uuid",
  "uploads": [
    { "uploadId": "...", "fileName": "doc1.pdf", "uploadStatus": "PROCESSING" },
    { "uploadId": "...", "fileName": "doc2.pdf", "uploadStatus": "PROCESSING" }
  ],
  "skippedEntries": ["readme.txt"]
}
```

**Errors:** `400` missing file/invalid metadata/no PDFs in ZIP, `401`/`403` not in `paymentMaker` group, `413` file too large, `500` DB or workflow-start failure.

### 2.3 `GET /api/fss/payments/gateway/v1/extraction-uploads/getUploads`

| Param | Type | Default | Description |
| --- | --- | --- | --- |
| `sort` | string | `recent` | `recent` = `uploaded_timestamp` DESC |
| `batchId` | string | - | Filter rows from the same ZIP upload |
| `entity` | string | - | Filter by routing key (e.g. `ID`) |
| `status` | string | - | Filter by `uploadStatus` |
| `myUploads` | boolean | `false` | Current user only |
| `page` / `size` | int | `0` / `20` | Paging |

**Response `200 OK`:**

```json
{
  "content": [
    {
      "uploadId": "a1b2c3d4-...",
      "batchId": null,
      "fileName": "3897122.pdf",
      "uploadedTimestamp": "2026-07-30T10:22:00Z",
      "uploadStatus": "READY_FOR_REVIEW",
      "overallConfidence": 97.23,
      "uploadedBy": "maker_user",
      "paymentRef": null,
      "entity": "ID"
    }
  ],
  "page": 0, "size": 20, "totalElements": 42
}
```

**UI behaviour:** poll every 5s while any row has `uploadStatus` in `UPLOADED`/`PROCESSING`; disable row click while `PROCESSING`.

**Upload status values:** `UPLOADED`, `PROCESSING`, `READY_FOR_REVIEW`, `SUBMITTED`, `HANDOFF_IN_PROGRESS`, `REJECTED`, `APPROVED`, `COMPLETED`, `FAILED`, `CANCELLED` — see [LLD §5.3](./IDP_LLD.md#53-status-lifecycle).

### 2.4 `GET /api/fss/payments/gateway/v1/extraction-uploads/getDetail?uploadId={id}`

`uploadId` (= `fss_payment_upload_meta.id`) is a required `@RequestParam`. Does **not** return the file blob.

**Response `200 OK`:**

```json
{
  "uploadId": "a1b2c3d4-...",
  "fileName": "3897122.pdf",
  "entity": "ID",
  "status": "READY_FOR_REVIEW",
  "uploadedBy": "maker_user",
  "uploadedAt": "2026-07-30T10:22:00Z",
  "processedAt": null,
  "ingestDetails": [
    {
      "detailId": "detail-uuid-1",
      "instructionIndex": 0,
      "status": "READY_FOR_REVIEW",
      "confidenceScore": 97.23,
      "txRef": "3897122-001",
      "subActivityId": "Redemption",
      "activityId": "BT",
      "trnTyp": "Redemption",
      "valueDate": "2025-10-29",
      "clientName": "PT BNI ASSET MANAGEMENT",
      "debitAmount": "5000000000",
      "lowConfidenceFieldCount": 0,
      "messageId": null,
      "handoffAt": null,
      "paymentId": null,
      "retry": 0,
      "errorDesc": null,
      "extractedData": { "header": { "..." }, "initiationDetail": { "..." } }
    }
  ],
  "camundaTasks": [
    { "taskId": "camunda-task-id", "taskDefinitionKey": "Extraction_MakerReview", "assignee": "maker_user" }
  ]
}
```

`ingestDetails` maps to `fss_payment_data_ingest_details` — one object per instruction. `extractedData` is the per-instruction `{ header, initiationDetail }` JSON. API query param remains `uploadId` (maps to `fss_payment_upload_meta.id`; legacy alias `extractionUploadId` accepted if needed).

**Errors:** `404` unknown id, `403` cannot view this upload.

### 2.5 `POST /api/fss/payments/gateway/v1/extraction-uploads/fields?uploadId={id}&detailId={detailId}`

Saves maker edits for **one instruction** (`detailId` = `fss_payment_data_ingest_details.id`). Re-denormalizes summary columns on save.

**Request body:**

```json
{
  "fields": [
    { "section": "TRANSACTION", "legIndex": null, "fieldName": "BankChg", "fieldValue": "No" },
    { "section": "DEBIT", "legIndex": 0, "fieldName": "DrAccNo", "fieldValue": "30681655612" }
  ]
}
```

**Response `200 OK`:** `{ "uploadId": "...", "detailId": "...", "status": "READY_FOR_REVIEW", "savedAt": "..." }`

### 2.6 `POST /api/fss/payments/gateway/v1/extraction-uploads/re-extract?uploadId={id}`

**Allowed when:** meta `status` is `READY_FOR_REVIEW`, `REJECTED`, or `FAILED`, caller is `paymentMaker`. Resets all detail rows: `retry+1`, clear `extracted_data`/`error_desc`, `status=PROCESSING`; re-fire extraction.

**Response `202 Accepted`:** `{ "uploadId": "...", "status": "PROCESSING" }`

---

## 3. Maker / checker action APIs — gateway façade (required)

> **v6 correction:** Earlier versions (v4) had the portal call `workflow-management` directly for Submit / Cancel / Approve / Reject. That **does not** update `fss_payment_upload_meta`, `fss_payment_data_ingest_details`, or `fss_payment_upload_audit`. **All four UI action buttons must go through the gateway** — the gateway owns DB + audit, then calls the existing `WorkflowServicesController` contract **server-side** via the same `WorkflowService` client already used for `startWorkflowProcess`.

**Base path:** `/api/fss/payments/gateway/v1/extraction-uploads`  
**Auth:** JWT; `paymentMaker` for submit/cancel; `paymentChecker` for approve/reject.

### 3.1 Action endpoints

| Method | Path | Caller | Gateway DB status after success |
| --- | --- | --- | --- |
| `POST` | `/performAction?uploadId={id}` | Maker / Checker | Per `action` in body — **preferred** (see [LLD §9.4](./IDP_LLD.md#94-extractionuploadactionservice--single-entry-switch-inside-service)) |
| `POST` | `/submit?uploadId={id}` | Maker | meta `SUBMITTED`; all details `SUBMITTED` |
| `POST` | `/cancel?uploadId={id}` | Maker | meta `CANCELLED`; all details `CANCELLED` |
| `POST` | `/approve?uploadId={id}` | Checker | meta `HANDOFF_IN_PROGRESS` → `APPROVED` when `ExtractionPaymentHandoffHandler` completes |
| `POST` | `/reject?uploadId={id}` | Checker | meta `REJECTED`; all details `REJECTED` |

**Request body (`performAction` — all actions):**

```json
{ "action": "SUBMIT", "remarks": "optional except REJECT" }
```

`action` values: `SUBMIT`, `CANCEL`, `APPROVE`, `REJECT`.

**Request body (cancel / reject on alias endpoints):**

```json
{ "remarks": "Required for reject; optional for cancel" }
```

**Response `200 OK` (all actions):**

```json
{
  "uploadId": "a1b2c3d4-...",
  "status": "SUBMITTED",
  "action": "MAKER_SUBMIT",
  "actedAt": "2026-07-30T11:00:00Z",
  "actedBy": "maker_user"
}
```

**Errors:** `400` invalid status transition, `403` wrong role, `404` unknown upload, `409` no active Camunda task / already actioned, `502` workflow-management call failed (DB rolled back or left unchanged — see §3.3).

### 3.2 `ExtractionUploadActionService` orchestration (gateway Java)

Each action runs in this order:

```
1. Load fss_payment_upload_meta (+ details); verify caller role and allowed status transition
2. Verify active Camunda task matches expected taskDefinitionKey (Extraction_MakerReview | Extraction_CheckerReview)
3. BEGIN transaction
4. INSERT fss_payment_upload_audit (action, actor, before_status, after_status, remarks, details JSON snapshot optional)
5. UPDATE fss_payment_upload_meta.status (+ remarks, approved_by/approved_at on approve)
6. UPDATE fss_payment_data_ingest_details.status in bulk where applicable
7. COMMIT
8. workflowClient.setTaskDetails(uploadId, vars)  // same vars as §3.4 table
9. workflowClient.completeCurrentTask(uploadId)
10. On workflow HTTP failure: compensating UPDATE to restore prior status + audit row FAILURE note; return 502
11. On approve success: return immediately; meta → APPROVED when ExtractionPaymentHandoffHandler finishes (async)
```

**Idempotency:** reject duplicate `POST` with same action when status already terminal → `409`.

### 3.3 Why not UI → workflow-management directly?

| Concern | Direct WFM call | Gateway façade |
| --- | --- | --- |
| `fss_payment_upload_meta.status` | Stale until external task runs (often never for user tasks) | Updated in step 5 |
| `fss_payment_data_ingest_details.status` | Not updated | Synced per action |
| `fss_payment_upload_audit` | Not written | Written in step 4 |
| UI table/modal | Wrong status on refresh | Correct immediately |
| Checker `remarks` | Only in Camunda variables | Persisted on meta + audit |

External task handlers (`ExtractionTriggerHandler`, `ExtractionPaymentHandoffHandler`) **still** update DB for system-driven steps (extraction complete, payment handoff). User-driven steps **must** use §3.1.

### 3.4 Camunda variables (internal — gateway sets these server-side)

The portal **does not** call these endpoints. The gateway uses the existing `WorkflowService` / REST client against `51786-workflow-management`:

| UI action (gateway path) | Active task | Variables (`setTaskDetails`) | `completeCurrentTask` |
| --- | --- | --- | --- |
| `POST .../submit` | `Extraction_MakerReview` | `{ "makerAction": "SUBMIT" }` | yes |
| `POST .../cancel` | `Extraction_MakerReview` | `{ "makerAction": "CANCEL", "remarks": "..." }` | yes |
| `POST .../approve` | `Extraction_CheckerReview` | `{ "extractionApproved": true }` | yes |
| `POST .../reject` | `Extraction_CheckerReview` | `{ "extractionApproved": false, "remarks": "..." }` | yes |

Underlying WFM paths (unchanged):

| Method | Path |
| --- | --- |
| `PUT` | `/api/fss/payments/workflow/v1/set/setTaskDetails/{uploadId}` |
| `PUT` | `/api/fss/payments/workflow/v1/complete/completeCurrentTask/{uploadId}` |

`uploadId` = Camunda business key = `fss_payment_upload_meta.id`.

### 3.5 Status transitions (gateway-enforced)

| Current meta `status` | Action | New meta `status` |
| --- | --- | --- |
| `READY_FOR_REVIEW`, `REJECTED` | `submit` | `SUBMITTED` |
| `READY_FOR_REVIEW`, `REJECTED` | `cancel` | `CANCELLED` |
| `SUBMITTED` | `approve` | `HANDOFF_IN_PROGRESS` → `APPROVED` (handler) |
| `SUBMITTED` | `reject` | `REJECTED` |

System-driven only (not UI buttons): `PROCESSING`, `FAILED`, `COMPLETED` — set by `ExtractionTriggerHandler` / `ExtractionPaymentHandoffHandler` / reconciliation.

### 3.6 Superseded (v4)

v4 stated no gateway proxy was needed because Cancel became a plain gateway completion. **v6 supersedes that:** the issue is not Camunda message correlation — it is **gateway table and audit consistency**. Workflow-management endpoints are reused **inside** the gateway, not from the browser.

---

## 4. Internal gateway API

### 4.1 `GET /api/fss/payments/gateway/v1/extraction-uploads/internal/getContent?extractionUploadId={id}`

Streams the uploaded file bytes from `fss_payment_upload_content` (via `fss_payment_upload_meta.file_content_id`) for `ExtractionTriggerHandler`. `uploadId` is a required `@RequestParam`.

---

## 5. Extraction service APIs - `51786-idp-extraction-service` (already built, unchanged)

**Base path:** `/v1/extract` · **Status:** Built · **Caller:** `payment-gateway-service` only.

### 5.1 `POST /v1/extract` (synchronous)

**Request:**

```json
{
  "templateId": "id-payment-v1",
  "correlationId": "extraction-upload-id",
  "fileName": "3897122.pdf",
  "fileContent": "<base64-encoded-bytes>",
  "locale": "id-ID",
  "metadata": { "deptId": "FundServices", "processId": "Cash", "subProcessId": "CI", "entity": "ID" }
}
```

Gateway sends `entity` in metadata. If the built extraction service still reads `country` from metadata, `ExtractionServiceClient` adds `"country": entity` when building the request — adapter only, not a separate upload field.

**Response `200 OK` (`status=COMPLETED`):** LLM `structuredOutput` contains **`initiationDetail` array only**; gateway merges `header` before persist. Full stored shape: [LLD §6.2](./IDP_LLD.md#62-real-structuredoutput-shape-verbatim-confirmed-against-the-built-fixture) and [Design doc §6.3](./IDP_Document_Ingestion_Design.md#63-llm-output-vs-stored-structured_output).

**Response `422 Unprocessable Entity` (`status=FAILED`):**

```json
{
  "extractionId": "ext-audit-uuid", "templateId": "id-payment-v1", "correlationId": "...",
  "status": "FAILED", "structuredOutput": null, "ocrTraceId": null,
  "processingTimeMs": 12000, "errorCode": "OCR_TIMEOUT", "errorDescription": "OCR read exceeded 420000ms"
}
```

**Gateway handler must treat 200 and 422 as two distinct, both-successful-HTTP-call outcomes** - see [Design doc §4.4](./IDP_Document_Ingestion_Design.md#44-how-gateway-knows-extraction-is-done). Only connection errors/timeouts are retryable.

### 5.2 `GET /v1/extract/{extractionId}`

Retrieve extraction audit by technical id - used for gateway crash-recovery (idempotency) and ops troubleshooting. Same response shape as §5.1.

---

## 6. UI action -> API map

`GW` = `/api/fss/payments/gateway/v1/extraction-uploads` · `WFM` = `/api/fss/payments/workflow/v1`

| User action | API | Upload status after |
| --- | --- | --- |
| Open upload tab | `GET {GW}/getUploads?sort=recent&entity=ID` | - |
| Click Upload | `POST {GW}` | `PROCESSING` |
| Table auto-refresh | `GET {GW}/getUploads?sort=recent` (poll 5s) | - |
| Click enabled row | `GET {GW}/getDetail?extractionUploadId={id}` | - |
| Save draft | `POST {GW}/fields?uploadId={id}&detailId={detailId}` | unchanged |
| Submit to checker | `POST {GW}/performAction?uploadId={id}` `{ "action": "SUBMIT" }` or `POST {GW}/submit?uploadId={id}` | `SUBMITTED` |
| Approve | `POST {GW}/performAction?uploadId={id}` `{ "action": "APPROVE" }` or `POST {GW}/approve?uploadId={id}` | `HANDOFF_IN_PROGRESS` → `APPROVED` |
| Reject | `POST {GW}/performAction?uploadId={id}` `{ "action": "REJECT", "remarks": "..." }` or `POST {GW}/reject?uploadId={id}` | `REJECTED` |
| Re-extract | `POST {GW}/re-extract?uploadId={id}` | `PROCESSING` |
| Cancel upload | `POST {GW}/performAction?uploadId={id}` `{ "action": "CANCEL" }` or `POST {GW}/cancel?uploadId={id}` | `CANCELLED` |
| View payment | Navigate to existing payment detail (uses `paymentId` / `paymentRef`) | - |

---

## 7. Timeouts and limits

Unchanged real values from the built extraction service, plus new gateway-side config - see [Design doc §4.5](./IDP_Document_Ingestion_Design.md#45-timeout-configuration-10-min-ocrllm-budget).

---

## 8. Error handling summary

| API | Failure | Client behaviour |
| --- | --- | --- |
| `POST {GW}` (create upload) | 5xx | Show error; user retries upload |
| Extraction (internal) | `422` | Upload -> `FAILED`; row clickable for re-extract |
| Extraction (internal) | timeout | Retried by handler; row stays `PROCESSING`, then `FAILED` after retries exhausted |
| `POST {GW}/fields` | `409` wrong status | "Cannot edit in current status" |
| `POST {GW}/re-extract` | `409` already processing | Disable button |
| `workflow-management` task calls (via gateway) | `502` workflow down after DB commit | Reconciliation restores status; user refreshes |
| `POST {GW}/submit` etc. | `409` wrong status | "Action not allowed in current status" |

---

## 9. Implementation status

| API | Service | Status |
| --- | --- | --- |
| `POST /v1/extract`, `GET /v1/extract/{id}` | `51786-idp-extraction-service` | **Built** |
| `POST/GET {GW}`, `{GW}/getUploads`, `{GW}/getDetail`, `{GW}/fields`, `{GW}/re-extract` (`GW = /api/fss/payments/gateway/v1/extraction-uploads`) | `payment-gateway-service` | **Planned** |
| `GET {GW}/internal/getContent` | `payment-gateway-service` | **Planned** |
| `POST {GW}/performAction`, `/submit`, `/cancel`, `/approve`, `/reject` | `payment-gateway-service` | **Planned** — gateway façade; updates DB + audit then calls WFM server-side |
| `PUT .../set/setTaskDetails`, `PUT .../complete/completeCurrentTask` | `workflow-management` | **Existing** — called **by gateway only**, not by portal UI for this feature |

# FSS Payments Document Ingestion - API Reference
> **Status:** Final v12 — File ID `YYENTYXXXXX` (variable-length sequence); `getUploads`/`getDetail` instruction aggregates; single user review; `performAction` only (optional `/submit`/`/cancel` aliases); entity-only routing; **gateway owns `header`**, extraction returns `initiationDetail` array; **re-extract removed**; per-endpoint authorization + entity-scoped `404`; three confidence scopes; file status as a fold over instruction statuses; reconciled size-limit chain; full error-code catalog; `getDetail` standardized on `uploadId`
> **Created:** 2026-08-02 · **Corrected:** 2026-08-03 (v2) · 2026-08-03 (v3) · 2026-08-03 (v4) · 2026-08-04 (v6/v7) · 2026-08-07 (v9) · **Corrected:** 2026-08-10 (v12)
> **Related:** [README.md](./README.md) · [IDP_LLD.md §9](./IDP_LLD.md#9-java-implementation-guide) · [IDP_UX_Design.md §8](./IDP_UX_Design.md) · [FSSPaymentsDocIngestion.bpmn](./FSSPaymentsDocIngestion.bpmn) · [../51786-idp-extraction-service/extraction-main](../51786-idp-extraction-service/extraction-main) · [../51786-workflow-management/src/main/java/com/sc/fss/iap/workflow/controller/WorkflowServicesController.java](../51786-workflow-management/src/main/java/com/sc/fss/iap/workflow/controller/WorkflowServicesController.java)

This document lists **all APIs** in the document ingestion & extraction upload flow: what each endpoint does, who calls it, and how it fits the end-to-end journey. Extraction service endpoints are **built**, with one pending change to `initiationDetail` cardinality ([§5](#5-extraction-service-apis---51786-idp-extraction-service-built-one-change-required)); gateway upload/action endpoints are **planned**; workflow-management task APIs are **existing** but invoked **server-side by the gateway** for user review actions (portal must not call them directly — see [§3](#3-user-review-action-apis--gateway-facade-required)).

---
## 1. API landscape
| Service | Base path (incl. servlet context path) | Caller | Status |
|---|---|---|---|
| `51786-payment-gateway-service` | `/api/fss/payments/gateway/v1/extraction-uploads` | Portal UI (`paymentMaker`, read-only for `paymentChecker`) — upload, list, detail, fields, **`performAction`** | **Planned** (new) |
| `51786-payment-gateway-service` | `/api/fss/payments/gateway/v1/extraction-uploads/internal` | `51786-idp-extraction-service`, ops | **Planned** (new, internal network only) |
| `51786-workflow-management` | `/api/fss/payments/workflow/v1/set`, `/v1/complete`, `/v1/assign` | **payment-gateway-service only** (server-side `WorkflowService` client) — **not** portal UI for this feature | **Existing, reused server-side** |
| `51786-idp-extraction-service` | `/v1/extract` | `payment-gateway-service` | **Built** |

> **v7 correction — entity-only routing.** `country` and `entity` are the same concept; use **`entity` only** on upload APIs, DB (`fss_payment_upload_meta.entity`), Camunda process variable, and registry lookup. Phase 1: `entity=ID` (Indonesia). If the built extraction service metadata map still expects a `country` key, `ExtractionServiceClient` maps `entity` → `country` at the HTTP boundary only — see [LLD §4](./IDP_LLD.md#4-entity-routing--template-config).

> **v12 — three corrections, each reversing an earlier position.** **(1) The gateway builds `header`;** the extraction service returns `initiationDetail` as an **array** and nothing else ([§5](#5-extraction-service-apis---51786-idp-extraction-service-built-one-change-required)). **(2) `POST .../re-extract` is removed** — retry is a re-upload, and `FAILED` is terminal ([§2.6](#26-removed-post-re-extract)). **(3) `confidence` is a real payload value at three scopes** (file, instruction, field), not an average the gateway computes ([§2.3](#response-fields-content--maps-to-uploadsummarydto)). Also new: a per-endpoint authorization matrix ([§2](#2-portal-upload-apis---payment-gateway-service)), the multipart password parts ([§2.2](#22-post-apifsspaymentsgatewayv1extraction-uploads)), and an exhaustive error-code catalog ([§8.2](#82-error-code-catalog)).

> **v9 — list/detail aggregates.** `getUploads` and `getDetail` expose `instructionCount` and file-level **`confidence`** from denormalized meta columns. **`paymentRef` is not on `getUploads`** — payment links on `getDetail.ingestDetails[]` only (modal). See [UX §2.4.2](./IDP_UX_Design.md#242-table-column-rule--instructions-only-no-payment-ref).

> **v8 — single user review.** Ingestion human actions are **`SUBMIT`** and **`CANCEL`** only, via **`POST .../performAction`** on **payment-gateway-service** (`@PreAuthorize` for `paymentMaker`). Optional thin aliases `/submit` and `/cancel` may delegate to the same service method. **No** ingestion `/approve` or `/reject` — checker step removed from `FSSPaymentsDocIngestion.bpmn`. Gateway updates `fss_payment_upload_meta`, `fss_payment_data_ingest_details`, and `fss_payment_upload_audit`, then calls `setTaskDetails` + `completeCurrentTask` **server-side**.

> **v6 correction — gateway façade (still required).** Direct UI → workflow-management leaves gateway tables stale even when Camunda advances.

> **v4 correction - BPMN redesign removes the need for a new message-correlation endpoint.** In `FSSPaymentsDocIngestion.bpmn` (process id `FSS_Payments_Document_Ingestion`), extraction result and user Submit/Cancel are plain gateway decisions (`ExtractionResultGateway`, `UserReviewGateway`) evaluated the instant the preceding task completes. **The `/v1/correlate/correlateMessage` endpoint proposed in v3 §3.4 is no longer needed** - Cancel uses the same two-call WFM pattern from the gateway after DB + audit updates. See [README §Naming change (v3)](./README.md#naming-change-v3v4).

> **v3 correction (still valid) - base paths and naming verified against real source.** `payment-gateway-service` has **no** global `server.servlet.context-path`; every controller hardcodes its own full path in `@RequestMapping` (confirmed in `PaymentController`, `PaymentsEnrichmentController`, `BulkUploadController`, `SummaryController`, `TransactionDetailController`), always following `/api/fss/payments/{module}/v{n}/...`. None of those controllers use REST path variables for ids - lookups are always `@RequestParam` query parameters, and sub-actions are camelCase verb paths (`getByDTO`, `getUploadsByDTO`, `retry`, `getMessage/inbound`); `@PatchMapping` is never used in this service. `workflow-management`, by contrast, **does** set `server.servlet.context-path=/api/fss/payments/workflow` in `application.properties`, on top of which `WorkflowServicesController` adds `@RequestMapping("v1")` - every call in §3 needs that `/api/fss/payments/workflow` prefix. `51786-idp-extraction-service` sets no context path; `ExtractionController` maps directly to `/v1/extract`.

```mermaid
sequenceDiagram
    participant UI as Portal
    participant GW as payment-gateway-service
    participant WFM as workflow-management
    participant EXT as 51786-idp-extraction-service
    UI->>GW: POST /api/fss/payments/gateway/v1/extraction-uploads
    GW->>WFM: POST /api/fss/payments/workflow/v1/start/startWorkflowProcess?processKey=FSS_Payments_Document_Ingestion
    WFM->>GW: External task Trigger_Data_Extraction
    GW->>EXT: POST /v1/extract (sync)
    EXT-->>GW: 200 structuredOutput OR 422 status=FAILED
    GW->>WFM: complete Trigger_Data_Extraction with extractionStatus (ExtractionResultGateway evaluates immediately)
    UI->>GW: GET /api/fss/payments/gateway/v1/extraction-uploads/getDetail?uploadId={id}
    UI->>GW: POST /api/fss/payments/gateway/v1/extraction-uploads/fields?uploadId={id} body { instructions: [ { detailId, fields[] } ] }
    UI->>GW: POST .../performAction?uploadId={id} body { action: SUBMIT | CANCEL }
    GW->>GW: Validate role + status; INSERT audit; UPDATE meta/details status
    GW->>WFM: setTaskDetails + completeCurrentTask (server-side WorkflowService client)
    WFM->>GW: External task Trigger_Payment_From_Extraction (on SUBMIT only)
    GW->>WFM: startMessageCorrelation (registry-resolved message, cross-process handoff to IAP_ID_Payments only)
```

---

## 2. Portal upload APIs - `payment-gateway-service`

**Base path:** `/api/fss/payments/gateway/v1/extraction-uploads` (hardcoded in the new controller's `@RequestMapping`, matching every sibling controller in this service - there is no `server.servlet.context-path` here).

**Auth:** existing portal JWT, with a rule **per endpoint** — an endpoint with no rule is a defect, not an open endpoint ([LLD §9.9.1](./IDP_LLD.md#991-authorization--per-endpoint-and-data-scoped)):

| Endpoint | Authority |
| --- | --- |
| `POST` *(bare)* — upload | `paymentMaker` |
| `GET /getUploads`, `GET /getDetail` | `paymentMaker` **or** `paymentChecker` |
| `POST /fields` — Save draft | `paymentMaker` **only** — a checker may read but never edit extracted values |
| `POST /performAction`, `/submit`, `/cancel` | `paymentMaker` |

**Authority is not sufficient on its own: every lookup is entity-scoped.** A `uploadId` is an unguessable UUID, but unguessable is not an access control. A maker with Level-1 access to one `ENTY` must not read or act on another entity's upload even holding its id, so repository lookups take both identifiers and a miss returns **`404`, not `403`** — distinguishing them would confirm that an upload they cannot see exists. `getUploads` filters by the caller's entities in SQL, never in Java after fetching.

The `actedBy` / actor value is always taken from the security context. No endpoint accepts an actor in the request body.

### 2.1 Summary table

| Method | Path (relative to `/api/fss/payments/gateway/v1/extraction-uploads`) | Purpose | UI use |
| --- | --- | --- | --- |
| `POST` | *(bare)* | Upload PDF or ZIP (+ metadata); persist row(s); start `FSS_Payments_Document_Ingestion` per PDF | Upload button |
| `GET` | `/getUploads` | List uploads (recent first, filters) | Upload tab table |
| `GET` | `/getDetail?uploadId={id}` | Upload detail + latest extraction run + fields (no blob) | Detail modal |
| `POST` | `/fields?uploadId={id}` | User saves draft field edits — **all dirty instructions in one call**, `detailId` per item in the body | Save draft |
| `POST` | `/performAction?uploadId={id}` | User review action — body `{ "action": "SUBMIT"\|"CANCEL", "remarks": "..." }` — **required** (see [LLD §9.4](./IDP_LLD.md#94-extractionuploadactionservice--single-entry-performaction--switch)) |
| `POST` | `/submit?uploadId={id}` | **Optional alias** — same as `performAction` with `SUBMIT` | Submit for routing |
| `POST` | `/cancel?uploadId={id}` | **Optional alias** — same as `performAction` with `CANCEL` | Cancel upload |

**User review actions** must go through the gateway — **`POST .../performAction`** (aliases optional) — **not** `workflow-management` directly. See [§3](#3-user-review-action-apis--gateway-facade-required).

This mirrors the exact style already used in this service: bare `POST` for the primary create action (as in `BulkUploadController`'s bare `@PostMapping`), camelCase `get*` sub-paths for lookups (as in `getUploadsByDTO`/`getByDTO`), ids passed as `@RequestParam` query parameters rather than `{id}` path variables (as in `retryTransaction(@RequestParam String transactionId, ...)`), and `POST` - never `@PatchMapping`, which is not used anywhere in this service - for the mutating "edit" call.

### 2.2 `POST /api/fss/payments/gateway/v1/extraction-uploads`

**Content-Type:** `multipart/form-data`

| Part | Type | Required | Description |
| --- | --- | --- | --- |
| `file` | file | Yes | Single **PDF** (`.pdf`) **or ZIP** (`.zip`) containing one or more PDFs. ZIP: only `.pdf` entries are processed; all other members are ignored. ZIP with zero PDFs → `400`. |
| `entity` | string | Yes | **Routing key** — selects payment BPMN + LLM template; phase 1: `ID` |
| `passwordProtected` | boolean | No (default `false`) | Declares that the PDF, or the ZIP archive itself, is encrypted |
| `password` | string | Only when `passwordProtected=true` | Decryption password. **Never persisted, logged, audited, or placed in a Camunda variable** — it is used in-memory to decrypt and then zeroed ([LLD §9.8.3](./IDP_LLD.md#983-password-protected-pdf-handling)) |

**For a ZIP, the password opens the archive, not the PDFs inside it.** The supported case is an encrypted archive containing plain PDFs; a per-member-encrypted PDF inside a ZIP is out of scope for v1 and fails as `CORRUPT_FILE` on that entry.

**Because the password is never stored, a document that needs one must be re-uploaded with the password re-entered** — there is no server-side copy to reuse. This is the direct consequence of removing re-extract (§2.6), and the UI copy has to say "upload the file again" rather than "retry".

**Removed in v11:** `deptId`, `processId`, `subProcessId`, `activityId`, `subActivityId`. These are country-specific, are not collected by the UI, and are no longer persisted on `fss_payment_upload_meta`. Where a country needs them downstream, the gateway resolves them from entity configuration ([LLD §4](./IDP_LLD.md#4-entity-routing--template-config)).

**Server actions (single PDF):**

1. Capture the upload timestamp; generate File ID `YYENTYXXXXX` — `YY` from the timestamp year, `ENTY` from the caller's Level-1 access, `XXXXX` from the sequence for that `(year, ENTY)`.
2. Insert `fss_payment_upload_content` + `fss_payment_upload_meta` (`status=UPLOADED` → `PROCESSING`).
3. `POST {workflow-management}/v1/start/startWorkflowProcess?processKey=FSS_Payments_Document_Ingestion&referenceId={uploadId}` with process variable `entity`.
4. On extraction complete: one `fss_payment_data_ingest_details` row per `initiationDetail` instruction.

**Server actions (ZIP bulk):**

1. Generate `batchId` (UUIDv7) for the upload request.
2. Unpack ZIP and identify all eligible `.pdf` entries (case-insensitive). Non-PDF entries are collected in `skippedEntries` (not stored).
3. Sort identified PDFs by file upload timestamp ascending. Resolve equal timestamps by normalized ZIP source path ascending, then discovery order.
4. Assign one File ID `YYENTYXXXXX` per PDF in that order — each file gets its **own** sequence value. File name is never used as identity, so equal names in different ZIP folders receive different IDs.
5. Persist each PDF with `batch_id=batchId` and its File ID.
6. If no PDF entries found → `400` with `"No PDF files found in archive"`.
7. One `FSS_Payments_Document_Ingestion` instance **per extracted PDF**.

**Response `201 Created` (single PDF):**

```json
{
  "uploadId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "fileId": "26IDJY00001",
  "uploadStatus": "PROCESSING"
}
```

**Response `201 Created` (ZIP bulk):**

```json
{
  "batchId": "batch-uuid",
  "uploads": [
    { "uploadId": "...", "fileId": "26IDJY00002", "fileName": "doc1.pdf", "sourcePath": "folder-a/doc1.pdf", "uploadStatus": "PROCESSING" },
    { "uploadId": "...", "fileId": "26IDJY00003", "fileName": "doc1.pdf", "sourcePath": "folder-b/doc1.pdf", "uploadStatus": "PROCESSING" }
  ],
  "skippedEntries": [
    { "path": "readme.txt", "reason": "NOT_PDF" },
    { "path": "deep/sub/invoice.pdf", "reason": "EXCEEDS_MAX_DEPTH" },
    { "path": "__MACOSX/._doc1.pdf", "reason": "SYSTEM_ENTRY" }
  ]
}
```

`reason` is one of `NOT_PDF`, `EXCEEDS_MAX_DEPTH`, `SYSTEM_ENTRY` (see [IDP_LLD.md §9.8.2](./IDP_LLD.md#982-zip-nested-folder-traversal)). Directory entries are structural and never appear here.

**Errors:** `400` missing file / invalid metadata / no PDFs in ZIP / password failure, `401`/`403` not in `paymentMaker` group, `413` `UPLOAD_SIZE_EXCEEDED`, `500` DB or workflow-start failure.

ZIP-specific `400` codes: `ZIP_TOO_MANY_FILES`, `ZIP_SIZE_EXCEEDED`, `ZIP_DEPTH_EXCEEDED`, `CORRUPT_ARCHIVE`, `NO_PDF_IN_ARCHIVE`, plus the password codes — full catalog in [§8.2](#82-error-code-catalog). A limit breach or unreadable archive rejects the **entire** upload — no partial batch is persisted.

### 2.3 `GET /api/fss/payments/gateway/v1/extraction-uploads/getUploads`

One row per **PDF** (`fss_payment_upload_meta`). Instruction-level data is **not** repeated here — use aggregates + `getDetail` for drill-down.

| Param | Type | Default | Description |
| --- | --- | --- | --- |
| `sort` | string | `recent` | `recent` = `uploaded_at` DESC |
| `batchId` | string | - | Filter rows from the same ZIP upload |
| `entity` | string | - | Filter by routing key (e.g. `ID`) |
| `status` | string | - | Filter by `uploadStatus` |
| `myUploads` | boolean | `false` | Current user only |
| `page` / `size` | int | `0` / `20` | Paging |

#### Response fields (`content[]` — maps to `UploadSummaryDto`)

| Field | Type | Source / rule | UI ([UX §2.4](./IDP_UX_Design.md#24-upload-tab-content-tab-2), [§4.4](./IDP_UX_Design.md#44-uploads-table-landing)) |
| --- | --- | --- | --- |
| `uploadId` | string | `fss_payment_upload_meta.id` | Row key, deep link. Canonical 36-character UUID string; the stored value is a UUIDv7 (**not** sequential-integer-like — clients must not assume it is ordered, parseable, or enumerable) |
| `batchId` | string \| null | Meta `batch_id` | ZIP batch filter |
| `fileId` | string | Meta `file_id` — `YYENTYXXXXX` ([LLD §5.1.1](./IDP_LLD.md#511-file-id-generation)) | Stable per-file identity. **Variable length** — treat as opaque, do not assume 11 chars and do not sort as a string |
| `fileName` | string | Meta `file_name` | File name column |
| `uploadedTimestamp` | ISO-8601 | Meta `uploaded_at` | Uploaded column |
| `uploadStatus` | string | Meta `status` | Status chip |
| `uploadedBy` | string | Meta `uploaded_by` | By column |
| `entity` | string | Meta `entity` | Routing (phase 1: `ID`) |
| `instructionCount` | int | Meta `instruction_count` — denormalized, refreshed on write (§5.1.2) | Instructions column — e.g. `1`, `18` |
| `confidence` | number \| null | Meta `confidence` — **whole-file** confidence taken from the extraction payload's document-level score when present; `AVG` of per-instruction scores only as a fallback | Confidence column — see [LLD §6.4](./IDP_LLD.md#64-confidence--three-scopes-three-homes) |

**Three confidence scopes, three different numbers.** The extraction payload carries a score for the whole file, one per extracted instruction, and one per field. `confidence` here is the **file** score — it is not an aggregate the gateway invents, and it will not generally equal the average of `ingestDetails[].confidenceScore`. Clients must not recompute one from the other. The exact payload key and unit (0–1 vs 0–100) are settled by the repo scan ([scan prompt](./IDP_LLD_REPO_SCAN_PROMPT.md)); the API renders whatever the shared contract library declares, normalized to 0–100 with two decimals.

**Not on list response:** `paymentRef` — modal only ([UX §2.4.2](./IDP_UX_Design.md#242-table-column-rule--instructions-only-no-payment-ref)). **Not in v1:** `lowConfidenceInstructionCount` on list.

**Gateway implementation:** `getUploads` reads **`fss_payment_upload_meta`** only — including `file_id`, `instruction_count`, and `confidence`; no join to details on poll. Refreshed by `ExtractionUploadAggregateService` after extraction ([LLD §5.1.2](./IDP_LLD.md#512-file-level-aggregates-on-meta-denormalized--not-computed-on-list-read)).

**Response `200 OK` (single-instruction file, ready for review):**

```json
{
  "content": [
    {
      "uploadId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "batchId": null,
      "fileId": "26IDJY00001",
      "fileName": "3897011.pdf",
      "uploadedTimestamp": "2026-07-30T09:15:00Z",
      "uploadStatus": "READY_FOR_REVIEW",
      "uploadedBy": "maker_user",
      "entity": "ID",
      "instructionCount": 1,
      "confidence": 97.23
    }
  ],
  "page": 0,
  "size": 20,
  "totalElements": 42
}
```

**Response `200 OK` (multi-instruction file — Subscription + Redemption, etc.):**

```json
{
  "content": [
    {
      "uploadId": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
      "batchId": null,
      "fileId": "26IDJY00002",
      "fileName": "3897122.pdf",
      "uploadedTimestamp": "2026-07-30T10:22:00Z",
      "uploadStatus": "READY_FOR_REVIEW",
      "uploadedBy": "maker_user",
      "entity": "ID",
      "instructionCount": 18,
      "confidence": 91.2
    }
  ],
  "page": 0,
  "size": 20,
  "totalElements": 42
}
```

**Response `200 OK` (completed — still shows instruction count, not payment ref):**

```json
{
  "content": [
    {
      "uploadId": "c3d4e5f6-a7b8-9012-cdef-123456789012",
      "batchId": null,
      "fileId": "26IDJY00003",
      "fileName": "3896880.pdf",
      "uploadedTimestamp": "2026-07-29T14:00:00Z",
      "uploadStatus": "COMPLETED",
      "uploadedBy": "maker_user",
      "entity": "ID",
      "instructionCount": 1,
      "confidence": 96.5
    }
  ],
  "page": 0,
  "size": 20,
  "totalElements": 42
}
```

**While `uploadStatus` is `UPLOADED` or `PROCESSING` (before detail rows exist):** `instructionCount=0`, `confidence=null`.

**UI behaviour:** poll every 5s while any row has `uploadStatus` in `UPLOADED`/`PROCESSING`; disable row click while `PROCESSING`. Show `confidence` as percentage when non-null.

**Upload status values:** `UPLOADED`, `PROCESSING`, `READY_FOR_REVIEW`, `SUBMITTED`, `TRIGGERING_PAYMENT`, `COMPLETED`, `FAILED`, `CANCELLED` — see [LLD §5.3](./IDP_LLD.md#53-status-lifecycle).

### 2.4 `GET /api/fss/payments/gateway/v1/extraction-uploads/getDetail?uploadId={id}`

`uploadId` (= `fss_payment_upload_meta.id`) is a required `@RequestParam`. Legacy alias `extractionUploadId` accepted if needed. Does **not** return the file blob.

**Upload-level fields** (same aggregates as `getUploads` — for modal header without re-aggregation):

| Field | Type | Rule |
| --- | --- | --- |
| `instructionCount` | int | Meta `instruction_count` |
| `confidence` | number \| null | Meta `confidence` — the **file**-level score from the payload, not an average of `ingestDetails[].confidenceScore` (§2.3) |

**`status` here is the file status, `ingestDetails[].status` is the instruction status — they are two different values and the modal needs both.** The file status is a *fold* over its instructions with `FAILED` ranked ahead of the rest, so a file reading `FAILED` may still contain instructions that extracted cleanly and sit at `READY_FOR_REVIEW`. Rendering only the header chip would hide usable rows; rendering only row chips would hide that the file as a whole cannot be submitted. The fold precedence is [LLD §5.3.2](./IDP_LLD.md#532-the-fold--how-metastatus-is-derived-from-its-children).

**Response `200 OK`:**

```json
{
  "uploadId": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
  "fileId": "26IDJY00002",
  "fileName": "3897122.pdf",
  "entity": "ID",
  "status": "READY_FOR_REVIEW",
  "uploadedBy": "maker_user",
  "uploadedAt": "2026-07-30T10:22:00Z",
  "processedAt": null,
  "instructionCount": 18,
  "confidence": 91.2,
  "ingestDetails": [
    {
      "detailId": "detail-uuid-1",
      "instructionIndex": 0,
      "status": "READY_FOR_REVIEW",
      "confidenceScore": 97.23,
      "trnTyp": "Redemption",
      "txRef": "3897122-001",
      "valueDate": "2025-10-29",
      "clientName": "PT BNI ASSET MANAGEMENT",
      "debitAmount": "5000000000",
      "lowConfidenceFieldCount": 0,
      "messageId": null,
      "paymentRef": null,
      "retry": 0,
      "errorDesc": null,
      "extractedData": { "header": { "..." }, "initiationDetail": { "..." } }
    },
    {
      "detailId": "detail-uuid-2",
      "instructionIndex": 1,
      "status": "READY_FOR_REVIEW",
      "confidenceScore": 88.0,
      "trnTyp": "Subscription",
      "txRef": "3897122-002",
      "valueDate": "2025-10-30",
      "clientName": "PT BNI ASSET MANAGEMENT",
      "debitAmount": "1000000000",
      "lowConfidenceFieldCount": 2,
      "messageId": null,
      "paymentRef": null,
      "retry": 0,
      "errorDesc": null,
      "extractedData": { "header": { "..." }, "initiationDetail": { "..." } }
    }
  ],
  "camundaTasks": [
    { "taskId": "camunda-task-id", "taskDefinitionKey": "Extraction_UserReview", "assignee": "user_id" }
  ]
}
```

#### `ingestDetails[]` (maps to `fss_payment_data_ingest_details`)

| Field | Stored? | UI left panel ([UX §4.2](./IDP_UX_Design.md#42-instruction-list-left-panel)) |
| --- | --- | --- |
| `instructionIndex` | Column | Row `#` (`+ 1` in UI) |
| `trnTyp` | **Column** | Transaction type — the grouping/filter key |
| `txRef` | Derived | TxRef column |
| `valueDate`, `clientName`, `debitAmount` | Derived | Optional columns |
| `confidenceScore` | Column | Conf |
| `lowConfidenceFieldCount` | Derived | ⚠ badge — count of fields with `Confidence < 90` |
| `messageId` | Column | `IAP_ID_Payments` business key set at handoff |
| `paymentRef` | Derived | Post-handoff payment link — resolved from the message row at read time; `null` until the payment is saved |
| `extractedData` | Column (`TEXT`) | Right panel field grids — **one instruction only** |

**Stored vs derived:** only `trnTyp` is promoted to a database column, because instructions are grouped by transaction type. Fields marked *Derived* are parsed out of `extracted_data` by the gateway when it builds this response — they are not persisted separately ([LLD §5.1](./IDP_LLD.md#51-three-table-model)). The response shape is unchanged for the frontend.

**Removed in v11:** `subActivityId` and `activityId` (grouping and filtering now key on `trnTyp` alone), `handoffAt` (handoff timing lives in `fss_payment_upload_audit`, and a non-null `messageId` marks the handoff as done), and `paymentId` — the payment reference is resolved on read via `messageId`, so no write-back into the frozen `ID_payments.bpmn` handler is needed ([LLD §5.1](./IDP_LLD.md#51-three-table-model)).

**Grouping:** API returns a **flat** `ingestDetails[]`. The modal left panel groups or filters by `trnTyp` **client-side** — no `GET .../getDetailGrouped` endpoint in v1.

`extractedData` is the per-instruction `{ header, initiationDetail }` JSON — `initiationDetail` is a **single object** here even though the extraction response carries an array, because the gateway fans the array out to one row per instruction before persisting. `header` is **built by the gateway**, not returned by the extraction service ([§5.1](#51-post-v1extract-synchronous)): it is copied identically onto every sibling instruction of the same file, so the UI can render it from any row without a separate document-level fetch.

`ingestDetails[].status` is one of `READY_FOR_REVIEW`, `SUBMITTED`, `TRIGGERING_PAYMENT`, `COMPLETED`, `FAILED`, `CANCELLED`. There is no `UPLOADED` or `PROCESSING` on an instruction row — rows are not created until extraction returns, so those two states only ever exist on the file.

**Errors:** `404` unknown id **or an upload outside the caller's entity** — the two are deliberately indistinguishable (§2).

### 2.5 `POST /api/fss/payments/gateway/v1/extraction-uploads/fields?uploadId={id}`

Saves the maker's draft field edits for **one or more instructions in a single call** — every row touched in the modal, sent when the maker clicks **Save draft**. `detailId` is a **body key, not a query parameter**. Re-denormalizes summary columns on save.

**Why batched.** The modal lets the maker edit instruction 1, arrow down to instruction 7, edit that too, then press one *file-level* footer button. A per-instruction endpoint persists only the last selected row and silently discards the rest, while still returning success — the worst failure shape on this screen, because the discarded values go on to become real payments at handoff. One call carrying all dirty rows makes the request match the button.

**A single-instruction save is `instructions[]` of length 1.** There is no second endpoint and no array-vs-object branch, the same rule [LLD §6.2.3](./IDP_LLD.md#623-document-scope-vs-instruction-scope--the-fan-out) applies to `initiationDetail`.

**Request body:**

```json
{
  "instructions": [
    {
      "detailId": "0190f2a1-7c3d-7e11-9b2a-000000000001",
      "fields": [
        { "fieldGroup": "TransactionDetails", "occurrenceIndex": null, "fieldName": "ClntNm", "fieldValue": "PT BNI ASSET MANAGEMENT" }
      ]
    },
    {
      "detailId": "0190f2a1-7c3d-7e11-9b2a-000000000002",
      "fields": [
        { "fieldGroup": "DebitDetails",  "occurrenceIndex": 0, "fieldName": "DrAccNo", "fieldValue": "30681655612" },
        { "fieldGroup": "CreditDetails", "occurrenceIndex": 0, "fieldName": "CrAmt",   "fieldValue": "5000000000" }
      ]
    }
  ]
}
```

| Element | Rule |
| --- | --- |
| `instructions[]` | 1..*N*, non-empty. Every `detailId` must belong to `uploadId` |
| `detailId` | `fss_payment_data_ingest_details.id` |
| `fields[]` | 1..*N* **absolute** field assignments — not a diff |
| `fieldGroup` | The **verbatim JSON container key** inside `initiationDetail`. **Entity-scoped, not a fixed enum** — for `ID`: `TransactionDetails`, `DebitDetails`, `CreditDetails` |
| `occurrenceIndex` | 0-based position within a **repeating** group; `null` when the group does not repeat. For `ID`: `null` on `TransactionDetails`, the `details[]` index on `DebitDetails` / `CreditDetails` |
| `fieldName` | Must appear in the [§6.2.1](./IDP_LLD.md#621-transactiondetails-known-name-catalog) / [§6.2.2](./IDP_LLD.md#622-debitdetailsdetailsdata--creditdetailsdetailsdata-known-name-catalog) catalog |
| `fieldValue` | New value only. The client never sends the old value or a `Confidence` — both are server-owned |

**Atomic.** All instructions commit or none. A partial save leaves the maker unable to tell what persisted.

**Response `200 OK`** — returns the refreshed summary for each saved instruction, so the modal's left panel repaints without re-fetching `getDetail` for all *N* rows. Each entry is the same `IngestDetailSummaryDto` shape as [§2.4](#24-get-apifsspaymentsgatewayv1extraction-uploadsgetdetailuploadidid)'s `ingestDetails[]`, without `extractedData`, plus `changedFieldCount`:

```json
{
  "uploadId": "a1b2c3d4-...",
  "savedAt": "2026-07-30T11:00:00Z",
  "instructions": [
    { "detailId": "...0001", "status": "READY_FOR_REVIEW", "changedFieldCount": 1, "trnTyp": "Redemption",   "txRef": "3897122-001", "…": "…" },
    { "detailId": "...0002", "status": "READY_FOR_REVIEW", "changedFieldCount": 2, "trnTyp": "Subscription", "txRef": "3897122-002", "…": "…" }
  ]
}
```

`changedFieldCount` counts only fields whose stored value **actually changed** — the gateway compares each assignment against `extracted_data` and ignores no-ops, so re-sending an untouched row yields `0`.

**Validation and errors:**

| Code | Condition |
| --- | --- |
| `400` | Empty `instructions[]`; `fieldGroup` not in the entity's catalog; `occurrenceIndex` out of range; `occurrenceIndex` non-null on a non-repeating group; `fieldName` outside the catalog for that group; any attempt to address a `header` field |
| `403` | Caller not `paymentMaker`, **or any `detailId` does not belong to `uploadId`** |
| `404` | Unknown `uploadId`, **or a `uploadId` outside the caller's entity scope** (§2); a `detailId` that exists nowhere |
| `409` `INVALID_STATUS_TRANSITION` | Any targeted instruction is not `READY_FOR_REVIEW` — the response names the offending `detailId` |
| `409` `CONCURRENT_EDIT` | Another actor changed one of the rows between this client's read and its save. **Re-fetch `getDetail` and re-apply — do not replay the request**, which would silently discard the other write ([LLD §5.5.1](./IDP_LLD.md#551-concurrency--the-diff-rule-creates-a-lost-update-so-it-needs-a-lock)) |

**`detailId` ownership is a security check, not a lookup convenience.** `detailId` is now client-supplied in a body scoped by a *different* identifier, so the gateway must confirm each one's `upload_id` equals the `uploadId` in the query string, and reject the whole batch otherwise. Without that check an authenticated maker can edit another upload's instruction under cover of their own `uploadId` — the enumeration risk [LLD §5.1.3](./IDP_LLD.md#513-surrogate-keys--native-uuid-generated-as-uuidv7) already flags.

**Header fields are unreachable by construction.** `{ fieldGroup, occurrenceIndex, fieldName }` is resolved **relative to the `initiationDetail` node**, never from the document root, so a header edit cannot be *expressed* — no denylist is needed to satisfy the read-only-header rule in [LLD §6.2.3](./IDP_LLD.md#623-document-scope-vs-instruction-scope--the-fan-out).

**The addressing is entity-scoped, deliberately.** `fieldGroup` carries the real JSON key rather than an invented token like `DEBIT`, and its legal values come from the entity's field catalog — for `ID`, [§6.2.1](./IDP_LLD.md#621-transactiondetails-known-name-catalog) and [§6.2.2](./IDP_LLD.md#622-debitdetailsdetailsdata--creditdetailsdetailsdata-known-name-catalog). A country whose template has different containers, different nesting, or no debit/credit split ships a new catalog and `ExtractedDataFieldAccessor` implementation; **this request contract, the response, and the audit schema do not change**. That is why the payload avoids `section` / `legIndex`: both encode `id-payment-v1`'s structure, and an audit column is the worst place to discover a country-specific assumption.

**Idempotent.** `fields[]` carries absolute values rather than a diff, so replaying the same request produces the same stored state — and, because no-ops are not audited, writes no phantom history.

#### 2.5.1 What this endpoint audits

Every real change is recorded at **field grain**, because the maker is overriding machine-extracted values that become live payment instructions. One save produces:

| Rows | Table | Content |
| --- | --- | --- |
| Exactly 1 | `fss_payment_upload_audit` | `action=FIELD_EDIT`, actor, and a `details` summary such as `{"instructions":2,"fields":3}` |
| 1 per changed field | `fss_payment_field_audit` | `detail_id`, `extraction_id`, `field_group`, `occurrence_index`, `field_name`, `old_value`, `new_value`, `edited_by`, `created_at` |

Three rules, specified in [LLD §5.5](./IDP_LLD.md#55-field-edit-write-rules-mandatory):

1. **The server computes the diff.** The client sends `fieldValue` only; the gateway derives `old_value` from the stored `extracted_data`. A client-supplied before-value would be an unverified claim, not an audit record — which is why the request body has no field for one.
2. **No-ops write nothing.** Re-sending an unchanged value produces no audit row and no `changedFieldCount`. If nothing changed across the whole batch, not even the `FIELD_EDIT` action row is written.
3. **`Confidence` is never overwritten.** The edit replaces `Value` only, so the original LLM score survives in `extracted_data` — which is why the audit row does not copy it.

All audit writes share the transaction that updates `extracted_data`, so a rejected batch leaves no trail fragment.

### 2.6 Removed: `POST .../re-extract`

**There is no re-extract endpoint.** Earlier versions specified `POST .../re-extract?uploadId={id}` returning `202 Accepted` and resetting detail rows back to `PROCESSING`. It is **removed from v1**: a failed or unsatisfactory extraction is handled by **uploading the document again**, which produces a new `uploadId`, a new File ID, and a new workflow instance.

**Why removal is the cheaper answer.** Re-extract looks like one endpoint but is actually a second write path into rows that already carry maker edits, audit history, and possibly a Camunda user task. Getting it right means deciding what happens to field edits made before the retry, whether the existing process instance is reused or a stale `Extraction_UserReview` task is cancelled first, and how `fss_payment_field_audit` rows stay meaningful when the `extraction_id` they reference is superseded. Re-upload sidesteps all of it: the failed row stays as an immutable record of what happened, and the retry is an ordinary upload with no special state machine.

**Consequences for clients:**

| Concern | Rule |
| --- | --- |
| `FAILED` | **Terminal.** No transition leaves it. The row is not clickable for retry and exposes no action |
| `retry` on `ingestDetails[]` | Reflects **payment-handoff** attempts only, never extraction attempts. It is not a re-extract counter |
| Duplicate detection | Re-uploading the same document is expected and permitted. The two uploads are independent rows with different File IDs; v1 does **not** deduplicate by content hash |
| Audit | Nothing links the replacement upload to the failed one. Operators correlate on `fileName` + `uploadedBy` + timestamp |

---

## 3. User review action APIs — gateway facade (required)

> **v6 correction:** Earlier versions had the portal call `workflow-management` directly for Submit / Cancel. That **does not** update `fss_payment_upload_meta`, `fss_payment_data_ingest_details`, or `fss_payment_upload_audit`. **UI actions must go through the gateway** — the gateway owns DB + audit, then calls the existing `WorkflowServicesController` contract **server-side** via the same `WorkflowService` client already used for `startWorkflowProcess`.

**Base path:** `/api/fss/payments/gateway/v1/extraction-uploads`  
**Auth:** JWT; `@PreAuthorize` on `performAction` for **`paymentMaker`** only (`SUBMIT` and `CANCEL`).

### 3.1 Primary endpoint

| Method | Path | Caller | Gateway DB status after success |
| --- | --- | --- | --- |
| `POST` | `/performAction?uploadId={id}` | User (`paymentMaker`) | Per `action` in body — see [LLD §9.4](./IDP_LLD.md#94-extractionuploadactionservice--single-entry-performaction--switch)) |

**Optional aliases** (thin wrappers — same `ExtractionUploadActionService.performAction`, no duplicate logic):

| Method | Path | Equivalent |
| --- | --- | --- |
| `POST` | `/submit?uploadId={id}` | `{ "action": "SUBMIT" }` |
| `POST` | `/cancel?uploadId={id}` | `{ "action": "CANCEL", "remarks": "..." }` |

Implement aliases only if the portal team wants dedicated URLs; **`performAction` alone is sufficient.**

**Removed from ingestion API (v8):** `/approve`, `/reject`, and `APPROVE` / `REJECT` in `UploadAction` — extraction checker step removed from BPMN. Payment maker/checker in `IAP_ID_Payments` is unchanged and uses separate APIs after handoff.

**Request body (`performAction` — all actions):**

```json
{ "action": "SUBMIT", "remarks": "optional" }
```

`action` values: `SUBMIT`, `CANCEL`.

**Request body (cancel on alias endpoint):**

```json
{ "remarks": "optional" }
```

**Response `200 OK` (all actions):**

```json
{
  "uploadId": "a1b2c3d4-...",
  "status": "SUBMITTED",
  "action": "USER_SUBMIT",
  "actedAt": "2026-07-30T11:00:00Z",
  "actedBy": "user_id"
}
```

**Errors:** `400` invalid status transition, `403` wrong role, `404` unknown upload, `409` no active Camunda task / already actioned, `502` workflow-management call failed (DB rolled back or left unchanged — see §3.3).

### 3.2 `ExtractionUploadActionService` orchestration (gateway Java)

Each action runs in this order:

```
1. Load fss_payment_upload_meta (+ details); verify caller role and allowed status transition
2. Verify active Camunda task matches expected taskDefinitionKey (`Extraction_UserReview`)
3. BEGIN transaction
4. INSERT fss_payment_upload_audit (action, actor, before_status, after_status, remarks, details JSON snapshot optional)
5. UPDATE fss_payment_upload_meta.status (+ remarks on cancel)
6. UPDATE fss_payment_data_ingest_details.status in bulk where applicable
7. COMMIT
8. workflowClient.setTaskDetails(uploadId, vars)  // same vars as §3.4 table
9. workflowClient.completeCurrentTask(uploadId)
10. On workflow HTTP failure: compensating UPDATE to restore prior status + audit row FAILURE note; return 502
11. On submit success: return with meta `SUBMITTED`; handoff handler advances to `TRIGGERING_PAYMENT` then `COMPLETED` (async)
```

**Idempotency:** reject duplicate `POST` with same action when status already terminal → `409`.

### 3.3 Why not UI → workflow-management directly?

| Concern | Direct WFM call | Gateway façade |
| --- | --- | --- |
| `fss_payment_upload_meta.status` | Stale until external task runs (often never for user tasks) | Updated in step 5 |
| `fss_payment_data_ingest_details.status` | Not updated | Synced per action |
| `fss_payment_upload_audit` | Not written | Written in step 4 |
| UI table/modal | Wrong status on refresh | Correct immediately |
| User `remarks` on cancel | Only in Camunda variables | Persisted on meta + audit |

External task handlers (`ExtractionTriggerHandler`, `ExtractionPaymentHandoffHandler`) **still** update DB for system-driven steps (extraction complete, payment handoff). User-driven steps **must** use §3.1.

### 3.4 Camunda variables (internal — gateway sets these server-side)

The portal **does not** call these endpoints. The gateway uses the existing `WorkflowService` / REST client against `51786-workflow-management`:

| UI action (gateway path) | Active task | Variables (`setTaskDetails`) | `completeCurrentTask` |
| --- | --- | --- | --- |
| `POST .../submit` | `Extraction_UserReview` | `{ "reviewAction": "SUBMIT" }` | yes |
| `POST .../cancel` | `Extraction_UserReview` | `{ "reviewAction": "CANCEL", "remarks": "..." }` | yes |

Underlying WFM paths (unchanged):

| Method | Path |
| --- | --- |
| `PUT` | `/api/fss/payments/workflow/v1/set/setTaskDetails/{uploadId}` |
| `PUT` | `/api/fss/payments/workflow/v1/complete/completeCurrentTask/{uploadId}` |

`uploadId` = Camunda business key = `fss_payment_upload_meta.id`.

### 3.5 Status transitions (gateway-enforced)

| Current meta `status` | Action | New meta `status` |
| --- | --- | --- |
| `READY_FOR_REVIEW` | `submit` | `SUBMITTED` → `TRIGGERING_PAYMENT` → `COMPLETED` (handler) |
| `READY_FOR_REVIEW` | `cancel` | `CANCELLED` |

Any other current status rejects the action with `409` — including a second `submit` on an already-`SUBMITTED` upload, which is the common double-click case rather than an exotic one.

System-driven only (not UI buttons): `PROCESSING`, `FAILED`, `TRIGGERING_PAYMENT`, `COMPLETED` — set by `ExtractionTriggerHandler` / `ExtractionPaymentHandoffHandler` / reconciliation.

**`COMPLETED`, `CANCELLED`, and `FAILED` are terminal.** No action, system or user, leaves them; with re-extract removed (§2.6) `FAILED` has no outbound edge at all.

**The gateway writes the file status by folding its instructions, never by direct assignment.** `submit` and `cancel` move the *instruction* rows, then `ExtractionUploadAggregateService` recomputes `fss_payment_upload_meta.status` from them — it is the single writer of that column ([LLD §5.3.2](./IDP_LLD.md#532-the-fold--how-metastatus-is-derived-from-its-children)). A caller therefore cannot see a file status that contradicts the rows underneath it, and a partially-handed-off multi-instruction file reads `TRIGGERING_PAYMENT` until the last instruction lands.

### 3.6 Superseded (v4)

v4 stated no gateway proxy was needed because Cancel became a plain gateway completion. **v6 supersedes that:** the issue is not Camunda message correlation — it is **gateway table and audit consistency**. Workflow-management endpoints are reused **inside** the gateway, not from the browser.

---

## 4. Internal gateway API

### 4.1 `GET /api/fss/payments/gateway/v1/extraction-uploads/internal/getContent?uploadId={id}`

Streams the uploaded file bytes from `fss_payment_upload_content` (via `fss_payment_upload_meta.file_content_id`) for `ExtractionTriggerHandler`. `uploadId` is a required `@RequestParam` — the same name as on every public endpoint, including here where an earlier revision used `extractionUploadId`.

**This endpoint returns raw customer payment documents and must not be reachable from the browser.** It is called server-side by the gateway's own external-task handler, so it belongs on the internal network path only, excluded from the portal's ingress. It carries no entity scoping of its own because it has no end user to scope to — which is exactly why exposing it would bypass the §2 access rules entirely.

---

## 5. Extraction service APIs - `51786-idp-extraction-service` (built; **one change required**)

**Base path:** `/v1/extract` · **Status:** Built, transport contract stable · **Caller:** `payment-gateway-service` only.

**The one change:** `structuredOutput.initiationDetail` must be an **array**, one element per payment instruction found in the document. The built service and its `id-payment-v1` fixtures emit a single object, which caps every file at one instruction and makes multi-instruction documents — the normal case for this workload — silently lossy. The transport envelope, `extractionId`, status semantics, and error codes are unaffected; only the LLM template and the `initiationDetail` node's cardinality change ([LLD §14](./IDP_LLD.md#14-implementation-order) gate 0a).

**The header stays out of the extraction response.** `header` is document-scope data the gateway already holds — File ID, entity, upload timestamp, uploader — so asking the LLM to reproduce it invites hallucinated values in fields that are authoritative elsewhere, with no way for the gateway to tell an extracted header from a correct one. The gateway builds `header` itself and merges it onto each instruction before persisting ([LLD §6.1a](./IDP_LLD.md#61a-division-of-the-contract--decided-the-service-returns-instructions-the-gateway-owns-the-header)).

### 5.1 `POST /v1/extract` (synchronous)

**Request:**

```json
{
  "templateId": "id-payment-v1",
  "correlationId": "extraction-upload-id",
  "fileName": "3897122.pdf",
  "fileContent": "<base64-encoded-bytes>",
  "locale": "id-ID",
  "metadata": { "entity": "ID" }
}
```

Country-specific keys (`deptId`, `processId`, …) are no longer sent from the upload form. If an entity's LLM template requires them, `ExtractionServiceClient` adds them from entity configuration when building the request.

Gateway sends `entity` in metadata. If the built extraction service still reads `country` from metadata, `ExtractionServiceClient` adds `"country": entity` when building the request — adapter only, not a separate upload field.

**Response `200 OK` (`status=COMPLETED`):** `structuredOutput` contains **`initiationDetail` array only** — no `header` node — plus confidence scores at file, instruction, and field scope. The gateway merges its own `header` before persist. Full stored shape: [LLD §6.2](./IDP_LLD.md#62-structuredoutput-shape) and [Design doc §6.3](./IDP_Document_Ingestion_Design.md#63-llm-output-vs-stored-structured_output).

The response POJOs come from the **shared contract library** both services already depend on, so the gateway deserializes into the published types rather than redeclaring them — a duplicated DTO here would drift on the next template change and fail silently, since Jackson ignores unknown fields by default. The library coordinates, version, and confidence field names are confirmed by the repo scan ([scan prompt](./IDP_LLD_REPO_SCAN_PROMPT.md)).

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
| Click enabled row | `GET {GW}/getDetail?uploadId={id}` | - |
| Save draft | `POST {GW}/fields?uploadId={id}` — body carries every dirty instruction | unchanged |
| Submit for routing | `POST {GW}/performAction?uploadId={id}` `{ "action": "SUBMIT" }` or `POST {GW}/submit?uploadId={id}` | `SUBMITTED` → `TRIGGERING_PAYMENT` → `COMPLETED` |
| Retry a failed file | *(no API)* — re-upload via `POST {GW}`, which mints a new `uploadId` (§2.6) | new row `PROCESSING`; failed row unchanged |
| Cancel upload | `POST {GW}/performAction?uploadId={id}` `{ "action": "CANCEL" }` or `POST {GW}/cancel?uploadId={id}` | `CANCELLED` |
| View payment | Open row → `getDetail` → **View payment** on `ingestDetails[]` with a non-null `paymentRef` (modal only — not list table) | - |

---

## 7. Timeouts and limits

Unchanged real values from the built extraction service, plus new gateway-side config - see [Design doc §4.5](./IDP_Document_Ingestion_Design.md#45-timeout-configuration-10-min-ocrllm-budget).

**Size limits are a chain, and the smallest link decides the real ceiling.** A PDF that clears the multipart limit can still be rejected further along, so the values must be reconciled rather than set independently ([LLD §7](./IDP_LLD.md#7-error-handling--resilience)):

| Hop | Limit | Note |
| --- | --- | --- |
| Multipart upload | `spring.servlet.multipart.max-file-size` | Rejects with `413`, before any DB write |
| Single PDF (post-ZIP-expansion) | `extraction.upload.max-file-size` | Applies to each PDF **inside** a ZIP too, not just a directly-uploaded one |
| ZIP archive | `extraction.upload.zip.max-total-size`, `max-file-count` | A breach rejects the **whole** upload; no partial batch is persisted |
| Extraction request body | WebClient `maxInMemorySize` | The PDF is **base64-encoded** here, so it occupies ≈ **1.34×** its byte size. A codec ceiling set to the raw file limit will fail on files that passed every earlier check |

That base64 expansion is the one easy to get wrong: the failure appears as a WebClient buffer error deep in the handler, on large-but-legal files, long after the upload was accepted — so the codec ceiling must be derived from the file limit rather than configured to match it.

---

## 8. Error handling summary

| API | Failure | Client behaviour |
| --- | --- | --- |
| `POST {GW}` (create upload) | 5xx | Show error; user retries upload |
| Extraction (internal) | `422` | Upload → `FAILED` (**terminal**); user re-uploads the document (§2.6) |
| Extraction (internal) | timeout | Retried by handler; row stays `PROCESSING`, then `FAILED` after retries exhausted |
| `POST {GW}/fields` | `409` wrong status | "Cannot edit in current status" |
| `workflow-management` task calls (via gateway) | `502` workflow down after DB commit | Reconciliation restores status; user refreshes |
| `POST {GW}/performAction` etc. | `409` wrong status | "Action not allowed in current status" |

### 8.1 Error response body

Every 4xx and 5xx from these endpoints uses one shape, produced by a single `@RestControllerAdvice` ([LLD §9.11](./IDP_LLD.md#911-exception-model--http-mapping)):

```json
{
  "errorCode": "INVALID_PASSWORD",
  "message": "Incorrect password — file could not be opened",
  "uploadId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "timestamp": "2026-07-30T09:15:00Z"
}
```

`errorCode` is the contract; `message` is display copy and may be reworded without a version bump. A client that branches on `message` will break. `uploadId` is present only where one exists — an upload rejected during validation has no id yet.

### 8.2 Error code catalog

Authoritative mapping lives in [LLD §9.11](./IDP_LLD.md#911-exception-model--http-mapping); this table is the client-facing view of it.

| `errorCode` | HTTP | Raised when |
| --- | --- | --- |
| `VALIDATION_FAILED` | `400` | Bean-validation failure on a request body or param — `message` names the offending field |
| `UNSUPPORTED_FILE_TYPE` | `400` | Upload is neither `.pdf` nor `.zip` |
| `UNKNOWN_ENTITY` | `400` | `entity` absent, blank, or with no registered configuration. A client or config fault, so it must **not** surface as `500` |
| `NO_PDF_IN_ARCHIVE` | `400` | ZIP parsed but contains zero eligible `.pdf` entries |
| `ZIP_TOO_MANY_FILES` | `400` | Eligible PDF count above `max-extracted-pdf-count` |
| `ZIP_SIZE_EXCEEDED` | `400` | Uncompressed content exceeds `max-uncompressed-size-mb` (ZIP-bomb guard) |
| `ZIP_DEPTH_EXCEEDED` | `400` | Every PDF sits beyond `max-folder-depth`, so nothing is eligible |
| `CORRUPT_ARCHIVE` | `400` | Archive unreadable, truncated, or carrying an illegal entry path |
| `CORRUPT_FILE` | `400` | PDF bytes unparseable |
| `PASSWORD_REQUIRED` | `400` | `passwordProtected=true` with a blank password |
| `INVALID_PASSWORD` | `400` | A password was supplied and decryption still failed |
| `ENCRYPTED_FILE_PASSWORD_REQUIRED` | `400` | File is encrypted but the caller did not declare it as protected |
| `FORBIDDEN` | `403` | Caller lacks the endpoint's authority, or a `detailId` in the batch belongs to a different `uploadId` |
| `NOT_FOUND` | `404` | Unknown id **or** an id outside the caller's entity scope (§2) |
| `UNKNOWN_FIELD_ADDRESS` | `400` | A `{ fieldGroup, occurrenceIndex, fieldName }` triple does not resolve against the stored payload |
| `INVALID_STATUS_TRANSITION` | `409` | Requested action is not legal from the current status |
| `CONCURRENT_EDIT` | `409` | Optimistic-lock failure — another actor changed the row after it was read |
| `UPLOAD_SIZE_EXCEEDED` | `413` | Any size ceiling in §7 breached |
| `FILE_ID_ALLOCATION_FAILED` | `500` | File ID sequence statement failed. Nothing was persisted; the user can retry |
| `INTERNAL_ERROR` | `500` | Everything else. Carries no stack trace, exception class, or SQL text |

**Three of these are near-neighbours the client must keep apart, because each implies a different next action.** `CORRUPT_FILE` means the bytes are unusable and a password will not help. `INVALID_PASSWORD` means retry with a different password. `PASSWORD_REQUIRED` means the user forgot to supply one at all. Collapsing them into one "upload failed" toast is what sends a user hunting for a password on a truncated file.

**`CONCURRENT_EDIT` must trigger a re-fetch, never a blind retry.** It means someone else changed the row after this client read it; replaying the same body would overwrite their write with values computed from stale data — the lost update the `409` exists to prevent.

**`EXTRACTION_FAILED` is not in the table because no gateway endpoint returns it.** Extraction runs behind an external task, so its `422` is consumed by the handler, never relayed to the portal: the outcome reaches the client as `uploadStatus=FAILED` plus `ingestDetails[].errorDesc` on the next poll or `getDetail`. A client waiting for an HTTP error to learn that extraction failed will wait forever — the signal is the status, and `errorDesc` carries the reason ([UX §5.1](./IDP_UX_Design.md#51-error-copy)).

**Unmapped exceptions are `500` by design, and that is why this catalog must stay exhaustive for expected faults.** Any client-caused condition arriving at the generic handler is a mapping defect: it hides a fixable user error behind a server error, and pages an on-call engineer for a malformed request. New failure modes get a code here before they ship.

---

## 9. Implementation status

| API | Service | Status |
| --- | --- | --- |
| `POST /v1/extract`, `GET /v1/extract/{id}` | `51786-idp-extraction-service` | **Built — one change pending:** `initiationDetail` must become an array (§5). Blocks everything downstream of it |
| `POST/GET {GW}`, `{GW}/getUploads`, `{GW}/getDetail`, `{GW}/fields` (`GW = /api/fss/payments/gateway/v1/extraction-uploads`) | `payment-gateway-service` | **Planned** |
| `POST {GW}/re-extract` | — | **Removed** from v1 (§2.6). Retry is a re-upload |
| `GET {GW}/internal/getContent` | `payment-gateway-service` | **Planned** |
| `POST {GW}/performAction` (`SUBMIT` / `CANCEL`; optional `/submit`, `/cancel` aliases) | `payment-gateway-service` | **Planned** — gateway façade; updates DB + audit then calls WFM server-side |
| `PUT .../set/setTaskDetails`, `PUT .../complete/completeCurrentTask` | `workflow-management` | **Existing** — called **by gateway only**, not by portal UI for this feature |

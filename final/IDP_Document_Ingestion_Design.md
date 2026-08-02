# IDP Document Ingestion — Production-Safe Design (Two BPMN Files)

> **Status:** **Final** — production-safe design (see [README.md](./README.md))  
> **Created:** 2026-07-30 · **Finalized:** 2026-08-02  
> **Related:** [README.md](./README.md) · [IDP_LLD.md](./IDP_LLD.md) (implementation LLD — routing §4) · [IDP_UX_Design.md](./IDP_UX_Design.md) · [../ID_payments.md](../ID_payments.md) · [../ID_payments.bpmn](../ID_payments.bpmn) · [IDP_Document_Ingestion.bpmn](./IDP_Document_Ingestion.bpmn) · [../after_ocr-llm-output.json](../after_ocr-llm-output.json)

---

## 1. Design decision summary

| Topic | Decision |
|-------|----------|
| BPMN strategy | **Two files** linked by Camunda message correlation |
| New file | `IDP_Document_Ingestion.bpmn` — upload, OCR/LLM, IDP maker/checker |
| Prod file | `IAP_ID_Payments.bpmn` — **additive only** (4th message start + one init task + merge) |
| LLM / extraction / IDP human tasks | **Only** in `IDP_Document_Ingestion.bpmn` |
| Existing prod triggers | **Unchanged** — automated, bulk, manual |
| Cross-process link | Country-specific message start on each payment BPMN (e.g. `IAP_ID_IDP_Trigger` for Indonesia) + BPMN text annotations |
| Country / payment routing | **`IDPPaymentRouteRegistry`** (config in gateway) resolves `country` / `entity` → message name — **not** a BPMN gateway in `IDP_Document_Ingestion` — see [§2.1](#21-country--payment-routing-responsibilities) and [IDP_LLD.md §4](./IDP_LLD.md#4-country--entity-routing) |
| Handoff implementation | External task `Trigger_IDP_Payment` → registry lookup → `WorkflowService.startMessageCorrelation(messageName, ...)` |
| Manual payment keying | `IAP_ID_Manual_Payment` stays as-is; IDP upload is a **separate** process |
| Document storage (Phase 1) | **Split tables** — `fss_idp_upload` (metadata) + `fss_idp_upload_content` (BLOB); lazy-fetched via separate repository/entity; **no** `Store_IDP_Document`; **no** document-service |
| Extraction naming | **`fss_idp_extraction_run`** — not “job” (avoids confusion with schedulers); one OCR+LLM pass per row |
| Confidence | Per-field `Confidence` in LLM JSON; `overall_confidence` column on each extraction run row |

### 1.1 Domain glossary

| Term | Meaning |
|------|---------|
| **Upload** | `fss_idp_upload` — user document + status |
| **Extraction run** | `fss_idp_extraction_run` — one OCR+LLM pass; re-extract = new run |
| **Review payload** | `structured_output` — JSON maker/checker edit (includes per-field confidence) |

Camunda passes **`idpUploadId`** only between steps — not OCR/LLM payloads.

---

## 2. Architecture overview

```mermaid
flowchart TB
    subgraph IDP["IDP_Document_Ingestion.bpmn — SINGLE shared process (all countries)"]
        UP(["Upload / Start"]) --> EXTRACT["Trigger_IDP_Extraction"]
        EXTRACT --> WAIT{{"IDPExtractionCompleted"}}
        WAIT --> GW1{"Extraction OK?"}
        GW1 -->|fail| FAIL["End / notify"]
        GW1 -->|success| IDPMK{{"IDP_MakerReview"}}
        IDPMK --> IDPCK{{"IDP_CheckerReview"}}
        IDPCK --> GW2{"idpApproved?"}
        GW2 -->|reject| IDPMK
        GW2 -->|approve| TRIGGER["Trigger_IDP_Payment"]
        TRIGGER --> END_IDP(["End"])
    end

    subgraph ROUTE["Runtime routing — country / entity from fss_idp_upload"]
        REG["IDPPaymentRouteRegistry"]
    end

    subgraph PAY["IAP_ID_Payments.bpmn (PROD — additive only, phase 1 example)"]
        A1(["IAP_ID_AutomatedTrigger"]) --> IP["Initialize_IAP_Payments"]
        B1(["IAP_ID_BulkTrigger"]) --> IB["Initialize_IAP_Bulk_Payments"]
        M1(["IAP_ID_Manual_Payment"]) --> MK["IAP_ID_MakerPayment"]
        IDP_START(["IAP_ID_IDP_Trigger NEW"]) --> INIT["Initialize_IAP_From_IDP NEW"]
        IP --> IID["Initialize_IAP_ID_Payments"]
        IB --> IID
        INIT --> IID
        IID --> ENR["OTT → Local Charges → Enrich → Save → …"]
        ENR --> GW["PaymentValidationGateway → …"]
    end

    subgraph CN_PAY["ChinaETF_TPlusN.bpmn — phase 2 example"]
        CN_START(["CN_ETF_IDP_Trigger"]) --> CN_INIT["Initialize_CN_ETF_From_IDP"]
        CN_INIT --> CN_ENR["Existing CN enrichment spine"]
    end

    UP --> EXTRACT
    TRIGGER --> REG
    REG -->|"country=ID"| IDP_START
    REG -->|"country=CN, product=ETF"| CN_START
```

**Annotation on `IAP_ID_IDP_Trigger`:**  
*"Triggered from IDP_Document_Ingestion via Trigger_IDP_Payment when country=ID (registry resolves message name)"*

**Annotation on `Trigger_IDP_Payment`:**  
*"External-task hook only — not a country-routing BPMN gateway. Handler loads upload row, calls IDPPaymentRouteRegistry, then correlates the resolved message (e.g. IAP_ID_IDP_Trigger)."*

### 2.1 Country / payment routing responsibilities

`IDP_Document_Ingestion.bpmn` **is** a decision engine for **IDP-internal** flow only (extraction success/fail, checker approve/reject). It is **not** the country/payment routing decision engine.

| Layer | Owns country routing? | Responsibility |
|-------|----------------------|----------------|
| `IDP_Document_Ingestion.bpmn` | **No** | Upload → extract → IDP maker/checker. Gateways: `IDPExtractionGateway`, `IDPCheckerGateway` only. `Trigger_IDP_Payment` is a **single external-task step** — a hook, not a branching gateway. |
| Country payment BPMN (e.g. `IAP_ID_Payments.bpmn`) | **Defines entry only** | Declares message start events (`IAP_ID_IDP_Trigger`, future `CN_ETF_IDP_Trigger`, …) and `Initialize_*_From_IDP` merge into existing enrichment. Does **not** receive routing logic from the common IDP BPMN. |
| `IDPPaymentRouteRegistry` (gateway config) | **Yes — selection** | Maps `country` + `entity` (+ optional `processId` / `subProcessId`) from `fss_idp_upload` → `messageName` + `processDefinitionKey`. Phase 1: `ID` → `IAP_ID_IDP_Trigger` / `IAP_ID_Payments`. |
| `IDPTriggerPaymentHandler` (gateway Java) | **Executes** | On `Trigger_IDP_Payment` external task: load upload → `registry.resolve(...)` → `startMessageCorrelation(messageName, ...)` → update `payment_workflow_key`. No per-country `if` chains in handler code. |

**Principle:** Message names are **declared in country BPMN** (deploy contract). **Which** message to fire is **selected at runtime** from upload metadata via the registry. Adding a country does **not** require changing `IDP_Document_Ingestion.bpmn` — only a new message start on that country's payment BPMN plus one registry row.

**Implementation detail:** registry interface, YAML/DB config, resolution order, and handler sequence diagram — [IDP_LLD.md §4](./IDP_LLD.md#4-country--entity-routing) and [§3.3](./IDP_LLD.md#33-handoff-flow-trigger_idp_payment).

---

## 3. Production safety — what must not change

### 3.1 Existing entry paths (frozen)

| Start event | Message / type | First step | Producer (Java) |
|-------------|----------------|------------|-----------------|
| `IAP_ID_AutomatedTrigger` | Message `IAP_ID_AutomatedTrigger` | `Initialize_IAP_Payments` | `IAPIDPaymentsMessageHandler` |
| `IAP_ID_BulkTrigger` | Message `IAP_ID_BulkTrigger` | `Initialize_IAP_Bulk_Payments` | `FssPaymentsSHNBatchUpload` |
| `IAP_ID_Manual_Payment` | Plain start | `IAP_ID_MakerPayment` | Manual UI / generic start |

**Do not modify:** sequence flows, gateway conditions, task IDs, or external task topics on these paths.

### 3.2 Frozen downstream (shared enrichment + completion)

| Region | Elements |
|--------|----------|
| Enrichment chain | `Initialize_IAP_ID_Payments` → OTT → local charges → internal/external enrich → derive client → `Save_Payment_Transaction` |
| Gateways | `PaymentValidationGateway`, `CheckerApprovalGateway`, `PaymentCheckGateway`, `FutureDateValidationGateway` |
| Human tasks | `IAP_ID_MakerPayment`, `IAP_ID_CheckerPayment`, `IAP_ID_Claim_Payment` |
| Cut-off / S2B | `VerifyPaymentCutoffStatus` → queue → `S2BFileGenerated` → complete → SSTM feedback |
| Boundary events | `CancelPaymentInstruction`, `ValueDateReached` |
| Messages | `IAP_ID_AutomatedTrigger`, `IAP_ID_BulkTrigger`, `CancelPaymentInstruction`, `ValueDateReached`, `S2BFileGenerated` |

### 3.3 Safe additions to `IAP_ID_Payments.bpmn` only

| Addition | Type | Notes |
|----------|------|-------|
| `IAP_ID_IDP_Trigger` | Message start event | 4th entry |
| `Initialize_IAP_From_IDP` | External service task | Topic `Initialize_IAP_From_IDP` |
| Sequence flow | Start → init → `Initialize_IAP_ID_Payments` | **Add** incoming on `Initialize_IAP_ID_Payments` (3rd incoming, alongside automated + bulk) |
| `Message_IAP_ID_IDP_Trigger` | BPMN message definition | New unique name |
| Text annotation | On new start | Cross-process documentation |

**Do not:** rename process id `IAP_ID_Payments`, remove existing incomings on `Initialize_IAP_ID_Payments`, or edit gateway condition expressions.

### 3.4 Camunda deployment behavior

| Scenario | Impact |
|----------|--------|
| In-flight instances on old definition version | Continue on version they started — **unaffected** |
| New automated/bulk/manual after deploy | Use latest version — must behave identically on old paths |
| New IDP path | Only starts when `IAP_ID_IDP_Trigger` correlated |

---

## 4. BPMN file 1: `IDP_Document_Ingestion.bpmn` (NEW)

### 4.1 Process metadata

| Attribute | Value |
|-----------|-------|
| Process ID | `IDP_Document_Ingestion` |
| Process name | `IDP_Document_Ingestion` |
| Executable | `true` |
| Owning module | `51786-workflow-management` |

### 4.2 Process flow

| Step | BPMN ID | Type | Topic / message |
|------|---------|------|-----------------|
| Start | `IDP_Upload_Start` | Plain start | REST upload persists file + triggers `startWorkflow` |
| Trigger extraction | `Trigger_IDP_Extraction` | External | `Trigger_IDP_Extraction` |
| Wait for extraction | `Event_IDPExtractionCompleted` | Intermediate catch | `IDPExtractionCompleted` |
| Extraction gateway | `IDPExtractionGateway` | Exclusive | `${idpStatus=='READY_FOR_REVIEW'}` |
| Maker review | `IDP_MakerReview` | User task | `${paymentMaker}` |
| Checker review | `IDP_CheckerReview` | User task | `${paymentChecker}` |
| Checker gateway | `IDPCheckerGateway` | Exclusive | `${idpApproved==true}` → handoff |
| Trigger payment | `Trigger_IDP_Payment` | External | `Trigger_IDP_Payment` |
| End | `Event_IDP_End` | End event | After successful handoff |

### 4.3 Optional boundary / failure paths

| Element | Message | Attached to |
|---------|---------|-------------|
| `CancelIDPUpload` | `CancelIDPUpload` | `IDP_MakerReview` |
| `IDPExtractionFailed` | `IDPExtractionFailed` | Extraction wait or gateway fail path |

### 4.4 BPMN messages (IDP process only)

| Message name | Producer | Consumer |
|--------------|----------|----------|
| `IDPExtractionCompleted` | **payment-gateway-service** (`IDPTriggerExtractionHandler`) after extract succeeds | `Event_IDPExtractionCompleted` |
| `IDPExtractionFailed` | **payment-gateway-service** on permanent extract failure | Boundary (optional) |
| `CancelIDPUpload` | Upload cancel API | Boundary on maker review |

**Important:** `IDP_Document_Ingestion` does **not** define `IAP_ID_IDP_Trigger` or any country payment messages — those belong to country payment BPMNs. Handoff message selection is via [§2.1](#21-country--payment-routing-responsibilities) / [IDP_LLD.md §4](./IDP_LLD.md#4-country--entity-routing).

Extraction service **never** correlates Camunda messages.

### 4.4.1 How gateway knows extraction is done (Phase 1 vs Phase 2)

| Phase | Extract API | How gateway knows it's done | BPMN |
|-------|-------------|----------------------------|------|
| **Phase 1 (recommended)** | `POST /v1/extract` — **synchronous** HTTP (blocks until OCR+LLM finish) | HTTP response = completion; handler persists run + correlates message in **same** `IDPTriggerExtractionHandler` invocation | Keep `Event_IDPExtractionCompleted`; gateway correlates immediately after sync call |
| Phase 2 (optional) | `POST /v1/extract` returns `202 Accepted` + `extractionId` | Gateway **polls** `GET /v1/extract/{extractionId}` or receives **callback** to gateway internal URL | Same message wait; async worker in gateway correlates when poll/callback says `COMPLETED` |

**Phase 1 choice:** synchronous HTTP matches the built extraction service and avoids “who notifies whom” complexity. The API is sync; only the **Camunda message catch** is async-shaped — gateway fills that gap in one handler after the HTTP call returns.

#### `IDPTriggerExtractionHandler` sequence (Phase 1 sync)

```
1. Load blob; insert fss_idp_extraction_run (PENDING)
2. POST /v1/extract (sync; gateway read-timeout 15 min, Camunda lock PT15M)
3. On HTTP 200 + valid JSON:
     - UPDATE run: structured_output, overall_confidence, run_status=READY_FOR_REVIEW
     - UPDATE upload: upload_status=READY_FOR_REVIEW
     - complete(Trigger_IDP_Extraction)
     - correlate(IDPExtractionCompleted, businessKey=idpUploadId, vars)
4. On HTTP error / timeout / invalid JSON:
     - UPDATE run: run_status=FAILED, error_*
     - UPDATE upload: upload_status=FAILED
     - fail external task (or correlate IDPExtractionFailed if boundary used)
     - do NOT correlate success message
```

#### Failure & shutdown scenarios (Phase 1)

| Scenario | System state | Recovery |
|----------|--------------|----------|
| **Extraction service down / timeout** | Run `FAILED` or still `PENDING`; Camunda task not completed | `ExternalTaskHelper` retries handler; increment `retry_count`; after max → `FAILED` |
| **Gateway crashes during HTTP call** | Camunda lock expires; upload `PROCESSING` | New worker re-claims task; idempotent: if `idp_extraction_audit` already `COMPLETED` for `correlationId`, call `GET /v1/extract/{extractionId}` instead of re-running OCR |
| **Gateway crashes after HTTP 200, before DB save** | Orphan audit in extraction service | Retry re-calls extract OR recovery uses `GET /v1/extract/{extractionId}` from audit `correlation_id` |
| **Gateway saves DB but correlate fails** | Run `READY_FOR_REVIEW`, upload stuck `PROCESSING`, Camunda at message wait | Reconciliation job: find mismatched rows → re-correlate `IDPExtractionCompleted`; do not re-extract |
| **Camunda down** | Handler finished extract + DB; task complete/correlate pending | Handler throws → task not completed → retry when Camunda returns |
| **Duplicate handler (lock race)** | Two workers same upload | Guard: only one run with `is_latest=true` in `PROCESSING`; second worker no-ops or loads existing result |

**Source of truth for UI:** `fss_idp_upload.upload_status` + `fss_idp_extraction_run.run_status` (gateway DB), not Camunda alone.

**Reconciliation (recommended):** scheduled job in gateway — uploads `PROCESSING` older than N minutes with run `READY_FOR_REVIEW` → correlate message; uploads `PROCESSING` with run `FAILED` → mark failed end.

### 4.4.2 Timeout configuration (10 min OCR+LLM budget)

See [IDP_LLD.md](./IDP_LLD.md) §7.8 for full hierarchy. Summary:

| Layer | Value | Owner |
|-------|-------|-------|
| OCR + LLM processing budget | **10 min** (`600000` ms) | idp-extraction-service |
| Inbound `/v1/extract` server timeout | **15 min** (`900000` ms) | idp-extraction-service Tomcat |
| Gateway HTTP client read (sync extract) | **15 min** (`900000` ms) | payment-gateway-service |
| Camunda `Trigger_IDP_Extraction` lock | **PT15M** | payment-gateway BPMN / handler |
| OCR client read | **7 min** (`420000` ms) | idp-extraction-service |
| LLM client read | **4 min** (`240000` ms) | idp-extraction-service |
| Stale `PROCESSING` reconciliation | **20 min** | payment-gateway scheduler |

**Critical:** infra proxies (load balancer, API gateway) must use **≥ 15 min** idle/read timeout on the gateway → extraction path.

`IDPTriggerExtractionHandler` uses extended lock and HTTP read timeout so the sync call is not cut off at 60 s. On timeout: mark run `FAILED`, fail external task, let Camunda retry per policy.

### 4.5 Java handlers (gateway-service + idp-extraction-service)

| Topic | Service | Handler (planned) |
|-------|---------|-------------------|
| `Trigger_IDP_Extraction` | payment-gateway-service | `IDPTriggerExtractionHandler` |
| `Trigger_IDP_Payment` | payment-gateway-service | `IDPTriggerPaymentHandler` |
| OCR + LLM | idp-extraction-service | `POST /v1/extract` only — **no workflow client** |
| Workflow after extract | payment-gateway-service | `IDPTriggerExtractionHandler` completes Camunda task / correlates message |

---

## 5. BPMN file 2: `IAP_ID_Payments.bpmn` (PROD — additive diff)

### 5.1 Before vs after entry paths

| # | Path | Before | After |
|---|------|--------|-------|
| 1 | Automated | `IAP_ID_AutomatedTrigger` → init → enrichment | **Same** |
| 2 | Bulk | `IAP_ID_BulkTrigger` → bulk init → enrichment | **Same** |
| 3 | Manual keying | `IAP_ID_Manual_Payment` → maker | **Same** |
| 4 | IDP document | — | `IAP_ID_IDP_Trigger` → `Initialize_IAP_From_IDP` → enrichment |

### 5.2 Merge point

`Initialize_IAP_From_IDP` → `Initialize_IAP_ID_Payments` (existing enrichment spine).

`Initialize_IAP_ID_Payments` incomings after change:

1. `Initialize_IAP_Payments` (automated) — existing  
2. `Initialize_IAP_Bulk_Payments` (bulk) — existing  
3. `Initialize_IAP_From_IDP` (IDP) — **new**

### 5.3 New external task on payment BPMN

| Task | Topic | Handler | Responsibility |
|------|-------|---------|----------------|
| `Initialize_IAP_From_IDP` | `Initialize_IAP_From_IDP` | `IAPIDPInitializeHandler` | Load structured JSON from latest extraction run; map to `Payment` / `Message`; `MessageService.save` with `channel=IDP_UPLOAD` |

---

## 6. Message contract (cross-process API)

### 6.1 Handoff message

The **message name is not fixed in** `IDP_Document_Ingestion.bpmn`. It is resolved by `IDPPaymentRouteRegistry` from the upload's `country` / `entity`. Phase 1 example (Indonesia):

| Attribute | Value (phase 1 — Indonesia) |
|-----------|----------------------------|
| Message name | `IAP_ID_IDP_Trigger` (from registry when `country=ID`) |
| Producer | `IDPTriggerPaymentHandler` on external task `Trigger_IDP_Payment` |
| Consumer | Message start `IAP_ID_IDP_Trigger` in `IAP_ID_Payments` |
| Correlation key | `businessKey = idpUploadId` |
| Mechanism | `registry.resolve(country, entity, ...)` then `WorkflowService.startMessageCorrelation(messageName, ...)` |

Future countries use their own message names (e.g. `CN_ETF_IDP_Trigger`) declared on their payment BPMN — see [IDP_LLD.md §4.1](./IDP_LLD.md#41-idppaymentrouteregistry-java).

### 6.2 Variables passed at correlation

| Variable | Type | Value / source |
|----------|------|----------------|
| `idpUploadId` | String | Upload row PK |
| `extractionRunId` | String | Latest extraction run with `is_latest=true` |
| `messageId` | String | If `fss_services_message` pre-created |
| `idpApproved` | Boolean | `true` |
| `isApproved` | Boolean | `true` — enables bulk-like skip of payment checker when validation clean |
| `isRepaired` | Boolean | `false` |
| `channel` | String | `IDP_UPLOAD` |
| `paymentMaker` | String | From upload / config |
| `paymentChecker` | String | From upload / config |

### 6.3 Idempotency

- Before correlating: check `fss_idp_upload.payment_workflow_key IS NULL`
- On success: store child `IAP_ID_Payments` process instance id in `payment_workflow_key`
- Double approve → no second payment instance

---

## 7. Two maker-checker cycles

| Stage | Process | User tasks | Purpose |
|-------|---------|------------|---------|
| Extraction QC | `IDP_Document_Ingestion` | `IDP_MakerReview`, `IDP_CheckerReview` | Validate OCR/LLM fields |
| Payment QC | `IAP_ID_Payments` | `IAP_ID_MakerPayment`, `IAP_ID_CheckerPayment` | Payment repair / banking approval |

**Routing after IDP checker approves:**

- Set `isApproved=true` at handoff.
- After `Save_Payment_Transaction`, existing `PaymentValidationGateway` flow `Flow_12mndij` (`isRepaired==false and isApproved==true`) may **skip payment checker**.
- If save yields `TO_BE_REPAIR`, existing `ToBeRepaired` → `IAP_ID_MakerPayment` — no new logic.

**Do not** reuse `IAP_ID_MakerPayment` for extraction review — affects manual and rework paths.

---

## 8. Manual trigger clarification

| Concept | BPMN | UI action | Change |
|---------|------|-----------|--------|
| Manual payment keying | `IAP_ID_Manual_Payment` | User keys payment fields | **No change** |
| IDP document upload | `IDP_Document_Ingestion` start | User uploads PDF/image | **New** — starts IDP process, not manual payment |

Upload REST must call `startWorkflow(IDP_Document_Ingestion)`, **not** `IAP_ID_Manual_Payment`.

---

## 9. Services and components

| Component | Action | Prod impact |
|-----------|--------|-------------|
| `51786-workflow-management` | Deploy `IDP_Document_Ingestion.bpmn` + new version `IAP_ID_Payments.bpmn` | Additive on payment BPMN |
| `51786-idp-extraction-service` | **New** — OCR, LLM, `POST /v1/extract`, technical audit — domain-agnostic; **no Camunda** |
| `51786-payment-gateway-service` | IDP REST upload (meta + content tables); 3 new external task handlers | Existing handlers unchanged |
| `51786-document-service` | **Phase 2** — not used in Phase 1 | None on existing |
| `51786-payment-publisher-service` | No change (unless IDP-specific SSTM feedback later) | None |
| `51786-payments-impl` | No schema change on `fss_payment_txns` | None |
| Portal UI | Upload, processing status, IDP maker/checker screens | Feature-flagged |

### 9.1 New external task topics (isolated)

| Topic | Process | Handler |
|-------|---------|---------|
| `Trigger_IDP_Extraction` | IDP | `IDPTriggerExtractionHandler` |
| `Trigger_IDP_Payment` | IDP | `IDPTriggerPaymentHandler` |
| `Initialize_IAP_From_IDP` | IAP_ID_Payments | `IAPIDPInitializeHandler` |

Existing 14 topics on `IAP_ID_Payments` — **no topic renames, no handler changes** unless explicitly required.

### 9.2 Unchanged prod producers

| Class | Still starts |
|-------|--------------|
| `IAPIDPaymentsMessageHandler` | `IAP_ID_AutomatedTrigger` |
| `FssPaymentsSHNBatchUpload` (country ID) | `IAP_ID_BulkTrigger` |
| Manual UI | `IAP_ID_Manual_Payment` |

---

## 10. Database design

### 10.1 New tables

#### `fss_idp_upload` (metadata)

| Column | Type | Notes |
|--------|------|-------|
| `idp_upload_id` | VARCHAR(36) PK | UUID; business key |
| `batch_id` | VARCHAR(36) | Optional multi-doc batch |
| `file_name` | VARCHAR(255) | |
| `content_type` | VARCHAR(100) | MIME type from upload |
| `file_size` | BIGINT | Bytes — on meta row so list/detail never touch BLOB table |
| `country` | VARCHAR(10) | `ID` |
| `dept_id` | VARCHAR(50) | |
| `process_id` | VARCHAR(50) | |
| `sub_process_id` | VARCHAR(50) | |
| `activity_id` | VARCHAR(50) | |
| `sub_activity_id` | VARCHAR(50) | |
| `upload_status` | VARCHAR(30) | UPLOADED, PROCESSING, READY_FOR_REVIEW, SUBMITTED, APPROVED, REJECTED, CANCELLED, FAILED |
| `idp_workflow_key` | VARCHAR(100) | IDP process instance id |
| `payment_workflow_key` | VARCHAR(100) | IAP payment process instance id (after handoff) |
| `payment_id` | VARCHAR(36) | After Save_Payment_Transaction |
| `message_id` | VARCHAR(36) | Link to `fss_services_message` |
| `uploaded_by` | VARCHAR(100) | |
| `uploaded_timestamp` | TIMESTAMP | |
| `approved_by` | VARCHAR(100) | IDP checker |
| `approved_timestamp` | TIMESTAMP | |
| `remarks` | VARCHAR(500) | Reject reason |
| Audit columns | | created/updated by/timestamp |

#### `fss_idp_upload_content` (blob — 1:1 with upload)

| Column | Type | Notes |
|--------|------|-------|
| `idp_upload_id` | VARCHAR(36) PK, FK | → `fss_idp_upload` |
| `file_content` | BLOB | Uploaded bytes; **only** loaded when OCR/extraction requests content |
| `created_timestamp` | TIMESTAMP | |

**Rationale:** List, detail, and workflow queries hit `fss_idp_upload` only. BLOB I/O is isolated to `IdpUploadContentRepository` — no accidental `@Lob` fetch on meta entity, smaller persistence context, better index/cache behaviour.

#### `fss_idp_extraction_run`

| Column | Type | Notes |
|--------|------|-------|
| `extraction_run_id` | VARCHAR(36) PK | |
| `idp_upload_id` | VARCHAR(36) FK | |
| `extraction_token` | VARCHAR(512) | OCR API token |
| `ocr_job_id` | VARCHAR(100) | External OCR id |
| `run_status` | VARCHAR(30) | PENDING, OCR_IN_PROGRESS, LLM_IN_PROGRESS, READY_FOR_REVIEW, FAILED |
| `ocr_response_payload` | CLOB | |
| `llm_response_payload` | CLOB | |
| `structured_output` | CLOB | Target: [../after_ocr-llm-output.json](../after_ocr-llm-output.json) — every field includes `Confidence` |
| `overall_confidence` | DECIMAL(5,2) | 0–100; from `initiationDetail.Confidence1` (fallback: min field confidence) |
| `error_code` / `error_description` | VARCHAR | |
| `retry_count` | INT | |
| `is_latest` | BOOLEAN | Re-extract support |
| Timestamps | TIMESTAMP | started, completed, created |

#### `fss_idp_extracted_field` (optional — UI edits)

| Column | Type | Notes |
|--------|------|-------|
| `field_id` | BIGINT PK | |
| `extraction_run_id` | VARCHAR(36) FK | |
| `section` | VARCHAR(50) | HEADER, TRANSACTION, DEBIT, CREDIT |
| `leg_index` | INT | |
| `field_name` | VARCHAR(100) | |
| `field_value` | VARCHAR(500) | |
| `confidence` | DECIMAL(5,2) | Per-field context confidence 0–100 from LLM |
| `is_edited` | BOOLEAN | |
| `edited_by` / `edited_timestamp` | | |

#### `fss_idp_upload_audit` (optional)

| Column | Type | Notes |
|--------|------|-------|
| `audit_id` | BIGINT PK | |
| `idp_upload_id` | VARCHAR(36) | |
| `action` | VARCHAR(50) | UPLOAD, EXTRACT_COMPLETE, MAKER_SUBMIT, CHECKER_APPROVE, PAYMENT_TRIGGERED, … |
| `actor` | VARCHAR(100) | |
| `details` | CLOB | |
| `timestamp` | TIMESTAMP | |

### 10.2 Existing tables (usage only)

| Table | Usage |
|-------|-------|
| `fss_services_message` | `channel=IDP_UPLOAD`; enriched Payment after `Initialize_IAP_From_IDP` |
| `fss_payment_txns` | No schema change |
| `fss_serv_batch` | Optional `batch_id` link for multi-doc upload |

### 10.3 Entity relationships

```mermaid
erDiagram
    fss_idp_upload ||--|| fss_idp_upload_content : stores
    fss_idp_upload ||--o| fss_idp_extraction_run : has
    fss_idp_extraction_run ||--o{ fss_idp_extracted_field : contains
    fss_idp_upload ||--o| fss_services_message : creates
    fss_idp_upload ||--o| fss_payment_txns : results_in
```

---

## 11. JSON → Payment mapping

Handler: `Initialize_IAP_From_IDP` / `IAPIDPInitializeHandler`

| JSON path | Payment / Message field |
|-----------|-------------------------|
| `header.uniqueId` | External ref / functional_id |
| `initiationDetail.TxRef` | Transaction reference |
| `initiationDetail.TrnTyp` / `SubActivityId` | Transaction type |
| `initiationDetail.ClntNm` | Client name |
| `initiationDetail.ValDt` | Value date |
| `TransactionDetails[*]` | Payment attributes |
| `DebitDetails.details[*].data` | Debit legs |
| `CreditDetails.details[*].data` | Credit legs |
| `header.doc_ids[0]` | `idpUploadId` (Phase 1) or document-service ref (Phase 2) |

Reference output: [../after_ocr-llm-output.json](../after_ocr-llm-output.json)

### 11.1 Structured output & confidence (LLM contract)

#### Per-field confidence (required)

Every value extracted by the LLM must include a **context-based confidence** score (0–100):

| JSON location | Shape |
|---------------|-------|
| `initiationDetail.Confidence1` | Overall instruction confidence |
| `TransactionDetails[]` | `{ "Name", "Value", "Confidence" }` |
| `DebitDetails.details[].data[]` | `{ "Name", "Value", "Confidence" }` |
| `CreditDetails.details[].data[]` | `{ "Name", "Value", "Confidence" }` |

- `Confidence` = how strongly the value is supported by OCR text + document context.
- Use `0` when the field is missing or cannot be determined.
- Enforced in `id-payment-v1/system.st` and `ExtractedField` Java type.

#### `overall_confidence` (extraction run row)

| Column | Population |
|--------|------------|
| `fss_idp_extraction_run.overall_confidence` | Primary: `initiationDetail.Confidence1`; fallback: minimum of all field `Confidence` values |

Used in upload list/detail API and maker UI (e.g. highlight fields where `Confidence < 90`).

#### Maker edits

Maker updates **field values** in the review payload (`structured_output` / `fss_idp_extracted_field`). Original LLM `Confidence` values are retained for audit; values are not re-scored on manual edit in Phase 1.

---

## 12. UI screens (planned)

| Screen | Process task | Actions |
|--------|--------------|---------|
| IDP Upload | Starts `IDP_Document_Ingestion` | Upload, Cancel |
| Processing status | Between extraction tasks | Poll upload status + `overall_confidence` |
| IDP Maker review | `IDP_MakerReview` | Save, Submit to checker, Cancel, Re-extract |
| IDP Checker review | `IDP_CheckerReview` | Approve, Reject |
| Payment maker/checker | `IAP_ID_MakerPayment` / `IAP_ID_CheckerPayment` | Existing — only if validation routes there |

---

## 13. Deployment plan

| Phase | Deliverable | Prod risk |
|-------|-------------|-----------|
| P1 | DDL `fss_idp_*` tables | None |
| P2 | `IDP_Document_Ingestion.bpmn` + IDP service (mock OCR/LLM) | None on IAP |
| P3 | Gateway workers for IDP topics | None until IDP started |
| P4 | Additive deploy `IAP_ID_Payments.bpmn` v(N+1) | Old paths must regression-pass |
| P5 | Real OCR + LLM integration | IDP path only |
| P6 | UI behind feature flag | Controlled rollout |
| P7 | Canary E2E test | New path only |

**Deploy order:** tables → IDP BPMN + service → payment BPMN additive version → enable UI flag.

**Rollback:** disable feature flag; IAP automated/bulk/manual unaffected.

---

## 14. Regression test matrix

| # | Scenario | Expected | Must match prod |
|---|----------|----------|-----------------|
| 1 | Automated (mock JMS) | `Initialize_IAP_Payments` | Yes |
| 2 | Bulk upload ID | Bulk init → `isApproved=true` path | Yes |
| 3 | Manual start | `IAP_ID_MakerPayment` directly | Yes |
| 4 | IDP upload → approve → handoff | `Initialize_IAP_From_IDP` → enrichment | New |
| 5 | Automated `TO_BE_REPAIR` | `IAP_ID_MakerPayment` | Yes |
| 6 | Cut-off rework gateways | Maker/checker routing | Yes |
| 7 | Cancel on payment maker | `PaymentCancellation` | Yes |
| 8 | Future value date | On-hold / `ValueDateReached` | Yes |
| 9 | Double IDP checker approve | Single payment instance | New |
| 10 | IDP handoff failure | IDP process does not complete; retries | New |

---

## 15. Risk register

| Risk | Mitigation |
|------|------------|
| Editing shared gateway conditions | Freeze prod paths; code review BPMN diff |
| Message name collision | Unique `IAP_ID_IDP_Trigger` only for IDP handoff |
| Duplicate payment instances | `payment_workflow_key` idempotency check |
| Handoff fails silently | External task `Trigger_IDP_Payment` with retry (not bare message end) |
| Variable type mismatch (boolean vs string) | Match existing patterns; document in correlation payload |
| Two instances in Cockpit | Shared `businessKey=idpUploadId`; link columns on `fss_idp_upload` |
| Deploy IAP BPMN without `Initialize_IAP_From_IDP` worker | Deploy workers before enabling IDP UI |

---

## 16. Implementation checklist

### BPMN

- [ ] Create `IDP_Document_Ingestion.bpmn` in `51786-workflow-management`
- [ ] Add annotations on IDP handoff task and IAP_IDP start
- [ ] Additive change to `IAP_ID_Payments.bpmn` (4th start + `Initialize_IAP_From_IDP` + merge)
- [ ] Verify `Initialize_IAP_ID_Payments` has 3 incomings, none removed

### Java

- [ ] `IDPUploadController` — persists meta + content rows, starts IDP process only
- [ ] `IDPTriggerExtractionHandler`
- [ ] `IDPTriggerPaymentHandler` — registry lookup + `startMessageCorrelation` (phase 1: `IAP_ID_IDP_Trigger`)
- [ ] `IDPPaymentRouteRegistry` / `YamlPaymentRouteRegistry` — config: `country=ID` → `IAP_ID_IDP_Trigger`
- [ ] `IAPIDPInitializeHandler` — topic `Initialize_IAP_From_IDP`
- [ ] Confirm no changes to `IAPIDPaymentsMessageHandler`, `FssPaymentsSHNBatchUpload`

### IDP service

- [ ] `51786-idp-extraction-service` — OCR, LLM, audit only (no workflow client)
- [ ] Correlate / complete extraction step from **gateway** `IDPTriggerExtractionHandler` only

### Database

- [ ] `fss_idp_upload` + `fss_idp_upload_content` + `fss_idp_extraction_run` (with `overall_confidence`)
- [ ] `fss_idp_extraction_run`
- [ ] Optional: `fss_idp_extracted_field`, `fss_idp_upload_audit`

### UI

- [ ] Upload screen (feature-flagged)
- [ ] Processing status view
- [ ] IDP maker / checker task forms
- [ ] Do not change manual payment keying flow

### Testing & release

- [ ] Regression matrix §14 on IAP v(N+1) before prod
- [ ] E2E IDP → payment path in lower env
- [ ] Feature flag + rollback runbook

---

## 17. Document history

| Date | Change |
|------|--------|
| 2026-07-30 | Initial version — two BPMN files, prod-safe additive IAP change, message handoff, tables, deployment and test plan |
| 2026-08-02 | Phase 1 sync extract; gateway workflow; resilience §4.4.1; timeouts §4.4.2 |
| 2026-08-02 | Country routing §2.1; registry model; moved to `final/` pack |

---

## 18. Open questions

- OCR API contract (auth, callback, max file size)?
- LLM hosting and prompt versioning?
- Full field parity with legacy upload screens?
- PII retention policy for OCR/LLM CLOB payloads?
- SSTM feedback behavior for `channel=IDP_UPLOAD` payments?

# IDP Document Upload — Low Level Design (LLD)

> **Status:** **Final** — implementation LLD (see [README.md](./README.md))  
> **Created:** 2026-07-30 · **Finalized:** 2026-08-02  
> **Related:** [README.md](./README.md) · [IDP_UX_Design.md](./IDP_UX_Design.md) · [IDP_Document_Ingestion_Design.md](./IDP_Document_Ingestion_Design.md) · [IDP_Document_Ingestion.bpmn](./IDP_Document_Ingestion.bpmn) · [../ID_payments.md](../ID_payments.md) · [../after_ocr-llm-output.json](../after_ocr-llm-output.json)

---

## 1. Purpose and scope

This LLD translates the UX specification and ingestion design into implementable detail: **one common IDP Camunda process for all countries**, **additive triggers on each country-specific payment BPMN**, **database schema**, **REST contracts**, and **Java/UI implementation strategy**.

### 1.1 Design evolution (from prior docs)

| Prior decision ([IDP_Document_Ingestion_Design.md](./IDP_Document_Ingestion_Design.md)) | LLD refinement |
|------------------------------------------------------------------------------------------|----------------|
| Common `IDP_Document_Ingestion.bpmn` + ID-only `IAP_ID_IDP_Trigger` handoff | **Confirmed and extended:** common IDP BPMN is **country-agnostic**; handoff message is **selected at runtime** from `country` / `entity` chosen in UI |
| Indonesia-first (`country = ID`) | **Phase 1 = ID**; registry pattern allows CN ETF and others without new IDP BPMN |
| Two BPMN files (IDP + additive IAP) | **Same pattern per country payment process** — not a single merged IAP file (obsolete single-BPMN draft retired; see `draft/IDP_ID_Payments_Enhancement.md` for history only) |

### 1.2 In scope (v1)

- Landing-page tab UX ([IDP_UX_Design.md](./IDP_UX_Design.md)) for **Indonesia (ID)** payments
- Common `IDP_Document_Ingestion` workflow (upload → OCR/LLM → IDP maker/checker)
- Additive 4th entry on `IAP_ID_Payments` (`IAP_ID_IDP_Trigger`)
- DDL for `fss_idp_*` tables
- Gateway REST + external task handlers
- New `51786-idp-extraction-service`

### 1.3 Out of scope (v1)

- Batch multi-file upload
- New sidebar nav route
- China ETF IDP handoff (registry entry only — implement in phase 2)
- SSTM feedback changes for `channel=IDP_UPLOAD`
- `51786-document-service` integration (Phase 2 — Phase 1 uses in-table BLOB + lazy fetch)

### 1.4 Domain glossary (avoid “job” confusion)

| Term | Table / artifact | Meaning |
|------|------------------|---------|
| **Upload** | `fss_idp_upload` | User-facing document + business status (`idp_upload_id`) |
| **Upload content** | `fss_idp_upload_content` | PDF/image bytes (lazy-fetched) |
| **Extraction run** | `fss_idp_extraction_run` | One OCR + LLM pass on an upload; re-extract = new run (`extraction_run_id`) |
| **Review payload** | `structured_output` CLOB (+ optional `fss_idp_extracted_field`) | Canonical JSON the maker/checker view and edit |
| **Workflow** | Camunda `IDP_Document_Ingestion` | Orchestration only — passes `idpUploadId`, not payloads |

**Not** a Quartz/cron/scheduled job. Code and UI should say *upload*, *extraction run*, or *re-extract* — not *job*.

---

## 2. Architecture overview

### 2.1 Two-layer BPMN model

```mermaid
flowchart TB
    subgraph UI["Portal — Instruction Upload tab"]
        UP["Upload form: Country / Entity / metadata + file"]
        TBL["Uploads table + detail modal"]
    end

    subgraph COMMON["IDP_Document_Ingestion.bpmn — SINGLE shared process"]
        START(["IDP_Upload_Start"]) --> EXTRACT["Trigger_IDP_Extraction"]
        EXTRACT --> WAIT{{"IDPExtractionCompleted"}}
        WAIT --> GW1{"Extraction OK?"}
        GW1 -->|fail| FAIL["End / FAILED"]
        GW1 -->|success| MK{{"IDP_MakerReview"}}
        MK --> CK{{"IDP_CheckerReview"}}
        CK --> GW2{"idpApproved?"}
        GW2 -->|reject| MK
        GW2 -->|approve| TRIGGER["Trigger_IDP_Payment"]
        TRIGGER --> END_IDP(["End"])
    end

    subgraph ROUTE["Runtime routing — country / entity from upload row"]
        REG["IDPPaymentRouteRegistry"]
    end

    subgraph ID_PAY["IAP_ID_Payments.bpmn — additive only"]
        ID_START(["IAP_ID_IDP_Trigger"]) --> ID_INIT["Initialize_IAP_From_IDP"]
        ID_INIT --> ID_ENR["Initialize_IAP_ID_Payments → enrichment spine"]
    end

    subgraph CN_PAY["ChinaETF_TPlusN.bpmn — phase 2 example"]
        CN_START(["CN_ETF_IDP_Trigger"]) --> CN_INIT["Initialize_CN_ETF_From_IDP"]
        CN_INIT --> CN_ENR["Existing CN enrichment spine"]
    end

    UP --> START
    TRIGGER --> REG
    REG -->|"country=ID"| ID_START
    REG -->|"country=CN, product=ETF"| CN_START
    TBL --> MK
    TBL --> CK
```

**Principle:** OCR, LLM, extraction QC (IDP maker/checker), and upload persistence live **only** in `IDP_Document_Ingestion`. Payment enrichment, cut-off, S2B, and payment maker/checker stay in **existing country BPMNs** — unchanged except for one new message start + one init external task per country.

### 2.2 Service interaction

```mermaid
flowchart LR
    UI["Portal UI"] --> GW["payment-gateway-service"]
    GW --> WFM["workflow-management"]
    GW --> IDP["idp-extraction-service"]
    IDP --> GW
    IDP --> WFM
    WFM --> GW
    GW --> CORE["payments-impl"]
    GW --> MSG["fss_services_message"]
    PUB["payment-publisher-service"] --> WFM
```

| Service | Responsibility |
|---------|----------------|
| `51786-payment-gateway-service` | `IDPUploadController` (meta + content tables), IDP external task workers, `Trigger_IDP_Payment` routing, `Initialize_*_From_IDP` handlers, internal content API for extraction |
| `51786-idp-extraction-service` | OCR client, LLM structuring, `POST /v1/extract`, technical audit (`idp_extraction_audit`) — **no Camunda, no workflow client** |
| `51786-workflow-management` | Deploy `IDP_Document_Ingestion.bpmn` + additive versions of country payment BPMNs |
| `51786-document-service` | **Phase 2 only** — not invoked in Phase 1 |
| `51786-payment-publisher-service` | No IDP changes in v1 (existing cut-off / status path after handoff) |

### 2.3 Orchestration boundary (critical)

The extraction service is a **common capability** — reusable beyond payments. It must **not** depend on Camunda, payment domain tables, or workflow clients.

| Responsibility | Owner |
|----------------|--------|
| `POST /v1/extract` — OCR + LLM + validate + audit | `51786-idp-extraction-service` |
| Persist `fss_idp_upload`, `fss_idp_extraction_run`, maker payload | `51786-payment-gateway-service` |
| Start / advance Camunda (`Trigger_IDP_Extraction`, complete task, correlate messages) | `51786-payment-gateway-service` |
| `correlationId` on extract request | Opaque caller reference (e.g. `idpUploadId`) — extraction service does not interpret it |

```mermaid
sequenceDiagram
    participant WFM as Camunda
    participant GW as payment-gateway-service
    participant EXT as idp-extraction-service

    WFM->>GW: External task Trigger_IDP_Extraction
    GW->>GW: Load blob, create extraction run row
    GW->>EXT: POST /v1/extract (correlationId=idpUploadId)
    EXT->>EXT: OCR → LLM → audit
    EXT-->>GW: structured JSON + extractionId
    GW->>GW: Save structured_output, overall_confidence
    GW->>WFM: complete task + correlate IDPExtractionCompleted
    Note over GW,WFM: Gateway owns workflow — extraction service never calls WorkflowService
```

---

## 3. BPMN design

### 3.1 Common process: `IDP_Document_Ingestion.bpmn`

| Attribute | Value |
|-----------|-------|
| Process ID | `IDP_Document_Ingestion` |
| Executable | `true` |
| Module | `51786-workflow-management` |
| Country scope | **Global** — one definition for all countries |

#### Process variables (set at start / maintained through flow)

| Variable | Type | Set by | Purpose |
|----------|------|--------|---------|
| `idpUploadId` | String | Upload REST | Business key; correlation id |
| `country` | String | UI upload form | Routes payment handoff |
| `entity` | String | UI (optional) | Legal entity / booking entity when distinct from country |
| `deptId`, `processId`, `subProcessId`, `activityId`, `subActivityId` | String | UI metadata dropdowns | Passed to OCR/LLM context |
| `paymentMaker`, `paymentChecker` | String | Config / upload | IDP user task assignees |
| `idpStatus` | String | Extraction + handlers | `PROCESSING`, `READY_FOR_REVIEW`, `FAILED`, … |
| `idpApproved` | Boolean | Checker complete task | Checker decision |
| `extractionRunId` | String | Extraction service | Latest run id (`is_latest=true`) |
| `paymentTriggerMessage` | String | `Trigger_IDP_Payment` (optional audit) | Message name correlated |

#### Task catalog

| BPMN ID | Type | Topic / message | Handler |
|---------|------|-----------------|---------|
| `IDP_Upload_Start` | Start | REST `startWorkflow` after meta + content rows persisted | `IDPUploadService` |
| `Trigger_IDP_Extraction` | External | `Trigger_IDP_Extraction` | `IDPTriggerExtractionHandler` |
| `Event_IDPExtractionCompleted` | Intermediate catch | `IDPExtractionCompleted` | **payment-gateway-service** correlates after extract (Phase 1: immediately after sync HTTP; Phase 2: after poll/callback) |
| `IDPExtractionGateway` | Exclusive | `${idpStatus=='READY_FOR_REVIEW'}` | — |
| `IDP_MakerReview` | User task | `${paymentMaker}` | Portal modal |
| `IDP_CheckerReview` | User task | `${paymentChecker}` | Portal modal |
| `IDPCheckerGateway` | Exclusive | `${idpApproved==true}` | — |
| `Trigger_IDP_Payment` | External | `Trigger_IDP_Payment` | `IDPTriggerPaymentHandler` |
| `Event_IDP_End` | End | — | — |

#### Boundary events

| Event | Message | Attached to | Action |
|-------|---------|-------------|--------|
| `CancelIDPUpload` | `CancelIDPUpload` | `IDP_MakerReview` | Set status `CANCELLED`; end or cancel path |
| `IDPExtractionFailed` | `IDPExtractionFailed` | Extraction wait (optional) | Set status `FAILED` |

#### BPMN messages (IDP process only)

| Message | Producer | Consumer |
|---------|----------|----------|
| `IDPExtractionCompleted` | **payment-gateway-service** (`IDPTriggerExtractionHandler` or async worker) | `Event_IDPExtractionCompleted` |
| `IDPExtractionFailed` | **payment-gateway-service** | Boundary (optional) |
| `CancelIDPUpload` | Upload cancel API | Boundary on maker review |

**Important:** `IDP_Document_Ingestion` does **not** define `IAP_ID_IDP_Trigger` or any country payment messages — those belong to country payment BPMNs.

### 3.2 Country payment BPMN — additive pattern (template)

Apply the **same additive diff** to each country payment process when IDP is enabled for that country.

#### Indonesia — `IAP_ID_Payments.bpmn` (phase 1)

| Addition | Detail |
|----------|--------|
| Message start | `IAP_ID_IDP_Trigger` (message `IAP_ID_IDP_Trigger`) |
| External task | `Initialize_IAP_From_IDP` (topic `Initialize_IAP_From_IDP`) |
| Merge | `Initialize_IAP_From_IDP` → `Initialize_IAP_ID_Payments` (3rd incoming) |
| Annotation | *"Triggered from IDP_Document_Ingestion via Trigger_IDP_Payment when country=ID"* |

Existing paths **frozen:** `IAP_ID_AutomatedTrigger`, `IAP_ID_BulkTrigger`, `IAP_ID_Manual_Payment` — no edits to sequence flows, gateway expressions, or existing external task topics.

#### China ETF — `ChinaETF_TPlusN.bpmn` (phase 2 example)

| Addition | Detail |
|----------|--------|
| Message start | `CN_ETF_IDP_Trigger` |
| External task | `Initialize_CN_ETF_From_IDP` |
| Merge | Into existing CN enrichment entry (equivalent of `Initialize_IAP_ID_Payments`) |

Same pattern for `ChinaETF_TplusZero` if product requires separate process.

### 3.3 Handoff flow (`Trigger_IDP_Payment`)

```mermaid
sequenceDiagram
    participant CK as IDP Checker (UI)
    participant GW as payment-gateway-service
    participant REG as IDPPaymentRouteRegistry
    participant WFM as Camunda engine
    participant PAY as IAP_ID_Payments instance

    CK->>GW: Complete IDP_CheckerReview (approve)
    WFM->>GW: External task Trigger_IDP_Payment
    GW->>GW: Load fss_idp_upload by businessKey
    GW->>GW: Assert payment_workflow_key IS NULL
    GW->>REG: resolve(country, entity, metadata)
    REG-->>GW: PaymentRoute(message=IAP_ID_IDP_Trigger, processKey=IAP_ID_Payments)
    GW->>WFM: startMessageCorrelation(IAP_ID_IDP_Trigger, businessKey=idpUploadId, vars)
    WFM->>PAY: New instance → Initialize_IAP_From_IDP
    GW->>GW: Update payment_workflow_key, upload_status=APPROVED
    GW->>WFM: Complete Trigger_IDP_Payment
```

#### Correlation payload (minimum)

| Variable | Value |
|----------|-------|
| `idpUploadId` | Upload PK |
| `extractionRunId` | Latest `is_latest=true` extraction run |
| `country` | From upload row |
| `entity` | From upload row |
| `channel` | `IDP_UPLOAD` |
| `idpApproved` | `true` |
| `isApproved` | `true` (bulk-like skip of payment checker when validation clean) |
| `isRepaired` | `false` |
| `paymentMaker` / `paymentChecker` | From upload / config |
| `messageId` | If pre-created on `fss_services_message` |

#### Idempotency

1. Before correlate: `SELECT payment_workflow_key FROM fss_idp_upload WHERE idp_upload_id = ?` — abort if non-null.
2. On success: store child process instance id in `payment_workflow_key`.
3. Double approve → single payment instance.

---

## 4. Country / entity routing

> **Cross-reference:** [IDP_Document_Ingestion_Design.md §2.1](./IDP_Document_Ingestion_Design.md#21-country--payment-routing-responsibilities) (same model, design-review wording).

### 4.0 Routing responsibilities (who decides what)

`IDP_Document_Ingestion.bpmn` is a decision engine for **IDP QC only** (`IDPExtractionGateway`, `IDPCheckerGateway`). It does **not** branch by country. `Trigger_IDP_Payment` is one external-task step — not a country-routing gateway.

| Layer | Owns country routing? | Responsibility |
|-------|----------------------|----------------|
| `IDP_Document_Ingestion.bpmn` | **No** | Common upload → extract → IDP maker/checker for all countries. Ends after `Trigger_IDP_Payment` completes. |
| Country payment BPMN | **Defines entry only** | Message start + `Initialize_*_From_IDP` per country (`IAP_ID_IDP_Trigger`, `CN_ETF_IDP_Trigger`, …). Names are BPMN/deploy contracts. |
| `IDPPaymentRouteRegistry` | **Yes — selection** | Config maps `country` / `entity` from `fss_idp_upload` → `messageName` + `processDefinitionKey`. |
| `IDPTriggerPaymentHandler` | **Executes** | Registry lookup + `startMessageCorrelation` — no scattered `if (country.equals(...))` in handler code. |

Adding a country: (1) additive message start on that country's payment BPMN, (2) one registry row. **No change** to `IDP_Document_Ingestion.bpmn`.

### 4.1 `IDPPaymentRouteRegistry` (Java)

Config-driven registry — **no country logic inside BPMN**.

```java
// Conceptual — payment-gateway-service
public record PaymentRoute(
    String messageName,      // e.g. IAP_ID_IDP_Trigger
    String processDefinitionKey, // e.g. IAP_ID_Payments
    String initializeTopic     // e.g. Initialize_IAP_From_IDP — for validation/docs
) {}

public interface IDPPaymentRouteRegistry {
    PaymentRoute resolve(String country, String entity,
                         String processId, String subProcessId);
}
```

#### Phase 1 registry entries

| country | entity | processId (optional) | messageName | processDefinitionKey | initializeTopic |
|---------|--------|----------------------|-------------|----------------------|-----------------|
| `ID` | * | * | `IAP_ID_IDP_Trigger` | `IAP_ID_Payments` | `Initialize_IAP_From_IDP` |

#### Phase 2 stubs (config only — no BPMN deploy until ready)

| country | entity | messageName | processDefinitionKey |
|---------|--------|-------------|----------------------|
| `CN` | `ETF` | `CN_ETF_IDP_Trigger` | `ChinaETF_TPlusN` |
| `CN` | `ETF_T0` | `CN_ETF_T0_IDP_Trigger` | `ChinaETF_TplusZero` |

**Resolution order:** exact match `(country, entity, processId, subProcessId)` → `(country, entity)` → `(country, *)` → error with clear ops message.

Store routes in `application.yml` or DB table `fss_idp_payment_route` (see §5) for runtime changes without redeploy.

### 4.2 UI country / entity selection

From [IDP_UX_Design.md](./IDP_UX_Design.md) upload zone:

```
Country [ID]  Dept ▼  Process ▼  SubProcess ▼  Activity ▼  SubAct ▼
```

| UI field | DB column | Workflow variable | Routing use |
|----------|-----------|-------------------|-------------|
| Country | `country` | `country` | Primary route key |
| Dept / Process / … | `dept_id`, … | same | OCR context + optional route refinement |
| Entity (if separate dropdown added) | `entity` | `entity` | Secondary route key |

For ID v1, country defaults to `ID` on ID payments landing; multi-country portal can expose country picker later using the same tab component.

---

## 5. Database design

### 5.1 New tables

#### `fss_idp_upload` (metadata only)

| Column | Type | Null | Notes |
|--------|------|------|-------|
| `idp_upload_id` | VARCHAR(36) | N | PK, UUID, Camunda business key |
| `batch_id` | VARCHAR(36) | Y | Optional multi-doc batch |
| `file_name` | VARCHAR(255) | N | |
| `content_type` | VARCHAR(100) | Y | MIME type |
| `file_size` | BIGINT | Y | Size in bytes — kept on meta row for list/detail without joining blob |
| `country` | VARCHAR(10) | N | Route key — **not hardcoded ID in schema** |
| `entity` | VARCHAR(50) | Y | Legal / booking entity |
| `dept_id` | VARCHAR(50) | Y | |
| `process_id` | VARCHAR(50) | Y | |
| `sub_process_id` | VARCHAR(50) | Y | |
| `activity_id` | VARCHAR(50) | Y | |
| `sub_activity_id` | VARCHAR(50) | Y | |
| `upload_status` | VARCHAR(30) | N | See §5.4 |
| `idp_workflow_key` | VARCHAR(100) | Y | IDP process instance id |
| `payment_workflow_key` | VARCHAR(100) | Y | Country payment instance id after handoff |
| `payment_trigger_message` | VARCHAR(100) | Y | Audited message name from registry |
| `payment_id` | VARCHAR(36) | Y | After `Save_Payment_Transaction` |
| `message_id` | VARCHAR(36) | Y | `fss_services_message` link |
| `uploaded_by` | VARCHAR(100) | N | |
| `uploaded_timestamp` | TIMESTAMP | N | Table sort: DESC |
| `approved_by` | VARCHAR(100) | Y | IDP checker |
| `approved_timestamp` | TIMESTAMP | Y | |
| `remarks` | VARCHAR(500) | Y | Reject / cancel reason |
| `created_by` | VARCHAR(100) | N | Audit |
| `created_timestamp` | TIMESTAMP | N | |
| `updated_by` | VARCHAR(100) | Y | |
| `updated_timestamp` | TIMESTAMP | Y | |

**Indexes:**

- `idx_idp_upload_status_ts` on (`upload_status`, `uploaded_timestamp` DESC)
- `idx_idp_upload_country_ts` on (`country`, `uploaded_timestamp` DESC)
- `idx_idp_upload_user` on (`uploaded_by`, `uploaded_timestamp` DESC)
- `uq_idp_payment_workflow` on (`payment_workflow_key`) WHERE `payment_workflow_key IS NOT NULL` (optional — idempotency)

#### `fss_idp_upload_content` (blob — 1:1 with upload)

| Column | Type | Null | Notes |
|--------|------|------|-------|
| `idp_upload_id` | VARCHAR(36) | N | PK, FK → `fss_idp_upload` |
| `file_content` | BLOB | N | Uploaded bytes |
| `created_timestamp` | TIMESTAMP | N | |

**Why split:** Keeps list/detail/workflow queries off the BLOB column. In Java, `IdpUploadEntity` has **no** `@Lob` field; content is loaded only via `IdpUploadContentRepository.findById(idpUploadId)` when the extraction pipeline needs it.

#### `fss_idp_extraction_run`

| Column | Type | Null | Notes |
|--------|------|------|-------|
| `extraction_run_id` | VARCHAR(36) | N | PK |
| `idp_upload_id` | VARCHAR(36) | N | FK → `fss_idp_upload` |
| `extraction_token` | VARCHAR(512) | Y | OCR API token |
| `ocr_job_id` | VARCHAR(100) | Y | External OCR id |
| `run_status` | VARCHAR(30) | N | PENDING, OCR_IN_PROGRESS, LLM_IN_PROGRESS, READY_FOR_REVIEW, FAILED |
| `ocr_response_payload` | CLOB | Y | Retention policy applies |
| `llm_response_payload` | CLOB | Y | |
| `structured_output` | CLOB | Y | Target: [../after_ocr-llm-output.json](../after_ocr-llm-output.json) — includes per-field `Confidence` |
| `overall_confidence` | DECIMAL(5,2) | Y | Document-level score 0–100; see §5.6 |
| `error_code` | VARCHAR(50) | Y | |
| `error_description` | VARCHAR(500) | Y | |
| `retry_count` | INT | N | Default 0 |
| `is_latest` | BOOLEAN | N | Default true; false on re-extract |
| `started_timestamp` | TIMESTAMP | Y | |
| `completed_timestamp` | TIMESTAMP | Y | |
| `created_timestamp` | TIMESTAMP | N | |

**Indexes:**

- `idx_idp_run_upload` on (`idp_upload_id`, `is_latest`)
- `idx_idp_run_status` on (`run_status`)

#### `fss_idp_extracted_field` (recommended for modal PATCH)

| Column | Type | Null | Notes |
|--------|------|------|-------|
| `field_id` | BIGINT | N | PK, identity |
| `extraction_run_id` | VARCHAR(36) | N | FK |
| `section` | VARCHAR(50) | N | HEADER, TRANSACTION, DEBIT, CREDIT |
| `leg_index` | INT | Y | 0-based leg |
| `field_name` | VARCHAR(100) | N | |
| `field_value` | VARCHAR(500) | Y | |
| `confidence` | DECIMAL(5,2) | Y | Per-field context confidence 0–100 from LLM; preserved when flattening from `structured_output` |
| `is_edited` | BOOLEAN | N | Default false |
| `edited_by` | VARCHAR(100) | Y | |
| `edited_timestamp` | TIMESTAMP | Y | |

**Indexes:** `idx_idp_field_run` on (`extraction_run_id`, `section`, `leg_index`)

#### `fss_idp_upload_audit` (recommended)

| Column | Type | Null | Notes |
|--------|------|------|-------|
| `audit_id` | BIGINT | N | PK |
| `idp_upload_id` | VARCHAR(36) | N | |
| `action` | VARCHAR(50) | N | UPLOAD, EXTRACT_COMPLETE, MAKER_SUBMIT, CHECKER_APPROVE, PAYMENT_TRIGGERED, … |
| `actor` | VARCHAR(100) | Y | |
| `details` | CLOB | Y | JSON snippet |
| `timestamp` | TIMESTAMP | N | |

#### `fss_idp_payment_route` (optional — alternative to YAML)

| Column | Type | Notes |
|--------|------|-------|
| `route_id` | BIGINT PK | |
| `country` | VARCHAR(10) | |
| `entity` | VARCHAR(50) | Nullable = wildcard |
| `process_id` | VARCHAR(50) | Optional filter |
| `sub_process_id` | VARCHAR(50) | Optional |
| `message_name` | VARCHAR(100) | e.g. `IAP_ID_IDP_Trigger` |
| `process_definition_key` | VARCHAR(100) | |
| `initialize_topic` | VARCHAR(100) | |
| `is_active` | BOOLEAN | |
| `priority` | INT | Higher wins on match |

### 5.2 Existing tables (usage)

| Table | Usage |
|-------|-------|
| `fss_services_message` | `channel = IDP_UPLOAD`; enriched Payment after `Initialize_*_From_IDP` |
| `fss_payment_txns` | No schema change; row created by existing `Save_Payment_Transaction` |
| `fss_serv_batch` | Optional `batch_id` on upload |
| `act_*` | Camunda state for IDP + payment instances |

### 5.3 ER diagram

```mermaid
erDiagram
    fss_idp_upload ||--|| fss_idp_upload_content : stores
    fss_idp_upload ||--o{ fss_idp_extraction_run : has
    fss_idp_extraction_run ||--o{ fss_idp_extracted_field : contains
    fss_idp_upload ||--o{ fss_idp_upload_audit : audited_by
    fss_idp_upload ||--o| fss_services_message : creates
    fss_idp_upload ||--o| fss_payment_txns : results_in
    fss_idp_payment_route ||..o{ fss_idp_upload : routes
```

### 5.4 Upload status lifecycle

| Status | Set by | UI table | Row clickable |
|--------|--------|----------|---------------|
| `UPLOADED` | Insert on REST | — | No |
| `PROCESSING` | After start workflow | Processing | No |
| `READY_FOR_REVIEW` | Extraction complete | Ready review | Yes (maker) |
| `SUBMITTED` | Maker submit | With checker | Yes |
| `REJECTED` | Checker reject | Rejected | Yes (maker) |
| `APPROVED` | Handoff triggered | Approved | Yes (read-only) |
| `COMPLETED` | Payment completed | Completed | Yes |
| `FAILED` | Extraction fail | Failed | Yes |
| `CANCELLED` | Maker cancel | Cancelled | Optional |

Map `upload_status` ↔ `idpStatus` process variable on every transition.

### 5.5 DDL sketch (Oracle / generic)

```sql
CREATE TABLE fss_idp_upload (
    idp_upload_id        VARCHAR2(36)  PRIMARY KEY,
    batch_id             VARCHAR2(36),
    file_name            VARCHAR2(255) NOT NULL,
    content_type         VARCHAR2(100),
    file_size            NUMBER(19),
    country              VARCHAR2(10)  NOT NULL,
    entity               VARCHAR2(50),
    dept_id              VARCHAR2(50),
    process_id           VARCHAR2(50),
    sub_process_id       VARCHAR2(50),
    activity_id          VARCHAR2(50),
    sub_activity_id      VARCHAR2(50),
    upload_status        VARCHAR2(30)  NOT NULL,
    idp_workflow_key     VARCHAR2(100),
    payment_workflow_key VARCHAR2(100),
    payment_trigger_message VARCHAR2(100),
    payment_id           VARCHAR2(36),
    message_id           VARCHAR2(36),
    uploaded_by          VARCHAR2(100) NOT NULL,
    uploaded_timestamp   TIMESTAMP     NOT NULL,
    approved_by          VARCHAR2(100),
    approved_timestamp   TIMESTAMP,
    remarks              VARCHAR2(500),
    created_by           VARCHAR2(100) NOT NULL,
    created_timestamp    TIMESTAMP     NOT NULL,
    updated_by           VARCHAR2(100),
    updated_timestamp    TIMESTAMP
);

CREATE TABLE fss_idp_upload_content (
    idp_upload_id        VARCHAR2(36)  PRIMARY KEY REFERENCES fss_idp_upload(idp_upload_id),
    file_content         BLOB          NOT NULL,
    created_timestamp    TIMESTAMP     NOT NULL
);

CREATE TABLE fss_idp_extraction_run (
    extraction_run_id    VARCHAR2(36)  PRIMARY KEY,
    idp_upload_id        VARCHAR2(36)  NOT NULL REFERENCES fss_idp_upload(idp_upload_id),
    extraction_token     VARCHAR2(512),
    ocr_job_id           VARCHAR2(100),
    run_status           VARCHAR2(30)  NOT NULL,
    ocr_response_payload CLOB,
    llm_response_payload CLOB,
    structured_output    CLOB,
    overall_confidence   NUMBER(5,2),
    error_code           VARCHAR2(50),
    error_description    VARCHAR2(500),
    retry_count          NUMBER(5)     DEFAULT 0 NOT NULL,
    is_latest            CHAR(1)       DEFAULT 'Y' NOT NULL,
    started_timestamp    TIMESTAMP,
    completed_timestamp  TIMESTAMP,
    created_timestamp    TIMESTAMP     NOT NULL
);

-- fss_idp_extracted_field, fss_idp_upload_audit, fss_idp_payment_route — same columns as §5.1
```

### 5.6 Structured output & confidence scores

#### LLM output shape (required)

Every extracted field in `structured_output` must include a **context-based confidence** score (0–100). This matches the legacy ID payment screen and [../after_ocr-llm-output.json](../after_ocr-llm-output.json).

| Location | Field | Type | Meaning |
|----------|-------|------|---------|
| `initiationDetail` | `Confidence1` | string/number | **Instruction-level** overall confidence from LLM |
| `TransactionDetails[*]` | `Name`, `Value`, `Confidence` | object | Transaction attribute + per-field confidence |
| `DebitDetails.details[*].data[*]` | `Name`, `Value`, `Confidence` | object | Debit leg field + confidence |
| `CreditDetails.details[*].data[*]` | `Name`, `Value`, `Confidence` | object | Credit leg field + confidence |

Example field object:

```json
{
  "Name": "TrnRef",
  "Value": "IDPYRE015001",
  "Confidence": "92.76"
}
```

**LLM prompt rules** (`51786-idp-extraction-service` template `id-payment-v1/system.st`):

1. Emit `Confidence` for **every** field in `TransactionDetails`, `DebitDetails`, and `CreditDetails`.
2. Use `0` when the field is absent or cannot be determined from the document context.
3. Confidence reflects how strongly the value is supported by OCR text and document context (not OCR engine score alone).
4. Emit `initiationDetail.Confidence1` as the overall instruction confidence.

**Java template:** `ExtractedField` (`Name`, `Value`, `Confidence`); `InitiationDetail.confidence1` → `Confidence1`.

#### `overall_confidence` column (extraction run row)

Denormalized on `fss_idp_extraction_run` for list views and sorting without parsing CLOB.

| Source | Rule |
|--------|------|
| Primary | `initiationDetail.Confidence1` from LLM structured output |
| Fallback | Minimum of all per-field `Confidence` values in `structured_output` |
| After maker edit | Unchanged unless business rule recalculates; edited fields set `is_edited=true` in `fss_idp_extracted_field` |

Populated when persisting the run after `POST /v1/extract` returns.

#### Maker review & confidence

- Maker **edits field values** in the review payload (`PATCH /uploads/{id}/fields`).
- On save: update `structured_output` and/or `fss_idp_extracted_field` for the current run (`is_latest=true`).
- **Original** LLM `Confidence` is retained for audit.
- UI highlights fields where `Confidence < 90` (configurable threshold).

#### Extraction service audit (`idp_extraction_audit`)

Technical trace only (OCR raw, LLM raw). Gateway `fss_idp_extraction_run` is the **business** record used by portal and payment handoff.

---

## 6. REST API design

> **Full API reference:** [IDP_API_Reference.md](./IDP_API_Reference.md) — request/response examples, UI action map, extraction service contract.

Base path: `/v1/idp` (gateway-service). Auth: existing portal JWT + `paymentMaker` / `paymentChecker` groups.

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/uploads` | Multipart upload + metadata; inserts meta + content rows; starts `IDP_Document_Ingestion` |
| `GET` | `/uploads` | List with `sort=recent`, filters: `status`, `country`, `myUploads` |
| `GET` | `/uploads/{id}` | Detail for modal (meta + latest extraction run + fields; **no** blob) |
| `GET` | `/internal/uploads/{id}/content` | **Internal** — streams blob from `fss_idp_upload_content` for idp-extraction-service |
| `PATCH` | `/uploads/{id}/fields` | Maker save draft |
| `POST` | `/uploads/{id}/re-extract` | New extraction run; upload → `PROCESSING` |
| `POST` | `/workflow/complete` | Existing generic complete — maker submit, checker approve/reject, cancel |

### 6.1 `POST /v1/idp/uploads` request

```json
{
  "country": "ID",
  "entity": null,
  "deptId": "FundServices",
  "processId": "Cash",
  "subProcessId": "CI",
  "activityId": "BT",
  "subActivityId": "Redemption"
}
```

Multipart: `file` (PDF/image).

### 6.2 `POST /v1/idp/uploads` response

```json
{
  "idpUploadId": "a1b2c3d4-...",
  "uploadStatus": "PROCESSING",
  "idpWorkflowKey": "..."
}
```

### 6.3 List response row

```json
{
  "idpUploadId": "...",
  "fileName": "3897122.pdf",
  "uploadedTimestamp": "2026-07-30T10:22:00Z",
  "uploadStatus": "READY_FOR_REVIEW",
  "overallConfidence": 97.23,
  "uploadedBy": "you",
  "paymentRef": null,
  "country": "ID"
}
```

---

## 7. Code implementation strategy

### 7.1 Module layout

```
51786-payment-gateway-service/
  src/main/java/.../idp/
    controller/IDPUploadController.java
    service/
      IDPUploadService.java
      IdpUploadContentService.java
      IDPFieldService.java
      IDPPaymentRouteRegistry.java (+ YamlPaymentRouteRegistry)
    workflow/handler/idp/
      IDPTriggerExtractionHandler.java
      IDPTriggerPaymentHandler.java
    workflow/handler/iap/id/
      IAPIDPInitializeHandler.java      // topic Initialize_IAP_From_IDP
    mapper/
      IDPStructuredOutputMapper.java    // JSON → Payment / fields
    model/entity/
      IdpUploadEntity.java              // metadata only — no @Lob
      IdpUploadContentEntity.java       // blob table — separate persistence unit path
      IdpExtractionRunEntity.java
      IdpExtractedFieldEntity.java
    repository/
      IdpUploadRepository.java
      IdpUploadContentRepository.java
      IdpExtractionRunRepository.java

51786-idp-extraction-service/   (NEW)
  src/main/java/.../
    client/OcrApiClient.java
    client/LlmStructuringClient.java
    client/IdpUploadContentClient.java
    service/ExtractionPipelineService.java
    service/ExtractionCorrelationService.java
    model/entity/ExtractionAuditEntity.java
    service/audit/ExtractionAuditService.java

51786-workflow-management/
  src/main/resources/
    IDP_Document_Ingestion.bpmn
    IAP_ID_Payments.bpmn          // additive v(N+1)
```

### 7.2 Implementation patterns (align with [../ID_payments.md](../ID_payments.md))

| Pattern | Application |
|---------|-------------|
| External task workers | All IDP service tasks use `@ExternalTaskSubscription` + `ExternalTaskHelper` retry — same as `IAPPaymentTransactionHandler` |
| Message correlation | `WorkflowService.startMessageCorrelation` for handoff and extraction complete |
| Payment enrichment | **No** IDP logic in enrichment chain — only `Initialize_*_From_IDP` builds `Message.enrichedMessage` |
| Process variables | Booleans for gateways (`idpApproved`); strings `"true"`/`"false"` for payment gateways after handoff — match existing IAP conventions |
| Human tasks | Portal completes via generic `/v1/workflow/complete` — not country-specific Java |

### 7.3 Key class responsibilities

#### `IDPUploadService`

1. Validate file type / size.
2. Insert `fss_idp_upload` metadata row (`UPLOADED`).
3. Insert `fss_idp_upload_content` with `file_content` (same transaction).
4. Call `WorkflowService.startWorkflow` with `processKey = IDP_Document_Ingestion`, `businessKey = idpUploadId`, variables from form.
5. Update `idp_workflow_key`, `upload_status = PROCESSING`.

**Must not** start `IAP_ID_Manual_Payment` or country payment processes directly.

#### `IdpUploadContentService`

1. **Only** service/repository that touches `fss_idp_upload_content`.
2. `getContent(idpUploadId)` — single-row `SELECT file_content` when OCR pipeline requests bytes.
3. Exposed via internal `GET /internal/uploads/{id}/content` for idp-extraction-service.
4. Never joined or eagerly loaded from `IdpUploadEntity`.

#### `IDPTriggerExtractionHandler`

1. Create `fss_idp_extraction_run` (`PENDING`, `is_latest=true`).
2. Fetch blob via `IdpUploadContentService`; call idp-extraction-service `POST /v1/extract` (**synchronous** — blocks until OCR+LLM finish; configure Camunda external-task lock ≥ 15 min).
3. On success: persist `structured_output`, `overall_confidence`, `run_status=READY_FOR_REVIEW`; set `upload_status=READY_FOR_REVIEW`.
4. `complete(Trigger_IDP_Extraction)` then `correlate(IDPExtractionCompleted, businessKey=idpUploadId, { idpStatus, extractionRunId })` — both from **gateway**, never extraction service.
5. On failure: set run/upload `FAILED`; fail external task; optionally correlate `IDPExtractionFailed` — do not correlate success.

See §7.6 for crash/timeout recovery and reconciliation job.

#### `IDPTriggerPaymentHandler`

1. Load upload; guard `payment_workflow_key`.
2. `registry.resolve(country, entity, processId, subProcessId)`.
3. Build correlation variables (§3.3).
4. `startMessageCorrelation(route.messageName(), ...)`.
5. Update `payment_workflow_key`, `payment_trigger_message`, `upload_status = APPROVED`.
6. Audit `PAYMENT_TRIGGERED`.
7. Complete external task.

#### `IAPIDPInitializeHandler` (`Initialize_IAP_From_IDP`)

1. Load latest extraction run `structured_output` (`is_latest=true`).
2. `IDPStructuredOutputMapper` → `Payment` / `PaymentData` (§7.4).
3. `MessageService.save` with `channel=IDP_UPLOAD`, `country=ID`.
4. Set `messageId` variable; complete task.
5. Flow continues to `Initialize_IAP_ID_Payments` — **no changes** to downstream handlers.

#### Country-specific init handlers (phase 2)

- `CNEFTPDPInitializeHandler` — same structure, CN-specific mapper.
- Consider abstract `AbstractIDPInitializeHandler` with injected `PaymentMapper` per country.

### 7.4 JSON → Payment mapping (ID)

Handler: `Initialize_IAP_From_IDP` / `IAPIDPInitializeHandler`

| JSON path | Payment / Message field |
|-----------|-------------------------|
| `header.uniqueId` | External ref / `functional_id` |
| `initiationDetail.TxRef` | Transaction reference |
| `initiationDetail.TrnTyp` / `SubActivityId` | Transaction type |
| `initiationDetail.ClntNm` | Client name |
| `initiationDetail.ValDt` | Value date |
| `TransactionDetails[*]` | Payment attributes |
| `DebitDetails.details[*].data` | Debit legs |
| `CreditDetails.details[*].data` | Credit legs |
| `header.doc_ids[0]` | Document linkage |

Reference: [../after_ocr-llm-output.json](../after_ocr-llm-output.json)

Keep mapper **per country** when CN field shapes diverge; share leg-building utilities.

### 7.5 idp-extraction-service pipeline

```mermaid
flowchart LR
    A["Run created PENDING"] --> LF["Fetch blob from fss_idp_upload_content"]
    LF --> B["OCR_IN_PROGRESS"]
    B --> C["LLM_IN_PROGRESS"]
    C --> D{"OK?"}
    D -->|yes| E["Gateway persists run + advances workflow"]
    D -->|no| F["FAILED + optional IDPExtractionFailed"]
    E --> G["Populate structured_output + optional field rows"]
```

On success:

1. Update extraction run + upload status; set `overall_confidence`.
2. Optionally flatten to `fss_idp_extracted_field` (including per-field `confidence`).
3. **Gateway** correlates `IDPExtractionCompleted` (after sync HTTP returns and DB is updated) — never the extraction service.

### 7.6 Error handling & resilience

| Layer | Strategy |
|-------|----------|
| External tasks | `ExternalTaskHelper` retries; `Trigger_IDP_Extraction` lock **PT15M**, HTTP read **900000 ms** — see §7.8 |
| Extract completion (Phase 1) | **Synchronous** `POST /v1/extract` — gateway knows done when HTTP returns; then DB update → complete task → correlate `IDPExtractionCompleted` |
| Extract completion (Phase 2) | Async `202` + gateway poll `GET /v1/extract/{id}` or callback to gateway internal endpoint — still **no** workflow client in extraction service |
| Handoff | If correlate fails, **do not** complete `Trigger_IDP_Payment`; upload stays `SUBMITTED`; ops alert |
| Extraction retry | `retry_count` on run; re-extract creates new run, marks old `is_latest=false` |
| Stuck uploads | Reconciliation scheduler in gateway (see §7.6.1) |
| Payment gateways | Existing IAP behavior after handoff — no IDP changes |

#### 7.6.1 Reconciliation scheduler (gateway)

Periodic job (e.g. every 5 min) in **payment-gateway-service**:

| Condition | Action |
|-----------|--------|
| `upload_status=PROCESSING` AND run `run_status=READY_FOR_REVIEW` AND Camunda waiting on `IDPExtractionCompleted` | Re-correlate success message (idempotent) |
| `upload_status=PROCESSING` AND run `run_status=FAILED` for > N min | Correlate failure / end process; set `upload_status=FAILED` |
| `upload_status=PROCESSING` AND no run progress for > timeout | Mark `FAILED` or retry extract (config) |

#### 7.6.2 Failure matrix

| Scenario | Extraction service | Gateway DB | Camunda | Recovery |
|----------|-------------------|------------|---------|----------|
| Extract HTTP timeout | May have audit `FAILED` or none | Run `FAILED` or `PENDING` | Task lock expires → retry | Handler retry; optional `GET` audit by `correlationId` |
| Gateway crash mid-HTTP | Audit may be `COMPLETED` | Run incomplete | Lock expires → retry | Poll `GET /v1/extract/{extractionId}` or re-post if no audit |
| DB saved, correlate fails | — | Run `READY`, upload `PROCESSING` | Stuck at catch event | Reconciliation job correlates |
| Extraction service down | 503 | No change | Task retries | Backoff retry; UI shows Processing |
| Camunda down | Extract may finish | DB updated | Complete/correlate fails | Handler throws; retry when WFM up |

**Idempotency keys:** `idpUploadId` (business key) + `extraction_run_id`; before re-extracting, check `idp_extraction_audit` for `correlation_id=idpUploadId` and `job_status=COMPLETED`.

### 7.7 Feature flags

| Flag | Purpose |
|------|---------|
| `idp.instruction-upload.enabled` | Show upload tab |
| `idp.countries.enabled` | List e.g. `ID` only in v1 |
| `idp.extraction.mock` | Mock OCR/LLM in lower env |

### 7.8 Timeout configuration (OCR + LLM ≈ 10 min max)

Phase 1 uses **synchronous** `POST /v1/extract`. Every layer in the call chain must allow the full pipeline to finish without premature disconnect. Use a **10 min processing budget** with a **15 min envelope** on outer HTTP/Camunda locks.

#### Timeout hierarchy (outer ≥ inner)

```mermaid
flowchart TB
    CAM["Camunda lockDuration\nPT15M"]
    GW["Gateway HTTP read timeout\n900000 ms"]
    EXT_SRV["Extraction Tomcat connection-timeout\n900000 ms"]
    PIPE["Pipeline budget idp.extraction.max-pipeline-ms\n600000 ms"]
    OCR["OCR client read-timeout-ms\n420000 ms"]
    LLM["LLM read-timeout-ms\n240000 ms"]

    CAM --> GW --> EXT_SRV --> PIPE
    PIPE --> OCR
    PIPE --> LLM
```

| Layer | Service | Property | Recommended | Notes |
|-------|---------|----------|-------------|-------|
| Camunda external task lock | payment-gateway | `lockDuration` on topic `Trigger_IDP_Extraction` | **PT15M** (900 s) | Must exceed gateway HTTP read timeout; set in BPMN or `@ExternalTaskSubscription` |
| HTTP client → extraction | payment-gateway | `idp.extraction-client.read-timeout-ms` | **900000** (15 min) | Blocks until `/v1/extract` returns |
| HTTP connect | payment-gateway | `idp.extraction-client.connect-timeout-ms` | **10000** | |
| Tomcat connection (inbound) | idp-extraction-service | `server.tomcat.connection-timeout` | **900000** (15 min) | Must be ≥ gateway client read timeout |
| Pipeline budget (OCR+LLM) | idp-extraction-service | `idp.extraction.max-pipeline-ms` | **600000** (10 min) | Target max processing time; use for watchdog/logging |
| OCR API read | idp-extraction-service | `idp.api.read-timeout-ms` | **420000** (7 min) | Leaves headroom for LLM within 10 min budget |
| LLM API read | idp-extraction-service | `idp.llm.read-timeout-ms` | **240000** (4 min) | Spring AI / OpenAI client; wire when not in mock mode |
| Stale upload reconciliation | payment-gateway | `idp.upload.processing-stale-timeout-ms` | **1200000** (20 min) | Reconciliation job marks stuck `PROCESSING` rows |

**Rule:** `Camunda lock ≥ gateway read ≥ extraction server timeout ≥ max-pipeline-ms ≥ (OCR + LLM timeouts)`.

#### `51786-idp-extraction-service` — `application.yml`

```yaml
server:
  tomcat:
    connection-timeout: 900000   # 15 min — inbound POST /v1/extract

idp:
  extraction:
    max-pipeline-ms: 600000      # 10 min OCR+LLM budget
  api:
    connect-timeout-ms: 10000
    read-timeout-ms: 420000      # 7 min OCR
  llm:
    read-timeout-ms: 240000      # 4 min LLM structuring
```

#### `51786-payment-gateway-service` — `application.yml` (planned)

```yaml
idp:
  extraction-client:
    base-url: http://51786-idp-extraction-service:8095
    connect-timeout-ms: 10000
    read-timeout-ms: 900000        # 15 min — must not cut off sync extract
  upload:
    processing-stale-timeout-ms: 1200000
  camunda:
    external-tasks:
      Trigger_IDP_Extraction:
        lock-duration: PT15M
        retries: 2
```

#### `IDPTriggerExtractionHandler` — timeout behaviour

| Event | Gateway action |
|-------|----------------|
| HTTP read timeout to extraction service | Catch `ResourceAccessException` / timeout; `run_status=FAILED`; increment `retry_count`; **fail** external task (Camunda retries if retries remain) |
| HTTP 422 / validation error from extraction | Permanent failure — no retry; `upload_status=FAILED` |
| HTTP 200 but DB save fails | Do not complete/correlate; throw → Camunda retry |
| Success within budget | Persist run, complete task, correlate `IDPExtractionCompleted` |

#### Load balancer / API gateway (if any)

Ensure any reverse proxy between gateway and extraction service has **idle/read timeout ≥ 900 s** (15 min). A 60 s default proxy timeout will break sync extract even when app timeouts are correct.

---

## 8. Frontend implementation strategy

Align with [IDP_UX_Design.md](./IDP_UX_Design.md) §8.

### 8.1 Components

| Component | Responsibility |
|-----------|----------------|
| `IdPaymentsLandingPage` | Add `TabBar`; wrap existing dashboard |
| `InstructionUploadTab` | Upload form + table |
| `UploadForm` | Country (default ID), metadata dropdowns, file picker |
| `UploadsTable` | Recent-first, disabled rows when processing |
| `UploadDetailModal` | Field grids, role/status actions |
| `useUploadsTable` | Fetch, 5s poll when processing rows exist |

### 8.2 URL state

| Query | Behavior |
|-------|----------|
| `tab=instruction-upload` | Active upload tab |
| `uploadId={id}` | Open modal on load |
| default / `tab=dashboard` | Existing dashboard |

### 8.3 API integration map

| Interaction | API |
|-------------|-----|
| Tab load | `GET /v1/idp/uploads?sort=recent&country=ID` |
| Upload | `POST /v1/idp/uploads` |
| Poll | `GET /v1/idp/uploads?sort=recent` every 5s if processing |
| Modal | `GET /v1/idp/uploads/{id}` |
| Save draft | `PATCH /v1/idp/uploads/{id}/fields` |
| Submit / Approve / Reject | `POST /v1/workflow/complete` |
| Re-extract | `POST /v1/idp/uploads/{id}/re-extract` |

### 8.4 Role matrix (modal buttons)

See [IDP_UX_Design.md](./IDP_UX_Design.md) §5 — implement via `ActionBar` driven by `uploadStatus` + `paymentMaker` / `paymentChecker`.

---

## 9. Two maker-checker cycles

| Stage | Process | Tasks | Purpose |
|-------|---------|-------|---------|
| Extraction QC | `IDP_Document_Ingestion` | `IDP_MakerReview`, `IDP_CheckerReview` | Validate OCR/LLM fields |
| Payment QC | `IAP_ID_Payments` (etc.) | `IAP_ID_MakerPayment`, `IAP_ID_CheckerPayment` | Banking repair / approval |

After IDP checker approves:

- Handoff sets `isApproved=true`.
- `PaymentValidationGateway` may skip payment checker when `isRepaired=false` and `isApproved=true` (existing `Flow_12mndij`).
- `TO_BE_REPAIR` still routes to payment maker — no new logic.

**Do not** reuse `IAP_ID_MakerPayment` for extraction review.

---

## 10. Deployment plan

| Phase | Deliverable | Prod risk |
|-------|-------------|-----------|
| P1 | DDL `fss_idp_*` | None |
| P2 | `IDP_Document_Ingestion.bpmn` + mock extraction service | None on IAP |
| P3 | Gateway IDP handlers (`Store`, `Trigger_Extraction`, `Trigger_Payment` with registry) | None until IDP started |
| P4 | Additive `IAP_ID_Payments.bpmn` v(N+1) + `IAPIDPInitializeHandler` | Regression on old paths |
| P5 | Real OCR + LLM | IDP path only |
| P6 | UI tab behind feature flag | Controlled rollout |
| P7 | Canary E2E ID upload → payment | New path only |

**Order:** tables → IDP BPMN + extraction service → payment BPMN additive + init handler → enable UI flag.

**Rollback:** disable feature flag; automated/bulk/manual IAP paths unaffected.

---

## 11. Test strategy

### 11.1 Regression (IAP v(N+1) before prod)

| # | Scenario | Expected |
|---|----------|----------|
| 1 | Automated JMS | `Initialize_IAP_Payments` — unchanged |
| 2 | Bulk upload ID | Bulk init — unchanged |
| 3 | Manual start | `IAP_ID_MakerPayment` — unchanged |
| 4 | IDP upload → approve → handoff | `Initialize_IAP_From_IDP` → enrichment |
| 5 | IDP double approve | Single payment instance |
| 6 | IDP handoff failure | No duplicate payment; retriable task |
| 7 | Wrong country in registry | Clear error; no correlate |

### 11.2 IDP-specific

| # | Scenario | Expected |
|---|----------|----------|
| 1 | Upload PDF | Row `PROCESSING`, workflow started |
| 2 | Extraction complete | `READY_FOR_REVIEW`, row clickable |
| 3 | Maker submit | `SUBMITTED` |
| 4 | Checker reject | `REJECTED`, back to maker |
| 5 | Re-extract | New run, old `is_latest=false` |
| 6 | Cancel upload | `CANCELLED` |
| 7 | Modal deep link | `?tab=instruction-upload&uploadId=` |

---

## 12. Risk register

| Risk | Mitigation |
|------|------------|
| Editing shared IAP gateway conditions | Additive-only BPMN diff; code review |
| Message name collision | Country-prefixed names: `IAP_ID_IDP_Trigger`, `CN_ETF_IDP_Trigger` |
| Duplicate payment instances | `payment_workflow_key` idempotency |
| Registry misconfiguration | Startup validation; integration test per route |
| Two Cockpit instances | Shared `businessKey=idpUploadId`; link columns on upload row |
| Deploy payment BPMN without init worker | Deploy workers before UI flag |
| PII in CLOB columns | Retention policy; mask in non-prod |

---

## 13. Open questions

| # | Question | Owner |
|---|----------|-------|
| 1 | OCR API contract (auth, callback, max size) | IDP / integration |
| 2 | LLM hosting and prompt versioning | IDP |
| 3 | Final upload tab label ([IDP_UX_Design.md](./IDP_UX_Design.md) §2.1.1) | UX |
| 4 | Separate `entity` dropdown vs infer from metadata | UX / product |
| 5 | Route config: YAML vs `fss_idp_payment_route` table | Engineering |
| 6 | SSTM feedback for `channel=IDP_UPLOAD` | Payments |
| 7 | PDF preview in modal v1? | UX |

---

## 14. Document history

| Date | Change |
|------|--------|
| 2026-07-30 | Initial LLD — common IDP BPMN, per-country payment triggers, tables, code strategy |
| 2026-08-02 | Orchestration boundary; Phase 1 sync extract; resilience §7.6; timeout hierarchy §7.8 |
| 2026-08-02 | Routing §4.0; consolidated into `final/` pack |

# FSS Payments Document Ingestion — Final Documentation Pack

> **Status:** Final — consolidated for implementation and review  
> **Created:** 2026-08-02  
> **Updated:** 2026-08-04 (v4 — FSS rename, ZIP bulk upload, multi-instruction fan-out; v6 — gateway façade for maker/checker actions)  
> **Supersedes:** `draft/` working copies and `IDP_LLD_latest.md` (obsolete — safe to delete). Use **`final/`** as source of truth.

---

## Canonical documents — safe to delete

**For implementation, use only the files in `final/` listed below.** Everything else in this repo for this feature is historical.

### Use these (source of truth)

| Purpose | **Canonical file** | Notes |
|---------|-------------------|--------|
| **Low Level Design (backend/DB)** | **[IDP_LLD.md](./IDP_LLD.md)** | Schema, handlers, gateway façade, DDL, tests |
| Architecture & BPMN | [IDP_Document_Ingestion_Design.md](./IDP_Document_Ingestion_Design.md) | Two-BPMN strategy, ZIP, multi-instruction handoff |
| REST APIs | [IDP_API_Reference.md](./IDP_API_Reference.md) | Upload + user review action endpoints |
| Portal UX | [IDP_UX_Design.md](./IDP_UX_Design.md) | Tabs, table, modal, button → API map |
| BPMN deploy artifact | [FSSPaymentsDocIngestion.bpmn](./FSSPaymentsDocIngestion.bpmn) | Process id `FSS_Payments_Document_Ingestion` |
| Start here | [README.md](./README.md) | Index and reading order |

### Safe to delete (obsolete)

| Path | Superseded by |
|------|----------------|
| **`final/IDP_LLD_latest.md`** | **`final/IDP_LLD.md`** |
| **`draft/`** (entire folder) | **`final/`** |

---

## Document set

| # | Document | Audience | Purpose |
|---|----------|----------|---------|
| 1 | [IDP_Document_Ingestion_Design.md](./IDP_Document_Ingestion_Design.md) | Architects, tech leads, BPMN reviewers | Production-safe two-BPMN strategy, ZIP upload, multi-instruction handoff, deployment & regression plan |
| 2 | [IDP_LLD.md](./IDP_LLD.md) | Backend, DB, integration engineers | **Implementation guide** — patterns, interfaces, Camunda handlers, DDL |
| 3 | [IDP_UX_Design.md](./IDP_UX_Design.md) | Frontend, UX, product | Landing-page tabs, upload table, detail modal, role-based actions |
| 4 | [IDP_API_Reference.md](./IDP_API_Reference.md) | Frontend, backend, integration | **All REST APIs** — request/response shapes, UI action map, extraction service contract |

### BPMN artifact (in this folder)

| File | Description |
|------|-------------|
| [FSSPaymentsDocIngestion.bpmn](./FSSPaymentsDocIngestion.bpmn) | Common country-agnostic document ingestion process — deploy to `51786-workflow-management` |

### Related repo artifacts (outside `final/`)

| Path | Description |
|------|-------------|
| [IDP_LLD_REPO_SCAN_PROMPT.md](./IDP_LLD_REPO_SCAN_PROMPT.md) | **Run in active code workspace** before implementation — discovers reusable classes |
| [../ID_payments.bpmn](../ID_payments.bpmn) | Indonesia payment process — **additive** change: `IAP_ID_Extraction_Trigger` + `Initialize_IAP_From_Extraction` |
| [../ID_payments.md](../ID_payments.md) | Existing IAP_ID_Payments workflow reference |
| [../after_ocr-llm-output.json](../after_ocr-llm-output.json) | Reference extraction JSON (single instruction; production uses `initiationDetail[]`) |
| [../51786-idp-extraction-service/](../51786-idp-extraction-service/) | Built extraction service (`POST /v1/extract`, mock mode) |

---

## Recommended reading order

0. **Repo scan** — [IDP_LLD_REPO_SCAN_PROMPT.md](./IDP_LLD_REPO_SCAN_PROMPT.md) in active code workspace; fill LLD §13.2
1. **Design** §1–§2 — decisions and architecture
2. **UX** §2–§5 — what users see and do (PDF + ZIP upload, multi-instruction modal)
3. **LLD** §2–§6 — services, BPMN task map, DDL, structured output contract
4. **[API Reference](./IDP_API_Reference.md)** — endpoints, payloads, UI → API map
5. **LLD** §4 — `ExtractionPaymentRouteRegistry` (entity handoff)
6. **Design** §14–§16 — regression matrix and implementation checklist

---

## Architecture at a glance

```mermaid
flowchart LR
    UI["Portal — Instruction Upload tab"]
    GW["payment-gateway-service"]
    EXT["idp-extraction-service"]
    WFM["Camunda"]
    PAY["Entity payment BPMN e.g. IAP_ID_Payments"]

    UI -->|POST extraction-uploads PDF/ZIP| GW
    GW -->|startWorkflow per PDF| WFM
    WFM -->|Trigger_Data_Extraction| GW
    GW -->|POST /v1/extract sync| EXT
    EXT -->|initiationDetail[] only| GW
    GW -->|merge header + complete task| WFM
    UI -->|performAction SUBMIT/CANCEL| GW
    GW -->|setTaskDetails + completeCurrentTask| WFM
    WFM -->|Trigger_Payment_From_Extraction| GW
    GW -->|N × startMessageCorrelation per instruction| WFM
    WFM --> PAY
```

### Key design rules

| Rule | Detail |
|------|--------|
| One ingestion BPMN for all countries | `FSS_Payments_Document_Ingestion` — upload, OCR/LLM, single user review |
| Entity routing **not** in ingestion BPMN | `Trigger_Payment_From_Extraction` fans out via `ExtractionPaymentRouteRegistry` keyed by `entity` |
| Entry points in entity payment BPMNs | e.g. `IAP_ID_Extraction_Trigger` on `IAP_ID_Payments.bpmn` when `entity=ID` |
| ZIP bulk upload | One upload row + workflow **per PDF** inside the archive; non-PDF entries ignored |
| Multi-instruction PDF | LLM returns `initiationDetail[]`; gateway merges `header`; **one payment trigger per instruction** |
| Extraction service is domain-agnostic | No Camunda, no payment tables — gateway owns workflow |
| User review actions via gateway façade | Portal calls `POST .../performAction` (`SUBMIT` / `CANCEL`); optional `/submit`/`/cancel` aliases — gateway updates DB + `fss_payment_upload_audit`, then WFM server-side |
| Phase 1 = sync extract | Gateway blocks on `POST /v1/extract` (~10 min budget, 15 min HTTP envelope) |
| Phase 1 entity | Indonesia (`entity=ID`) — registry row only |

---

## Naming change (v3/v4)

| Legacy (v1) | Current (v4) |
|-------------|----------------|
| `IDP_Document_Ingestion` | `FSS_Payments_Document_Ingestion` |
| `IDP_Document_Ingestion.bpmn` | `FSSPaymentsDocIngestion.bpmn` |
| `fss_idp_*` / `fss_doc_extract_*` | `fss_payment_upload_*` + `fss_payment_data_ingest_details` |
| `IAP_ID_IDP_Trigger` | `IAP_ID_Extraction_Trigger` |
| `Initialize_IAP_From_IDP` | `Initialize_IAP_From_Extraction` |
| `channel=IDP_UPLOAD` | `channel=DOC_EXTRACTION` |
| `country` (upload routing) | **`entity`** — same value, single field (phase 1: `ID`) |
| BPMN message wait for extraction | `ExtractionResultGateway` on task completion |

---

## Phase 1 scope summary

| In scope | Out of scope (v1) |
|----------|-------------------|
| ID landing tab UX (upload + table + modal) | New sidebar nav route |
| Single PDF + ZIP bulk upload (PDFs only) | Non-PDF file types as direct upload |
| Multi-instruction PDF → N payment instances | China ETF handoff (registry stub only) |
| `fss_payment_upload_*` + `fss_payment_data_ingest_details` + `fss_payment_upload_audit` (`retry`, `error_desc` on details) | `document-service` integration |
| `51786-idp-extraction-service` | SSTM feedback for `DOC_EXTRACTION` channel |
| Additive `IAP_ID_Payments.bpmn` change | |

---

## Implementation status (high level)

| Component | Status |
|-----------|--------|
| `51786-idp-extraction-service` | Built (sync `/v1/extract`, mock mode, `id-payment-v1` template) |
| `FSSPaymentsDocIngestion.bpmn` | In `final/` — ready for workflow-management |
| `IAP_ID_Payments.bpmn` additive diff | In repo root `ID_payments.bpmn` |
| `51786-payment-gateway-service` | Planned — upload APIs + `ExtractionUploadActionService` (user review façade) documented in LLD |
| Portal UI | Specified in UX doc — not built |

---

## Document history

| Date | Change |
|------|--------|
| 2026-07-30 | Initial draft pack in `draft/` |
| 2026-08-02 | Routing responsibilities, registry model, sync extract, timeouts |
| 2026-08-02 | **Final pack** — consolidated into `final/` with README index |
| 2026-08-02 | Added [IDP_API_Reference.md](./IDP_API_Reference.md) |
| 2026-08-05 | **v7** — Entity-only routing synced across API Reference + Design + UX |
| 2026-08-04 | **v5** — Three-table schema; `retry` + `error_desc` on ingest details |

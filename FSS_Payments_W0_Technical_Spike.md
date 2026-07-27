# W0 — Foundation Technical Spike Checklist

> **Program:** FSS Payments Platform Re-Architecture  
> **Wave:** W0 — Foundation (8–10 weeks indicative, no production traffic)  
> **Parent:** [FSS_Payments_Target_Architecture.md](./FSS_Payments_Target_Architecture.md) §18  
> **Stakeholder summary:** [FSS_Payments_Executive_Summary.md](./FSS_Payments_Executive_Summary.md)

**W0 goal:** Prove the platform skeleton end-to-end in a **non-prod environment** — upload PDF → Python extraction → event → process-worker starts Camunda → stub save — with observability, security, and schema in place for W1 Indonesia IAP.

---

## 1. Spike exit criteria (definition of done)

| # | Criterion | Verified by |
|---|---|---|
| E1 | PDF uploaded via BFF/ingestion → stored in document-platform | Demo + integration test |
| E2 | Python intelligence completes job → callback → ingestion → `PaymentInstructionExtracted` event | Event trace + DB row |
| E3 | process-worker consumes event → starts `PaymentInstruction_ID_IAP` (skeleton BPMN) on **workflow-payments** | Camunda Cockpit |
| E4 | Generic `InitializeInstruction` + `SavePayment` handlers complete (stub or real core API) | Worker logs + core DB |
| E5 | `instruction_id` visible in traces across all hops (OpenTelemetry) | Trace UI |
| E6 | Dual Camunda engines deployed: **workflow-payments** + **workflow-admin** (admin may be empty BPMN) | Deploy checklist |
| E7 | No Camunda client in ingestion service dependency tree | CI dependency check |
| E8 | Architecture decision record (ADR) for event bus choice signed | ADR doc |
| E9 | W1 backlog groomed with dependencies marked ready | Jira/ADO |

---

## 2. Week-by-week spike plan (suggested)

| Week | Focus |
|---|---|
| **1–2** | ADRs, repo scaffolding, CI/CD templates, schema v1, local dev stack |
| **3–4** | document-platform + ingestion upload path; intelligence Python API + worker |
| **5–6** | Event bus wiring; process-worker skeleton; workflow-payments deploy |
| **7–8** | BFF stub; E2E demo; observability; security (mTLS/HMAC); perf smoke |
| **9–10** | Hardening, documentation, W1 handoff, spike retrospective |

---

## 3. Architecture decision records (complete in W0)

| ADR | Question | Options | Owner | Due |
|---|---|---|---|---|
| ADR-001 | Event transport | Solace (extend) vs Kafka (new) | Platform architect | Week 1 |
| ADR-002 | Intelligence job queue | Celery+Redis vs RQ vs Solace pull | ML lead | Week 1 |
| ADR-003 | process-worker packaging | Single repo multi-module vs mono-module | Tech lead | Week 2 |
| ADR-004 | Schema migration tool | Flyway per service vs shared migration svc | DBA | Week 2 |
| ADR-005 | BFF auth | Existing enterprise IdP integration pattern | Security | Week 3 |
| ADR-006 | document-platform | Extend 51786-document-service vs new repo | Platform | Week 2 |
| ADR-007 | Parallel-run strategy (W1) | Shadow vs percentage router | Product + platform | Week 8 |

---

## 4. Repository & service scaffolding

| Task | Service / artifact | Checklist |
|---|---|---|
| Create repo `fss-payment-ingestion-service` | Java 17+, Spring Boot 3 | [ ] parent POM [ ] health endpoint [ ] Dockerfile [ ] Helm chart skeleton |
| Create repo `fss-payment-document-intelligence` | Python 3.11+, FastAPI | [ ] pyproject [ ] Dockerfile [ ] worker process [ ] lint/test CI |
| Create repo `fss-payment-process-worker` | Java, camunda-bpm-client | [ ] external task client config [ ] plugin SPI interface |
| Create repo `fss-payment-core-service` | Java | [ ] `pmt_payment` entity [ ] save/get API |
| Create repo `fss-payment-experience-api` | Java BFF | [ ] upload proxy to ingestion |
| Extend or fork `fss-document-platform-service` | Java | [ ] upload/download API contract |
| Create `fss-workflow-payments` deploy | Camunda 7 | [ ] engine-rest [ ] separate DB schema `act_pay_*` |
| Create `fss-workflow-admin` deploy | Camunda 7 | [ ] separate DB schema `act_adm_*` |
| Shared `fss-payment-bpmn-lib` | BPMN only | [ ] `SP_MakerChecker` placeholder subprocess |

---

## 5. Database & schema (W0 scope)

| Task | Detail | Checklist |
|---|---|---|
| DDL `pmt_instruction` | See target arch §7.2 | [ ] migration script [ ] indexes |
| DDL `pmt_instruction_event` | Append-only | [ ] migration script |
| DDL `pmt_payment` | Core aggregate | [ ] migration script [ ] version column |
| DDL `pmt_channel_route` | Seed ID/IAP/DOCUMENT_UI row | [ ] seed data |
| DDL `pmt_extraction_profile` | Seed `ID_IAP` + `pipeline_id` | [ ] seed data |
| DDL `doc_intel_processing_job` | Python service (optional) | [ ] migration |
| **No prod migration yet** | W0 uses dedicated dev DB only | [ ] |

---

## 6. `payment-ingestion-service` spike tasks

| Task | Checklist |
|---|---|
| `POST /internal` channel API (or public behind BFF only) | [ ] |
| Multipart upload validation (size, PDF mime) | [ ] |
| document-platform client integration | [ ] |
| Persist `pmt_instruction` lifecycle | [ ] |
| Submit extraction job to Python service | [ ] |
| `POST /v1/channels/extraction/callback` with HMAC validation | [ ] |
| Publish `PaymentInstructionExtracted` to bus | [ ] |
| Write `pmt_instruction_event` on each transition | [ ] |
| Feature flag: `payments.ingestion.enabled` | [ ] |
| **Verify:** no `camunda` / `workflow` dependencies in `pom.xml` | [ ] |

---

## 7. `payment-document-intelligence` spike tasks

| Task | Checklist |
|---|---|
| `POST /v1/extraction/jobs` (202 Accepted) | [ ] |
| Celery/RQ worker consumes job | [ ] |
| Fetch PDF from document-platform by `documentId` | [ ] |
| Pipeline `id_iap_stub_v0` — rule-based or minimal OCR stub | [ ] |
| Callback to ingestion with `extractedFields` JSON | [ ] |
| Persist `doc_intel_processing_job` | [ ] |
| Include `modelVersion` in callback | [ ] |
| Timeout + retry policy documented | [ ] |
| Contract test: ingestion ↔ intelligence (Pact or wiremock) | [ ] |

---

## 8. Event bus spike tasks

| Task | Checklist |
|---|---|
| Topic / queue names defined (see §9) | [ ] |
| Publisher in ingestion (after extraction complete) | [ ] |
| Consumer in process-worker | [ ] |
| Dead-letter queue + replay procedure doc | [ ] |
| Message schema JSON Schema or Avro registered | [ ] |
| Idempotency: consumer dedup by `instruction_id` | [ ] |

---

## 9. Event catalog (W0 minimum)

| Event | Schema version | Topic / queue name (example) |
|---|---|---|
| `PaymentInstructionExtracted` | `1.0` | `payments/instruction/extracted` |

**Payload minimum fields:** `instructionId`, `country`, `product`, `channel`, `documentId`, `extractedPayload` (or ref), `correlationTimestamp`, `schemaVersion`.

---

## 10. `payment-process-worker` spike tasks

| Task | Checklist |
|---|---|
| Connect to **workflow-payments** `engine-rest` only | [ ] |
| Subscribe: `InitializeInstruction`, `SavePayment` (generic topics) | [ ] |
| `PluginRegistry` loads `ID_IAP` initializer stub | [ ] |
| On `PaymentInstructionExtracted`: `startProcess(PaymentInstruction_ID_IAP)` | [ ] |
| `SavePaymentHandler` calls core-service API | [ ] |
| Shared `ExternalTaskHelper` / retry utility (no copy-paste per handler) | [ ] |
| Safe default variables on retry exhaustion | [ ] |
| Read `pmt_instruction` for extracted payload | [ ] |

---

## 11. `workflow-payments` BPMN spike

| Artifact | Checklist |
|---|---|
| `PaymentInstruction_ID_IAP.bpmn` skeleton | [ ] Start → InitializeInstruction → SavePayment → End |
| Deploy to workflow-payments engine | [ ] |
| Message start or API start from worker documented | [ ] |
| Process variables: `instructionId`, `messageId`, `country`, `product`, `isApproved=false` | [ ] |

---

## 12. `payment-core-service` spike

| Task | Checklist |
|---|---|
| `POST /v1/payments` — create from instruction | [ ] |
| `GET /v1/payments/{paymentId}` | [ ] |
| Link `instruction_id` on payment row | [ ] |
| Optimistic locking on update | [ ] |
| OpenAPI spec published to artifact repo | [ ] |

---

## 13. `payment-experience-api` spike (minimal)

| Task | Checklist |
|---|---|
| `POST /payments/instructions/upload` → ingestion | [ ] |
| `GET /payments/instructions/{id}` aggregated status | [ ] |
| Auth integration stub (or dev bypass documented) | [ ] |
| No direct calls to process-worker or intelligence | [ ] |

---

## 14. Observability & security

| Task | Checklist |
|---|---|
| OpenTelemetry agent on Java services | [ ] |
| `instruction_id` as trace baggage | [ ] |
| Structured logging JSON format | [ ] |
| mTLS between services (dev certs) | [ ] |
| HMAC on intelligence → ingestion callback | [ ] |
| Secrets via vault / K8s secrets (no secrets in git) | [ ] |
| Dashboard: instruction status counts by state | [ ] |

---

## 15. Local developer experience

| Task | Checklist |
|---|---|
| `docker-compose` or Tilt: ingestion, intelligence, worker, core, postgres, redis, solace/kafka | [ ] |
| One-command E2E script: upload sample PDF → wait → verify payment row | [ ] |
| Sample PDF fixtures for ID IAP in repo | [ ] |
| README per service with run instructions | [ ] |

---

## 16. Legacy discovery (parallel work — unblocks W1–W2)

| Discovery item | Why | Checklist |
|---|---|---|
| Checkout / locate S2B service source | W1 file-service | [ ] repo URL [ ] owner |
| Confirm `S2BFileGenerated` correlation producer today | File workflow | [ ] doc |
| Map all `processDefinitionKeyIn` in gateway/publisher | Migration scope | [ ] spreadsheet |
| SWOOSH field mapping for ID IAP | Intelligence pipeline W1 | [ ] field list |
| Camunda Tasklist groups for ID maker/checker | W1 UI | [ ] group names |
| Solace topic names for ID automated trigger | W2 | [ ] topic list |

---

## 17. W0 demo script (acceptance demo)

1. Operator (or script) uploads `sample_id_iap.pdf` via BFF with `country=ID`, `product=IAP`.
2. BFF returns `instructionId`; status `EXTRACTION_SUBMITTED`.
3. Within SLA, status becomes `EXTRACTION_COMPLETE` (intelligence callback).
4. Event consumed; Camunda instance visible in Cockpit for `PaymentInstruction_ID_IAP`.
5. External tasks complete; `pmt_payment` row exists with linked `instruction_id`.
6. Distributed trace shows single trace ID across BFF → ingestion → intelligence → worker → core.
7. `pmt_instruction_event` contains ≥4 events: `DOCUMENT_UPLOADED`, `EXTRACTION_COMPLETED`, `WORKFLOW_STARTED`, `PAYMENT_SAVED`.

---

## 18. W1 readiness gate (handoff from W0)

Before starting W1 Indonesia IAP production path:

| Gate | Owner |
|---|---|
| All E1–E9 exit criteria met | Platform lead |
| ADR-001 through ADR-006 approved | Architecture board |
| Security review of callback + mTLS | InfoSec |
| Performance baseline: extraction p95 documented | ML lead |
| `id_iap_v1` pipeline spec signed (fields + confidence rules) | Product + country team |
| BPMN `SP_MakerChecker` subprocess drafted | Workflow lead |
| Parallel-run approach agreed (ADR-007) | Product |
| W1 backlog estimated with team capacity | PM |

---

## 19. Roles & RACI (W0)

| Area | Responsible | Accountable | Consulted | Informed |
|---|---|---|---|---|
| Platform scaffolding | Platform squad | Tech lead | Country teams | Sponsors |
| Python intelligence | ML squad | ML lead | Product | Platform |
| Camunda / BPMN | Workflow lead | Architect | Country teams | Ops |
| Schema & DBA | DBA | DBA lead | Service owners | — |
| Event bus | Platform architect | Architect | Enterprise messaging | All squads |
| Security | InfoSec | CISO delegate | Platform | Sponsors |

---

## 20. Reference links

| Document | Purpose |
|---|---|
| [FSS_Payments_Target_Architecture.md](./FSS_Payments_Target_Architecture.md) | Full target design |
| [ID_payments.md](./ID_payments.md) | Legacy IAP_ID behavior reference |
| [ID_payments_UI_OCR_Rearchitecture.md](./ID_payments_UI_OCR_Rearchitecture.md) | Incremental design (superseded for target) |

---

*Last updated: aligned with target architecture W0 wave. Adjust week estimates per team size (recommended W0 core team: 6–8 engineers + 1 ML + 1 workflow + 1 DBA part-time).*

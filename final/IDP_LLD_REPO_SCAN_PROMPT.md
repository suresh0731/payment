# Repo scan prompt — ESS Payments Document Ingestion

Use this prompt in a workspace that contains the real Java repos (`51786-payment-gateway-service`, `51786-workflow-management`, `51786-idp-extraction-service`, etc.).

---

## Prompt (copy below this line)

```
You are implementing ESS Payments Document Ingestion per the specification in `final/IDP_LLD.md` (v7 implementation guide).

**Before writing any new code**, scan the repositories and produce a **Reuse & alignment report** with concrete file paths, class names, and code citations.

### 1. Scan targets (search each repo)

| What to find | Search hints | Used for |
|--------------|--------------|----------|
| External task handlers | `@ExternalTaskSubscription`, `ExternalTaskHandler`, `ExternalTaskHelper` | `ExtractionTriggerHandler`, `ExtractionPaymentHandoffHandler`, `IAPExtractionInitializeHandler` |
| Workflow client | `WorkflowService`, `startWorkflowProcess`, `setTaskDetails`, `completeCurrentTask`, `startMessageCorrelation` | Upload start, action façade, handoff |
| Message / payment persistence | `MessageService`, `fss_services_message`, `DOC_EXTRACTION`, `IAPIDPaymentsMessageHandler` | Handoff before correlation |
| Country/entity factories | `CountrySpecificProcessorFactory`, `CountryExcelMsgProcessorFactory`, `*Factory`, `*Registry` | `ExtractionPaymentRouteRegistry`, `EntityHandlerRegistry` pattern |
| IAP ID handlers | `IAPIDPaymentTransactionHandler`, `IAPPaymentBulkTransactionHandler`, `Initialize_IAP` | Package placement, handler base class |
| Bulk upload controller | `BulkUploadController`, `retryTransaction`, `@RequestParam` id style | `ExtractionUploadController` URL conventions |
| Extraction service POJOs | `InitiationDetail`, `ExtractedField`, `IdPaymentOutput`, `POST /v1/extract` | Jackson models — **reuse, do not duplicate** |
| Jackson config | `ObjectMapper`, `@JsonProperty`, existing DTO packages | `ExtractedDataSerializer` / deserialization of `extracted_data` CLOB |
| JPA entities/repos | `*Entity`, `*Repository` in payment-gateway | New `ingestion` package layout |
| application.yml routing | `country`, `entity`, processor maps, list-of-objects config | `extraction.payment-routes`, `extraction.entity-templates` |
| Camunda BPMN deploy path | `src/main/resources/*.bpmn`, `IAP_ID_Payments` | Deploy `ESS_Payments_Document_Ingestion.bpmn` |

### 2. Report format

For each row in IDP_LLD.md §9.4 (interface/class inventory), output:

```markdown
### <ClassName>
- **Reuse:** [existing class path] OR **New**
- **Pattern match:** [similar class] — cite 5–15 lines showing the pattern to copy
- **Package:** recommended package based on neighbors
- **Gaps:** anything the LLD assumes that does not exist in repo
```

### 3. Validation checks

- Confirm `entity` is the only routing key (no separate `country` column in new DDL).
- Confirm extraction service returns `initiationDetail[]` and which Jackson types deserialize it.
- List Java version in parent POM (8 / 11 / 17) and which modern features are already used (`record`, `Optional`, `var`, `Map.of`, etc.).
- Identify the **base class or helper** used by existing IAP external task handlers (retries, complete/fail, variable read).

### 4. Output deliverables

1. Reuse & alignment report (markdown)
2. Updated package tree proposal aligned to real repo structure
3. List of LLD sections that need correction after scan (if any)
4. Do **not** implement yet unless I ask — discovery only

Reference spec: `final/IDP_LLD.md` sections 2.4, 4, 6, 9, 13.
```

---

## After the scan

1. Paste the **Reuse & alignment report** back into `IDP_LLD.md` §13.2 (or attach as `IDP_LLD_REPO_FINDINGS.md`).
2. Update §9.4 class inventory with real package paths from the scan.
3. Implement following §9 in order: config → entities/repos → POJOs → registry → handlers → controller.

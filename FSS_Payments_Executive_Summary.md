# FSS Payments Platform Re-Architecture — Executive Summary

> **Audience:** Sponsors, product owners, architecture review boards  
> **Detail:** [FSS_Payments_Target_Architecture.md](./FSS_Payments_Target_Architecture.md) · **W0 plan:** [FSS_Payments_W0_Technical_Spike.md](./FSS_Payments_W0_Technical_Spike.md)

---

## The problem

The payments platform grew through layered additions: Solace queues, country-specific workflows, and two overloaded Java services (`payment-gateway-service` and `payment-publisher-service`) that mix ingestion, Camunda workers, and outbound messaging. OCR for payment documents depends on legacy SWOOSH data embedded in queue messages. A new UI for PDF upload and modern AI extraction cannot be added cleanly without increasing operational risk and long-term maintenance cost.

**Symptoms:** hard to add countries, confusing service names, one Camunda engine for payments and admin approvals, missing S2B file-generation ownership, duplicated database access across services.

---

## The proposal

Replace the legacy stack with a **purpose-built platform** organized by business capability — not by historical module names. Accept a **big-bang target architecture** delivered in **controlled waves**, with a defined end date for retiring gateway, publisher, and legacy BPMNs.

### Target capabilities (what leadership gets)

| Capability | Outcome |
|---|---|
| **Document-first payments** | Operators upload PDFs; a **Python AI service** extracts fields; auditable end-to-end |
| **Unified ingress** | One service for UI, Solace, bulk, and manual channels — config-driven routing |
| **Clear operations** | Payments workflow engine **separated** from admin approval workflows |
| **Extensibility** | New country/product = configuration + plugins, not a new microservice per region |
| **Owned file generation** | Dedicated file service closes the S2B gap |
| **Single worker fleet** | One Camunda worker service with plugins replaces scattered gateway/publisher handlers |

### New service landscape (high level)

```text
UI → Experience API (BFF)
        → Ingestion Service (all channels)
        → Document Intelligence (Python OCR/LLM)
        → Process Worker (Camunda tasks)
        → Core Service (payments database)
        → File Service (bank files) + Outbound Adapter (Solace feedback)
```

**Retired after migration:** `payment-gateway-service`, `payment-publisher-service`, legacy `IAP_ID_Payments` / `ChinaETF_*` BPMNs, SWOOSH for new channels.

---

## Program timeline (indicative)

| Wave | Focus | Duration | Production impact |
|---|---|---|---|
| **W0** | Platform foundation (no prod traffic) | 8–10 weeks | None |
| **W1** | Indonesia IAP — document UI | 6–8 weeks | Parallel / pilot % |
| **W2** | Indonesia IAP — Solace path | 4–6 weeks | ID cutover |
| **W3–W4** | China ETF T+0 and T+N | 16–20 weeks | CN cutover |
| **W5** | Admin workflows to separate engine | 4 weeks | Low |
| **W6** | Decommission legacy | 4 weeks | Legacy off |

**Total program:** ~18–24 months depending on CN ETF complexity and parallel-run duration.

---

## Investment vs return

| Investment | Return |
|---|---|
| New services (ingestion, intelligence, worker, file, BFF) + dual workflow engines | Lower cost per new country; faster UI and AI features |
| Data migration (`instruction` + `payment` model) | Reliable audit trail; simpler support |
| Parallel run and reconciliation | Safer cutover than endless dual-stack |
| Python ML platform | Model upgrades without Java payment deploys |

---

## Risks and how we manage them

| Risk | Mitigation |
|---|---|
| Cutover regression | Shadow traffic + automated reconciliation before flip |
| Program scope creep | Thin product BPMNs + shared library; plugins per country |
| Operational complexity (two Camunda engines) | Separate namespaces; payment SLA on payments engine only |
| Cross-team delivery | Platform team owns framework; country teams own plugins |

---

## Decisions needed from stakeholders

1. **Approve big-bang target** with hard decommission of gateway/publisher (vs indefinite strangler).
2. **Confirm event bus** for W0 spike: extend Solace vs introduce Kafka.
3. **Prioritize W1** Indonesia document UI as first production value.
4. **Staffing:** platform squad (W0–W1) + country squads (plugins, UAT).
5. **S2B / file-generation:** commit to in-scope `payment-file-service` (checkout external S2B repo if exists).

---

## Success criteria (program exit)

- [ ] 100% Indonesia IAP traffic on new stack (document + Solace).
- [ ] China ETF T+0 and T+N on target architecture.
- [ ] Gateway and publisher **removed** from production.
- [ ] Admin approvals on `workflow-admin` only.
- [ ] Document UI live with Python extraction and full audit (`instruction_id` trace).
- [ ] No production dependency on SWOOSH for document-ui channel.

---

*For technical foundation work, see [FSS_Payments_W0_Technical_Spike.md](./FSS_Payments_W0_Technical_Spike.md).*

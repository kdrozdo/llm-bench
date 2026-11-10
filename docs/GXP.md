# Life sciences / GxP design

Status: draft · Builds on [DESIGN.md](DESIGN.md)

Target: pharma, biotech, CDMOs, and medical devices. Maintenance and calibration of GMP equipment, utilities (WFI, clean steam, HVAC/cleanroom), and lab instruments.

---

## 1. Regulatory frame

| Source | What it means for us |
|---|---|
| 21 CFR Part 11 / EU GMP Annex 11 | Audit trail, e-signatures with meaning, record integrity, access control, readable exports |
| EU GMP Annex 15 | Qualification status (IQ/OQ/PQ), requalification, change control on qualified equipment |
| GAMP 5 (2nd ed.) incl. AI/ML appendix | Risk-based validation, supplier involvement, configuration (Cat 4) instead of customization |
| FDA Computer Software Assurance (CSA) | Assurance effort scaled to risk; critical thinking over scripted testing. We supply the evidence |
| EU GMP **Annex 22 (AI)**, draft published 2025 | Draft position: critical GMP uses limited to static, deterministic models; generative/LLM use in non-critical roles only, with a human in the loop. **Verify current status before relying on it** |
| FDA draft guidance on AI for regulatory decision-making (2025) | Risk-based credibility assessment of a model for its "context of use" |
| ALCOA+ | Every record attributable, legible, contemporaneous, original, accurate, complete, consistent, enduring, available |

**Design consequence:** the product must be able to show an auditor, for every AI function, its *GMP impact class*, its *context of use*, the *human control*, and the *evidence that it works*. We treat that as a product feature, not as paperwork.

## 2. AI classification rule

Every AI function is registered with a GMP impact class, and the policy engine enforces the cap:

| Class | Definition | Allowed AI role | Autonomy cap |
|---|---|---|---|
| **C: Critical** | Directly decides product quality, patient safety, or the content of a GMP record | None. Deterministic logic only (e.g. cal pass/fail = tolerance math) | n/a |
| **A: Assistive-GMP** | Drafts or checks GMP content that a qualified human reviews and signs | Draft, pre-review, flag, summarize | **L1 max**: human edits/approves with e-signature |
| **N: Non-GMP** | Non-GMP assets, scheduling, inventory, search, analytics | Full agent | Up to L3 per policy |

Hard rules:
- **Agents never sign.** Part 11 signatures are human only. An agent is an *actor* in the audit trail (agent ID + model version + prompt/config version), never a *signer*.
- **The signed record is what the human approved.** The AI draft, the human's diff, and the evidence links are kept as metadata on the record, so the record shows where content came from.
- **On GMP assets, failure codes and closeout fields are L1**, even where the same action is L2 on non-GMP assets.
- **Calibration results, OOT determination, qualification status, and validated state come from deterministic code only.**

## 3. Domain extensions (beyond the Maximo-style core)

- **GMP flag and qualification status** on Location/Asset: GMP/non-GMP, qualified state, requalification due, linked qualification documents. A WO on a GMP asset can trigger "impact to qualified state?" questions.
- **Calibration as a first-class module:**
  - instruments, test points, tolerances (process vs. manufacturer), standards used and their traceability
  - as-found/as-left results, automatic pass/OOT
  - **OOT impact assessment**: which batches, tests, or products used the instrument since its last good calibration. The agent assembles the evidence; QA decides
- **Electronic equipment logbooks** (use, cleaning, maintenance, status). These are paper in most plants and are a constant source of inspection findings.
- **Controlled master data:** GMP job plans, PMs, cal procedures, and tolerances are versioned, have effective dates, and change only through change control with e-signature.
- **QMS linkage:** deviation, CAPA, and change-control references on WOs. Integration with Veeva Vault QMS, TrackWise Digital, MasterControl. Agents *draft* a deviation initiation; the QMS stays the system of record for quality events.
- **Environmental & utilities signals:** EMS/BMS alarms (cleanroom ΔP, temperature/RH, WFI conductivity/TOC) as condition triggers.

## 4. Where AI earns its keep in life sciences

| Function | Class | Value |
|---|---|---|
| **GMP WO pre-review**: completeness, GDP errors, missing readings/signatures, out-of-range values, "this needs a deviation/change control?" | A | Large drop in QA review cycle time and rejected-WO loops |
| Closeout copilot (voice/photo → structured record) | A (GMP) / N | Contemporaneous, complete records (the "C" in ALCOA+) |
| OOT impact assessment evidence pack | A | Days of manual tracing reduced to minutes |
| Audit/inspection readiness pack per asset/system | A | Instant "show me everything on this autoclave since last requal" |
| PM/cal interval optimization proposals | A → feeds change control | Evidence-based interval extension, which today is mostly avoided due to effort |
| Job plan / SOP drafting from OEM manuals | A | Faster master-data creation, but always through change control |
| Scheduling around campaigns, cleaning, and room status | N | Maintenance windows aligned with production campaigns |
| Spares / storeroom | N | Critical spares for GMP-critical assets |

## 5. Revised wedge

**GMP work-order pre-review plus a closeout copilot.**

- Reading the incumbent CMMS (Maximo, SAP PM, Blue Mountain RAM) and *flagging* issues is low risk and low validation burden. We don't write to the validated system of record at first.
- It targets a bottleneck everyone in pharma maintenance knows: QA reviewing completed GMP WOs.
- Measurable: QA review cycle time, right-first-time documentation rate, inspection findings related to maintenance records.
- After that, the closeout copilot raises right-first-time at the source, and calibration and OOT impact assessment follow as the next module.

## 6. What we ship to customers (supplier validation package)

- **Vendor QMS**: ISO 9001-aligned SDLC, plus SOC 2 Type II / ISO 27001, ready for supplier audits.
- **Per-release validation evidence, generated from CI**: requirements → risks → automated tests trace matrix, test execution records, release notes with GMP impact.
- **AI evidence per function**: context of use, impact class, eval dataset description, eval results for the pinned model version, known limitations.
- **Release trains:**
  - *Continuous*: non-GxP tenants
  - *Validated*: quarterly, with an N-1 supported version, change notes 30+ days ahead, and a customer UAT window
- **Model change is change control.** A model version is pinned per tenant/cell. A model upgrade is a release: the eval suite is re-run and the report becomes validation evidence. Nothing changes silently underneath a validated customer.

## 7. Data integrity mechanics

- Append-only event log, hash-chained per tenant, with periodic anchor exports. Covers audit trail completeness and tamper evidence.
- Every event records actor type (human / agent / integration), identity, timestamp from a trusted time source, reason, and before/after values.
- E-signature: re-authentication at signing (SSO step-up allowed), meaning (reviewed/approved/performed), signature shown on the record and in exports.
- Retention policies per record type; human-readable export (PDF/A + JSON) for inspections and decommissioning.

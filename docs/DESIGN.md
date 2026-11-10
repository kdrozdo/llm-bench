# AI-native CMMS: Design v0 (draft)

Status: draft for discussion · Audience: people who know Maximo-class EAM

---

## 1. Thesis

Maximo, SAP PM, Infor EAM and the rest are **systems of record built around forms**: WO, PM, Job Plan, Asset, Location, Item, PO. Their value depends on humans typing good data into those forms, and in practice that rarely happens:

- Failure codes default to `OTHER` / `NOCAUSE`. The real story is in the free-text long description, where reports can't use it.
- Actual labor and materials are entered from memory at the end of the shift, or never.
- Job plans are copied once and then go stale. PM frequencies are set by OEM manuals or habit and never revisited.
- Planners spend most of their time on clerical work: finding parts, chasing permits, re-typing scope.
- Reliability engineers export to Excel, because the data they need is there but isn't structured.

**An AI-native CMMS turns the record into a by-product of doing the work.** Agents do the clerical work and the structuring. Humans do the work and approve decisions. Deterministic engines (solvers, rules) handle anything that has to be exact.

It is not "Maximo plus a chatbot." The test: if you delete the LLM, the product's core loop no longer works.

## 2. Design principles

1. **Capture without typing.** Voice, photos, nameplate OCR, and sensor events are the main ways data gets in. Forms remain for review and correction.
2. **Agents propose; policy decides; humans own risk.** Every agent action is a typed *proposal* with evidence, a confidence score, and a risk class. A policy engine decides whether it auto-applies or needs approval. Think of it as Maximo signature security plus workflow, applied to AI actions.
3. **LLMs orchestrate; solvers optimize; rules guarantee.** Scheduling uses a constraint solver, not an LLM. Reorder points use inventory math. LOTO isolation lists come from verified data, never generated text.
4. **One domain API for everything.** The UI, mobile app, integrations, and agents all call the same typed commands, so there is no back-door write path for AI.
5. **Event-sourced history.** Every change is an event with an actor (human / agent / integration), a reason, and evidence links. That gives audit, regulatory traceability, and training data in one place.
6. **Familiar domain model.** Keep the concepts the industry already knows (Asset, Location, Classification, Job Plan, PM, WO, Failure Hierarchy, Item/Storeroom, Meter), so migration from Maximo is a mapping exercise and doesn't require re-education.
7. **Continuous evaluation.** Each agent has an eval suite built from labelled historical data, run on every model or prompt change. (This repo's name, `llm-bench`, fits.)

## 3. Core loops (where AI changes the work)

| Loop | Legacy CMMS | AI-native |
|---|---|---|
| **Intake** (SR → WO) | Requester fills a form, often with the wrong asset. A planner re-triages it. | Requester sends a photo or voice note or text. The agent identifies the asset (location, QR, nameplate, photo match), dedupes against open WOs, proposes priority and classification, and creates the WO. |
| **Planning** | Planner copies an old job plan and hand-picks parts. | The agent drafts scope, tasks, labor, materials, tools, and permits from asset history, similar WOs, OEM manuals, and job plan templates. It checks stock and lead times, then flags gaps. The planner edits and approves. |
| **Scheduling** | Spreadsheet or Gantt, by hand. Weekly freeze. | A solver (e.g. OR-Tools / CP-SAT) builds the schedule under constraints: crafts, shifts, permits, outages, material availability, priority/criticality. The LLM explains trade-offs and handles "what if" questions in natural language. |
| **Execution** | Printed work packet or a mobile form. | A technician copilot works offline-first, takes voice input, and gives step-by-step guidance grounded in the manual and in past fixes on this asset. It also captures readings and photos as it goes. |
| **Close-out** | Tech types "replaced pump seal, ok." Failure codes are skipped. | The agent turns the narrative, photos, and readings into actuals, problem/cause/remedy codes, meter readings, and follow-up WOs. The tech confirms with one tap, and a supervisor reviews only the low-confidence items. |
| **Reliability** | Annual bad-actor review in Excel. | Always on: bad-actor detection, failure-mode clustering across free text, PM optimization (adjust frequency, convert to condition-based, delete), and RCA drafts with evidence. |
| **Inventory** | Static min/max, obsolete stock, emergency buys. | Usage-driven reorder suggestions, critical-spares analysis tied to asset criticality, duplicate-item detection (the classic item master mess), and obsolescence alerts. |

Close-out is the keystone. It is the cheapest loop to automate, the easiest to measure, and it **feeds data quality into every other loop**. It is the best wedge.

## 4. Architecture

```
 ┌──────────────── Clients ────────────────┐
 │ Web (planner/sched/rel.)  Mobile (offline│
 │ voice/photo)  Chat/Teams  Requester portal│
 └───────────────────┬──────────────────────┘
                     │ same typed API
 ┌───────────────────▼──────────────────────┐
 │           Domain Command API             │  validation, permissions,
 │   (CreateWO, ApproveProposal, Issue...)  │  idempotency, audit
 └──────┬───────────────┬───────────────────┘
        │               │
 ┌──────▼──────┐  ┌─────▼──────────────────────────────┐
 │ Policy /    │  │ Agent runtime                       │
 │ Autonomy    │◄─┤ intake · planner · tech copilot ·   │
 │ engine      │  │ closeout · reliability · storeroom ·│
 │ (risk class,│  │ compliance                          │
 │ thresholds, │  │ tools = Command API + read models   │
 │ approvals)  │  │ + retrieval + solvers               │
 └──────┬──────┘  └─────┬──────────────────────────────┘
        │               │
 ┌──────▼───────────────▼───────────────────────────────┐
 │ Deterministic engines: scheduler (CP-SAT), PM gen,    │
 │ reorder/inventory, meter/condition rules, SLA timers  │
 └──────┬───────────────────────────────────────────────┘
 ┌──────▼───────────────────────────────────────────────┐
 │ Data plane                                            │
 │ Postgres: domain tables + append-only event log       │
 │ Time-series (meters, IoT, historian mirror)           │
 │ Document store + vector index (manuals, WO text, P&IDs)│
 │ Object store (photos, video, audio)                   │
 └──────┬───────────────────────────────────────────────┘
 ┌──────▼──────── Integration ───────────────────────────┐
 │ Maximo (OSLC/REST, MIF) · SAP/Oracle ERP · PI/historian│
 │ IoT/MQTT · IdP (SAML/OIDC) · e-permit / LOTO systems   │
 └────────────────────────────────────────────────────────┘
```

### 4.1 Domain model (Maximo-recognizable)

- **Location** (functional location, hierarchy + systems) and **Asset** (serialized, moves between locations). Keep the Maximo split. People understand it.
- **Classification** + specification attributes. Agents fill these from nameplates and datasheets.
- **Asset graph**: parent/child plus typed relations (`feeds`, `drives`, `isolates`, `redundant_with`). Needed for impact analysis, LOTO, and RCA. Stored in Postgres as an adjacency table, with no separate graph database at first.
- **Failure hierarchy**: Failure class → Problem → Cause → Remedy, optionally aligned to ISO 14224. Agents propose codes, and the hierarchy itself can evolve: the agent suggests new codes when the clustered free text shows a mode the hierarchy is missing.
- **Job Plan / PM / Route / Meter**, as in Maximo, with one addition: a **PM rationale** record (the failure mode it addresses, source, last review). That makes PM optimization auditable.
- **Work Order** with tasks, planned/actual labor, materials, tools, plus **Evidence** (photos, audio, sensor snippets) as first-class children.
- **Proposal**: the core new object. `{type, target, diff, evidence[], confidence, risk_class, agent, model_version, status}`.

### 4.2 Proposal & autonomy model

| Level | Meaning | Example |
|---|---|---|
| L0 | Suggest only (inline hint) | "Similar WO 4471 used part X" |
| L1 | Draft, human must approve | Job plan draft, PM frequency change |
| L2 | Auto-apply, human can revert within N hours | Failure code on routine WO with confidence ≥ 0.9 |
| L3 | Auto-apply, audited | Dedupe SR into an existing open WO |

Policy is configured per site, asset criticality, and action type, and every rule can be audited. Hard floors that are never automated: LOTO/isolation, permit issuance, safety-critical element (SCE) PM deferral, and regulatory compliance sign-off.

### 4.3 Grounding & knowledge

- Index sources per asset: OEM manuals, datasheets, P&IDs, past WO long descriptions, and RCA reports.
- Every generated statement in a job plan or answer carries **citations** to those sources. Uncited safety content is blocked.
- Tenant data never trains shared models. Per-tenant retrieval only.

### 4.4 Deterministic engines

- **Scheduler**: CP-SAT model covering labor by craft/shift, permits, outage windows, material availability (kitting), and priority × criticality. It re-solves on events.
- **PM generation**: time-based, meter-based, and condition-based. Same semantics as Maximo PM / PM sequence / nested job plans.
- **Inventory**: usage-based forecasts with lead times, and criticality-weighted service levels.

### 4.5 Evaluation & ops

- Golden datasets per agent, labelled by reliability engineers: failure-code accuracy, asset-ID accuracy, job-plan completeness, dedupe precision.
- Shadow mode first: the agent proposes, and its proposals are compared with what humans actually did. Promote an action type to L2 only once its measured precision clears the threshold.
- Track per tenant: proposal acceptance rate, edit distance on approvals, and reverts.

## 5. Migration & coexistence with Maximo

Most target customers already run Maximo or SAP, and nobody rips out an EAM quickly. Two entry modes:

1. **Overlay (recommended wedge):** Maximo stays the system of record. We sync assets, locations, WOs, job plans, and the failure hierarchy through OSLC/REST. Agents write back through the Maximo APIs (as proposals → approved → posted). Value in weeks, with no migration.
2. **Replacement:** for mid-market sites, greenfield sites, or customers leaving Maximo 7.6 instead of upgrading to MAS. The same agent layer runs on our own data plane.

The overlay and our own system of record share one domain model and one adapter interface, so a customer can move from overlay to replacement site by site.

## 6. Phased roadmap

| Phase | Scope | Success metric |
|---|---|---|
| 0 (8–10 wks) | Read-only connector to the incumbent CMMS, **GMP WO pre-review agent**, cell + model gateway skeleton (SaaS and bare-K8s reference envs), shadow-mode evals | Pre-review flags agree with QA findings at high precision; QA review cycle time down |
| 0b | Close-out copilot (voice/text → codes, actuals, follow-ups), Part 11 e-signature, hash-chained audit trail | Right-first-time WO documentation rate; close-out time down 50% |
| 1 | Calibration module + OOT impact assessment; intake agent; tech copilot (offline mobile, RAG over manuals) | OOT assessment time; SR→WO triage time |
| 2 | Planner agent + kitting check; reliability: bad actors, failure-mode clustering | Planned-work %; planner backlog/hours per WO |
| 3 | Scheduler (CP-SAT) + PM optimization with rationale records | Schedule compliance; PM hours removed without failure increase |
| 4 | Own system of record (replacement mode), storeroom agent | First full-replacement site |

## 7. Decisions log

| # | Decision | Detail |
|---|---|---|
| D1 | **Target segment: life sciences** (pharma, biotech, CDMO, med device) | See [GXP.md](GXP.md). GxP drives the AI classification rule, e-signatures, and validated release trains |
| D2 | **Cloud-native first, with an isolated deployment option** from day one | See [DEPLOYMENT.md](DEPLOYMENT.md). Cell-based architecture, minimal dependencies, model gateway with hosted and self-hosted providers |
| D3 | **Wedge revised: GMP WO pre-review + closeout copilot** | Read-only on the incumbent system of record at first, which keeps validation burden low and targets the QA review bottleneck |

## 8. Open questions

1. **Incumbents at target customers:** mostly Maximo, SAP PM, or Blue Mountain RAM? This decides which connector we build first.
2. **Customer size:** big pharma (overlay on validated Maximo/SAP, long sales cycle) or biotech/CDMO (greenfield replacement more likely, faster)?
3. **Calibration:** in scope for the first release, or the second module after pre-review?
4. **QMS integration priority:** Veeva Vault QMS, TrackWise Digital, or MasterControl?
5. **Failure taxonomy:** ISO 14224 baseline, or learn from each customer's hierarchy?
6. **Mobile:** our own offline app from the start, or integrate with existing mobile tools first?
7. **APM overlap:** our own condition monitoring, or ingest alerts from existing APM/EMS/BMS?

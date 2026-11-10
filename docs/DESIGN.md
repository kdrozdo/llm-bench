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

Two entry modes. **For biotech (D4) replacement comes first**; the overlay is kept for larger customers later.

1. **Overlay (later, for large pharma):** Maximo stays the system of record. We sync assets, locations, WOs, job plans, and the failure hierarchy through OSLC/REST. Agents write back through the Maximo APIs (as proposals → approved → posted). Value in weeks, with no migration.
2. **Replacement (launch mode):** for biotech, greenfield sites, or customers leaving a legacy tool. The same agent layer runs on our own data plane.

The overlay and our own system of record share one domain model and one adapter interface, so a customer can move from overlay to replacement site by site.

## 6. Phased roadmap

| Phase | Scope | Success metric |
|---|---|---|
| 0 (MVP, roughly 4–6 months, small team) | Core system of record (Location/Asset, WO, PM, Job Plan, **Calibration**), Part 11 e-signature, hash-chained audit trail, controlled master data; **onboarding agent** (documents → asset register, PM/cal plan); **vendor cal-certificate ingestion**; close-out copilot; GMP pre-review; offline mobile; shared validated cell + model gateway | Time from contract to go-live (target: weeks); right-first-time WO rate; QA review cycle time |
| 1 | OOT impact assessment, intake agent, tech copilot (RAG over manuals), inspection-readiness pack | OOT assessment time; time to answer inspector requests |
| 2 | Planner agent + kitting check; reliability: bad actors, failure-mode clustering | Planned-work %; planner backlog/hours per WO |
| 3 | Scheduler (CP-SAT) + PM optimization with rationale records | Schedule compliance; PM hours removed without failure increase |
| 4 | Overlay connector (Maximo / SAP PM) for larger customers, storeroom agent | First overlay customer |

## 7. Decisions log

| # | Decision | Detail |
|---|---|---|
| D1 | **Target segment: life sciences** (pharma, biotech, CDMO, med device) | See [GXP.md](GXP.md). GxP drives the AI classification rule, e-signatures, and validated release trains |
| D2 | **Cloud-native first, with an isolated deployment option** from day one | See [DEPLOYMENT.md](DEPLOYMENT.md). Cell-based architecture, minimal dependencies, model gateway with hosted and self-hosted providers |
| D3 | ~~Wedge: GMP WO pre-review on top of the incumbent system~~ | Superseded by D6 |
| D4 | **Customer segment: biotech** (clinical → commercial, small or mid-size, 1–5 sites) | Greenfield or a weak incumbent (spreadsheets, a generic CMMS, a legacy validated tool). We are the **system of record from day one**: the replacement mode in §5 comes first, and the Maximo overlay is deprioritized |
| D5 | **QMS integration deferred** | Deviation, CAPA, and change-control references are stored as typed external IDs and links on the WO/asset. No integration in v1. Our own change control covers CMMS master data only |
| D6 | **Wedge: GxP-ready CMMS that onboards itself** | See [GXP.md §5](GXP.md#5-wedge-for-biotech). AI builds the asset register, PM plan, and calibration plan from the customer's existing documents. Close-out copilot and GMP pre-review are built into the product. Shared SaaS cell on the validated release train is the default; isolated deployment stays an architectural guarantee, not a launch priority |

## 8. Open questions

1. **Calibration depth:** biotech has many lab instruments and usually outsourced calibration vendors. Calibration in v1 (recommended), or v2?
2. **Data we onboard from:** what do biotech customers actually have? Spreadsheets, IQ/OQ PDFs, vendor cal certs, exports from a generic CMMS, or Blue Mountain?
3. **Failure taxonomy:** ISO 14224 baseline (for equipment, plus a simplified lab-instrument set), or start minimal?
4. **Mobile:** our own offline app from the start? (Recommended: yes, since we are the system of record.)
5. **Monitoring signals:** ingest EMS/BMS alarms in v1, or later?

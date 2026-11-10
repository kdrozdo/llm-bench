# Core data model (v0)

Status: draft · Builds on [DESIGN.md](DESIGN.md), [GXP.md](GXP.md), [DEPLOYMENT.md](DEPLOYMENT.md)

Scope: the system-of-record entities for the biotech MVP, plus the three new mechanisms (controlled master data, proposals, and the signed, hash-chained event log). The DDL below is a sketch for discussion, not final.

---

## 0. Conventions

- **IDs:** a UUIDv7 primary key (time-ordered), plus a human number per site (`WO-24-000123`, `CAL-...`) from a gap-free sequence. Auditors like gap-free numbers.
- **Tenancy:** every table has `tenant_id`, even in a database-per-tenant cell (DEPLOYMENT §7.4).
- **Hierarchy:** Organization → Site, like Maximo ORGID/SITEID. Most records are site-scoped. Classifications and the AI function registry are org-scoped.
- **State + event in one transaction.** Domain tables hold current state for querying. Each command writes the state change *and* an event row atomically.
  - This is pragmatic event logging, not pure event sourcing: reports and validation are simpler, and the audit trail is still complete.
  - Nothing writes domain tables except command handlers.
- **Snapshots, not live joins, in GMP records.** A WO copies the GMP impact, the job plan revision, the tolerances, and so on *at the time of creation*. A signed record must not change meaning when the master data changes later.

## 1. Places and things

```sql
location (id, tenant_id, site_id, code, description,
          parent_id, path ltree,             -- hierarchy
          type,                              -- building/room/area/system/position
          cleanroom_grade NULL,              -- A/B/C/D, ISO class
          gmp_area bool, status)

asset (id, tenant_id, site_id, asset_num, description,
       location_id, parent_asset_id NULL,
       classification_id, specs jsonb,       -- validated against the classification's schema
       manufacturer, model, serial_num,
       gmp_impact,                           -- direct | indirect | none  (C&Q impact assessment)
       qualification_status,                 -- not_required | pending | qualified | requal_due | suspended
       operational_status,                   -- in_service | out_of_service | quarantined | retired
       criticality,                          -- 1..5
       is_instrument bool, is_reference_standard bool,
       created_from_proposal_id NULL)

asset_relation (from_asset_id, to_asset_id, type)  -- feeds | drives | isolates | redundant_with | measures

classification (id, path, description,
                spec_schema jsonb, schema_version) -- JSON Schema; agents fill specs against it
```

Notes:
- `gmp_impact` follows ISPE C&Q impact assessment terms (direct/indirect/none), which biotech QA already uses.
- `quarantined` exists because of OOT handling: an instrument found out of tolerance is quarantined until the impact assessment is done.
- `is_reference_standard` marks calibration standards, so traceability checks can be computed (§4).

## 2. Controlled master data (one pattern for everything versioned)

Job plans, PM plans, calibration procedures, tolerances, and checklists all follow the **document + revision** pattern:

```sql
controlled_doc (id, tenant_id, site_id, doc_type,  -- job_plan | pm_plan | cal_procedure | checklist
                number, title, current_effective_rev_id NULL)

controlled_rev (id, doc_id, rev_label,             -- A, B, C...
                status,                            -- draft | in_review | approved | effective | superseded | withdrawn
                body jsonb,                        -- typed by doc_type (tasks, cal points, frequency...)
                body_hash,                         -- sha256 of canonical body
                effective_from, change_reason,
                created_from_proposal_id NULL)
```

Rules:
- Revisions are **immutable once `in_review`**. Any change means a new draft revision.
- `approved` → `effective` requires the signatures that the doc type's policy demands (e.g. author + SME + QA for GMP), each bound to `body_hash` (§6).
- Generators (PM scheduler, cal due engine) only read the **effective** revision, and WOs snapshot `rev_id` + `body`.
- This is our own change control **for CMMS master data only**. Change control for the product or process stays in the customer's QMS (D5); we just store its reference.

PM plan body (sketch):
```json
{ "asset_ids": ["..."], "job_plan_rev_id": "...",
  "trigger": { "type": "time", "interval": "P6M", "window": { "before": "P7D", "after": "P14D" },
               "basis": "fixed" },
  "rationale": { "failure_modes": ["..."], "source": "OEM manual §7.2", "doc_refs": ["..."] } }
```
The grace window is explicit and part of the controlled body. "Is this PM overdue?" is then a deterministic, auditable answer.

## 3. Work orders

```sql
work_order (id, tenant_id, site_id, wo_num, type,  -- CM | PM | CAL | QUAL | EM (emergency) | INSP
            asset_id, location_id,
            -- snapshot at creation:
            gmp_impact_snapshot, qualification_status_snapshot,
            job_plan_rev_id NULL, job_plan_body jsonb NULL,
            pm_plan_rev_id NULL, due_date, window_start, window_end,
            status,              -- §3.1
            priority, problem_description,
            failure_class, problem_code, cause_code, remedy_code,
            requires_qa_review bool,   -- derived from GMP snapshot + type
            created_from_proposal_id NULL)

wo_task (id, wo_id, seq, instruction, kind,        -- step | reading | check | photo | signature
         expected jsonb NULL,                      -- e.g. {min, max, unit} from the plan snapshot
         result jsonb NULL,                        -- actual value / pass / text
         result_eval,                              -- computed: in_range | out_of_range | n/a
         performed_by, performed_at_device, received_at_server, offline_captured bool)

wo_labor (wo_id, person_id, craft, start, finish, hours)
wo_material (wo_id, item_id, qty, lot_or_serial)
evidence (id, wo_id, kind, object_key, sha256, captured_at_device, received_at_server, captured_by)
external_ref (id, record_type, record_id, system, ref_type, ref_number, url)  -- deviation/CAPA/CC (D5)
```

Contemporaneous capture: every reading keeps **device time and server time** and whether it was captured offline. The pre-review agent flags gaps that don't make sense, such as a reading timed after its own signature.

### 3.1 WO status machine (GMP)

```
draft → approved → scheduled → in_progress → completed ──► qa_reviewed → closed
  │         │                      │            (tech signs:      (QA signs:
  └─────────┴──────── cancelled ◄──┘             "performed")      "reviewed")
                                                     │
                                       returned ◄────┘  (QA returns with comments → in_progress)
```
- Non-GMP WOs skip `qa_reviewed` (`requires_qa_review = false`).
- `completed` is blocked by deterministic checks:
  - all required tasks have results
  - out-of-range results carry a comment
  - the required evidence is present
- Pre-review agent findings are attached at `completed`. They help QA but never block transitions. Only deterministic checks block.
- Seven states, not Maximo's configurable dozens. Customers configure *who signs*, not *the state machine*. That keeps validation simple.

## 4. Calibration

```sql
cal_profile (asset_id, cal_procedure_doc_id,       -- controlled doc; points + tolerances live in its revision
             interval, window, performed_by,       -- in_house | vendor
             vendor_id NULL)

-- cal_procedure revision body (controlled):
--   points: [{ id, parameter, nominal, unit,
--              process_tol: {type: abs|pct_reading|pct_span, value},
--              mfr_tol:     {...},
--              min_tur: 4.0 }]

cal_event (id, wo_id, asset_id, procedure_rev_id, procedure_body jsonb,  -- snapshot
           source,                                 -- in_house | vendor_certificate
           certificate_doc_id NULL,
           standards_used: [{ asset_id, cal_due_at_time, valid bool }],  -- computed
           result,                                 -- computed: pass | pass_adjusted | oot | fail_mfr
           impact_assessment_required bool)        -- computed: as_found outside process_tol

cal_reading (cal_event_id, point_id, phase,        -- as_found | as_left
             reading, standard_value, error, error_eval,     -- computed
             entered_by, entry_method)             -- manual | extracted_verified
```

Deterministic rules (Critical class, no AI):
- `error_eval` comes from the tolerance math on the snapshotted procedure.
- `result`:
  - any **as-found** outside process tolerance → `oot` → asset `quarantined` + impact assessment required
  - as-found in tolerance but adjusted → `pass_adjusted`
- **Standards traceability:** each standard used must itself be in calibration on the event date (its own last cal_event result and due date), otherwise the event cannot be completed.
- The **vendor's own pass/fail statement is stored but never used.** We recompute against *our* process tolerance, which is a common inspection finding.

AI role (Assistive):
- extract readings from the vendor certificate PDF into `cal_reading` drafts, with the page and bounding-box citation for each value
- a human verifies against the PDF side by side and signs "verified"
- `entry_method = extracted_verified` records that path

## 5. Proposals (how AI writes anything)

```sql
ai_function (id, name, impact_class,               -- C | A | N  (GXP §2); C functions cannot exist here
             max_autonomy,                         -- L0..L3
             context_of_use text, eval_suite_id)

ai_function_qualification (function_id, model_id, model_version, prompt_version,
                           eval_run_id, metrics jsonb, passed bool,
                           effective_autonomy,     -- may be lower than max (eval-gated)
                           qualified_at)

proposal (id, tenant_id, site_id,
          function_id, agent_run_id, model_id, model_version, prompt_version,
          target_type, target_id NULL, target_base_hash NULL,   -- NULL target = create
          operation,                               -- create | update | attach | flag
          payload jsonb,                           -- full object (create) or JSON Patch (update)
          evidence jsonb,                          -- [{doc_id, page, bbox, quote} | {record_type, id}]
          confidence numeric,
          autonomy_applied,                        -- policy decision at creation
          status,                                  -- pending | accepted | accepted_with_edits | rejected
                                                   -- | auto_applied | reverted | stale
          decided_by NULL, decided_at NULL, signature_id NULL,
          final_payload jsonb NULL,                -- what was actually applied
          human_diff jsonb NULL,                   -- final vs. proposed: the training/eval signal
          batch_id NULL)                           -- bulk onboarding review
```

Rules:
- **Staleness:** if `target_base_hash` no longer matches the target's current state, the proposal becomes `stale` and cannot be applied. That prevents an agent overwriting a human edit.
- Applying a proposal runs the **same command** a human would, with `proposal_id` attached to the resulting event. There is no separate write path.
- **Effective autonomy** = min(policy for this site/asset class, `ai_function.max_autonomy`, the eval-qualified autonomy for the deployed model). GMP impact `direct` caps it at L1.
- **Onboarding batches:** the setup agent emits thousands of proposals grouped in `batch_id`, and the SME reviews by exception: sorted by confidence, sampled, with bulk accept requiring an e-signature. The signature covers a manifest hash of all accepted payloads.

## 6. Events and signatures (Part 11 core)

```sql
event (tenant_id, seq bigint,                      -- gap-free per tenant
       event_id uuid, stream_type, stream_id,      -- e.g. work_order / <id>
       event_type,                                 -- WorkOrderCompleted, CalReadingRecorded, ...
       payload jsonb,                              -- before/after for changed fields
       actor_type,                                 -- human | agent | integration | system
       actor_id, on_behalf_of NULL,                -- agent acting in a user's session
       proposal_id NULL, reason NULL,
       occurred_at,                                -- trusted server time (NTP-disciplined)
       client_time NULL,
       prev_hash, hash)                            -- hash = sha256(prev_hash || canonical(event))

signature (id, tenant_id, signer_id, signer_name_snapshot,
           meaning,                                -- performed | verified | reviewed | approved
           record_type, record_id, record_rev NULL,
           content_hash,                           -- hash of the record as rendered for signing
           auth_method,                            -- password_reauth | sso_step_up
           auth_time, signed_at, event_seq)
```

Invariants:
1. **Append-only:** the application DB role has no UPDATE/DELETE on `event` or `signature`, enforced with DB grants and a trigger, not just code.
2. **Hash chain per tenant.** A periodic job writes the head `(seq, hash)` to WORM storage (S3 Object Lock or an on-prem equivalent). Verification is a built-in report an auditor can run.
3. **Signature/record linking (§11.70):** a signature binds to `content_hash`. If the record changes afterwards, the signature shows as **invalidated** and the record has to be signed again. Signatures cannot be cut or copied to another record.
4. **Signers are always humans.** `signer_id` must be a person account. Agent service identities cannot hold any signing permission.
5. **Readable audit trail:** every event type has a human-readable renderer ("Calibration reading as-found 4.02 bar entered by J. Smith, extracted from certificate CERT-123 p.2, verified"), exportable to PDF/A for inspections.

## 7. Documents and knowledge

```sql
source_doc (id, tenant_id, site_id, kind,          -- manual | iq | oq | pq | sop | cal_certificate | spreadsheet | drawing
            title, object_key, sha256, uploaded_by, uploaded_at,
            linked_assets uuid[])

doc_chunk (doc_id, page, bbox jsonb, text, embedding vector(N), chunk_hash)
```

- Every citation in a proposal points to `(doc_id, page, bbox)`, so the review UI shows the exact highlighted region.
- Source documents are immutable (same hash means the same doc). A new version is a new `source_doc` row with a `supersedes` link.

## 8. Open points

1. Gap-free per-tenant `seq` serializes writes per tenant. That's fine at biotech scale (hundreds of users), but measure it. Fallback: chain per stream plus a periodic Merkle root across streams.
2. Inventory/storeroom is minimal in MVP (item, issue to WO, lot/serial). Is that enough for biotech?
3. Do we need **logbooks** (GXP §3) in MVP as a separate entity, or as WO-like "log entries" on assets?
4. Training/qualification of *people* (who may perform or sign a GMP cal) is often required. Do we need a minimal `person_qualification` in MVP?

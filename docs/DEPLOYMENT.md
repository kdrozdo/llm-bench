# Deployment architecture: cloud-native with isolated option

Status: draft · Builds on [DESIGN.md](DESIGN.md), [GXP.md](GXP.md)

Goal: **one codebase and one set of artifacts** run as multi-tenant SaaS and as a fully isolated deployment (customer cloud, on-prem, air-gapped) without forking.

---

## 1. Topologies

| # | Topology | Who runs it | Typical customer |
|---|---|---|---|
| 1 | **Shared cell** (multi-tenant SaaS) | Us | Biotech, CDMO, mid-size pharma |
| 2 | **Dedicated cell** (single-tenant, our cloud, chosen region, customer-managed keys) | Us | Large pharma wanting isolation without ops |
| 3 | **Customer-hosted** (their cloud account / VPC, Kubernetes) | Customer, or us under contract | Pharma with strict cloud policy |
| 4 | **Air-gapped / on-prem** | Customer | Sites with no egress, some government/defense-adjacent manufacturing |

**Priority for the biotech segment (D4):** topology 1 on the validated release train is the default and launch target. Topologies 2–4 stay *architecturally guaranteed* (CI keeps the bare-Kubernetes reference environment green) but are sold on demand.

## 2. Cell-based architecture

A **cell** is a complete, independent stack: the app services, a Postgres cluster, object storage, the model gateway, and workers. Tenants are placed into cells.

- Shared cells serve many tenants. Each tenant gets **its own database** in the cell's Postgres cluster. Pharma auditors find this easy to understand, and it gives per-tenant backup/restore, residency, and clean offboarding.
- Cells are grouped by **release train** (continuous vs. validated), so a validated tenant is never upgraded by a deploy meant for continuous tenants.
- A dedicated cell or a customer-hosted install is just a cell with one tenant. Topologies 2–4 are the same software.

```
           SaaS control plane (ours)
   tenant provisioning · release orchestration · billing · opt-in telemetry
        │                │                 │
   ┌────▼────┐     ┌─────▼────┐      ┌─────▼──────┐        ┌──────────────┐
   │ Shared  │     │ Shared   │      │ Dedicated  │        │ Isolated cell │
   │ cell A  │     │ cell B   │      │ cell (T9)  │        │ (customer DC) │
   │continuous│    │validated │      │ validated  │        │ local operator│
   │ T1..Tn  │     │ T20..Tm  │      │            │        │ no phone-home │
   └─────────┘     └──────────┘      └────────────┘        └──────────────┘
                                                             updates via signed
                                                             offline bundles
```

## 3. Minimal dependency footprint (the key constraint)

Everything must run where the customer has only Kubernetes and nothing else, so we deliberately keep the dependencies small:

| Need | Choice | Why |
|---|---|---|
| Orchestration | Kubernetes + our **operator** / Helm chart | Same install everywhere |
| Relational + event log | **PostgreSQL 16+** (managed in cloud; CloudNativePG on-prem) | One database to operate and validate |
| Queue / jobs / outbox | Postgres (`SKIP LOCKED` job tables + outbox) | Avoids running Kafka/RabbitMQ on-prem; revisit only at real scale |
| Vector search | **pgvector** | No separate vector DB |
| Full-text search | Postgres FTS | Same |
| Time series (meters, signals) | Postgres native partitioning first; TimescaleDB optional | Can add later without redesign |
| Files (photos, manuals, audio) | **S3-compatible** API (S3 / GCS-interop / Azure via gateway / MinIO on-prem) | One storage abstraction |
| Identity | OIDC / SAML to the customer IdP; SCIM provisioning | Pharma always brings its own IdP |
| Secrets / keys | Cloud KMS or Vault; BYOK in dedicated/hosted | Customer-managed keys |

In SaaS we use managed equivalents (managed Postgres, S3), but the app only ever sees the same interfaces.

## 4. AI in an isolated world: the model gateway

All model calls go through one internal **model gateway**. Agents never call a provider directly.

**Provider adapters:**
- hosted frontier models via the vendor API, or via the customer's own cloud account (AWS Bedrock / Google Vertex), which keeps traffic in their cloud boundary and region
- **self-hosted open-weights models** (served with vLLM or similar) for air-gapped installs
- the same pattern for supporting models: embeddings, speech-to-text, OCR, and vision all have hosted and local implementations

**Capability tiers and eval-gated features:**
- Each agent function declares the capability tier it needs.
- On install and on every model change, the eval suite runs against the configured models.
- A function whose evals don't meet its threshold is automatically **downgraded** (e.g. L1 → L0 suggestions only) or disabled, and the result is recorded in the validation evidence.
- Air-gapped customers therefore get an honest, measured feature set rather than a silently worse one.

**The gateway also handles:** model version pinning per tenant, prompt/config versioning, PII/GMP-data redaction rules, per-tenant usage limits, and request/response logging into the audit trail (stored inside the cell).

**Rough air-gapped sizing (to validate with benchmarks):**
- A capable open-weights model for drafting and pre-review needs a small GPU node, roughly 2–4 data-center GPUs.
- Smaller deployments can run a smaller model plus CPU-only embeddings and STT, accepting lower tiers.

## 5. Control plane vs. isolated operations

| Concern | SaaS / dedicated | Isolated |
|---|---|---|
| Upgrades | Orchestrated per release train | Signed, versioned **offline bundle** (images + migrations + validation evidence) applied by the operator |
| Licensing | Control plane | Signed license file, offline verification |
| Telemetry | Default on (ops metrics only) | Off by default; optional export bundle for support |
| Backups / DR | Ours, per-tenant PITR | Operator-managed, documented RPO/RTO, customer storage |
| Model updates | Release with eval report | Part of the offline bundle, with weights where licensable |

## 6. Mobile & edge

- An offline-first mobile app syncs against whichever cell it belongs to. Sync is through the same command API, with idempotency keys and conflict rules.
- On-device speech-to-text where possible, so voice capture works in cleanrooms and basements with no connectivity.
- Optional **site edge node** for plants with poor WAN: caches assets and documents and buffers events.

## 7. Engineering rules that keep this possible

1. No direct calls to a cloud-proprietary service from app code. Everything goes behind an interface with an on-prem implementation, tested in CI.
2. CI runs the full test and eval suite on **two reference environments**: managed cloud and a "bare" Kubernetes + CloudNativePG + MinIO + local model.
3. Database migrations are forward-only, versioned, and validated as part of the release bundle.
4. Tenant ID is part of every request context and every row access path, even in single-tenant cells. One code path.

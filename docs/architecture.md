# StemMatch Architecture

> **Status: proposed.** This document describes the agreed target architecture.
> Unless a component is listed under [Implemented](#1-implementation-status), it does not exist yet.
> [CLAUDE.md](../CLAUDE.md) remains the working source of truth for scope, constraints, and workflow. If the two disagree, CLAUDE.md wins and this document should be corrected.

StemMatch is an educational/portfolio project. It is not a medical device or clinical decision-support system, and all donor/patient data is synthetic.

---

## 1. Implementation status

### Implemented

| Item | Location |
|---|---|
| Project instructions and constraints | [CLAUDE.md](../CLAUDE.md) |
| Project README with non-clinical disclaimer | [README.md](../README.md) |
| Repository hygiene (`.gitignore`, `.gitattributes`) | repository root |
| This architecture document | `docs/architecture.md` |

### Proposed (not yet implemented)

Everything else in this document, including all services, the database schema, API contracts, CI, containers, and infrastructure.

---

## 2. Proposed monorepo structure

```text
stemmatch/
├── apps/
│   └── web/              # React + TypeScript frontend          (proposed)
├── services/
│   ├── platform/         # Node.js + Express application backend (proposed)
│   └── matching/         # Python + FastAPI matching service     (proposed)
├── infra/                # Infrastructure configuration: Docker, Terraform,
│                         # observability config                   (proposed)
├── docs/                 # Architecture, API contracts, domain docs
│   └── architecture.md   # this file                              (implemented)
├── CLAUDE.md
└── README.md
```

| Path | Purpose | Notes |
|---|---|---|
| `apps/web/` | React/TypeScript UI | Presentation only. No authoritative matching logic; never relies on hidden UI for access control. |
| `services/platform/` | Node.js/Express backend | Application APIs, identity/roles, donor record lifecycle, consent, provenance, authorization. |
| `services/matching/` | Python/FastAPI matching service | HLA parsing/normalization, deterministic compatibility rules, explanations, matching/search endpoints. |
| `infra/` | Infrastructure configuration | Introduced in later stages (see CLAUDE.md §4). No Docker, Kafka, Redis, or AWS configuration yet. |
| `docs/` | Documentation | Architecture, API contracts (e.g. `docs/api/`), domain specs (e.g. `docs/domain/`). |

Each service must remain independently buildable and testable. Services communicate only through documented API contracts, never by importing each other's code.

---

## 3. Ownership

| Area | Owner | Primary branch |
|---|---|---|
| HLA matching engine, domain matching logic, Python/FastAPI matching service, normalization, compatibility rules, explanations, matching tests | **Ashlesha** (project owner) | `feature/matching-engine` |
| Node.js/Express platform backend, PostgreSQL application data model and database integration, platform-side infrastructure | **Collaborator** | `feature/platform-backend` |
| API contracts between services, the HLA projection schema (see §4), integration, cross-service review | **Shared** | Coordinated via PRs into `develop` |

Changes that cross an ownership boundary must be discussed and reviewed by both contributors before merging.

---

## 4. HLA data ownership model (approved)

```text
                 writes (authoritative)
  apps/web ──▶ services/platform ─────────────▶ PostgreSQL (source of truth)
                     │                              │
                     │ sync HTTP (match request)    │ normalized HLA projection
                     ▼                              │ (read-only access)
               services/matching ◀──────────────────┘
               (owns matching calculations)
```

1. **PostgreSQL is the source of truth** for all persisted state. Redis or any future index is derived data only.
2. **The Node platform service owns authoritative writes** for donor and application data: donor lifecycle, consent, provenance, and access policy.
3. **The Python matching service reads a normalized HLA projection** through **read-only** database access (a dedicated read-only role, scoped to the projection only). It does not write donor or application records.
4. **The Python matching service owns matching calculations**: normalization rules, compatibility evaluation, and explanations.
5. **Kafka is not required merely to connect these services.** Synchronous HTTP between the platform and matching services, plus a shared database read path, is sufficient for the initial design. Kafka is considered only where an asynchronous workflow is justified (CLAUDE.md §7).
6. **A separate search index is a possible future optimization** (CLAUDE.md §4, stage 5), introduced only if benchmarks on synthetic data show it is needed. It would be derived from PostgreSQL, not a second source of truth.

### Open decisions for this model (shared)

These are not yet decided and must be agreed by both contributors before implementation:

- Exact form of the projection: table, view, or materialized view, and who maintains it.
- Which schema/columns the projection exposes. It should contain only what matching needs (e.g. an opaque donor ID, normalized HLA values, eligibility/consent flags), not identifying details.
- How consent and authorization are enforced on match requests, and whether the projection pre-filters consent-ineligible records.
- Freshness and consistency expectations between writes and the projection.
- Projection schema versioning and migration ownership.

---

## 5. Development environment: OneDrive considerations

The repository is currently located inside a OneDrive-synced folder. **It has not been moved**; this section documents the risks so the team can decide.

Risks of developing inside a OneDrive-synced folder:

- **`node_modules/`**: tens of thousands of small files cause heavy sync traffic, slow installs, and file locks during sync that can make `npm install` fail (`EPERM`/`EBUSY` errors). Files On-Demand can also make dependencies appear present but not be downloaded locally.
- **Python virtual environments (`.venv/`)**: contain many files and platform-specific binaries. Sync locks can break installs, and syncing a venv to another machine produces a broken environment.
- **Database and container volumes**: PostgreSQL, Redis, or Kafka data directories must never be synced. Concurrent modification by the sync client can corrupt database files, and files can be locked while the database is running.
- **Git internals (`.git/`)**: sync locks or conflict copies (e.g. `index-MACHINE.lock`, duplicated objects) can corrupt the repository or cause confusing failures.
- **Build outputs and caches**: generate frequent churn and sync conflict copies with no value.
- **Secrets**: a local `.env` file would be uploaded to cloud storage even though `.gitignore` keeps it out of Git.

Mitigations (in rough order of preference):

1. Move the repository to a non-synced location (e.g. `C:\dev\stemmatch`), with GitHub as the backup/sync mechanism.
2. If it stays in OneDrive, exclude `node_modules/`, `.venv/`, and build/cache folders from sync where possible, and keep all container volumes on Docker-managed named volumes rather than bind mounts inside the repository.
3. Never store real secrets in a `.env` inside a synced folder.

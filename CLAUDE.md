# StemMatch — Claude Code Project Instructions

> This file is the working source of truth for Claude Code when operating in this repository.
> Follow the approved scope and architecture below. Do not silently redesign the system or expand scope.

## 1. Project overview

**StemMatch** is a portfolio-grade, intelligent donor-matching platform focused on HLA (human leukocyte antigen) compatibility. Its engineering goals are:

- Correct, deterministic HLA normalization and compatibility evaluation.
- Explainable match results: show which loci/alleles contributed to a result and why.
- Efficient donor search that can scale to a benchmark of one million **synthetic** donor records.
- Secure handling of donor-like records, consent, provenance, and access boundaries.
- A modular architecture that demonstrates modern backend, data, distributed-systems, cloud, and DevOps engineering.

StemMatch is an educational/portfolio software project. It is **not** a medical device, clinical decision-support system, transplant recommendation, or substitute for qualified medical professionals. Do not present generated scores as clinical probabilities or clinical advice.

## 2. Non-negotiable data and safety constraints

- Use synthetic, fabricated data only.
- Never request, import, commit, log, or reproduce real donor or patient data, personal health information, credentials, access tokens, or real genetic/HLA records.
- Do not use production datasets containing identifiable or sensitive patient/donor records.
- Synthetic HLA examples must be clearly labelled as synthetic and must not be represented as clinically validated.
- **Permitted:** public HLA nomenclature and allele names (for example, published WHO/IMGT-HLA allele designations such as `A*02:01`) may be used in technical references, documentation, and synthetic fixtures. A nomenclature name is a reference identifier, not a record about a person.
- **Prohibited:** real individuals' HLA typing records (any genotype or typing result associated with a real person, even if de-identified), and any identifiable donor or patient information. Synthetic fixtures may combine public allele names, but must never be copied or derived from a real person's typing.
- Keep secrets in environment variables or an appropriate secrets manager. Commit only `.env.example` with placeholder values.
- Do not add real credentials to source files, test fixtures, screenshots, logs, issues, or documentation.
- Enforce authorization and tenant boundaries in the application design; never assume that hiding a UI element is access control.
- Avoid claims of clinical validity. Any matching methodology, assumptions, and limitations must be documented.

If a request conflicts with these constraints, stop and explain the conflict before making changes.

## 3. Approved architecture and technology stack

The approved high-level stack is:

| Layer | Technology | Responsibility |
|---|---|---|
| Web frontend | React + TypeScript | User interface and workflows |
| Application backend | Node.js + Express | Application APIs, identity/access workflows, donor and consent management |
| Matching service | Python + FastAPI | HLA normalization, deterministic matching, explainability, search-oriented matching APIs |
| Primary database | PostgreSQL | System of record and relational integrity |
| Cache | Redis | Carefully scoped caching and performance support |
| Event streaming | Kafka | Asynchronous domain events and distributed workflows |
| Containers | Docker | Reproducible local development and service packaging |
| Cloud | AWS | Deployment and cloud infrastructure |
| Infrastructure as code | Terraform | Repeatable infrastructure provisioning |
| CI/CD | GitHub Actions | Automated quality checks and delivery workflows |
| Observability | Prometheus + Grafana | Metrics, dashboards, and operational visibility |
| Research tools | SciPy / PyMC | Optional, later-stage probabilistic research—not a replacement for deterministic correctness |

Do not introduce additional major infrastructure or frameworks without explaining the need, cost, operational burden, and alternatives, then asking for approval.

### Service boundaries

- **React/TypeScript frontend:** presentation and user interaction. It must not contain authoritative matching logic or bypass backend authorization.
- **Node.js/Express application backend:** application-facing API, user and role workflows, donor record lifecycle, consent and access policies, and coordination with the matching service.
- **Python/FastAPI matching backend:** domain-specific HLA parsing/normalization, deterministic compatibility logic, match explanations, and matching/search endpoints.
- **PostgreSQL:** authoritative persisted state, relational constraints, and transaction boundaries.
- **Redis:** derived/temporary performance layer only. PostgreSQL remains the source of truth.
- **Kafka:** asynchronous integration where justified. Events must be designed for retries, duplicates, and recovery.

Keep the Node and Python services independently testable. Define and document API contracts rather than coupling their internal implementation details.

## 4. Core engineering priorities

Work in this order. Do not optimize later-stage infrastructure while core correctness is still uncertain.

1. **Foundation:** repository structure, local development conventions, configuration, CI checks, test strategy, and domain glossary.
2. **HLA domain and deterministic matching:** normalization, validation, missing/ambiguous values, compatibility rules, explanations, and extensive correctness tests.
3. **Data and application backend:** PostgreSQL schema, constraints, donor lifecycle, access control, consent, provenance, and Node/Python API contracts.
4. **Frontend:** clear workflows, validation, loading/error states, and explainable match-result presentation.
5. **Search and performance:** indexed/optimized search and reproducible benchmarks using synthetic data, targeting one million synthetic donor records.
6. **Distributed workflows:** Kafka integration with reliability and recovery controls.
7. **Cloud and DevOps:** Docker, AWS deployment, Terraform, GitHub Actions, Prometheus, and Grafana.
8. **Research extensions:** investigate probabilistic matching with SciPy/PyMC only after the deterministic baseline is correct, tested, and documented.

Federation across multiple institutions is a future extension, not a reason to complicate the first implementation.

## 5. HLA matching correctness requirements

The deterministic matching engine is a critical domain component. Treat correctness as more important than speed or feature count.

- Define supported HLA loci, allele representations, input formats, and normalization rules explicitly.
- Preserve the distinction between an absent value, an unknown value, an ambiguous value, and a validated allele.
- Never silently convert missing or ambiguous data into a confident match.
- Validate malformed and unsupported inputs and return actionable, structured errors.
- Make compatibility rules explicit, versioned, deterministic, and testable.
- Return an explanation for each match result, including the relevant loci/alleles, applied rules, unresolved inputs, and limitations.
- Keep raw input, normalized representation, and derived match output conceptually distinct.
- Add unit, boundary, negative, regression, and property-oriented tests where appropriate.
- Use only synthetic fixtures.
- Do not claim that a matching score is a clinical probability or that the algorithm is clinically validated.

Before implementing or changing a matching rule, document the rule and its assumptions. Ask for approval if the change alters the agreed domain interpretation or externally visible result semantics.

## 6. Data integrity, privacy, and authorization

The data model and application must account for:

- Explicit donor/patient-like record relationships where applicable.
- Role-based authorization and tenant isolation.
- Consent state and consent versioning.
- Data provenance and auditability.
- Retention/deletion policy hooks.
- Database-level constraints for important invariants.
- Safe handling of errors and logs so sensitive values are not exposed.

Use synthetic entities and test accounts only. Do not build an assumption that every authenticated user can access every record. Validate authorization on the server for every protected operation.

## 7. Distributed-system reliability requirements

When Kafka or asynchronous processing is introduced, design for failure rather than assuming exactly-once execution.

- Use an outbox pattern for reliable publication of database-backed domain events where appropriate.
- Make consumers idempotent.
- Define retry policies and dead-letter handling.
- Make replay and recovery behavior explicit.
- Include correlation IDs across service boundaries and logs.
- Avoid publishing secrets or sensitive payloads in events.
- Document event schemas and compatibility expectations.
- Add tests for duplicate delivery, retry, failure, and recovery paths.

Do not add Kafka to a workflow that is simpler and safer as a synchronous database transaction. Explain the trade-off before introducing asynchronous complexity.

## 8. Team ownership and collaboration

The repository is being developed collaboratively. Respect the agreed ownership boundaries.

### Project owner — Ashlesha

Owns:
- HLA matching engine and domain matching logic.
- Python/FastAPI matching service.
- HLA normalization, deterministic compatibility rules, explanations, and matching tests.
- Integration decisions for the matching service, coordinated with the collaborator.

Primary branch:
- `feature/matching-engine`

### Collaborator

Owns:
- Node.js/Express application backend.
- PostgreSQL application data model and database integration.
- Platform-side infrastructure and application integration work, coordinated with the project owner.

Primary branch:
- `feature/platform-backend`

### Shared responsibilities

- Agree on API contracts and data ownership before integrating services.
- Coordinate schema and interface changes that affect both workstreams.
- Review cross-service changes together.
- Keep work within the assigned ownership area unless both contributors agree to a change.

Do not overwrite, reformat, or refactor another contributor's work unnecessarily. If a change crosses ownership boundaries, explain the dependency and coordinate before modifying it.

## 9. Git and GitHub workflow

Approved branch model:

- `main`: stable, reviewed project state.
- `develop`: integration branch for completed feature work.
- `feature/matching-engine`: project owner's matching-engine work.
- `feature/platform-backend`: collaborator's platform-backend work.

Rules:

1. Do day-to-day work on the appropriate feature branch, not directly on `main`.
2. Open pull requests from feature branches into `develop`.
3. Run relevant tests and checks before requesting review.
4. Keep pull requests focused and reasonably small.
5. Do not merge unfinished or unreviewed work into `main`.
6. Promote tested, integrated releases from `develop` to `main` through a deliberate review.
7. Never force-push shared branches unless the project owner explicitly approves it.
8. Do not rewrite shared Git history or delete remote branches without approval.
9. Use clear commit messages describing the change.
10. Before any Git operation that could discard work, inspect status and explain the impact.

Useful commands (do not run destructive commands without approval):

```bash
git status
git branch -a
git switch develop
git switch feature/matching-engine
git switch feature/platform-backend
git pull
```

If a local feature branch does not yet exist, create it to track its matching remote branch, for example:

```bash
git switch --track origin/feature/matching-engine
```

Use the equivalent command for the platform backend branch when needed.

## 10. Repository and code-quality conventions

- Inspect the existing repository before proposing or creating a directory structure.
- Preserve existing files and working code. Do not replace project files wholesale without a clear reason and approval.
- Prefer small, cohesive modules with explicit interfaces.
- Keep business/domain logic separate from HTTP handlers, persistence, and UI components.
- Use clear names, type hints/types, validation, and structured errors.
- Add or update tests with behavior changes.
- Update documentation when an API contract, setup step, domain rule, or operational behavior changes.
- Avoid unnecessary abstractions, premature optimization, and speculative features.
- Do not add dependencies without explaining why they are needed.
- Pin or constrain dependency versions using the package manager's normal conventions.
- Keep formatting and linting consistent with the existing project configuration.

## 11. Claude Code working protocol

At the beginning of a task:

1. Read this `CLAUDE.md`.
2. Inspect the current branch and working tree with `git status` and `git branch --show-current`.
3. Inspect the relevant files before editing.
4. Identify the task's scope, affected services, and ownership boundary.
5. State a concise implementation plan before multi-file or architectural changes.

While working:

- Make the smallest complete change that satisfies the request.
- Do not invent requirements or silently expand scope.
- Do not change the approved architecture, matching semantics, API contracts, database ownership, or Git workflow without asking.
- Do not modify unrelated files.
- Do not run destructive commands, reset work, discard changes, force-push, or merge branches without explicit approval.
- Do not claim a test, build, migration, deployment, or command succeeded unless it was actually run and its result was checked.
- If blocked, report the exact blocker and the safest next step.

At the end of a task, report:

- What changed (files and behavior).
- Commands/tests actually run and their results.
- Any remaining risks, limitations, or follow-up work.
- Whether the working tree is clean or has uncommitted changes.

## 12. Definition of done

A task is not complete merely because code was written. Depending on the task, completion should include:

- Correct implementation aligned with approved requirements.
- Relevant automated tests added or updated and passing.
- Validation and error cases handled.
- No secrets or real sensitive data introduced.
- Documentation updated where behavior or setup changed.
- Cross-service contracts checked where applicable.
- Actual test/build results reported accurately.

For a matching-engine change, correctness tests and explicit explanation of edge cases are mandatory. For database changes, include integrity constraints and migration considerations. For asynchronous changes, include duplicate/retry/recovery considerations.

## 13. Current Development Priorities

These gates apply to all work in this repository. A later stage does not begin until the gate before it has been met.

1. **Foundation first.** The foundation setup (project instructions, repository hygiene, architecture documentation) must be reviewed and merged into `develop` before any application implementation begins.
2. **HLA domain specification before matching code.** The HLA domain specification (supported loci, input formats, normalization rules, absent/unknown/ambiguous/validated states, compatibility rules, explanation format, and limitations) must be written and approved before matching-engine implementation starts. See §5.
3. **Contracts before integration.** The Node/Python API contracts and the ownership, schema, and access model of the normalized HLA projection must be agreed by both contributors before the services are integrated. See `docs/architecture.md`.
4. **Follow the roadmap and ownership.** Work must follow the approved development order in §4 and the branch ownership in §8–§9. Cross-boundary changes require coordination with the owning contributor.
5. **Inspect before every task.** At the start of each task, Claude must inspect the current repository state (branch, working tree, and relevant files) rather than relying on assumptions or earlier sessions. See §11.
6. **No unapproved expansion.** Do not expand the architecture, add infrastructure or frameworks, or implement work beyond the currently approved stage without explicit permission from the project owner.

---

**Guiding principle:** Build a correct, explainable, secure, testable matching platform first. Add scale, distributed systems, and cloud complexity only when the foundations justify them.

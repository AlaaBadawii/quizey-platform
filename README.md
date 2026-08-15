# Quizey

**An AI-first Assessment & Training Platform — backend-first, built to scale from "an exam can be taken and graded" toward a continuous, measurable learning loop.**

Exams are the mechanism. Continuous, measurable learning improvement is the product.

---

## About this repository

This is the **public project page** for Quizey. The source code lives in a private repository; this repository intentionally contains only this README. It documents the product vision, the architecture actually implemented, the engineering decisions behind it, and the current status — nothing more and nothing less.

---

## Table of contents

- [Product vision](#product-vision)
- [What Quizey provides today](#what-quizey-provides-today)
- [Backend architecture](#backend-architecture)
- [Domain model](#domain-model)
- [Assessment lifecycle](#assessment-lifecycle)
- [Grading engine](#grading-engine)
- [Reliability engineering](#reliability-engineering)
- [API surface](#api-surface)
- [Testing and engineering quality](#testing-and-engineering-quality)
- [AI-first direction](#ai-first-direction)
- [Production and infrastructure](#production-and-infrastructure)
- [Technology stack](#technology-stack)
- [Key engineering decisions](#key-engineering-decisions)
- [Current status and roadmap](#current-status-and-roadmap)

---

## Product vision

Quizey is designed around three users with three different goals:

| User | Who they are | Their actual goal |
|------|--------------|-------------------|
| **Creator** | Teacher, trainer, coach | Understand what learners know — not "create an exam" |
| **Learner** | Student, trainee, candidate | Know what to study next — not "get a high score" |
| **Organization** | School, company, training program | Understand group performance, not one individual |

Every assessment feeds a single loop:

```
Assess → Understand → Improve → Measure Again → Repeat
```

The assessment changes by context (school exam, coding interview, medical training, language learning); the loop stays the same. Quizey's job is to be the infrastructure for that loop, regardless of domain.

The long-term vision is **AI-first**: AI assists assessment authoring, grading with a human in the loop, and personalized recommendations. That direction shapes the architecture today, but it is deliberately **not built yet** — the current phase exists to make the underlying assessment engine and data trustworthy first. The rationale is explicit in the project plan: *AI without clean underlying data just hallucinates confidently.* (See [AI-first direction](#ai-first-direction).)

Product principles that drive engineering decisions:

1. **Every feature must improve learning** — it must trace back to Creator, Learner, or Organization value.
2. **Scores are not enough** — every score must produce actionable feedback.
3. **Every assessment creates data** — data creates insight; insight improves learning.
4. **AI never replaces the teacher** — every AI-assisted decision keeps a human in the loop until proven trustworthy.
5. **Every AI decision must be explainable** — never "the AI says so."

---

## What Quizey provides today

Quizey is a backend REST API. There is no frontend yet; the API is the product surface.

### Authentication & authorization
- Registration with role (`student` / `teacher`), email + password validation, password hashing.
- JWT access + refresh token pair; token refresh and logout (refresh-token blacklisting).
- Role-based access control enforced at the route layer, with the **database as the single source of truth** for a user's role (no JWT role claims).

### Exam & content management
- Exam creation, editing, listing, and retrieval with ownership enforcement (teachers manage only their exams).
- Question types: **multiple choice**, **true/false**, and **multi-select**.
- Publish validation per question type (option counts, correct-answer counts, rubric presence for manual-review questions).
- **Version families via copy-on-write**: published exams are immutable; editing one creates a new version. Only one version per family is published at a time. Question options and full grading configuration — including rubrics — are deep-copied onto the new version.
- Question **soft-delete** (`deleted_at`) when student answers exist, so grading and audit trails stay intact; hard delete when ungraded.
- Question locking on publish (`locked_at`) — no edits after publication.

### Assessment delivery
- **Attempt lifecycle**: start/resume, pause/resume, submit, lazy expiry, results.
- Attempts are **family-aware** — a student resumes the in-progress attempt anywhere in the exam's version family and new attempts target the newest published version.
- Configurable exam rules, enforced at start: `max_attempts` (across the whole version family), an availability window (`available_from` / `available_until`), and an optional `access_code`.
- Timing model: per-attempt active-time limit (`duration_minutes`, or untimed), an absolute wall-clock `deadline_at`, and per-exam pause policy (`allow_pausing`). Attempt telemetry records what actually happened (`started_at`, real `submitted_at`, paused time), separate from the configured rules.
- **Lazy expiration**: an attempt that runs out of active time or passes its wall-clock deadline is auto-finalized and graded on the next read/write. No background scheduler is required.

### Grading
- **Auto-grading** for multiple choice and true/false (all-or-nothing).
- **Multi-select partial credit** with an exam-level numeric wrong-selection penalty.
- **Weighted questions** (`weight`) and **bonus questions** (`is_bonus`) with a defined, tested order of operations.
- **Manual grading** for open-ended answers, driven by **rubrics**: a teacher-facing grading queue and a rubric-score grading endpoint.
- Per-question feedback in results (correctness, awarded points, and, for manual items, criterion scores).

### Integrity
- An **append-only audit log** (ORM guards + database triggers) recording key grading/submission events.
- **Exactly-once semantics** on mutating endpoints via an idempotency layer with a 24-hour retention window.
- All-or-nothing transactions via a nestable `atomic()` context manager.

---

## Backend architecture

Quizey is a **Flask modular monolith** following a strict layering rule:

```
Route → Service → Model → Database
```

Routes are thin: they parse JSON, enforce role gates, delegate to services, and return responses. Services contain all business logic and validation. Models define the schema. A grading **coordinator + strategy registry** provides a plugin-like extension point for question types.

```
app/
├── auth/          # Blueprint: register, login, refresh, logout, profile
├── exams/         # Blueprint: exam/question/option CRUD, publish, versioning, attempts entry
├── attempts/      # Blueprint: attempt lifecycle routes
├── grading/       # Blueprint + grading coordinator + question-type strategies
├── agents/        # Reserved for the future AI layer (empty today)
├── models/        # SQLAlchemy models (see Domain model)
├── services/      # Business logic: auth, exam, question, option, attempt, grading, manual grading, audit
├── extensions/    # Flask extension instances (db, jwt, migrate)
├── config/        # Environment-based configuration (development / testing / production)
└── utils/         # Cross-cutting: RBAC, idempotency, transactions, response helpers
```

### Architecture overview

```mermaid
flowchart TB
    Client[REST clients] --> BP

    subgraph Flask["Flask application (factory + blueprints)"]
        BP[Blueprints: auth · exams · attempts · grading]
        BP --> SVC
        subgraph SVC["Service layer (business logic)"]
            AU[auth_service]
            EX[exam_service · question_service · option_service]
            AT[attempt_service]
            MG[manual_grading_service]
            AD[audit_service]
        end
        subgraph GRAD["Grading engine"]
            CO[grading coordinator]
            ST[Strategies: multiple_choice · true_false · multi_select]
            CTX[GradingContext — batch-loaded lookups]
        end
        subgraph XC["Cross-cutting"]
            RBAC[RBAC role gates]
            IDEM[Idempotency layer]
            TX[atomic transactions]
            SM[State machine]
        end
    end

    SVC --> GRAD
    SVC --> DB[(Database)]
    XC --> SVC
```

Key structural properties:

- **Application factory** (`create_app`) with environment-driven configuration and a `/health` endpoint.
- **Blueprints per domain**, each registered under `/api/v1/...`.
- **No business logic in routes** — services are the only place domain rules live, and they are unit-tested independently of HTTP.
- **A grading engine with two layers**: a coordinator that owns scoring rules and a registry of per-question-type strategies. Strategies are **N+1-free by design** — the coordinator batch-loads every lookup a strategy might need into a `GradingContext`, and strategies never issue database queries.
- **Cross-cutting concerns are utilities, not layers**: RBAC decorators, an idempotency decorator, a nestable transaction context manager, and a reusable state-machine mixin.

---

## Domain model

14 tables, versioned through Alembic migrations (12 domain tables + token blacklist + idempotency keys).

```
users ──< exams ──< questions ──< options
  │         │  ^        │
  │         │  └─ root_exam_id (self-referential → version family)
  │         │
  │         └──< attempts (state machine) ──< answers ──< answer_options (multi-select)
  │                  │                              │
  │                  │                              ├──< answer_evaluations ──< evaluation_criterion_scores
  │                  │                              │
  │                  │                              └── (Answer.selected_option_id → options)
  │                  │
  │                  └── (timing telemetry: started_at, submitted_at, paused_at, total_paused_seconds)
  │
  ├──< idempotency_keys
  ├──< audit_logs (immutable)
  └──< token_blocklist

questions ──< rubrics ──< rubric_criteria      (grading contract for a question)
```

| Table | Purpose |
|-------|---------|
| `users` | Accounts with role (`student` / `teacher`); email/username unique |
| `exams` | Assessment config: title, published flag, `max_attempts`, timing/access rules, `multi_select_penalty`, `exam_type`, `root_exam_id` (version family) |
| `questions` | Question text, `question_type`, `points`, `weight`, `is_bonus`, `requires_manual_review`, `locked_at`, `deleted_at` (soft delete) |
| `options` | Answer options with `is_correct` flag |
| `attempts` | One row per attempt: `status` (state machine), timing telemetry, `score` |
| `answers` | One row per (attempt, question) — unique constraint; single-select via `selected_option_id`, free-text via `text` |
| `answer_options` | Join rows for multi-select selections (unique per answer+option) |
| `rubrics` | Grader-agnostic grading contract for a question (one-to-one) |
| `rubric_criteria` | Evaluation axes with `label`, `max_points` (fractional), `position` |
| `answer_evaluations` | Durable per-answer grading result (grader-agnostic `mechanism`); `pending` vs `graded` |
| `evaluation_criterion_scores` | Per-criterion awarded points + snapshots of the criterion definition |
| `idempotency_keys` | Enforces exactly-once semantics with a 24h TTL |
| `audit_logs` | Immutable, append-only record of sensitive actions |
| `token_blocklist` | Revoked refresh-token `jti` values |

### Notable design properties

- **Version families via `root_exam_id`** (self-referential foreign key). A family is the root plus every exam whose `root_exam_id` points at it. `get_family_exam_ids()` resolves the full lineage from any member. Attempt limits are enforced across the family; in-progress attempts stay frozen to the version they started.
- **Copy-on-write versioning.** Published exams are immutable. Editing one deep-copies questions, options (including correctness), and grading configuration (weight, bonus flag, manual-review flag, and rubric + criteria) into a new draft version. The old version remains live and gradable while the new draft is being edited.
- **Grading results are snapshotted, not recomputed.** `evaluation_criterion_scores` stores the criterion label / max points / position at grade time, so a grade remains historically reproducible even if the rubric later evolves. Rubrics themselves ride on exam versioning (published content is immutable).
- **Pending is not zero.** A manual answer awaiting review has `total_awarded = NULL` and is never presented as a numeric score. An attempt stays `SUBMITTED` while any manual evaluation is pending.
- **Soft delete only where audit matters.** Questions with student answers are soft-deleted; ungraded questions are hard-deleted. Exams have no soft-delete column.

---

## Assessment lifecycle

Attempt statuses are enforced by a **state machine** (`StateMachineMixin`) at the ORM level — assigning an illegal status raises `InvalidTransitionError` before anything hits the database.

```mermaid
stateDiagram-v2
    [*] --> in_progress : start / resume
    in_progress --> paused : pause (if allow_pausing)
    paused --> in_progress : resume
    in_progress --> submitted : submit
    in_progress --> submitted : lazy expiry
    paused --> submitted : lazy expiry
    submitted --> graded : grading complete
    graded --> archived : archive
    archived --> [*]
```

Lifecycle rules worth calling out:

- **Resume over restart.** A student with an `in_progress` or `paused` attempt anywhere in the family resumes it (after an expiry check) rather than starting a new one. Availability-window and access-code gates apply only to *new* starts — a frozen attempt is exempt.
- **Lazy expiry, no scheduler.** On any read/write, an attempt whose active time is exhausted or whose wall-clock deadline has passed is atomically finalized (`submitted_at = now`) and graded in the same transaction. An expired attempt can never be resurrected.
- **Pause stops the active clock, not the wall clock.** `total_paused_seconds` accumulates paused intervals; a configured `deadline_at` keeps advancing while paused.
- **Double submission is provably impossible.** Submission requires the `in_progress` status, is re-checked inside the transaction, and the route is idempotent — a retried request cannot double-grade, and a second submit after grading returns 409.
- **Manual review blocks finalization.** Grading a submitted attempt with any pending manual answer keeps it `SUBMITTED`; the final pending manual evaluation transitions it to `GRADED`.

### Timing model

Assessment configuration lives on the exam; attempt telemetry lives on the attempt:

- **`Exam.duration_minutes`** — active-time limit (`NULL` = untimed).
- **`Exam.deadline_at`** — absolute wall-clock cutoff (`NULL` = none).
- **`Attempt.started_at` / `submitted_at`** — `submitted_at` is always the *real* finalization timestamp, never the configured deadline.
- **`Attempt.paused_at` / `total_paused_seconds`** — drive derived `total_elapsed_seconds` and `active_time_seconds`.

Effective remaining time is the smaller of active-duration remaining and wall-clock remaining; `NULL` for a fully untimed attempt; `0` once expired.

---

## Grading engine

Grading is split into two responsibilities:

1. **Strategies** (one per question type) produce a *neutral* verdict — exact-match correctness plus, for multi-select, raw counts. They never award points and never query the database.
2. The **coordinator** owns all scoring rules and applies them in a fixed, documented order.

### Per-question scoring pipeline

```
raw performance
  → base / partial score        (all-or-nothing, or multi-select partial credit)
  → multi-select penalty        (exam-level, floored at 0, multi-select only)
  → weight multiplier           (effective = base × weight)
  → bonus constraint            (contribution clamped ≥ 0)
  → exam aggregation
```

### Grading features implemented

| Feature | Behavior |
|---------|----------|
| **Multiple choice / true/false** | All-or-nothing: `points` if correct, else `0`. |
| **Multi-select partial credit** | `fraction = max(0, selected_correct − selected_wrong × penalty) / total_correct`, clamped to `[0, 1]`; awarded as `points × fraction`. The penalty is an **exam-level numeric policy** (`multi_select_penalty`, default `0.0` = no deduction). A `0` penalty is valid and triggers an advisory warning, never a rejection. |
| **Weighted questions** | `effective_points = points × weight` (weight > 0, default `1.0`); applies to normal and bonus questions. |
| **Bonus questions** | Excluded from the normal denominator; earned points are *added* to the score; a bonus can never reduce a score (contribution clamped ≥ 0). Percentage is intentionally **not capped at 100** — a perfect normal score plus earned bonus can exceed 100%. |
| **Manual grading** | Open-ended answers are graded against a rubric. Rubric scores are normalized onto the question's points (`rubric_fraction = total_awarded / Σ max_points`), then weight/bonus rules apply unchanged. |
| **Per-answer evaluation** | One durable `AnswerEvaluation` per answer (`answer_id` UNIQUE), grader-agnostic via a `mechanism` field (`manual` today; `ai` reserved). Criterion scores are snapshotted for historical reproducibility. |

### Manual grading flow

1. A question marked `requires_manual_review` accepts free-text answers instead of option selections.
2. On submit, a durable `pending` evaluation is created — the attempt stays `SUBMITTED`.
3. The teacher pulls `GET /grading/exams/{id}/grading-queue` (oldest-submitted first, teacher-owned exams only).
4. The teacher posts per-criterion `awarded_points` (0 ≤ awarded ≤ max, fractional allowed) to `POST /grading/answers/{id}/grade`. The server derives the total, validates every criterion (missing / unknown / duplicate / out-of-range → 422), persists evaluation + snapshots, and re-grades the attempt **in one transaction**. The grade route is idempotent.
5. Grading the final pending answer transitions the attempt to `GRADED`.

Unanswered manual questions score `0` (never "pending"). Regrading in place is allowed (last-writer-wins); immutable override history is a planned follow-up.

---

## Reliability engineering

Beyond CRUD, the project invests in correctness and auditability:

- **Idempotency.** The `@idempotent` decorator (client-supplied `Idempotency-Key` header) gives exactly-once semantics for answer submission, attempt submission, and manual grading. A unique (user, key, endpoint) constraint means concurrent retries resolve to a single winner; responses are replayed from storage for 24h; stale in-progress locks are cleaned by a CLI command.
- **Atomic transactions.** The `atomic()` context manager is nestable — only the outermost call commits/rolls back — so multi-step operations (e.g., submit + grade + audit) commit or fail as a unit.
- **State-machine enforcement.** Illegal status transitions raise at the ORM level, not just at the service layer.
- **Immutable audit log.** `audit_logs` rows cannot be updated or deleted: SQLAlchemy `before_update`/`before_delete` listeners raise, and database triggers `RAISE(ABORT)` even against raw SQL. Currently wired for attempt submission and manual grading.
- **Batch-loaded grading.** The coordinator loads options and selections in a fixed number of queries; strategies never cause N+1 queries.
- **Ownership + role checks.** Every mutating route enforces exam ownership *and* role (teacher for authoring/grading, student for attempts).

---

## API surface

All endpoints are prefixed with `/api/v1`.

| Area | Endpoint | Purpose |
|------|----------|---------|
| **Auth** | `POST /auth/register`, `POST /auth/login` | Register (role student/teacher), log in → JWT pair |
| | `POST /auth/refresh`, `POST /auth/logout` | Refresh access token, revoke refresh token |
| | `GET /auth/me` | Current user profile |
| **Exams** | `GET /exams`, `POST /exams` | List (teacher's) exams, create draft |
| | `GET /exams/{id}`, `PUT /exams/{id}` | Read; update (or auto-create version if published) |
| | `POST /exams/{id}/publish`, `POST /exams/{id}/version` | Publish with validation; explicit version clone |
| | `POST /exams/{id}/questions`, `PUT /exams/{id}/questions/{qid}`, `DELETE /exams/{id}/questions/{qid}` | Question CRUD |
| | `POST /questions/{qid}/options`, `PUT /questions/{qid}/options/{oid}` | Option CRUD |
| **Attempts** | `POST /exams/{id}/attempts`, `GET /exams/{id}/attempts` | Start (or resume), list attempts |
| | `GET /attempts/{id}`, `GET /attempts/{id}/result` | Attempt detail, graded result with per-question feedback |
| | `POST /attempts/{id}/answers` | Submit an answer (idempotent) |
| | `POST /attempts/{id}/pause`, `POST /attempts/{id}/resume` | Pause / resume (per exam policy) |
| | `POST /attempts/{id}/submit` | Submit for grading (idempotent) |
| **Grading** | `GET /grading/exams/{id}/grading-queue` | Pending manual-review answers (teacher) |
| | `POST /grading/answers/{id}/grade` | Persist rubric scores (teacher, idempotent) |
| **Health** | `GET /health` | Liveness check |

---

## Testing and engineering quality

The test suite is the project's primary quality gate. I run it as part of every milestone.

- **393 tests pass** (plus 23 state-machine subtests), run with `pytest` against in-memory SQLite with foreign keys enabled. Verified on the current tree.
- **Organization:**
  - `tests/factories/` — factory functions for users, exams, questions, options, attempts, answers.
  - `tests/scenarios/` — reusable end-to-end scenario builders (full exam-taking flow).
  - `tests/concurrency.py` — a `threading.Barrier`-based harness that fires the same call simultaneously and asserts exactly-one-winner outcomes (e.g., 10 concurrent submits → exactly 1 graded).
  - `tests/base.py` — shared `BaseTestCase` wiring the app factory, in-memory DB, and teardown.
- **Major coverage areas:**

| Area | What it pins down |
|------|-------------------|
| **Auth** (34 tests) | Registration validation, login, refresh, logout/blacklist, profile |
| **Attempt lifecycle** (28 tests) | Start/resume, pause/resume, lazy expiry (all four timing cases), double-submit protection |
| **Attempt service** (12 tests) + **concurrency** (6 tests) | Family-aware behavior, exactly-one-graded under race |
| **Grading — multi-select** (37 tests) | Scoring matrix, partial credit, penalty, payload rules, publish validation |
| **Grading — manual** (34 tests) | Pending-vs-zero semantics, rubric normalization, queue, grade validation, idempotent regrade, SUBMITTED → GRADED transition |
| **Grading — weight/bonus** (12 tests) | Order-of-operations, bonus never negative, percentage not capped |
| **Grading — rubric** (11 tests) | Model relationships, version-copy preservation |
| **Assessment rules** (32 tests) | Availability window, access code, family-wide max attempts, frozen-at-start |
| **Exam service** (35) + **question service** (17) + **exam routes** (13) | CRUD, publish validation, copy-on-write versioning |
| **Idempotency** (11 tests) | Exactly-once, replay, stale-lock cleanup |
| **Transactions** (7 tests) | Nestable all-or-nothing semantics, rollback on failure |
| **RBAC** (11 tests) | Role gates at decorator and route level |
| **Audit log** (10 tests) | Append-only enforcement at ORM and trigger level |
| **Model tests** (82 tests) | Per-model constraints, relationships, soft delete, state machine |

The suite includes regression tests for bugs found during manual QA (null-crash handling, version-title overflow, N+1 query elimination, dropped grading config during versioning).

---

## AI-first direction

**Current AI implementation: none.** There is no LLM integration, no agent framework, no retrieval, no embeddings, no AI grading — the `agents/` module is intentionally empty. Phase 2 of the project is deliberately AI-free: it exists to build the assessment engine, the reliability layer, and clean structured data that future AI can safely operate on.

What *does* exist today is an **AI-ready design** — architectural choices made so AI can plug in without rework:

- **Grader-agnostic evaluations.** `AnswerEvaluation.mechanism` is `"manual"` today and reserves `"ai"` for a future grading agent; the persisted result shape is identical either way.
- **Rubrics as grading contracts.** A rubric defines *how a question should be evaluated* independent of who (or what) evaluates. An AI grading agent can consume the same contract a teacher uses today.
- **An audit log built for explainability.** Every future AI decision is expected to record its reasoning — the append-only audit infrastructure exists to support that.

Planned AI stages (Phase 3, not built):

- **3.1 AI-assisted assessment authoring** — question generation from source material, difficulty estimation, quality checks (teacher-in-the-loop; nothing auto-publishes).
- **3.2 Intelligent evaluation** — AI-assisted essay/coding grading against rubrics, confidence scores, a judge-critic pattern with human approval below a confidence threshold.
- **3.3 Personalized learning** — a per-student knowledge model fed by attempt history, driving weak-concept detection and personalized practice.
- **3.4 Agent platform** — multiple purpose-built agents (Exam Reviewer, Learning Coach, Teacher Assistant, Analytics Agent) with defined tools and memory, rather than a general-purpose chatbot.
- **3.5 AI engineering** — prompt versioning, evaluation pipelines, groundedness/hallucination detection, latency and cost tracking, and human-approval workflows.

The architecture intentionally sequences this: the data model (attempts, grades, rubrics, audit) is being built *before* the AI, so the AI layer stands on clean, trustworthy data.

---

## Production and infrastructure

**What exists today:**

- Environment-based configuration (`development` / `testing` / `production`) loaded from `.env`.
- MySQL for development (PyMySQL driver); in-memory SQLite for tests.
- Alembic migrations via Flask-Migrate (12 migrations, one per schema change).
- A `/health` liveness endpoint.
- A CLI command for idempotency-key cleanup.

**Deliberately not yet integrated** (planned, not present in the codebase):

- No containerization (Docker / Docker Compose).
- No CI/CD pipeline (GitHub Actions) and no deployment to any cloud (AWS, Kubernetes, etc.).
- No background job queue (Redis / Celery) — expiry is handled lazily by design.
- No external observability (APM, metrics, structured log shipping).

This is a **backend engineering project in active development**; production hardening (containerization, CI/CD, monitoring, backups) is an explicit upcoming stage of the roadmap rather than a present capability.

---

## Technology stack

| Layer | Technology |
|-------|------------|
| Language | Python 3.10 |
| Web framework | Flask |
| ORM | SQLAlchemy (Flask-SQLAlchemy) |
| Migrations | Alembic (Flask-Migrate) |
| Auth | Flask-JWT-Extended (JWT access + refresh, HS256) |
| Password hashing | Werkzeug |
| Email validation | `email-validator` |
| Database (dev) | MySQL (PyMySQL) |
| Database (tests) | SQLite in-memory |
| Config | `python-dotenv`, environment-based |
| Testing | `pytest` (unittest-style suites, factories, scenarios, concurrency harness) |

---

## Key engineering decisions

The decisions below are the ones that most shape the system — each is documented in the project's engineering handbook and pinned by tests.

1. **Copy-on-write version families.** Published exams are immutable; edits create a new draft version linked through a self-referential `root_exam_id`. This gives free, consistent "exam version X" snapshots — attempts stay frozen to the version they started, grades stay reproducible, and the old version stays live while a new one is drafted.

2. **Assessment config vs. attempt telemetry.** What an exam *allows* (duration, deadline, pause policy, availability) lives on `Exam`; what an attempt *actually did* (started, real submitted time, paused time) lives on `Attempt`. `submitted_at` is always the real finalization timestamp. This split keeps rules changeable without corrupting historical records.

3. **A state machine enforced at the ORM level.** `StateMachineMixin` validates every status assignment via SQLAlchemy `@validates` — illegal transitions fail regardless of code path. Combined with an idempotent submit route and an in-transaction status re-check, double submission is provably impossible.

4. **Lazy expiration over a scheduler.** No background worker is needed to enforce time limits: any read/write path checks expiry and atomically finalizes+grades expired attempts. Simpler to deploy, and correct for the current single-service topology.

5. **A grading engine with a strategy registry.** Question types are pluggable (`register("multi_select")`, etc.) behind a shared verdict contract, and the coordinator owns all scoring rules. Adding a question type means adding a strategy — not editing an `if/elif` chain.

6. **N+1-free grading by construction.** The coordinator batch-loads options and selections into a `GradingContext`; strategies cannot query the database. Grading a large attempt runs in a small, fixed number of queries.

7. **Explicit scoring semantics.** Weight, bonus, and multi-select penalty each have a documented order of operations and invariants (bonus contribution ≥ 0, percentage not capped, penalty floored at 0). Ambiguity here silently corrupts scores, so every rule is pinned by tests.

8. **"Pending is not zero."** An unanswered-review manual grade is represented as `NULL`, never as a numeric 0, and the attempt stays `SUBMITTED` until every manual evaluation is graded. A student is never shown a wrong intermediate score.

9. **Grader-agnostic evaluations with snapshot history.** Evaluations carry a `mechanism` (manual today, AI reserved) and per-criterion snapshots of the rubric definition — grades remain explainable and reproducible even if rubrics change later.

10. **Immutable audit log with defense in depth.** ORM-level event guards *and* database triggers prevent tampering, so audit integrity survives raw SQL access.

11. **Exactly-once semantics via an idempotency layer.** Client keys + a unique DB constraint + 24h response replay make retry-sensitive endpoints (answer submission, attempt submission, manual grading) safe under network retries and double-clicks.

12. **Modular monolith with a strict layering rule.** Routes are thin, services hold business logic, models define schema. This keeps the codebase navigable and testable as it grows.

---

## Current status and roadmap

**Status: Phase 2 of 3, actively in progress.** Phase 1 (foundation) is complete. The Phase 2 milestones below reflect what is verified in the code and test suite.

### Phase 1 — Foundation ✅ Complete
Authentication, exam/question/option management, publish + versioning, attempts, auto-grading, and the full test suite.

### Phase 2 — Production assessment platform 🔶 In progress

| Stage | Status |
|-------|--------|
| **2.1 Assessment platform** | Prerequisites (state machine, idempotency, transactions, concurrency, RBAC, audit log) done. **2.1.1 Attempt lifecycle done** (pause/resume, expiry, telemetry). **2.1.2 Advanced grading — steps 1–6 done** (strategy pattern, partial credit, weight/bonus, rubrics, manual grading queue); **steps 7–8 pending** (grade-override history, full combination matrix). **2.1.3 Assessment rules — partial** (availability window, access code, family-wide max attempts enforced; seeded shuffle, question pools, passing score not built). **2.1.4 Integrity & audit — foundation done, wiring remains** (role gates present; broader audit wiring + immutable-submission guard pending). |
| **2.2 Publishing & version management** | Not started — scheduling, archive, version diff, rollback. |
| **2.3 Question bank** | Not started — reusable question library, tags, search. |
| **2.4 Instructor experience** | Not started — dashboard, analytics, bulk tools, exports. |
| **2.5 Student experience** | Not started — dashboard, progress, notifications. |
| **2.6 Platform services** | Not started — background jobs (Redis/Celery), caching, search, security hardening. |
| **2.7 DevOps & production** | Not started — containerization, CI/CD, monitoring, backups. |

### Phase 3 — AI-native learning platform 🔷 Planned
AI-assisted authoring, intelligent grading with human-in-the-loop, personalized learning, a multi-agent platform, and the AI engineering layer (prompt versioning, evaluation, observability). See [AI-first direction](#ai-first-direction) for detail. No Phase 3 code exists yet.

---

*Assess → Understand → Improve → Measure Again → Repeat.*

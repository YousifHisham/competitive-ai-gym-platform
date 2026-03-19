<!--
  Sync Impact Report
  ==================
  Version change: 0.0.0 → 1.0.0 (initial ratification)

  Added principles:
    - I. Modular Monolith Architecture
    - II. RESTful API Design
    - III. Code Quality & Simplicity
    - IV. Data Integrity First
    - V. Pragmatic Testing
    - VI. Explainable ML/AI Integration
    - VII. Frontend Clarity
    - VIII. Always Deployable
    - IX. Observability & Error Handling
    - X. Iterative Phased Development

  Added sections:
    - Security & Extensibility Constraints
    - Development Workflow & Quality Gates
    - Governance

  Removed sections: none
  Removed principles: none

  Templates requiring updates:
    - .specify/templates/plan-template.md        ... no update needed
    - .specify/templates/spec-template.md         ... no update needed
    - .specify/templates/tasks-template.md        ... no update needed
    - .specify/templates/commands/                ... no command files found

  Follow-up TODOs: none
-->

# Competitive AI Gym Platform Constitution

## Core Principles

### I. Modular Monolith Architecture

- The backend MUST start as a single FastAPI application with clear
  modular structure: routes, services, models, schemas.
- Concerns MUST be separated into distinct modules — no business logic
  in route handlers, no database queries in schemas.
- Design MUST accommodate future scalability, but microservice
  extraction is prohibited until a measurable bottleneck demands it.
- Premature architectural complexity MUST be justified via the
  Complexity Tracking table in plan.md.

### II. RESTful API Design

- All external interfaces MUST follow RESTful conventions with proper
  HTTP methods, status codes, and resource naming.
- Request validation MUST be separated from business logic — validation
  lives in schemas, logic lives in services.
- All API responses MUST use a consistent envelope format with proper
  HTTP status codes (2xx success, 4xx client error, 5xx server error).
- Every endpoint MUST validate all incoming data before processing.

### III. Code Quality & Simplicity

- The YAGNI principle governs all implementation decisions: do not
  build what is not yet needed.
- Code MUST be readable, maintainable, and modular — prefer explicit
  over clever.
- Unnecessary abstractions (repositories, factories, strategy patterns)
  are prohibited unless a concrete, current need justifies them.
- Premature optimization is prohibited. Optimize only when profiling
  reveals a measurable problem.

### IV. Data Integrity First

- PostgreSQL MUST be the primary data store with proper relational
  design.
- All data relationships MUST be enforced via database constraints
  (foreign keys, unique constraints, check constraints) — not
  application-level validation alone.
- Database schemas MUST be designed and reviewed before implementation
  begins for any feature that touches data.
- Migrations MUST be used for all schema changes.

### V. Pragmatic Testing

- Critical business logic (ranking algorithms, score calculations,
  matchmaking) MUST have unit tests.
- All API endpoints MUST have integration tests verifying request/
  response contracts.
- Full TDD is not required, but shipping a feature without tests is
  prohibited.
- A feature is not considered complete until its tests pass.

### VI. Explainable ML/AI Integration

- Initial models MUST start simple: linear models, rule-based systems,
  or basic statistical approaches.
- Interpretability MUST take priority over raw accuracy — every model
  prediction MUST be explainable.
- All ML/AI capabilities MUST be exposed via the REST API, never as
  standalone scripts or notebooks in production.
- Model complexity may increase only after simpler approaches are
  proven insufficient with evidence.

### VII. Frontend Clarity

- The frontend MUST be built with React.
- UI components MUST be separated from data fetching logic — no API
  calls inside presentation components.
- State management MUST remain simple (React built-in state, context).
  External state libraries (Redux, MobX) are prohibited unless a
  concrete, demonstrated need arises.

### VIII. Always Deployable

- Every completed project phase MUST produce a deployable artifact.
- No phase may end in a broken or partially integrated state.
- Deployment readiness is a quality gate for phase completion.

### IX. Observability & Error Handling

- All services MUST include basic structured logging for key
  operations.
- All errors MUST be caught, logged, and returned to the caller with
  appropriate HTTP status codes and messages.
- Silent failures (swallowed exceptions, empty catch blocks, ignored
  return values) are prohibited.

### X. Iterative Phased Development

- Development MUST proceed in phases: Backend, Data/ML, Full Stack.
- Each phase MUST be independently functional and testable before the
  next phase begins.
- Building multiple phases simultaneously is prohibited — complete one
  before starting the next.

## Security & Extensibility Constraints

- Authentication MUST use securely hashed passwords (bcrypt or argon2)
  and JWT-based token management.
- No sensitive data (passwords, tokens, secrets) may be stored in
  plain text — not in the database, not in configuration files, not in
  version control.
- The system MUST be designed for extensibility: social features,
  advanced ranking, and AI model improvements MUST be addable without
  rewriting existing modules.
- All user-facing input MUST be validated and sanitized at the API
  boundary.

## Development Workflow & Quality Gates

### Feature-Based Development

Every feature MUST follow this sequence:

1. **Understand** — read the spec, clarify ambiguities before coding.
2. **Implement** — build the feature following constitution principles.
3. **Test** — write and pass required tests (unit + API integration).
4. **Refactor** — clean up only what was touched; no drive-by
   refactoring.

### Git Discipline

- Every commit MUST be meaningful and describe what changed and why.
- One feature per commit — do not bundle unrelated changes.

### Quality Gate: Feature Completion

A feature is **not complete** unless all of the following are true:

- It works via the API (manually or programmatically verifiable).
- It has passing tests covering critical paths.
- The code is understandable by someone unfamiliar with it.
- The feature can be explained in plain language.

## Governance

- This constitution supersedes conflicting practices. When a team
  practice contradicts a principle above, the constitution wins.
- Amendments are permitted only if they improve clarity, scalability,
  or learning value **and** do not introduce unnecessary complexity.
- Every amendment MUST include: the change, the rationale, and an
  updated version number following semantic versioning.
- Simplicity MUST be preferred over sophistication in all governance
  decisions.
- All major architectural or principle changes MUST be documented in
  README or architecture notes.
- Compliance with this constitution MUST be verified during code
  review.

**Version**: 1.0.0 | **Ratified**: 2026-03-20 | **Last Amended**: 2026-03-20

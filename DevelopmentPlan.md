# Longreach Aviation: Application Architecture and Development Plan


## 1. Context and findings

- The database project defines a single PostgreSQL schema, `longreach`, with UUID primary keys. The database has not been created yet, but create scripts and insert scripts exist.
- The schema is person-centric. `persons` is the hub. `candidates` and `instructors` are role profiles of a person (unique `person_id`). All CV data (licences, ratings, flight and aircraft experience, medicals, qualifications, language assessments, documents, notes, preferred aircraft, employment history) hangs off `person_id`.
- Status history is written by DB triggers for organizations, positions, candidates, instructors, applications and training requests. The triggers read the transaction-local settings `app.actor` and `app.change_reason`.
- Login accounts live in `user_accounts` (see section 5). Candidate and instructor accounts link to a person; staff accounts do not.
- The package names will be `aero.longreach`.
- Local tooling: Java 21 and Maven are installed. Node, npm, Docker and psql are not.

## 2. Decisions made

| Topic | Decision |
|---|---|
| Authentication | Spring Security with our own `user_accounts` table |
| Session model | Server-side sessions (Spring Session JDBC) in an HttpOnly, Secure, SameSite cookie, with CSRF protection |
| Roles | `candidate`, `instructor`, `consultant`, `admin` (stored lowercase, mapped to `ROLE_…` authorities in Java) |
| Airline users | No airline login. Organizations and contacts are plain data managed by consultants |
| Hosting topology | One domain: the SPA and `/api` sit behind a single reverse proxy (required for cookie sessions) |
| Tooling | Install Node 22 LTS and Docker (or Colima) with Homebrew |
| First-release scope | Public job board in. Candidate-to-position matching deferred (see Phase 6) |
| Schema ownership | The `database` project is the single source of truth. The database is new, so there are no migrations and no migration tool. Schema changes are edits to the create scripts |
| Repositories | `longreach-backend` (needs `git init`), a new `longreach-frontend`, and the existing `database` |
| UI library | MUI |
| Schema revisions | Applied directly to the create scripts before any application code (see section 10) |
| Open | Cloud provider and any data-residency constraint (needed before Phase 7) |

## 3. Architecture

A modular monolith: one Spring Boot REST API, one PostgreSQL database, one React single-page app.

```
Browser (React SPA) --HTTPS/JSON--> Reverse proxy --> Spring Boot API --JDBC--> PostgreSQL (schema "longreach")
                                                           |
                                                           +--> Object storage (documents)
                                                           +--> SMTP (verification and reset emails)
```

Technical decisions:

- **Modular monolith, not microservices.** The data is tightly relational and the team is small. Splitting services would turn foreign keys into distributed-consistency problems. Module boundaries are kept clean (enforced by ArchUnit) so a module can be split out later.
- **Java 21 LTS and Spring Boot 4.x** (Maven wrapper). Boot 3.5 is at or past end of open-source support. The exact GA version is confirmed when scaffolding.
- **Layers per module:** Controller (HTTP and DTOs), Service (transactions, rules, authorization), Repository, Entity. Controllers never expose entities.
- **The application does not manage the schema.** It uses Spring Data JPA with `ddl-auto=validate`, so Hibernate only verifies at startup that entities match the tables the scripts created. DB-managed columns (`created_at`, `updated_at`) are mapped as non-insertable and non-updatable. Search and reporting queries use JPA Specifications or native queries.
- **Integration tests use a real PostgreSQL (Testcontainers), not H2,** built by running the `database` project's own `create_all.sql` and `insert_all.sql` through `psql` in the container's init step. This means tests exercise exactly the scripts that will build production, including the partial unique indexes, triggers, `set_config` and expression indexes that H2 cannot run.
- **Audit trigger contract.** An `AuditContext` component, invoked by an aspect on write service methods, runs `SELECT set_config('app.actor', <user UUID>, true)` (and `app.change_reason`) as the first statement of every write transaction. Transaction-local is required so values cannot leak between requests on pooled connections. Every status-changing endpoint has a test that checks the history row's actor and reason.
- **Row history (`latest_record`).** The DB enforces one current row through partial unique indexes but nothing flips the flag. A shared `VersionedRecordService` does, in one transaction: set the old row `latest_record = false`, then insert the new row (order matters, otherwise the unique index fails).
- **Status workflows live in Java.** The DB has lookup tables but no transition rules. Each workflow is an enum with a transition map. A test asserts the enums match the seeded lookup rows.
- **DTOs** are Java records mapped with MapStruct. No Lombok on entities (generated `equals`/`hashCode` is a known JPA pitfall). Candidate-facing and staff-facing DTOs are separate.

## 4. Backend package structure (package-by-feature)

Package-by-feature keeps each feature's code together and lets internals stay package-private. The feature packages mirror the `domains/` folders in the database project.

```
aero.longreach
├── App.java
├── common/        error handling (RFC 9457 ProblemDetail), paging, audit context, row-history helper, config
├── security/      SecurityConfig, role model, current-user resolver, ownership checks
├── auth/          login/logout, registration, email verification, password reset
├── user/          user accounts, roles, admin user management
├── reference/     countries, regions, aircraft types, manufacturers, job roles, licence/qualification/document types
├── person/        persons, contact details
├── organization/  organizations, contacts, status history
├── recruitment/   positions, requirements, status history, public job board
├── profile/       employment, licences, ratings, flight/aircraft experience, medicals,
│                  qualifications, language assessments, documents, notes, preferred aircraft
├── candidate/     candidates, preferences, status history
├── instructor/    instructors, status history
├── application/   applications, status workflow, history
├── training/      training types/sub-types, requests, history
└── simulator/     simulator types, simulators, sessions
    (each feature: api/ service/ domain/ repository/)
```

## 5. Security requirements

The system holds dates of birth, nationality, passport and licence scans and medical certificate data (likely health data under GDPR).

**Accounts (tables created by scripts in the `database` project):**

| Table | Purpose |
|---|---|
| `roles` | Lookup of role codes |
| `user_accounts` | `user_id`, `email` (unique, case-insensitive), `password_hash`, optional unique `person_id`, `enabled`, `email_verified_at`, `failed_login_count`, `locked_until`, `last_login_at`, `password_changed_at`, timestamps |
| `user_account_roles` | `(user_id, role_code)` with `granted_by` and `granted_at`; a user may hold several roles |
| `user_token_types` | Lookup: `email_verification`, `password_reset` |
| `user_tokens` | Single-use expiring tokens; only a hash of the token is stored |

Role-to-person consistency (a candidate account must have a `person_id`) is enforced by the application, not the database.

**Authentication (we own these):**

- Passwords: Spring's `DelegatingPasswordEncoder` (bcrypt by default; the algorithm is stored in the hash so it can be upgraded later without touching the schema).
- Password policy: minimum 12 characters and a common-password block (NIST-style, no character-class rules).
- Brute-force defence: per-account lockout plus per-IP rate limiting.
- No account enumeration: login, registration and "forgot password" respond identically whether or not the email exists.
- Registration requires email verification before the account can log in.
- Sessions: ID rotation on login, idle and absolute timeouts, server-side invalidation on logout.
- MFA (TOTP) for `consultant` and `admin` is a go-live requirement, delivered in Phase 7 at the latest.

**Authorization:**

- Method security with `@PreAuthorize` plus object-level ownership checks on every person-scoped endpoint. UUIDs are not access control.
- Field-level separation: candidates never see `consultant_notes`, the `notes` table or `changed_by`.
- `changed_by` in the history tables stores the `user_id` UUID, never an email.

**Other controls:**

- Documents in object storage (the DB keeps only `storage_path`); MIME and size checks, malware scan, authorized streaming downloads.
- TLS and HSTS, strict CORS, security headers, parameterised queries only.
- Rate limiting and CAPTCHA on public registration.
- Secrets from environment or a secret manager, never `application.properties`.
- Two database roles: an owner role used manually to run the create scripts, and a least-privilege application role with only the table privileges it needs (a grants script is added to the `database` project).
- No PII in logs; dependency scanning in CI.
- Privacy: consent capture, retention rules and an erasure/anonymisation design (see section 10).

## 6. REST API (`/api/v1`, JSON, UUID ids)

Conventions: paging with `?page&size&sort`, filters as query params, Bean Validation, errors as `application/problem+json`, OpenAPI generated by springdoc. Status changes are `POST …/status-changes` sub-resources because a change is an event that carries a reason and creates a history row.

| Area | Endpoints |
|---|---|
| **Auth** | `POST /auth/login`, `POST /auth/logout`, `GET /auth/csrf`, `POST /public/registrations`, `POST /auth/email-verifications`, `POST /auth/password-reset-requests`, `POST /auth/password-resets` |
| **Me** | `GET /me`, `GET /me/candidate`, `GET /me/applications` |
| **Admin users** | `GET/POST/PATCH /admin/users`, `PUT /admin/users/{id}/roles` (admin only) |
| **Public** | `GET /public/positions`, `GET /public/positions/{id}` (active positions only, no authentication) |
| **Reference** | `GET /reference/{countries, countries/{cc}/regions, aircraft-types, manufacturers, job-roles, licence-types, licence-rating-types, qualification-types, document-types, employment-types, training-types}`; admin `POST/PUT` on the editable ones |
| **Persons** | `GET/POST /persons`, `GET/PUT /persons/{id}`, `PUT /persons/{id}/contact-details` |
| **Organizations** | CRUD `/organizations`; `POST …/status-changes`; `GET …/status-history`; CRUD `…/contacts` (setting a new primary demotes the old one in the same transaction) |
| **Positions** | CRUD `/positions` (filters: status, organization, role, aircraft, country); `POST …/status-changes`; `GET …/status-history`; CRUD `…/requirements`; `GET …/applications` |
| **Candidates** | `GET /candidates` (staff, rich filters); `GET/PATCH /candidates/{id}`; `POST …/status-changes`; `GET …/status-history`; CRUD `…/preferences` |
| **Profile (person-scoped)** | Under `/persons/{personId}/`: `employment-history`, `licences` (+ `licences/{id}/ratings`), `flight-experience` (+ `/history`), `aircraft-experience`, `language-assessments`, `medical-certificates`, `qualifications`, `preferred-aircraft`, `documents` (multipart upload and authorized download), `notes` (staff only). `GET` returns current rows; `?includeHistory=true` returns all rows |
| **Applications** | `POST /applications`; `GET /applications` (staff); `GET /applications/{id}`; `PATCH …/consultant-notes` (staff); `POST …/status-changes` (staff transitions; candidates may only withdraw); `GET …/status-history` |
| **Training** | `POST/GET /candidates/{id}/training-requests`; `GET /training-requests` (staff); `POST …/status-changes`; `GET …/status-history` |
| **Simulator** | CRUD `/simulator-types`, `/simulators`; `GET /simulator-sessions?from&to&simulatorId`; `POST` to schedule; `POST …/status-changes` |
| **Instructors** | CRUD `/instructors`; `POST …/status-changes`; `GET …/status-history` |
| **Staff tools (later)** | `GET /dashboard/summary`, `GET /compliance/expiring?within=90`; `GET /positions/{id}/matching-candidates` (deferred) |

## 7. Frontend structure

React 19, TypeScript and Vite, as one SPA with three areas: public (job board, register, login), candidate portal and staff console.

```
src/
├── app/         router, providers, layout shells, route guards
├── api/         client generated from the OpenAPI spec (types cannot drift from the backend)
├── auth/        login, registration, reset forms; session via cookie (credentials: 'include' + CSRF header); route guards read GET /me
├── components/  shared UI (tables, forms, dialogs, status badges)
├── features/    one folder per backend module: candidates/, positions/, applications/, profile/, ...
│   └── each: pages/ components/ hooks/ (TanStack Query) schemas/ (Zod)
└── test/
```

- Server state: TanStack Query, no global store.
- Forms: React Hook Form with Zod.
- UI library: MUI.
- Tests: Vitest with React Testing Library, MSW for API mocks, Playwright for end-to-end.
- Local dev: the Vite dev server proxies `/api` to Spring Boot so cookies behave as they will in production.

## 8. Development and deployment approach

- **Schema.** The `database` project is the only definition of the schema. A new database is built by running two scripts once, in order: `create scripts/create_all.sql`, then `insert scripts/insert_all.sql`. To change the schema, edit the scripts and rebuild; there is no upgrade path and no migration tool.
- **Local stack (Docker Compose):** Postgres, MinIO (S3-compatible storage) and Mailpit (catches verification and reset emails). The `database` repo is mounted into the Postgres container, which runs the two scripts on first start. To reset, delete the volume.
- **Repository layout:** the repos are sibling checkouts (`../database`). The backend reads the scripts' location from a `DATABASE_SCRIPTS_DIR` setting; CI checks out both repos side by side.
- **Testing:** JUnit 5, Mockito, `@WebMvcTest` and `@DataJpaTest` slices, Testcontainers integration tests, Spring Security test support, ArchUnit. Tests run after every increment and results are reported.
- **CI:** GitHub Actions: build, test, lint, dependency scan, container image.
- **Deployment:** containerised, 12-factor. The API is a Docker image; the SPA is a static build served behind the same reverse proxy as `/api`; managed PostgreSQL with backups. Environments: dev, test, prod. Actuator for health probes and metrics; structured logs. Production database creation is a one-time manual run of the two scripts with `psql` as the owner role. Cloud provider to be decided.

## 9. Implementation phases

Each phase delivers working, tested software. After Phase 0, phases are vertical slices (backend and frontend together).

| Phase | Scope | Done when |
|---|---|---|
| **0. Foundations and schema** | Install Node and Docker. Validate the users scripts and apply the schema revisions in section 10 to the `database` project, with your review. Add the application-role grants script and update the ERD. New Spring Boot 4 project replacing the skeleton, `git init`, Testcontainers database built from the scripts, GitHub Actions | Scripts build an empty database cleanly; the app boots against it; a schema-validation test passes |
| **1. Platform core** | Security and sessions, registration, email verification, login/logout, password reset, user admin, `/me`, error handling, `AuditContext`, row-history helper, reference data API, persons. Frontend shell, login, API client | A user can register, verify and log in; audit actor verified by a DB test |
| **2. Organizations and positions** | Organizations, contacts, positions, requirements, status workflows, public job board. Staff screens | Staff manage an airline and publish a position; the public sees it |
| **3. Candidates and profile** | Candidate profile, all work-history sections, preferences, document upload. Candidate portal | A candidate can complete a CV |
| **4. Applications** | Apply, staff pipeline with enforced transitions, history, consultant notes | Full lifecycle works, with permission tests |
| **5. Training, simulator, instructors** | Training requests, instructors, simulator scheduling with double-booking protection | A session can be scheduled with no overlap |
| **6. Compliance and insight** | Expiry tracking, dashboard, email notifications; candidate matching if structured requirements are designed | Staff see expiring items |
| **7. Hardening and go-live** | MFA for staff, security review, performance, GDPR features, observability, production deployment, backup/restore drill | Production release |

## 10. Risks and schema issues

**Schema revisions, edited directly into the `database` scripts in Phase 0** (items 2, 4 and 5 need your input first):

1. **Simulator double-booking.** Nothing prevents it. Add a `btree_gist` exclusion constraint on `(simulator_id, tstzrange(scheduled_start_at, scheduled_end_at))` for active sessions.
2. **Instructors are linked to nothing.** They have no relation to simulator sessions or training. Their role needs defining.
3. **No concurrency guard.** Two consultants editing the same record can overwrite each other. Add a row counter column (for optimistic locking) to the editable tables.
4. **`latest_record` unique indexes are too strict in places.** `notes` allows one current note per `(person, note_type)`; `documents` one per `(person, document_type)`; `employment_history` cannot hold a rehire with the same employer and title.
5. **Status inconsistencies.** `'registration received'` has a space while every other status is snake_case; `simulator_sessions.session_status` is a CHECK constraint, not a lookup table.
6. **Free-text fields with no lookup:** `language_code`, `medical_class`, `note_type`, `requirement_category`.
7. **Erasure.** Deleting a `persons` row cascades to CV data, but candidates, instructors, applications and user accounts have no cascade and would block it. An erasure/anonymisation design is needed before go-live.
8. **Duplicate people.** `contact_details.primary_email` is not unique. Uniqueness is only enforced at the account level (`user_accounts.email`).

**Other risks:**

- **No upgrade path after go-live.** Once production holds real data, the create scripts cannot be re-run. Before the first production release we must decide how later schema changes are applied (a migration tool or reviewed manual scripts). Until then, editing the scripts is the right approach.
- **Matching** is deferred because `position_requirements` is free text; it needs a structured-requirements design first.
- **Same-site cookie sessions** require the SPA and API on one domain in every environment, including local dev (handled by the Vite proxy).
- **Owning authentication** means owning password storage, lockout, verification and reset. These are covered by the security requirements and tests above.
- **The users scripts are written but not yet validated** against a real PostgreSQL (done in Phase 0).

## 11. Open items

- Decisions on schema revisions 2, 4 and 5.
- Cloud provider and data-residency constraints (needed before Phase 7).
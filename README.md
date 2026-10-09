# 9CGPA backend

Didn't get a 9 CGPA? Build a 9/10 career.

Phase 6 adds personalized learning roadmaps, a leveled learning catalog, and task
progress to the existing account, profile, skill, assessment, and AI coach APIs.
Assessments and roadmaps work without an AI key or provider access. No frontend is included.

## Local development

Install Python 3.12 or 3.13 and uv, then run from this directory:

```powershell
$env:UV_CACHE_DIR = Join-Path (Get-Location) '.uv-cache'
uv sync
if (-not (Test-Path .env)) { Copy-Item .env.example .env }
# Set NINECGPA_DATABASE_URL in .env to your existing PostgreSQL database.
# Generate a new secret into the environment without printing it:
$env:NINECGPA_JWT_SECRET = uv run python -c "import secrets; print(secrets.token_urlsafe(48))"
uv run alembic upgrade head
uv run python -m app.db.seed_skills
uv run python -m app.db.seed_roles
uv run python -m app.db.seed_learning
uv run uvicorn app.main:create_app --factory --reload
```

Open `http://127.0.0.1:8000/docs` for the development API contract.
`GET /api/v1/health` returns HTTP 200 with
`{"status":"ok","service":"9cgpa-api"}`. This is a liveness endpoint,
not a database or AI readiness check. It requires no credentials.

Configure settings with `NINECGPA_` environment variables or a local `.env`.
Never commit `.env` or credentials. Production disables API documentation.
Set `NINECGPA_CORS_ORIGINS` to a JSON array of exact frontend origins if needed.
Origins must include an HTTP(S) scheme and optional port, with no wildcard,
credentials, path, trailing slash, query, or fragment. Environment variables
override `.env`; the file is read as UTF-8. Settings load when the application
factory is called, not during module import.
The database URL must use `postgresql+psycopg://`; URL-encode special characters
in credentials. Provision the database separately: migrations create tables,
not databases or database users. Migrations do not run at application startup.
Save the generated JWT secret securely for subsequent runs; generating a new one
invalidates existing tokens. Missing database/JWT settings keep health available
but make account operations fail closed with HTTP 503.

Bearer tokens are sent in the Authorization header, not cookies. CORS cookie
credentials remain disabled. The JWT secret must have at least 32 bytes; use
cryptographically random values as shown above. Defaults are HS256, issuer
`9cgpa-api`, audience `9cgpa-client`, and a 30-minute access token lifetime
(configurable from 1 to 60 minutes).

```powershell
# In restricted environments, keep pytest temporary files inside the workspace.
New-Item -ItemType Directory -Force .pytest_cache/verification-temp | Out-Null
$env:PYTEST_DEBUG_TEMPROOT = Join-Path (Get-Location) '.pytest_cache/verification-temp'
uv run pytest
uv run ruff check .
uv run ruff format --check .
uv run mypy app tests migrations
```

## Architecture and phase boundary

The established direction is REST routes -> services -> repositories/PostgreSQL.
`api` owns routes/dependencies, `schemas` owns Pydantic contracts, `services` owns
business rules and transaction commits, `repositories` owns scoped queries,
`models` owns SQLAlchemy entities, `db` owns sessions/metadata, and `core` owns
configuration, security, and error handling. Each request gets a session that is
closed on completion; failed transactions roll back. Engines dispose at shutdown.

Application code owns authorization, scoring rules, validation, and state changes.
AI capabilities produce validated structured proposals and never write directly
to the database. Each capability needs its own input/output schemas, controlled
instructions, failure handling, and tests. Company preparation must distinguish
supported facts from general practice and must not invent official processes.

Roadmaps, project generation, interviews, and coding evaluation are not implemented.
Phase 5 adds deterministic assessment/history; Phase 6 adds learning roadmaps/progress.
Phase 7 project generation has not started.
Live-provider verification requires locally configured
credentials and model access; mocked tests are not evidence of a live GPT call.

Handled application errors use `{"error":{"code":"...","message":"..."}}`
and omit submitted values and exception details. CORS preflight responses are
owned by the CORS middleware and use its standard response format.
Unexpected HTTP errors log the exception class only through Python logging;
the host process controls log handlers and destinations. The error middleware
handles these errors inside CORS, preserving CORS headers and preventing raw
exceptions from reaching the server logger. Once response headers are sent,
failures abort the response with a sanitized exception instead of sending a
second response. This is not a general log-redaction system: do not put secrets
in URLs, and configure server/proxy access logging appropriately before deployment.
Protected diagnostics and richer domain error contracts can be added with the
features that need them.

## Database and migration

`0001_accounts` creates UUID-keyed `users`, `user_profiles`, and `career_goals`.
Email is stored casefolded with a unique constraint and lowercase check.
Profiles have a unique user foreign key (one profile per account); goals have an
indexed user foreign key. Both foreign keys cascade on user deletion. Registration
creates user and profile in one transaction. Timestamps use timezone-aware
PostgreSQL columns with server creation defaults and ORM-managed update times.
Direct SQL updates must explicitly maintain `updated_at`.

Profile values are nullable. CGPA is a Decimal on a 0-10 scale with two decimal
places and serializes as a JSON string. Graduation year is 1900-2200; current
study year is 1-10. Experience level is `student`, `fresher`, or `experienced`.
These are explicit Phase 2 contract choices, not inferred assessments.

`uv run alembic current` shows the installed revision. `uv run alembic check`
checks model/migration drift after migration. Downgrading to `base` drops all
three account tables and their data; use only on a disposable database.
Docker configuration was not added. `uv.lock` is now present; retain it with the
project for reproducible dependency installation.

## Account API contract

All paths below are relative to `/api/v1`. Request bodies are JSON.

| Method | Path | Body / response |
| --- | --- | --- |
| POST | `/auth/register` | `email`, `password`; 201 user, including ID |
| POST | `/auth/login` | `email`, `password`; 200 access_token, token_type, expires_in |
| GET | `/users/me` | Authenticated user's public account fields |
| PATCH | `/users/me` | `email`, `current_password`; updated public account |
| GET | `/users/me/profile` | Profile, initially empty nullable fields |
| PATCH | `/users/me/profile` | At least one profile field; updated profile |
| POST | `/users/me/career-goals` | Required `target_role`; 201 goal |
| GET | `/users/me/career-goals` | Array; `limit` 1-100 (default 50), `offset` >= 0 |
| PATCH | `/users/me/career-goals/{goal_id}` | At least one goal field; updated goal |
| DELETE | `/users/me/career-goals/{goal_id}` | 204, empty body |

All `/users/me` routes require `Authorization: Bearer <access_token>`.
Passwords must be 12-128 characters at registration and are hashed with Argon2id.
Login uses one generic error for incorrect email, incorrect password, and inactive
accounts. JWT signature, fixed algorithm, issuer, audience, required claims,
expiry, not-before, UUID subject and token type are validated. The user must
still exist and be active on each authenticated request. Tokens contain no email,
profile information, or password hashes. Token responses disable caching.

Only email can be changed through the account PATCH, with current-password
verification. IDs, password hashes, timestamps, ownership, and activation state
are never writable through request schemas. Unknown fields are rejected.
Password changes, password reset, email verification, refresh tokens, logout/token
revocation, and administrative endpoints are outside this phase.

Profile PATCH fields: `full_name`, `university`, `degree`, `branch`,
`graduation_year`, `current_year`, `cgpa`, `bio`, `experience_level`.
Goal fields: `target_role`, `target_company`, `target_industry`,
`preferred_location`, `target_date` (ISO date). Omitted fields stay unchanged;
explicit null clears optional fields. Target role cannot be null or blank.
Goals outside the caller's ownership return the same 404 as nonexistent goals.

| Status | Error code |
| --- | --- |
| 401 | `invalid_credentials`, `authentication_required`, or `invalid_token` |
| 409 | `email_unavailable` (including concurrent duplicate writes) |
| 422 | `validation_error` |
| 404 | `not_found` for unavailable profile/goal; `http_error` for unknown routes |
| 503 | `service_unavailable` for missing account configuration |

## Tests and validation status

The normal account tests apply the Alembic migration to a fresh SQLite file;
foreign keys are enabled. SQLite is a test substitute only and is not accepted
by the application database configuration. Tests cover account flows, ownership,
JWT failures, inactive users, field validation, uniqueness, cascades, migration
upgrade/downgrade, schema drift, and PostgreSQL offline SQL generation.

For actual PostgreSQL verification, set `TEST_DATABASE_URL` to a dedicated test
database using `postgresql+psycopg://`, then run `uv run pytest` again. The account
suite creates and migrates a random `test_9cgpa_*` schema per test and drops only
that schema afterward. The test role needs CREATE SCHEMA permission. Never point
this variable at production. Without it, the explicit live-PostgreSQL check skips.
Tests isolate application settings from the developer's environment and `.env`.

Final Phase 2 verification on 2026-10-04: the development migration succeeded,
all 73 tests passed against the separate PostgreSQL test database, and Ruff lint,
format checks, and mypy passed. The live API flow, development documentation,
password storage, ownership checks, and captured-log secrecy checks also passed.
One non-blocking upstream Starlette warning remains about the httpx TestClient
integration. No warnings are suppressed. The verification server was stopped and
its two disposable accounts removed after testing.

## Manual verification

After the setup commands above, use a second PowerShell terminal. The temporary
password below is randomly generated for this local test and is not printed:

```powershell
$api = 'http://127.0.0.1:8000/api/v1'
$testPassword = uv run python -c "import secrets; print(secrets.token_urlsafe(24))"
$credentials = @{ email = 'phase2@example.com'; password = $testPassword } | ConvertTo-Json
$user = Invoke-RestMethod "$api/auth/register" -Method Post -ContentType 'application/json' -Body $credentials
$login = Invoke-RestMethod "$api/auth/login" -Method Post -ContentType 'application/json' -Body $credentials
$auth = @{ Authorization = "Bearer $($login.access_token)" }
Invoke-RestMethod "$api/users/me" -Headers $auth
$emailChange = @{ email = 'phase2-updated@example.com'; current_password = $testPassword } | ConvertTo-Json
Invoke-RestMethod "$api/users/me" -Method Patch -Headers $auth -ContentType 'application/json' -Body $emailChange
Invoke-RestMethod "$api/users/me/profile" -Headers $auth
Invoke-RestMethod "$api/users/me/profile" -Method Patch -Headers $auth -ContentType 'application/json' -Body '{"full_name":"Test Student","cgpa":"8.75"}'
$goal = Invoke-RestMethod "$api/users/me/career-goals" -Method Post -Headers $auth -ContentType 'application/json' -Body '{"target_role":"Backend engineer"}'
Invoke-RestMethod "$api/users/me/career-goals" -Headers $auth
Invoke-RestMethod "$api/users/me/career-goals/$($goal.id)" -Method Patch -Headers $auth -ContentType 'application/json' -Body '{"target_role":"AI engineer"}'
Invoke-RestMethod "$api/users/me/career-goals/$($goal.id)" -Method Delete -Headers $auth
Invoke-RestMethod "$api/users/me/career-goals" -Headers $auth
```

Expect registration/goal creation 201, reads/updates/login 200, deletion 204,
and an empty final goal list. Confirm no response includes password/hash fields.
Swagger `/docs` also exposes all contracts and its Authorize button accepts the
login access token.

Negative checks in `/docs`: repeat registration with the updated email (409),
login with a wrong password (401), clear authorization and GET `/users/me` (401),
authorize with an invalid token (401), PATCH CGPA to 11 (422). Register/login a
second user, create a goal as the first user, and try updating/deleting that ID
as the second user: both must return 404, and the second user's goal list must
exclude it. To verify expiration, start the server with
`NINECGPA_ACCESS_TOKEN_MINUTES=1`, obtain a fresh token, wait over a minute, and
confirm GET `/users/me` returns 401. Restore the setting afterward.


## Phase 3 skill system

The catalog is the normalized source of truth for skill names and categories.
`skill_categories` owns categories; `skills` references a category; `user_skills`
references a user and skill. User skill records do not duplicate catalog names.
Migration `0002_skills` adds these tables without changing `0001_accounts`.

Proficiency is a shared enum with a database CHECK constraint:
`BEGINNER`, `ELEMENTARY`, `INTERMEDIATE`, `ADVANCED`, `EXPERT`.
Confidence is a separate strict integer from 0 to 100, also constrained in the
database. `ADVANCED` with confidence 55 means the user reports advanced knowledge
but moderate confidence in that report. These are self-reported values, not
verified assessment scores. Evidence is optional plain text, at most 5,000
characters. `last_assessed_at` is read-only, initially null, and untouched by
self-report updates. No assessment or AI execution exists in this phase.

The database enforces one record per `(user_id, skill_id)`, one category name,
and case-insensitive skill-name uniqueness within each category. Category names
are uppercase and trimmed. Foreign keys cascade on account deletion, but restrict
catalog deletion while dependent entries exist. The user-skill unique index also
supports user-specific queries; skill/category foreign keys have lookup indexes.

### Catalog seeding

```powershell
uv run alembic upgrade head
uv run python -m app.db.seed_skills
# Optional custom catalog using the same validated JSON structure:
uv run python -m app.db.seed_skills --catalog path/to/catalog.json
```

The default data lives only in `app/db/skill_catalog.json`: seven categories and
38 skills covering PROGRAMMING, DSA, CS_FUNDAMENTALS, DATABASES, DATA, AI_ML, and
ENGINEERING. To add entries, extend that file or provide a custom catalog file;
no application logic change is needed. The entire catalog is validated before
writes. Seeding inserts missing entries in one transaction using database conflict
handling, so repeated runs preserve existing IDs, metadata, inactive flags, and
user records. It does not rename, remove, move, reactivate, or overwrite existing
entries. Failed runs roll back. Seeding is explicit, not performed on API startup.

### Skill endpoints

All paths are relative to `/api/v1`. All responses use the existing JSON/error
conventions. List endpoints accept `limit` (1-100, default 100) and `offset`
(nonnegative, default 0), with deterministic ordering.

| Method | Path | Access / behavior |
| --- | --- | --- |
| GET | `/skills` | Public; only active skills in active categories |
| GET | `/skills/categories` | Public; active categories |
| GET | `/users/me/skills` | Bearer authentication; only caller's records |
| PUT | `/users/me/skills` | Bearer authentication; atomic batch create/update |
| DELETE | `/users/me/skills/{skill_id}` | Bearer authentication; removes caller's association, 204 |

PUT accepts 1-100 unique skill IDs. It updates only supplied associations and
leaves omitted associations unchanged. Each supplied item requires proficiency
and confidence; omitted or null evidence clears its evidence. The response is
an array of the submitted records in request order. Unknown or inactive skills
(including skills in inactive categories) return 404 and roll back the entire
batch. Invalid bodies, enum values, duplicate IDs, non-integer/out-of-range
confidence, and attempted ownership/assessment fields return the existing 422
validation error. IDs are UUIDs. Repeated PUT calls do not create duplicates.

```json
{
  "skills": [
    {
      "skill_id": "COPY-A-SKILL-UUID-FROM-GET-SKILLS",
      "proficiency": "ADVANCED",
      "confidence": 55,
      "evidence": "Built FastAPI and machine learning projects"
    }
  ]
}
```

A returned record contains `skill` (`id`, `name`, category name, and effective
`is_active`), `proficiency`, `confidence`, `evidence`, and `last_assessed_at`.
Previously assigned inactive entries remain readable and removable, with
`skill.is_active=false`; they cannot be assigned or updated through PUT.
DELETE takes the catalog skill ID, not the association ID. A missing caller-owned
association returns 404, regardless of whether another user has that skill.

### Phase 3 verification and manual walkthrough

Verification on 2026-10-05: migration and model-drift checks passed against the
development database. Both seed runs preserved all Phase 2 records, and the second
seed preserved catalog IDs and timestamps. The full PostgreSQL suite passed all
113 tests, including all 73 prior-phase tests. SQLite fallback passed 112 tests
with its live-PostgreSQL check skipped. Ruff, formatting, and mypy passed.
Read-only checks against the configured development catalog and OpenAPI passed.
The upstream test-client deprecation and SQLite expression-index reflection
warnings remain visible; no errors or warnings were suppressed.

Run the quality gates with the PostgreSQL test URL configured in the test process:

```powershell
uv run pytest
uv run ruff check .
uv run ruff format --check .
uv run mypy app tests migrations
uv run alembic check
```

The existing test fixture creates and removes only isolated random schemas in the
separate test database. Tests do not seed or delete development data. The SQLite
fallback remains available by leaving `TEST_DATABASE_URL` unset; its live-PostgreSQL
check skips, and SQLite cannot reflect expression indexes for migration comparison.
PostgreSQL is the authoritative database for full schema verification.

To test manually after migration and seeding:

1. Start `uv run uvicorn app.main:create_app --factory --reload` and open
   `http://127.0.0.1:8000/docs`.
2. GET `/api/v1/skills/categories` and `/api/v1/skills`; expect seven categories
   and 38 skills on a fresh seed. Copy Python's skill UUID.
3. Register/login using the existing authentication endpoints. Enter the returned
   token in Swagger's Authorize dialog; do not share credentials or tokens.
4. PUT `/api/v1/users/me/skills` with the sample body above and the copied UUID.
   Expect 200, ADVANCED/55, the supplied evidence, and null `last_assessed_at`.
5. GET your skills; verify persistence. PUT again with confidence 72 and ensure
   one Python association remains. Add another skill, then PUT only Python and
   verify the other association remains.
6. Try confidence -1 or 101, an invalid proficiency, a malformed UUID, duplicate
   IDs, or `last_assessed_at` in the body: expect 422. A valid but nonexistent
   skill UUID returns 404, without partially applying other items in the batch.
7. Clear authorization and try GET/PUT/DELETE user skills: expect 401.
8. Login as a second account. Its list must exclude the first account's records;
   deleting a skill assigned only to the first account must return 404. Assigning
   that skill to the second account must leave the first account unchanged.
9. As the owner, DELETE your Python association; expect 204. Verify it is absent
   from GET; repeating the DELETE must return 404.
10. Run the seed again; catalog IDs and existing user associations must remain
    unchanged. The catalog itself must not be deleted by user-skill deletion.

## Phase 4: AI career coach

The request path is `api/ai.py` -> `services/career_coach.py` ->
`ai/orchestrator.py` -> `AIProvider` -> `OpenAIResponsesProvider`.
Routes authenticate and delegate. The service builds context through existing
repositories and releases its read transaction before a network request.
The orchestrator applies the versioned prompt and validates/redacts the answer.
`AIProvider`, `AIRequest`, `AIMessage`, and `AIResponse` are typed contracts;
another adapter can implement the protocol without changing the service.
The provider receives no session, repository, credentials from authentication,
or database mutation tools. Existing migrations and account/skill APIs are preserved.

### Configuration and official API

The adapter uses the documented `POST https://api.openai.com/v1/responses`
REST API through HTTPX, which was already installed for tests and is now a runtime
dependency. No unofficial SDK or guessed endpoint is used. The optional SDK could
not be installed in this restricted environment; it is not required by this implementation.
See the official [GPT-6 Astra model documentation](https://developers.openai.com/api/docs/models/gpt-6-astra)
and [Structured Outputs guide](https://developers.openai.com/api/docs/guides/structured-outputs?api-mode=responses).
Model availability and account access still require a successful live call.

| Setting | Default / behavior |
| --- | --- |
| `NINECGPA_AI_PROVIDER` | `disabled`; set `openai` to enable |
| `NINECGPA_AI_MODEL` | Unset; configure `gpt-6-astra` for Astra |
| `NINECGPA_AI_API_KEY` | Unset; supply locally or through a secret manager |
| `NINECGPA_AI_TEMPERATURE` | Unset; only configure for compatible models |
| `NINECGPA_AI_MAX_TOKENS` | 2048; allowed 128-16384 output tokens, including reasoning |
| `NINECGPA_AI_TIMEOUT` | 30 seconds; allowed 1-120, HTTP connect/read/write/pool timeouts |
| `NINECGPA_AI_PROMPT_VERSION` | `career-coach-v1` |
| `NINECGPA_AI_REQUESTS_PER_MINUTE` | 5 per user per process; allowed 1-60 |

Leave temperature unset for Astra: its reasoning modes do not support the
non-reasoning temperature control. The adapter fails closed for that combination.
See [model parameter compatibility](https://developers.openai.com/api/docs/guides/latest-model).
The timeout is an HTTP operation/inactivity limit, not an end-to-end deadline.
The client performs no automatic retries, does not follow redirects, and ignores
proxy environment variables. Provider clients close at application shutdown.
Missing/disabled AI configuration returns a sanitized 503; it does not prevent
the existing backend from starting. Configuration is loaded once by the app factory.
Use only model identifiers in the model setting, never credentials.

### Endpoint and context

`POST /api/v1/ai/career-coach` requires the existing Bearer JWT.
Input is a single trimmed message of 1-4000 characters; extra fields, including
user IDs and conversation IDs, are rejected with 422. Example request:

```json
{"message":"I want to become a backend engineer. What should I learn next?"}
```

The response contains `message`, configured `model`, `provider`,
`conversation_id` (currently null), and `usage` with `input_tokens` and
`output_tokens`. Missing provider usage is null, never fabricated. The message is
validated against a strict JSON schema and limited to 8000 characters.
Incomplete, malformed, empty, oversized, or structurally invalid provider output
returns 502. Successful responses use `Cache-Control: no-store`.

Context includes only the authenticated student's education fields, up to ten
career goals, and up to fifty assigned skills in deterministic order. Skills include
category, proficiency, confidence, evidence (at most 500 characters), assessment
date, and catalog-active status. Truncation flags identify omitted information.
Names, emails, bios, user/record IDs, timestamps unrelated to assessments, password
hashes, tokens, database settings, and keys are excluded. No missing company type,
assessment, or other unsupported data is invented. The complete serialized input
is capped at 65000 characters. Advice does not mutate the student's records.

`app/ai/prompts.py` holds the versioned coach instructions. The coach distinguishes
self-report, confidence, and evidence, prioritizes practical next steps, states
uncertainty, and must not invent company requirements or guarantee employment.
Student data and questions remain in a separate untrusted user message.

### Security, failures, and limits

Known configured secrets and recognizable credential patterns are redacted from
outbound text and returned advice. Context text is cleaned before truncation.
Verbatim prompt disclosure is rejected. These are defense-in-depth measures;
they cannot guarantee detection of every user-entered secret or every semantic
prompt-injection attempt. Do not submit credentials. Application secrets never
belong in prompts. No generated answer can modify application state.

The provider request uses `store=false`. This disables Responses storage; it is
not a promise of zero provider retention. Student context is transmitted to the
configured provider when the endpoint is called. Review provider data policies
and product consent requirements before production use.

Errors retain the existing `{"error":{"code":"...","message":"..."}}` format:
401 for missing/invalid authentication, 422 for invalid input or provider refusal,
429 for local/provider limits, 503 for unavailable configuration/provider auth or
limiter capacity, 504 for provider timeouts, and 502 for transport/provider/output
failures. No provider error body, stack trace, prompt, or secret is returned.

Safe INFO log records on `app.ai.orchestrator` include request ID, provider/model,
prompt version, latency, success/failure, failure status code, and token counts
when supplied. The host controls logging handlers/levels; inspect structured record
fields if using JSON logging. No full inputs, student context, answers, auth headers,
or provider exception messages are logged. Do not enable HTTP wire/debug logging
or request-body capture in deployment infrastructure.

The in-memory sliding-window limiter is thread-safe and bounds tracked users.
Provider failures consume attempts too. This is a single-process development
protection: worker counts multiply the allowance and restarts reset it. Replace
the `RequestLimiter` protocol implementation with a shared store before using
multiple workers. It is not a billing/token accounting system.

### Conversations and local testing

Conversation persistence is deferred to avoid storing personal chats before
history, retention, and deletion requirements are defined. Each call is single-turn;
no history/resume API or database migration is introduced. `ConversationStore`
is the future persistence boundary and always receives the authenticated user ID.
Its current stateless implementation discards messages and returns no conversation ID.

There is deliberately no fake provider switch for the running application.
Automated tests override the provider dependency with a test-only fake whose
answers are labeled `[TEST RESPONSE]`; adapter tests use HTTPX MockTransport.
Neither makes a network request or requires an API key. With AI disabled, local
API testing can verify authentication, documentation, and the safe 503 response.

```powershell
uv run alembic upgrade head
uv run pytest tests/test_ai.py tests/test_ai_provider.py
uv run pytest
uv run ruff check .
uv run ruff format --check .
uv run mypy app tests migrations
uv run uvicorn app.main:create_app --factory --reload
```

For PostgreSQL tests, the existing test fixture reads `TEST_DATABASE_URL` from
the process environment, not automatically from `.env`. A safe local invocation
that reads that one setting without printing it is:

```powershell
uv run python -c "import os,subprocess; from dotenv import dotenv_values; from sqlalchemy.engine import make_url; u=dotenv_values('.env').get('TEST_DATABASE_URL'); assert u and make_url(u).database=='ninecgpa_test'; e=os.environ.copy(); e['TEST_DATABASE_URL']=u; raise SystemExit(subprocess.call(['uv','run','pytest'],env=e))"
```

Follow the workspace-local pytest temporary-directory instructions above if needed.
Tests migrate isolated schemas in the test database and never write development data.
For a real call, set provider/model/key privately, leave Astra temperature unset,
restart, and open `/docs`. Register/login and authorize with your JWT, then call
the coach with the sample message. Expect 200 with real model metadata and usage;
this is the outstanding live-provider verification. Clear authorization and expect
401. With authorization, try an empty message or more than 4000 characters and
expect 422. Do not copy credentials or tokens into logs or reports.

### Phase 4 verification record

On 2026-10-05, the development migration and `alembic check` both passed;
there are no Phase 4 schema changes. All 168 tests passed against the dedicated
PostgreSQL test database: 113 existing tests and 55 new AI tests. Ruff, formatting,
and strict mypy passed. A real Uvicorn startup and HTTP checks for health, docs,
OpenAPI, and unauthenticated coach rejection passed; the verification server was
then stopped. The existing upstream TestClient deprecation warning remains visible.
Live GPT-6 Astra verification is **BLOCKED**: AI provider/model/key were not
configured locally. No real model call was made and no test response was presented
as a real model response. Phase 5 is not started.


## Phase 5: deterministic career assessment

Assessment routes use the existing authentication/session dependencies, a service,
a repository, and a pure scoring engine. They never request an AI provider and
need no AI API key, balance, or live model response. The current Phase 4 live-call
blocker is provider quota (`429 insufficient_quota`, reported from the user's
Windows diagnostic); it does not block assessment. No Phase 4 behavior was changed.
The typed `AssessmentResponse` is the future boundary for optional AI explanations;
there is no automatic AI enhancement or generated score.

### Role catalog and seeding

`CareerRole` stores a stable public slug, display name, description, and active flag.
`CareerRoleSkill` links existing catalog skills with required/recommended status,
weight (1-5), and target proficiency. UUIDs remain internal to the role tables;
API role IDs are slugs such as `backend-engineer`. The database-backed catalog fits
existing repositories and permits later administration without changing the engine.
No admin API is introduced now.

Ten reference roles are supplied in `app/db/role_catalog.json`: Backend Engineer,
Frontend Engineer, Full Stack Engineer, Data Analyst, Data Scientist, Machine
Learning Engineer, AI Engineer, DevOps / Cloud Engineer, Software Engineer, and
Product Engineer. These are 9CGPA reference profiles, not researched or official
company requirements. All initial targets are INTERMEDIATE; each requirement
can specify another proficiency independently. Backend and general software
profiles use a Python track with Java recommended, rather than requiring every
possible implementation language. DSA is represented by existing individual skills.

```powershell
uv run alembic upgrade head
uv run python -m app.db.seed_roles
# Optional alternative role catalog using the same validated JSON structure:
# uv run python -m app.db.seed_roles --catalog PATH_TO_ROLE_CATALOG.json
```

The command first seeds the original skill catalog and then supplemental
`app/db/role_skills.json` using the Phase 3 seed mechanism. That adds HTML and CSS
under PROGRAMMING, System Design under ENGINEERING, and Data Visualization under
DATA: 42 total skills on a fresh full seed, with seven existing categories.
The original Phase 3 catalog/seed remains unchanged. It then seeds ten roles and
127 requirements. All inserts use conflict handling and preserve existing IDs,
weights, targets, active flags, user skills, and assessment history. Repeated runs
do not overwrite administrative edits or delete obsolete requirements. To change
an existing requirement intentionally, use a reviewed data migration; editing the
seed file alone only inserts missing entries. Each skill-seed stage commits
independently; role/requirement seeding is one atomic transaction and rolls back
if a referenced skill is missing. A failed role stage may leave newly seeded
catalog skills, which are safe to reuse on retry.

### Score formula: career-readiness-v1

Proficiency points: BEGINNER=20, ELEMENTARY=40, INTERMEDIATE=60, ADVANCED=80,
EXPERT=100. For a present skill:

`score = min(100, round_half_up(points * (0.8 + confidence / 500)) + evidence_bonus)`

Confidence ranges from 0 to 100 and changes the multiplier only from 0.8 to 1.0.
Evidence bonus is 5 when non-whitespace evidence exists, otherwise 0. Evidence is
self-reported and not verified: quantity or quality does not earn additional points.
Missing skills receive zero, null proficiency/confidence, and no evidence bonus.
All calculations use integer arithmetic with explicit half-up rounding.

For each requirement:

- `target` is the configured target proficiency's points.
- `gap = max(0, target - score)`.
- `attainment = min(100, round_half_up(100 * score / target))`.
- `priority = round_half_up(weight * gap * (1 + (100 - confidence) / 500))`,
  using confidence=0 for missing skills. Priority is bounded 0-600.

Raw skill status is MISSING for an absent association; otherwise score below 40
is NEEDS_IMPROVEMENT, 40-59 DEVELOPING, 60-79 READY, and 80-100 STRONG.
Status describes the raw skill score, whereas attainment describes progress toward
the selected role's target. A skill can meet an INTERMEDIATE target without being
STRONG. A beginner with maximum confidence and evidence scores only 25.

Required and recommended category scores are the weight-averaged attainment,
rounded independently. Overall readiness is `round_half_up(0.9 * required_score
+ 0.1 * recommended_score)`. Without recommended requirements it is the required
score and recommended score is null. At least one required skill is necessary.
Missing required skills cannot be compensated for by optional skills beyond that
10% contribution. All score fields are bounded 0-100.

| Overall score | Readiness level |
| --- | --- |
| 0-39 | FOUNDATION |
| 40-59 | DEVELOPING |
| 60-74 | INTERVIEW_PREPARATION |
| 75-89 | INTERVIEW_READY |
| 90-100 | HIGHLY_READY |

Priority lists contain only positive gaps: required first, then descending priority,
descending target points, ascending confidence, and ascending skill UUID for ties.
Other result lists sort by skill UUID. `strengths` means target met (`gap=0`);
`skill_gaps` means target not met. These lists provide structured next-step signals,
not a generated learning roadmap. Identical input/requirements produce identical
scores, lists, and ordering. New assessment IDs and `generated_at` are event metadata
and naturally differ; neither influences scoring.

The product readiness metric is a self-report-based preparation aid, not a verified
assessment or employment guarantee. University, CGPA, and missing profile fields
never penalize skill readiness. Company names do not change generic role requirements.
Existing user proficiency, confidence, and last-assessed dates are never changed.

### Assessment APIs and persistence

All paths below are relative to `/api/v1/assessment` and require a Bearer JWT.

| Method | Path | Behavior |
| --- | --- | --- |
| GET | `/roles` | Active role summaries; `limit` 1-100, `offset` >=0 |
| GET | `/roles/{role_id}` | Role requirements and linked catalog skills |
| POST | `/career-readiness` | Compute and persist a snapshot; returns 201 |
| GET | `/history` | Caller-owned summaries, newest first; default limit 20 |
| GET | `/history/{assessment_id}` | Caller-owned snapshot; UUID path parameter |

Explicit selection uses `{"role_id":"backend-engineer"}`. With `{}` or a null
role ID, exactly one distinct active role must match the caller's career goals by
case-insensitive, trimmed display name or slug. No goal, no known match, or multiple
matches returns 422 `target_role_required`; explicit selection resolves ambiguity.
Extra body fields, including `user_id`, are rejected with the existing 422 validation
error. Invalid/unknown/inactive roles return 404. Missing required configuration or
inactive linked skills/categories returns 503 `role_unavailable`; requirements are
never silently omitted to inflate readiness. Unknown or other users' assessment
IDs both return 404. Assessment/history responses use `Cache-Control: no-store`.
Swagger documents authentication, request/response schemas, and error responses.

Example skill result within a response (illustrative, not a model response):

```json
{
  "skill": {"id":"00000000-0000-0000-0000-000000000001","name":"Python","category":"PROGRAMMING"},
  "required": true,
  "weight": 5,
  "target_proficiency": "INTERMEDIATE",
  "current_proficiency": "ADVANCED",
  "confidence": 55,
  "evidence_present": true,
  "score": 78,
  "attainment": 100,
  "status": "READY",
  "gap": 0,
  "priority": 0
}
```

The response envelope includes `id`, `role`, `algorithm_version`, `overall_score`,
`readiness_level`, `required_skills_score`, `recommended_skills_score`, `generated_at`,
`skill_results`, `skill_gaps`, `strengths`, and `priority_skills`.

`Assessment` and `AssessmentSkillResult` preserve numeric scores, requirement
weights/targets, proficiency/confidence, and evidence presence at assessment time.
They do not copy evidence text, personal profile data, or skill names/categories.
History therefore retains original scores after user skills or requirements change;
role/skill display metadata reflects the current catalog. Scores are never recomputed
on history reads. Deleting a user cascades through their assessments/results.
Referenced roles/skills cannot be deleted while snapshots refer to them. Role-skill
pairs and assessment-skill pairs are unique; enum/range checks and foreign-key indexes
protect integrity. These snapshots enable later progress analysis without implementing
roadmaps, analytics dashboards, or other future phases now.

### Phase 5 verification commands

Use the existing dedicated PostgreSQL `TEST_DATABASE_URL` process configuration and
workspace-local pytest temp-directory setup described above. Do not use the development
database as the test database.

```powershell
uv run pytest
uv run ruff check .
uv run ruff format --check .
uv run mypy .
uv run alembic current
uv run alembic check
uv run uvicorn app.main:create_app --factory --reload
```

In `/docs`, register/login and authorize, then list roles, retrieve
`backend-engineer`, and POST `{"role_id":"backend-engineer"}`. Expect 201, missing
skills for a new account, and zero overall score. Assign a skill using the existing
Phase 3 endpoint and assess again; inspect the score/gap/priority change. GET history
and a snapshot by its ID. Another account must receive 404 for that ID. Remove
authorization and expect 401. Use an unknown role and expect 404; omit the role
without a matching career goal and expect 422. No OpenAI request is made.

Migration downgrade/upgrade testing is performed only in isolated test schemas.
Do not downgrade the development database to test reversibility: downgrading Phase 5
removes assessment history and role data, although it preserves Phase 1-4 tables.

### Phase 5 verification record

Verification on 2026-10-05: all 224 PostgreSQL tests passed (168 prior tests plus
56 new assessment/migration tests). The SQLite assessment/migration fallback passed
58 tests with the PostgreSQL-only check skipped. Ruff, format checking, and `mypy .`
passed. Development is at `0003_assessment`; upgrade and metadata-drift checks passed.
Downgrade/upgrade passed in isolated test schemas, including preservation of an
existing account. Development records were compared before/after migration and seed;
all existing Phase 1-4 records were preserved. Two seed runs preserved every row,
ID, and timestamp after the first run.

A separate Uvicorn process on port 8015 passed real HTTP checks for health, docs,
OpenAPI, registration/login/JWT, profiles, career goals, skills, role catalog/detail,
assessment creation, goal resolution, repeatability, history ownership, and invalid
inputs. AI was disabled and its API key empty in that verification process. Temporary
accounts and their cascading assessment records were removed afterward; that server
was stopped without touching any pre-existing server. No OpenAI calls were made.
The existing TestClient deprecation and SQLite expression-index reflection warnings
remain visible. Phase 6 has not started.


## Phase 6: personalized learning roadmaps

The roadmap flow is authenticated routes -> `RoadmapService` -> existing Phase 5
assessment snapshot + learning repository -> pure `generate_plan` -> persisted
roadmap/tasks. It reuses Phase 5 gaps, strengths, target proficiency, and priority
ordering; it does not recompute readiness or ask an LLM to choose a score.
The Phase 4 provider and orchestration architecture are unchanged. Roadmaps work
with an absent, invalid, or quota-exhausted AI key because no AI provider is called.
The typed `RoadmapResponse`, skill mappings, objectives, and BUILD tasks provide
context for future project generation; no project generator is implemented here.

### Learning catalog

`app/db/learning_catalog.json` defines 42 modules, covering all 38 original skills
and the four Phase 5 additions. Every module maps to existing skills, contains four
leveled objectives, and has two actionable tasks per objective: 168 objectives and
336 tasks in total. This is a curriculum outline, not a course/resource marketplace.
Objectives inherit the module's skill mappings. Module difficulty represents its
entry level; individual objective/task difficulties are BEGINNER, ELEMENTARY,
INTERMEDIATE, or ADVANCED. Estimated effort is coarse, not a promised duration.
Task types are LEARN, PRACTICE, BUILD, REVIEW, and ASSESS; ASSESS means a student
self-check exercise, not automated coding or interview evaluation.

Modules/objectives/tasks and many-to-many skill mappings are database-backed.
Prerequisite edges represent foundational familiarity. For example, PostgreSQL
requires SQL, SQL requires DBMS, and Pandas requires NumPy, which requires Python.
Circular graphs, self-dependencies, and missing references are rejected during seed
validation; the merged database graph is validated before commit. Generation also
checks the stored graph and fails safely if administrative data contains a cycle.

```powershell
uv run alembic upgrade head
uv run python -m app.db.seed_roles
uv run python -m app.db.seed_learning
# Optional complete learning catalog with all referenced prerequisite slugs:
# uv run python -m app.db.seed_learning --catalog PATH_TO_LEARNING_CATALOG.json
```

The learning seed first invokes the existing original/supplemental skill seeds,
then inserts learning content in one transaction. It does not require role seeding,
but roles are needed for assessments. Stable module slugs, objective positions,
and task positions identify seed entries. Repeated runs preserve IDs, existing
content, active flags, estimates, and roadmap progress. Existing entries are never
overwritten or removed; use reviewed migrations for intentional content revisions,
and append rather than reorder seed positions. Missing skills or invalid merged
prerequisites roll back the learning stage. The preceding skill stages commit
independently and remain safe to rerun. No credentials or database URLs are logged.

### Selection and ordering: learning-roadmap-v1

Generation accepts an optional role slug. It selects the caller's latest saved
assessment for that role; without a role it selects their latest assessment across
roles. Latest is ordered by assessment timestamp descending, then UUID descending
for ties. No assessment returns 409 `assessment_required`; generation does not
silently create one. An unknown/inactive role returns 404.

The assessment snapshot governs role skill state. Reassess after changing role
skills to update gaps. For prerequisite skills outside that snapshot, generation
may use the caller's current skill records. Academic grades, names, company labels,
and profile text do not alter curriculum ranking or get copied into the roadmap.

Selection rules:

1. Only positive Phase 5 gaps select direct learning content. Target-met strengths
   are not repeated as direct learning phases.
2. Missing/BEGINNER skills start at BEGINNER; ELEMENTARY starts at ELEMENTARY;
   INTERMEDIATE starts at INTERMEDIATE; ADVANCED/EXPERT uses ADVANCED content.
3. When evidence is present and confidence is at least 60, advance the starting
   level by one, capped at ADVANCED, to avoid repeating demonstrated material.
4. Select objectives from that starting level through the target level (capped
   at ADVANCED), or the starting level if already above target. Expert-level gaps
   receive advanced practice/review; the catalog does not certify expert mastery.
5. All matching modules contributing to that level range are selected, with each
   module/task included once. Required level coverage must exist; absent/inactive
   content or empty required objectives returns 503 `learning_content_unavailable`.
6. Expand prerequisites. A prerequisite is considered known when all its mapped
   skills are assessment strengths, or have at least ELEMENTARY proficiency,
   confidence >=60, and evidence. Otherwise include foundational objectives
   through ELEMENTARY (or the module entry level if higher).
7. Topologically order selected modules. Prerequisites inherit the most important
   dependent gap's rank. Among currently unblocked modules, use Phase 5 priority
   rank first and module slug second. Objective/task positions order work within
   each module. This preserves required/weighted gap priorities while respecting
   prerequisite dependencies. Catalogs over 1000 modules fail with a structured 503
   rather than silently truncating a plan.

Each selected module becomes one roadmap phase. Phase count/titles therefore adapt
to actual gaps rather than always emitting five generic stages. Each phase includes
skills, objectives, priority, and a reason (`CLOSE_GAP` or `PREREQUISITE`) with the
source assessment skill ID. A shared prerequisite uses its most important triggering
skill as the primary reason. Matching skills and assessment IDs let the frontend
explain the selection without large generated paragraphs.

No random value or current time ranks tasks. Identical assessment, relevant skill
state, and catalog produce identical task ordering. Persisted UUIDs/timestamps are
metadata, not ranking inputs. The catalog content and task descriptions are authored
seed data, never presented as GPT-generated responses.

### Persistence and regeneration

`LearningRoadmap` belongs to a user and references a role and assessment. Its phases
reference modules, and its items reference learning tasks. Phase/item titles,
objectives, descriptions, difficulty, task type, and estimates are snapshotted so
later catalog edits do not rewrite old work or progress. The full assessment is
not duplicated. Role/skill display metadata and module skill mappings resolve from
the current catalog; historical task content and estimates remain stable.

Each user has one current roadmap across roles: ACTIVE or COMPLETED. A partial
unique database index enforces this, and a user-row lock serializes generation and
progress writes on PostgreSQL. Repeating generation with the same assessment,
generator version, and resulting plan returns the current roadmap unchanged,
including its progress. Changed assessment/content creates a new current roadmap
and marks the previous one SUPERSEDED, atomically. Invalid generation leaves the
previous current roadmap intact. Superseded records remain readable by owner ID
but reject progress edits with 409 `roadmap_superseded`.

A no-gap assessment produces an empty COMPLETED roadmap: zero effort, 100% completion,
no current phase, and no next task. This means no work was selected, not that the
student is guaranteed a job. User deletion cascades through roadmaps/phases/items;
assessment deletion also removes dependent roadmaps. Referenced catalog tasks/modules
are protected from deletion while roadmaps use them. Unique positions, unique links,
foreign keys, effort ranges, and enum constraints protect integrity.

### Progress and API contracts

All six endpoints require the existing Bearer JWT. Paths are relative to `/api/v1`.

| Method | Path | Behavior |
| --- | --- | --- |
| GET | `/learning/modules` | Active catalog summaries, limit 1-100 and offset >=0 |
| GET | `/learning/modules/{module_id}` | Module slug, objectives, tasks, skill mappings, prerequisite slugs |
| POST | `/roadmaps/generate` | Body `{}` or `{"role_id":"backend-engineer"}`; 200 for creation or reuse |
| GET | `/roadmaps/current` | Current ACTIVE/COMPLETED roadmap; 404 when absent |
| GET | `/roadmaps/{roadmap_id}` | Owner-only UUID lookup, including superseded history |
| PATCH | `/roadmaps/{roadmap_id}/items/{item_id}` | Body `{"status":"COMPLETED"}` or `{"status":"PENDING"}` |

Clients cannot submit ownership, assessment scores, estimates, or task text. Extra
body fields and invalid completion states return the existing 422 error. Unknown
or other users' roadmap/item IDs return 404. Authentication failures return 401.
Roadmap responses use `Cache-Control: no-store`. Swagger documents response schemas
and the existing structured 401/404/409/422/503 error envelope.

PATCH returns the full updated roadmap. Repeating the same status is a no-op.
Completing all tasks marks the roadmap COMPLETED; reopening a task makes it ACTIVE.
Task completion is self-reported and may occur out of order. Ordering guides the
next task; it does not lock later tasks or claim to validate learning. Completion
does not change UserSkill proficiency, evidence, last-assessed dates, or readiness.
Reassessment remains an explicit Phase 5 action.

Progress is completed task count / total task count * 100, rounded to two decimals.
Completed effort sums completed task estimates; remaining effort is total minus
completed. Integers in minutes are authoritative; hours are derived and rounded to
two decimals. Each phase and the roadmap expose both counts and effort. The next
task is the earliest pending item in phase/task order, and the current phase is
the first phase with pending tasks. The full ordered item list supports upcoming
work. Superseded-roadmap progress is historical and does not change current progress.

Illustrative response excerpt (IDs and additional task fields omitted here only):

```json
{
  "status": "ACTIVE",
  "generator_version": "learning-roadmap-v1",
  "total_tasks": 6,
  "completed_tasks": 1,
  "overall_progress": 16.67,
  "estimated_minutes": 285,
  "completed_minutes": 15,
  "remaining_minutes": 270,
  "estimated_hours": 4.75,
  "completed_hours": 0.25,
  "remaining_hours": 4.5,
  "phases": [{
    "order": 1,
    "module_id": "python",
    "title": "Python Learning Path",
    "reason": "CLOSE_GAP",
    "progress": 16.67
  }]
}
```

### Phase 6 checks and manual walkthrough

Use the dedicated PostgreSQL test database and existing workspace-local pytest
configuration described earlier. PostgreSQL is authoritative for row-lock and
concurrency checks; SQLite is only a lightweight fallback.

```powershell
uv run pytest
uv run ruff check .
uv run ruff format --check .
uv run mypy .
uv run alembic current
uv run alembic check
uv run uvicorn app.main:create_app --factory --reload
```

1. Open `/docs`, register/login, and authorize with your JWT.
2. GET `/learning/modules` and `/learning/modules/postgresql`. Inspect SQL prerequisite,
   four leveled objectives, actionable tasks, and effort estimates.
3. POST `/assessment/career-readiness` with `{"role_id":"backend-engineer"}`.
4. POST `/roadmaps/generate` with the same role. Expect 200 with phases, tasks, source
   assessment, progress, and next task. GET `/roadmaps/current` and by returned UUID.
5. Repeat generation without changing inputs: same roadmap ID and progress.
6. PATCH one returned item with `{"status":"COMPLETED"}`. Check count, percentage,
   completed/remaining minutes, and next task. Repeat PATCH: no double counting.
7. Create another assessment and regenerate. The previous roadmap stays readable
   by its ID with SUPERSEDED status; edits to it return 409.
8. A different user must receive 404 for the original roadmap and its item updates.
   Without authentication expect 401; invalid state or `user_id` in the body gives 422.
9. Re-run the learning seed and verify existing task IDs and progress remain intact.

Migration downgrade/re-upgrade checks belong only in isolated test schemas.
Never reset development data to verify this phase. Phase 7 has not started.

### Phase 6 verification record

Verification on 2026-10-05: all 277 PostgreSQL tests passed, including all 224 prior
tests and 53 new roadmap/catalog/engine tests. Concurrent generation from two
independent PostgreSQL connections produced one current roadmap. Ruff, format
checking, and `mypy .` passed. The existing TestClient deprecation warning remains
visible and unsuppressed.

Development is at `0004_roadmaps`; upgrade and schema-drift checks passed. Eight
new tables were added without changing existing Phase 1-5 records. Downgrade and
re-upgrade passed in isolated schemas, including a populated learning catalog and
preservation of a Phase 5 assessment. Two learning-seed runs preserved every row,
ID, and timestamp after the first run: 42 modules, 42 skill mappings, 168 objectives,
336 tasks, and 35 prerequisite links.

A separate Uvicorn process on port 8016 passed real HTTP checks for startup, health,
Swagger/OpenAPI, registration/login/JWT, learning catalog/detail, assessment creation,
roadmap generation/current/by-ID retrieval, repeat generation, completion, progress
arithmetic, duplicate completion, ownership, malformed input, regeneration, and
read-only superseded history. AI was disabled and its key empty in that process.
Disposable accounts and cascading roadmap data were removed afterward; only the
verification server was stopped. No OpenAI calls or package installations were needed.

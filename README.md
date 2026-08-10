# SageWire Onboarding Service

Reusable, versioned onboarding infrastructure for SageWire applications and services.

The SageWire Onboarding Service provides a stable API for defining onboarding flows, publishing immutable flow versions, creating onboarding sessions, collecting validated answers, determining completion eligibility, completing sessions, and recording an event ledger of important onboarding activity.

**Service:** `sagewire-onboarding`  
**Version:** `0.1.0`  
**Production:** `https://onboarding.sagewire.dev`  
**API version:** `v1`

---

## Why this service exists

Onboarding should not have to be rebuilt independently inside every SageWire application.

This service separates onboarding logic from the product UI so applications can share a common engine while still defining different onboarding experiences for users, organizations, locations, devices, and other subjects.

The core model is:

```text
Flow
  └── Version
       └── Steps
            └── Fields

Published Version
  └── Session
       └── Answers
            └── Completion
                 └── Event Ledger
```

A flow is the stable identity of an onboarding process. Its editable definition lives in draft versions. Once a version is published, it becomes immutable and can be used to create onboarding sessions.

---

## Current capabilities

- Create and manage onboarding flows
- Create versioned flow definitions
- Add steps and fields to draft versions
- Publish immutable flow versions
- Retire older published versions
- Activate and deactivate flows
- Create onboarding sessions from published definitions
- Write and update answers
- Bulk answer submission
- Validate required fields
- Reject unknown fields
- Reject answer changes after session completion
- Calculate onboarding progress
- Determine completion eligibility
- Complete eligible sessions
- Persist completion state
- Maintain an onboarding event ledger
- Filter and paginate service resources
- Stable validation and database error responses
- Process health and database readiness checks

---

## Technology

- Python
- FastAPI
- SQLAlchemy
- Pydantic
- SQLite
- Uvicorn
- Docker
- Pytest
- Coolify

The default production database path is:

```text
/data/onboarding.db
```

The service uses SQLAlchemy metadata to create missing SQLite tables at startup in `v0.1.0`. A future release may move schema changes to Alembic migrations.

---

## Service endpoints

### Service identity

```http
GET /
```

Returns service identity, version, status, and navigation links.

### Health

```http
GET /health
```

Lightweight process health check. This endpoint does not query the database.

Example:

```json
{
  "service": "sagewire-onboarding",
  "status": "ok",
  "version": "0.1.0"
}
```

### Readiness

```http
GET /ready
```

Confirms that the application can access its database.

Example:

```json
{
  "service": "sagewire-onboarding",
  "status": "ready",
  "version": "0.1.0",
  "database": "available"
}
```

### API documentation

```text
/docs
/redoc
/openapi.json
```

---

## API structure

All application API routes are namespaced under:

```text
/api/v1
```

Primary route families:

```text
/api/v1/flows
/api/v1/sessions
/api/v1/events
```

### Flows

Flows are stable onboarding identities. Their definitions are maintained through separately versioned records.

Supported operations include:

- create a flow
- list and filter flows
- retrieve a flow by UUID or stable flow key
- update flow metadata
- activate/deactivate a flow
- create and list versions
- publish a draft version
- retire a published version
- retrieve the currently published version
- create/update/delete steps on draft versions
- create/update/delete fields on draft versions

Published versions are immutable.

### Sessions

Sessions represent an actual onboarding attempt against a published flow version.

The session API supports the lifecycle needed by a client application to:

1. create a session
2. retrieve the current onboarding definition/state
3. submit answers
4. inspect progress and completion eligibility
5. complete the session when requirements are satisfied

Completed sessions reject subsequent answer modification.

### Events

Important onboarding activity is written to an event ledger. Events can retain actor and correlation information so activity can be traced across applications and services.

---

## Example production checks

```bash
curl https://onboarding.sagewire.dev/health
```

```bash
curl https://onboarding.sagewire.dev/ready
```

Interactive API documentation:

```text
https://onboarding.sagewire.dev/docs
```

---

## Local development

### 1. Clone the repository

```bash
git clone https://github.com/voidwalker-sagewire/sagewire-onboarding.git
cd sagewire-onboarding
```

### 2. Create a virtual environment

```bash
python -m venv .venv
source .venv/bin/activate
```

On Windows:

```powershell
.venv\Scripts\activate
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

### 4. Run the service

```bash
uvicorn app.main:app --reload --host 0.0.0.0 --port 8010
```

Then open:

```text
http://localhost:8010/docs
```

---

## Database configuration

The Docker image defaults to:

```text
DATABASE_URL=sqlite:////data/onboarding.db
```

For local development, set `DATABASE_URL` to the SQLite location you want to use.

Production deployments should mount persistent storage at:

```text
/data
```

so the onboarding database survives container replacement and redeployment.

---

## Docker

Build the image:

```bash
docker build -t sagewire-onboarding:0.1.0 .
```

Run it with persistent storage:

```bash
docker run --rm \
  -p 8010:8010 \
  -v sagewire-onboarding-data:/data \
  sagewire-onboarding:0.1.0
```

Test the container:

```bash
curl http://localhost:8010/health
curl http://localhost:8010/ready
```

---

## Tests

Run the test suite with:

```bash
pytest -v
```

The `v0.1.0` hardening suite covers the main happy path and important failure cases, including:

- health endpoint
- full onboarding flow
- missing required fields
- invalid email validation
- unknown field rejection
- premature completion rejection
- completed-session answer protection

At the completion of the `v0.1.0` hardening pass, the suite reported:

```text
7 passed
```

---

## Error handling

The service provides a consistent error shape for request validation failures, database failures, and unexpected application errors.

Example validation response structure:

```json
{
  "error": {
    "code": "REQUEST_VALIDATION_ERROR",
    "message": "The request did not pass validation.",
    "path": "/api/v1/...",
    "details": []
  }
}
```

Raw SQLAlchemy/database errors are not exposed directly to API clients.

---

## CORS

`v0.1.0` currently allows cross-origin requests from all origins:

```text
allow_origins = ["*"]
```

Credentials are disabled.

This is suitable for the current service integration model, but production policy can be tightened later if SageWire moves to an explicit origin allowlist.

---

## Deployment

The production service is deployed through Coolify using the repository Dockerfile.

```text
Repository: voidwalker-sagewire/sagewire-onboarding
Branch:     main
Domain:     https://onboarding.sagewire.dev
Database:   /data/onboarding.db
```

The `/health` endpoint is used for process health and `/ready` verifies database connectivity.

---

## Design principles

### Version onboarding definitions

Applications should not depend on a mutable onboarding definition. A session remains attached to the exact published flow version it started with.

### Keep products thin

Products should render the onboarding experience and submit answers. Validation, flow lifecycle, completion rules, persistence, and auditing belong in the onboarding service.

### Preserve the record

Onboarding is part of the larger SageWire record architecture. Important state transitions should be durable, attributable, and inspectable later.

### Reuse infrastructure

The service is not tied to one SageWire product. It is designed to support multiple applications and subject types through the same reusable API.

---

## Status

**SageWire Onboarding Service `v0.1.0` is deployed and operational.**

The initial production hardening milestone is complete. Future work can focus on additional flow capabilities, authentication/authorization policy, migrations, richer integrations, and broader product adoption without changing the core service boundary.

---

## SageWire

SageWire builds infrastructure for turning real-world activity into digital records and automated actions.

Developer platform: `https://sagewire.dev`  
Public site: `https://sagewire.io`

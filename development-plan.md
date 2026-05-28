# CI/CD Pipeline Optimizer — Phased Development Plan

> Project: 006-cicd-pipeline-optimizer · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | Python 3.12+ | ML/LLM-heavy workloads (XGBoost, SHAP, LLM calls); rich CI platform client libraries; YAML parsing ecosystem |
| API framework | FastAPI | Async-native for webhook ingestion; auto-generates OpenAPI 3.1; Pydantic v2 for request/response validation |
| Database | PostgreSQL 16 | Partitioned tables for high-volume builds/tests; JSONB for flexible metadata; UUID primary keys |
| Migrations | Alembic | Standard SQLAlchemy migration tool; version-controlled schema changes |
| ORM | SQLAlchemy 2.0 | Async support; mapped dataclasses; works with Alembic |
| Task queue | Celery + Redis | Async build ingestion, DORA metric computation, flaky test analysis, ML scoring jobs |
| Cache backend | Redis | Shared cache metadata registry; also serves as Celery broker |
| Object storage | S3-compatible (MinIO for self-hosted) | Content-addressed build cache artifact storage |
| Frontend | React 18 + Vite + Tailwind CSS | Dashboard for DORA metrics, build timelines, flaky tests; SPA with REST API backend |
| Charting | Recharts | Lightweight React charting for DORA trends, build duration histograms, cache hit rates |
| ML framework | scikit-learn + XGBoost | Test selection and failure prediction models (proven 89.7% accuracy in research) |
| Explainability | SHAP | Feature importance for flaky test root cause and failure prediction explanations |
| LLM integration | Anthropic Python SDK (Claude) | Flaky test root cause analysis, pipeline YAML generation, natural-language queries |
| CLI | Click | CLI tool for local cache operations, affected-change detection, and CI integration |
| CI YAML parsing | ruamel.yaml + custom parsers | Parse GitHub Actions, GitLab CI, Jenkins, CircleCI YAML without executing |
| Git analysis | GitPython + pygit2 | Commit history, changed-file detection, monorepo package graph |
| Containerisation | Docker + docker-compose | Self-hosted deployment; PostgreSQL, Redis, MinIO, API, worker, frontend |
| Testing | pytest + pytest-asyncio + httpx | Unit, integration, and E2E tests; httpx for async FastAPI test client |
| Linting | Ruff | Fast Python linter and formatter replacing flake8 + black + isort |
| Type checking | mypy (strict) | Catch type errors before runtime; enforced in CI |
| Package manager | uv | Fast Python package installer; lockfile for reproducible builds |

### Project Structure

```
cicd-pipeline-optimizer/
├── pyproject.toml
├── Dockerfile
├── docker-compose.yml
├── alembic.ini
├── alembic/
│   └── versions/
├── src/
│   ├── __init__.py
│   ├── main.py                    # FastAPI app factory
│   ├── config.py                  # Pydantic Settings
│   ├── database.py                # SQLAlchemy async engine + session
│   ├── models/                    # SQLAlchemy ORM models
│   │   ├── __init__.py
│   │   ├── tenant.py
│   │   ├── repository.py
│   │   ├── pipeline.py
│   │   ├── build.py
│   │   ├── job.py
│   │   ├── test_result.py
│   │   ├── flaky_test.py
│   │   ├── cache_entry.py
│   │   ├── dora_metric.py
│   │   └── ai_suggestion.py
│   ├── schemas/                   # Pydantic request/response schemas
│   │   ├── __init__.py
│   │   ├── builds.py
│   │   ├── tests.py
│   │   ├── dora.py
│   │   ├── cache.py
│   │   └── webhooks.py
│   ├── api/                       # FastAPI routers
│   │   ├── __init__.py
│   │   ├── builds.py
│   │   ├── repositories.py
│   │   ├── pipelines.py
│   │   ├── tests.py
│   │   ├── dora.py
│   │   ├── cache.py
│   │   ├── webhooks/
│   │   │   ├── __init__.py
│   │   │   ├── github.py
│   │   │   ├── gitlab.py
│   │   │   ├── jenkins.py
│   │   │   └── circleci.py
│   │   └── ai.py
│   ├── services/                  # Business logic
│   │   ├── __init__.py
│   │   ├── build_ingestion.py
│   │   ├── affected_changes.py
│   │   ├── cache_manager.py
│   │   ├── flaky_detector.py
│   │   ├── dora_calculator.py
│   │   ├── test_selector.py       # v1.1
│   │   ├── failure_predictor.py   # v1.1
│   │   └── pipeline_analyzer.py   # v1.1
│   ├── connectors/                # CI platform adapters
│   │   ├── __init__.py
│   │   ├── base.py
│   │   ├── github_actions.py
│   │   ├── gitlab_ci.py
│   │   ├── jenkins.py
│   │   └── circleci.py
│   ├── ml/                        # ML models
│   │   ├── __init__.py
│   │   ├── test_selection.py
│   │   ├── failure_prediction.py
│   │   └── flaky_attribution.py
│   ├── tasks/                     # Celery async tasks
│   │   ├── __init__.py
│   │   ├── ingest.py
│   │   ├── metrics.py
│   │   ├── flaky.py
│   │   └── ml.py
│   └── cli/                       # CLI tool
│       ├── __init__.py
│       ├── main.py
│       ├── cache.py
│       └── affected.py
├── frontend/
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx
│   │   │   ├── Builds.tsx
│   │   │   ├── FlakyTests.tsx
│   │   │   └── DoraMetrics.tsx
│   │   ├── components/
│   │   └── api/
│   └── public/
├── tests/
│   ├── conftest.py
│   ├── unit/
│   │   ├── test_affected_changes.py
│   │   ├── test_cache_manager.py
│   │   ├── test_flaky_detector.py
│   │   ├── test_dora_calculator.py
│   │   └── test_build_ingestion.py
│   ├── integration/
│   │   ├── test_webhook_github.py
│   │   ├── test_webhook_gitlab.py
│   │   ├── test_api_builds.py
│   │   └── test_api_dora.py
│   ├── e2e/
│   │   └── test_full_pipeline.py
│   └── fixtures/
│       ├── github_webhook_push.json
│       ├── gitlab_pipeline_complete.json
│       ├── junit_results.xml
│       └── sample_pipeline.yml
└── docs/
    └── openapi-overrides.yml
```

---

## Phase 1: Foundation

### Purpose
Establish the project skeleton, database schema, configuration system, and Docker development environment. After this phase, a developer can run `docker-compose up`, connect to PostgreSQL, and run migrations — but no business logic exists yet.

### Tasks

#### 1.1 — Project Scaffold and Configuration

**What**: Create `pyproject.toml`, FastAPI app factory, Pydantic Settings configuration, and Docker infrastructure.

**Design**:

```python
# src/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str = "postgresql+asyncpg://cicd:cicd@localhost:5432/cicd"
    redis_url: str = "redis://localhost:6379/0"
    s3_endpoint: str = "http://localhost:9000"
    s3_bucket: str = "cache"
    s3_access_key: str = "minioadmin"
    s3_secret_key: str = "minioadmin"
    api_host: str = "0.0.0.0"
    api_port: int = 8000
    log_level: str = "INFO"
    webhook_secret_github: str = ""
    webhook_secret_gitlab: str = ""
    anthropic_api_key: str = ""

    model_config = {"env_prefix": "CICD_"}
```

```yaml
# docker-compose.yml services:
# postgres:16, redis:7-alpine, minio/minio, api (uvicorn), worker (celery), frontend (nginx)
```

**Testing**:
- Unit: `Settings()` with env vars → correct values parsed
- Unit: `Settings()` with missing optional → defaults applied
- Integration: `docker-compose up -d` → all services healthy within 30s

#### 1.2 — Database Schema and Migrations

**What**: Implement all SQLAlchemy ORM models from data-model-suggestion-1 and generate Alembic migrations.

**Design**:

ORM models map directly to the DDL in data-model-suggestion-1.md. Key models:

```python
# src/models/build.py
from sqlalchemy import Column, Text, BigInteger, Boolean, ForeignKey, Index
from sqlalchemy.dialects.postgresql import UUID, JSONB, TIMESTAMPTZ
import uuid

class Build(Base):
    __tablename__ = "builds"
    __table_args__ = (
        Index("idx_builds_pipeline", "pipeline_id", "started_at"),
        Index("idx_builds_tenant", "tenant_id", "started_at"),
        Index("idx_builds_status", "tenant_id", "status"),
        Index("idx_builds_commit", "commit_sha"),
        {"postgresql_partition_by": "RANGE (started_at)"},
    )

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    tenant_id = Column(UUID(as_uuid=True), ForeignKey("tenants.id"), nullable=False)
    pipeline_id = Column(UUID(as_uuid=True), ForeignKey("pipelines.id"), nullable=False)
    external_id = Column(Text, nullable=False)
    branch = Column(Text, nullable=False)
    commit_sha = Column(Text, nullable=False)
    commit_message = Column(Text)
    author = Column(Text)
    trigger = Column(Text)
    status = Column(Text, nullable=False)
    started_at = Column(TIMESTAMPTZ)
    finished_at = Column(TIMESTAMPTZ)
    duration_ms = Column(BigInteger)
    queue_time_ms = Column(BigInteger)
    is_deployment = Column(Boolean, default=False)
    deployment_env = Column(Text)
    runner_type = Column(Text)
    runner_os = Column(Text)
    cost_microcents = Column(BigInteger)
    slsa_level = Column(BigInteger)
    provenance_uri = Column(Text)
    failure_prediction = Column(JSONB)
    created_at = Column(TIMESTAMPTZ, server_default="now()")
```

All 12 tables from data-model-suggestion-1: `tenants`, `users`, `repositories`, `pipelines`, `builds`, `jobs`, `test_results`, `flaky_tests`, `cache_entries`, `dora_metrics`, `ai_suggestions`, `audit_log`.

Partition creation for `builds`, `test_results`, `audit_log` — monthly partitions via Alembic migration with `CREATE TABLE builds_2026_06 PARTITION OF builds FOR VALUES FROM ('2026-06-01') TO ('2026-07-01')`.

**Testing**:
- Unit: all models importable, relationships resolve
- Integration: `alembic upgrade head` on fresh database → all tables created
- Integration: `alembic downgrade base` → all tables dropped
- Integration: insert a tenant → insert a repository → insert a pipeline → insert a build → FK constraints hold

#### 1.3 — Health Check and Base API

**What**: `/health` endpoint, CORS middleware, API key authentication middleware, and OpenAPI metadata.

**Design**:

```python
# GET /health → {"status": "ok", "version": "0.1.0", "db": "connected", "redis": "connected"}
# All /api/v1/* routes require X-API-Key header matching tenant API key
# OpenAPI spec at /openapi.json with title "CI/CD Pipeline Optimizer API"
```

Authentication middleware:
```python
async def api_key_auth(request: Request) -> Tenant:
    api_key = request.headers.get("X-API-Key")
    if not api_key:
        raise HTTPException(401, "Missing X-API-Key header")
    tenant = await get_tenant_by_api_key(api_key)
    if not tenant:
        raise HTTPException(401, "Invalid API key")
    return tenant
```

**Testing**:
- Unit: valid API key → tenant returned
- Unit: missing header → 401
- Unit: invalid key → 401
- E2E: `GET /health` → 200 with all checks passing
- E2E: `GET /openapi.json` → valid OpenAPI 3.1 document

---

## Phase 2: Build Ingestion and CI Platform Connectors

### Purpose
Enable the platform to receive build and test data from CI platforms. After this phase, users can connect a GitHub Actions, GitLab CI, Jenkins, or CircleCI pipeline and see builds flowing into the system.

### Tasks

#### 2.1 — Webhook Receiver for GitHub Actions

**What**: Receive `workflow_run.completed` and `check_suite.completed` webhook events from GitHub, validate HMAC-SHA256 signatures, and create Build + Job + TestResult records.

**Design**:

```python
# POST /api/v1/webhooks/github
# Headers: X-GitHub-Event, X-Hub-Signature-256
# Payload: GitHub workflow_run webhook payload

class GitHubWebhookPayload(BaseModel):
    action: str
    workflow_run: WorkflowRun | None = None
    check_suite: CheckSuite | None = None

async def handle_github_webhook(payload: dict, signature: str, event_type: str):
    verify_github_signature(payload, signature, settings.webhook_secret_github)
    if event_type == "workflow_run" and payload["action"] == "completed":
        build = map_workflow_run_to_build(payload["workflow_run"])
        jobs = await fetch_github_jobs(payload["workflow_run"]["id"])
        await ingest_build(build, jobs)
```

Mapping: `workflow_run.conclusion` → Build.status (`success`→`passed`, `failure`→`failed`, `cancelled`→`cancelled`). `workflow_run.run_started_at` → `started_at`. Jobs fetched via GitHub API `/actions/runs/{id}/jobs`.

**Testing**:
- Unit: valid HMAC signature → accepted
- Unit: invalid HMAC signature → 401
- Unit: `workflow_run.completed` with `conclusion=success` → Build with status `passed`
- Unit: `workflow_run.completed` with `conclusion=failure` → Build with status `failed`
- Integration (mocked GitHub API): webhook → Build + 3 Jobs created in database
- Fixture: `tests/fixtures/github_webhook_push.json`

#### 2.2 — Webhook Receiver for GitLab CI

**What**: Receive GitLab Pipeline and Job webhook events, validate X-Gitlab-Token, and create records.

**Design**:

```python
# POST /api/v1/webhooks/gitlab
# Headers: X-Gitlab-Token
# Events: Pipeline Hook (object_kind: "pipeline"), Job Hook (object_kind: "build")
```

Mapping: GitLab `pipeline.status` → Build.status (`success`→`passed`, `failed`→`failed`, `canceled`→`cancelled`). GitLab `builds[]` array → Job records. Test results parsed from JUnit XML artifacts fetched via GitLab API.

**Testing**:
- Unit: valid X-Gitlab-Token → accepted
- Unit: pipeline completed event → Build created with correct status
- Integration (mocked): pipeline with 4 jobs → Build + 4 Jobs in database
- Fixture: `tests/fixtures/gitlab_pipeline_complete.json`

#### 2.3 — Generic Build Ingestion API

**What**: REST endpoint for manually pushing build data from any CI platform (Jenkins, CircleCI, Buildkite, or custom).

**Design**:

```python
# POST /api/v1/builds
class BuildCreateRequest(BaseModel):
    repository: str                    # e.g., "org/repo"
    pipeline_name: str
    ci_platform: Literal["github_actions", "gitlab_ci", "jenkins", "circleci", "buildkite", "other"]
    external_id: str
    branch: str
    commit_sha: str
    commit_message: str | None = None
    author: str | None = None
    trigger: Literal["push", "pull_request", "schedule", "manual", "tag", "api"] = "push"
    status: Literal["queued", "running", "passed", "failed", "cancelled", "timed_out"]
    started_at: datetime
    finished_at: datetime | None = None
    is_deployment: bool = False
    deployment_env: str | None = None
    jobs: list[JobCreateRequest] = []
    test_results: list[TestResultCreateRequest] = []

# Response: 201 Created with Build object including generated ID
```

**Testing**:
- Unit: valid payload → Build created, 201 returned
- Unit: duplicate external_id → 409 Conflict
- Unit: missing required field → 422 with field name
- Integration: POST build with 2 jobs and 50 test results → all records in database with correct FK relationships
- Integration: POST build for new repository → repository auto-created

#### 2.4 — JUnit XML Test Result Parser

**What**: Parse JUnit XML test result files into `TestResult` records.

**Design**:

```python
# src/services/build_ingestion.py
def parse_junit_xml(xml_content: str) -> list[TestResultCreate]:
    """Parse JUnit XML into TestResult schemas.
    Handles: <testsuites>, <testsuite>, <testcase>, <failure>, <error>, <skipped>.
    Maps: testcase.classname → test_class, testcase.name → test_name,
          testsuite.name → test_suite, testcase.time → duration_ms (seconds * 1000).
    """
```

**Testing**:
- Unit: valid JUnit XML with 10 passed, 2 failed → 12 TestResultCreate objects with correct statuses
- Unit: JUnit XML with `<skipped>` → status = `skipped`
- Unit: JUnit XML with `<error>` → status = `error`, failure_message populated
- Unit: nested `<testsuites>` → all suites parsed
- Unit: empty XML → empty list (no error)
- Fixture: `tests/fixtures/junit_results.xml`

---

## Phase 3: Build Cache System

### Purpose
Implement distributed content-addressed build caching — the #1 table-stakes feature. After this phase, CI agents and local developer machines can push and pull cache artifacts, and the dashboard shows cache hit rates.

### Tasks

#### 3.1 — Content-Addressed Cache Storage

**What**: S3-compatible storage backend for cache artifacts, keyed by content hash.

**Design**:

```python
# src/services/cache_manager.py
class CacheManager:
    async def put(self, tenant_id: UUID, cache_key: str, data: BinaryIO, size_bytes: int) -> CacheEntry:
        """Upload artifact to S3 at key: {tenant_id}/{cache_key}.
        Creates/updates cache_entries record. Returns CacheEntry with presigned URL."""

    async def get(self, tenant_id: UUID, cache_key: str) -> CacheEntry | None:
        """Check if cache entry exists. If so, increment hit_count, update last_hit_at.
        Return presigned download URL (5-minute expiry). Return None on miss."""

    async def delete(self, tenant_id: UUID, cache_key: str) -> bool:
        """Delete artifact from S3 and remove cache_entries record."""

    async def evict_expired(self, tenant_id: UUID) -> int:
        """Delete entries where expires_at < now(). Return count deleted."""
```

Cache key format: content hash of inputs (source files, dependencies, build config). The CLI generates this hash locally; the server stores and serves artifacts.

```python
# API endpoints:
# PUT  /api/v1/cache/{cache_key}  — upload artifact (multipart form data)
# GET  /api/v1/cache/{cache_key}  — download artifact (redirect to presigned URL)
# HEAD /api/v1/cache/{cache_key}  — check existence (200 or 404)
# DELETE /api/v1/cache/{cache_key} — remove artifact
```

**Testing**:
- Unit: `put` → S3 upload called, cache_entries record created
- Unit: `get` on existing key → hit_count incremented, presigned URL returned
- Unit: `get` on missing key → None returned
- Unit: `evict_expired` → expired entries deleted, non-expired preserved
- Integration: PUT 1MB artifact → GET returns identical bytes
- Integration: HEAD existing key → 200; HEAD missing key → 404

#### 3.2 — Cache CLI Tool

**What**: CLI command `cicd cache push` and `cicd cache pull` for use in CI scripts and local development.

**Design**:

```python
# src/cli/cache.py
@click.group()
def cache():
    """Manage build cache artifacts."""

@cache.command()
@click.argument("key")
@click.argument("path", type=click.Path(exists=True))
@click.option("--server", envvar="CICD_SERVER_URL")
@click.option("--api-key", envvar="CICD_API_KEY")
def push(key: str, path: str, server: str, api_key: str):
    """Push a file or directory to the remote cache."""

@cache.command()
@click.argument("key")
@click.argument("path", type=click.Path())
@click.option("--server", envvar="CICD_SERVER_URL")
@click.option("--api-key", envvar="CICD_API_KEY")
def pull(key: str, path: str, server: str, api_key: str):
    """Pull a cached artifact. Exit code 0 if hit, 1 if miss."""

@cache.command()
@click.argument("paths", nargs=-1, type=click.Path(exists=True))
def hash(paths: tuple[str, ...]):
    """Compute content-addressed cache key from input paths."""
    # SHA-256 of sorted file contents
```

**Testing**:
- E2E: `cicd cache hash src/` → deterministic SHA-256 printed
- E2E: `cicd cache push abc123 ./dist` → artifact uploaded, exit code 0
- E2E: `cicd cache pull abc123 ./dist` → artifact downloaded, contents match
- E2E: `cicd cache pull missing-key ./dist` → exit code 1

---

## Phase 4: Affected-Change Detection

### Purpose
Implement monorepo-aware change detection so CI runs only rebuild/retest packages affected by the current changeset. This is the second table-stakes feature.

### Tasks

#### 4.1 — Package Graph Builder

**What**: Parse monorepo package manifests (`package.json`, `pom.xml`, `build.gradle`, `Cargo.toml`, `go.mod`) to build a dependency graph.

**Design**:

```python
# src/services/affected_changes.py
@dataclass
class PackageNode:
    name: str
    path: str              # relative to repo root
    dependencies: set[str] # names of packages this depends on
    build_system: str      # npm, maven, gradle, cargo, go

class PackageGraph:
    nodes: dict[str, PackageNode]  # name → node

    def dependents_of(self, package_name: str) -> set[str]:
        """Return all packages that transitively depend on the given package."""

    @classmethod
    def from_repository(cls, repo_path: Path) -> "PackageGraph":
        """Scan repo for package manifests, parse dependencies, build graph."""
```

**Testing**:
- Unit: repo with 3 npm packages (A depends on B, B depends on C) → graph with correct edges
- Unit: `dependents_of("C")` → `{"B", "A"}` (transitive)
- Unit: single-package repo → graph with 1 node, no edges
- Fixture: sample monorepo directory structure in `tests/fixtures/`

#### 4.2 — Changed-File to Affected-Package Mapping

**What**: Given a git diff (list of changed files), determine which packages are affected.

**Design**:

```python
def affected_packages(graph: PackageGraph, changed_files: list[str]) -> set[str]:
    """Map changed files to their owning package, then compute transitive dependents."""
    directly_changed = set()
    for f in changed_files:
        pkg = graph.package_for_file(f)
        if pkg:
            directly_changed.add(pkg)
    affected = set(directly_changed)
    for pkg in directly_changed:
        affected |= graph.dependents_of(pkg)
    return affected
```

**Testing**:
- Unit: file in package A changed → `{"A"}` affected (no dependents)
- Unit: file in package C changed (B→C, A→B) → `{"C", "B", "A"}` affected
- Unit: file outside any package (README.md) → empty set
- Unit: file in root config (tsconfig.json, .eslintrc) → all packages affected

#### 4.3 — Affected CLI Command

**What**: CLI command `cicd affected --base=main` that outputs affected package names.

**Design**:

```python
# cicd affected --base=main --head=HEAD --format=json
# Output: ["packages/api", "packages/shared"]

@click.command()
@click.option("--base", default="main")
@click.option("--head", default="HEAD")
@click.option("--format", type=click.Choice(["json", "text"]), default="text")
def affected(base: str, head: str, format: str):
    """List packages affected by changes between base and head."""
```

**Testing**:
- E2E: commit changes to one package → only that package and its dependents listed
- E2E: `--format=json` → valid JSON array
- E2E: no changes → empty output, exit code 0

---

## Phase 5: Flaky Test Detection

### Purpose
Detect flaky tests via statistical analysis of test result variance across builds. After this phase, the platform identifies tests that pass and fail non-deterministically and surfaces them in a ranked list.

### Tasks

#### 5.1 — Flaky Test Analysis Engine

**What**: Analyse test result history to compute flakiness rates and detect newly flaky tests.

**Design**:

```python
# src/services/flaky_detector.py
class FlakyDetector:
    WINDOW_SIZE = 100       # last N runs
    FLAKY_THRESHOLD = 0.02  # 2% failure rate with mixed pass/fail = flaky

    async def analyse_repository(self, tenant_id: UUID, repository_id: UUID) -> list[FlakyTestUpdate]:
        """Scan test_results for tests with mixed pass/fail in the last WINDOW_SIZE runs.
        A test is flaky if: total_runs >= 10 AND 0 < failure_rate < 0.95 AND
        it has both passed and failed within the window (not consistently failing).
        Returns list of FlakyTestUpdate with computed rates."""

    async def update_flaky_tests(self, updates: list[FlakyTestUpdate]) -> None:
        """Upsert into flaky_tests table. If a previously flaky test now passes
        consistently (0 failures in window), mark status='fixed'."""
```

Celery task runs this hourly per repository.

**Testing**:
- Unit: test with 100 runs, 5 failures → flakiness_rate = 0.05, status = `active`
- Unit: test with 100 runs, 0 failures → not flaky (excluded)
- Unit: test with 100 runs, 100 failures → not flaky (consistently failing, not flaky)
- Unit: test with 8 runs → excluded (below minimum threshold)
- Unit: previously flaky test with 0 recent failures → status = `fixed`
- Integration: insert 200 test_results for a test (alternating pass/fail) → flaky_test record created

#### 5.2 — Flaky Test API and Dashboard Data

**What**: REST endpoints for querying flaky tests per repository.

**Design**:

```python
# GET /api/v1/repositories/{repo_id}/flaky-tests?status=active&sort=flakiness_rate
# Response: paginated list of FlakyTest objects with rate, total_runs, last_flaky_at

class FlakyTestResponse(BaseModel):
    id: UUID
    test_suite: str
    test_name: str
    test_class: str | None
    flakiness_rate: float
    total_runs: int
    failed_runs: int
    status: str  # active, quarantined, fixed, ignored
    root_cause: str | None
    suggested_fix: str | None
    first_flaky_at: datetime
    last_flaky_at: datetime
```

**Testing**:
- Integration: 5 flaky tests in DB → GET returns 5, sorted by rate descending
- Integration: `?status=fixed` → only fixed tests returned
- Integration: empty repository → empty list, 200

---

## Phase 6: DORA Metrics Dashboard

### Purpose
Compute and display the four DORA metrics (deployment frequency, lead time, change failure rate, MTTR) per repository. This is the primary dashboard experience.

### Tasks

#### 6.1 — DORA Metrics Calculator

**What**: Compute DORA metrics from build and deployment records.

**Design**:

```python
# src/services/dora_calculator.py
class DoraCalculator:
    async def compute_daily(self, tenant_id: UUID, repo_id: UUID, date: date) -> DoraMetricRow:
        """
        deployment_frequency: count of builds WHERE is_deployment=true AND status='passed' for this date
        lead_time_seconds: median time from first commit in branch to deployment completion
        change_failure_rate: (failed deployments) / (total deployments) over rolling 30 days
        mttr_seconds: median time from failed deployment to next successful deployment
        Also computes: total_builds, avg_build_duration_ms, cache_hit_rate, test_pass_rate, flaky_test_count, total_cost_microcents
        """
```

Celery task runs nightly at 01:00 UTC, backfilling the previous day.

**Testing**:
- Unit: 5 successful deployments in a day → deployment_frequency = 5
- Unit: 3 commits merged, deployed 2h later → lead_time ≈ 7200s
- Unit: 1 failed out of 10 deployments (30d window) → change_failure_rate = 0.10
- Unit: failure at 10:00, recovery at 10:45 → mttr = 2700s
- Unit: no deployments → all metrics null (not zero)
- Integration: insert builds over 7 days → compute_daily returns correct values for each day

#### 6.2 — DORA API Endpoints

**What**: REST endpoints for querying DORA metrics with date ranges.

**Design**:

```python
# GET /api/v1/repositories/{repo_id}/dora?start=2026-05-01&end=2026-05-29
# Response: list of daily DoraMetric objects

# GET /api/v1/dora/summary?period=30d
# Response: aggregate DORA metrics across all repositories for the tenant
```

**Testing**:
- Integration: 30 days of metrics → GET returns 30 rows, correct date ordering
- Integration: `?period=7d` summary → weighted averages across repos
- Integration: no data for date range → empty list, 200

#### 6.3 — Frontend Dashboard

**What**: React SPA with DORA metrics dashboard, build list, and flaky test view.

**Design**:

Pages:
- **Dashboard** (`/`): Four DORA metric cards with sparkline trends (30d), top 5 flaky tests, recent builds
- **Builds** (`/builds`): Paginated build list with status, duration, cache hit rate; click to see jobs timeline
- **Flaky Tests** (`/flaky-tests`): Ranked table of flaky tests with rate, status, root cause (if available)
- **DORA Metrics** (`/dora`): Time-series charts for all four metrics with date range picker; repo selector

Components: `MetricCard`, `BuildTimeline`, `FlakyTestRow`, `DoraChart` (Recharts line chart).

**Testing**:
- E2E: load dashboard with seeded data → 4 metric cards visible with correct values
- E2E: click build row → job timeline expands with correct job names and durations
- E2E: flaky tests page → table sorted by rate, quarantine button works

---

## Phase 7: CI Platform Connectors (Polling Mode)

### Purpose
Add polling-based connectors for CI platforms that don't support webhooks (or where webhook setup is impractical). After this phase, users can connect any supported CI platform without configuring webhooks.

### Tasks

#### 7.1 — GitHub Actions Polling Connector

**What**: Periodically fetch workflow runs and test results via GitHub REST API.

**Design**:

```python
# src/connectors/github_actions.py
class GitHubActionsConnector(BaseCIConnector):
    async def sync(self, repo: Repository, since: datetime) -> list[BuildCreate]:
        """Fetch workflow runs completed after `since` via GET /repos/{owner}/{repo}/actions/runs.
        For each run, fetch jobs via GET /repos/{owner}/{repo}/actions/runs/{id}/jobs.
        Download JUnit artifacts and parse test results."""
```

Celery beat schedules sync every 5 minutes per connected repository.

**Testing**:
- Integration (mocked API): 3 new runs since last sync → 3 builds created
- Integration (mocked API): no new runs → no records created, sync cursor updated
- Unit: rate limit 429 → exponential backoff, retry after header respected

#### 7.2 — GitLab CI Polling Connector

**What**: Periodically fetch pipeline and job data via GitLab REST API.

**Design**: Similar to GitHub connector but uses GitLab Pipelines API (`GET /projects/{id}/pipelines`) and Jobs API. Test results from JUnit report artifacts.

**Testing**:
- Integration (mocked): pipeline with 5 jobs → Build + 5 Jobs created
- Unit: GitLab pipeline status mapping → correct Build.status

#### 7.3 — Jenkins Polling Connector

**What**: Fetch build data from Jenkins REST API.

**Design**: Uses Jenkins JSON API (`/job/{name}/api/json`, `/job/{name}/{build_number}/api/json`). Parses JUnit XML from build artifacts. Maps Jenkins `result` field to Build.status.

**Testing**:
- Integration (mocked): Jenkins build with 3 stages → Build + 3 Jobs
- Unit: Jenkins result `UNSTABLE` → Build status `failed`

#### 7.4 — CircleCI Polling Connector

**What**: Fetch pipeline and workflow data from CircleCI API v2.

**Design**: Uses CircleCI API v2 (`GET /project/{slug}/pipeline`, `GET /pipeline/{id}/workflow`, `GET /workflow/{id}/job`). Test results from JUnit artifacts.

**Testing**:
- Integration (mocked): CircleCI workflow with 4 jobs → Build + 4 Jobs
- Unit: CircleCI job status `infrastructure_fail` → Job status `failed`

---

## Phase 8: Predictive Test Selection (v1.1)

### Purpose
Train an ML model to predict which tests are most likely to fail given the current changeset, enabling developers to run a smaller test subset while maintaining failure detection coverage.

### Tasks

#### 8.1 — Feature Engineering Pipeline

**What**: Extract features from build history for test selection model training.

**Design**:

```python
# src/ml/test_selection.py
@dataclass
class TestSelectionFeatures:
    test_name: str
    historical_failure_rate: float     # failures / total runs (last 90 days)
    days_since_last_failure: int
    avg_duration_ms: float
    is_flaky: bool
    changed_files_overlap: float       # Jaccard similarity between changed files and test's historical failure file set
    author_failure_rate: float         # failure rate for this test when this author commits
    time_since_last_run_hours: float
    commit_message_keywords: list[str] # extracted from commit message

def build_training_dataset(tenant_id: UUID, repo_id: UUID) -> pd.DataFrame:
    """Join test_results with builds to create feature matrix.
    Label: 1 if test failed in this build, 0 if passed.
    Window: last 90 days of builds."""
```

**Testing**:
- Unit: build with 100 tests → 100 feature rows generated
- Unit: test with no history → sensible defaults (failure_rate=0, days_since_last_failure=999)
- Integration: 500 builds × 50 tests → 25,000 row DataFrame with correct columns

#### 8.2 — XGBoost Model Training and Scoring

**What**: Train XGBoost classifier and use it to rank tests by failure probability.

**Design**:

```python
class TestSelector:
    def train(self, features: pd.DataFrame, labels: pd.Series) -> None:
        """Train XGBoost binary classifier. Store model via joblib.
        Target: AUC-ROC > 0.85, following the 89.7% accuracy benchmark."""

    def select(self, features: pd.DataFrame, budget_pct: float = 0.20) -> list[str]:
        """Score all tests, return top budget_pct% ranked by predicted failure probability.
        Always include: tests whose files were directly changed, previously flaky tests."""

    def explain(self, test_name: str) -> dict[str, float]:
        """Return SHAP values for why this test was selected/excluded."""
```

Model stored in S3 as `models/{tenant_id}/{repo_id}/test_selector_v{version}.joblib`. Retrained weekly via Celery task.

**Testing**:
- Unit: model with synthetic data → AUC-ROC > 0.80
- Unit: `select` with budget_pct=0.20 → returns ~20% of tests
- Unit: `select` always includes flaky tests regardless of score
- Integration: train on real-shaped data → model serialises/deserialises correctly
- Integration: explain returns SHAP values summing approximately to prediction score

#### 8.3 — Test Selection API

**What**: Endpoint that accepts a changeset and returns the recommended test subset.

**Design**:

```python
# POST /api/v1/repositories/{repo_id}/test-selection
class TestSelectionRequest(BaseModel):
    commit_sha: str
    changed_files: list[str]
    budget_pct: float = 0.20  # run 20% of tests

class TestSelectionResponse(BaseModel):
    selected_tests: list[str]
    total_tests: int
    selected_count: int
    model_version: str
    confidence: float
```

**Testing**:
- Integration: POST with changed files → selected tests returned, count ≈ 20% of total
- Integration: POST for repo with no model → 503 "Model not yet trained"
- Unit: all changed files in one package → tests from that package ranked higher

---

## Phase 9: Flaky Test Root Cause Attribution (v1.1)

### Purpose
Use LLM analysis to explain *why* a test is flaky and suggest targeted fixes — moving from detection to remediation.

### Tasks

#### 9.1 — Flaky Test Root Cause Analyser

**What**: Send test source code and failure patterns to Claude for root cause classification.

**Design**:

```python
# src/ml/flaky_attribution.py
class FlakyAttributor:
    SYSTEM_PROMPT = """You are a CI/CD expert analysing flaky tests.
    Given the test source code and historical failure patterns, classify the root cause
    into exactly one of: shared_state, timing_dependency, network_call, race_condition,
    order_dependent, resource_leak, environment_specific, unknown.
    Provide a confidence score (0.0-1.0) and a specific suggested fix."""

    async def attribute(self, test_source: str, failure_history: list[dict]) -> FlakyAttribution:
        """Call Claude with test source and failure patterns.
        Return: root_cause, confidence, suggested_fix."""

@dataclass
class FlakyAttribution:
    root_cause: str
    confidence: float
    suggested_fix: str
    evidence: list[str]  # specific lines of code cited
```

Runs as a Celery task for newly detected flaky tests.

**Testing**:
- Unit (mocked LLM): test with `time.sleep` in setup → root_cause = `timing_dependency`
- Unit (mocked LLM): test with shared database state → root_cause = `shared_state`
- Unit (mocked LLM): unparseable LLM response → root_cause = `unknown`, confidence = 0.0
- Integration: flaky test detected → attribution task enqueued → flaky_tests record updated with root_cause

---

## Phase 10: Predictive Build Failure Triage (v1.1)

### Purpose
Predict build failures before they occur and surface early warnings with recommended investigation targets.

### Tasks

#### 10.1 — Failure Prediction Model

**What**: XGBoost model predicting build failure from pre-execution signals.

**Design**:

```python
# src/ml/failure_prediction.py
@dataclass
class FailurePredictionFeatures:
    recent_failure_rate_7d: float          # repo failure rate last 7 days
    author_failure_rate_30d: float         # this author's failure rate
    files_changed_count: int
    flaky_tests_in_scope: int             # flaky tests in affected packages
    time_since_last_success_hours: float
    branch_age_hours: float
    commit_count_in_pr: int
    has_merge_conflicts: bool

class FailurePredictor:
    def predict(self, features: FailurePredictionFeatures) -> FailurePrediction:
        """Return failure probability (0-1), risk_factors (top 3 SHAP contributors),
        and recommended_actions (list of strings)."""
```

**Testing**:
- Unit: high recent failure rate + flaky tests → probability > 0.7
- Unit: clean history, small change → probability < 0.3
- Unit: risk_factors correctly identifies top contributors via SHAP

#### 10.2 — Failure Prediction API and Notifications

**What**: Endpoint returning failure predictions for queued/running builds. Optional Slack/webhook notifications.

**Design**:

```python
# GET /api/v1/builds/{build_id}/prediction
# Response: FailurePrediction with probability, risk_factors, recommended_actions

# POST /api/v1/settings/notifications
class NotificationConfig(BaseModel):
    slack_webhook_url: str | None = None
    failure_threshold: float = 0.75  # only notify above this probability
```

**Testing**:
- Integration: build with high failure probability → prediction returned with risk factors
- Integration: probability > threshold → Slack webhook called (mocked)

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                  ─── required by everything
    │
Phase 2: Build Ingestion             ─── requires Phase 1
    │
    ├── Phase 3: Cache System         ─── requires Phase 1 (parallel with Phase 2)
    │
    ├── Phase 4: Affected Changes     ─── requires Phase 1 (parallel with Phase 2)
    │
Phase 5: Flaky Test Detection        ─── requires Phase 2
    │
Phase 6: DORA Dashboard              ─── requires Phase 2
    │
Phase 7: CI Polling Connectors       ─── requires Phase 2 (parallel with Phases 5-6)
    │
Phase 8: Predictive Test Selection   ─── requires Phase 2 + Phase 5
    │
Phase 9: Flaky Root Cause            ─── requires Phase 5
    │
Phase 10: Failure Prediction         ─── requires Phase 2
```

Parallelism opportunities:
- Phases 3, 4 can be developed concurrently with Phase 2 (no build data dependency)
- Phases 5, 6, 7 can be developed concurrently after Phase 2
- Phases 8, 9, 10 can be developed concurrently after their respective dependencies

---

## Definition of Done (per phase)

1. All tasks implemented with code matching the design specification.
2. All unit and integration tests pass (`pytest` with 0 failures).
3. Ruff linting passes with zero warnings (`ruff check .`).
4. mypy type checking passes in strict mode (`mypy --strict src/`).
5. Docker build succeeds (`docker build -t cicd-optimizer .`).
6. `docker-compose up` brings all services to healthy state.
7. Feature works end-to-end (manual or automated E2E test).
8. New API endpoints appear in auto-generated OpenAPI spec at `/openapi.json`.
9. Database migrations created and tested (`alembic upgrade head` + `alembic downgrade -1`).
10. New environment variables documented in `src/config.py` with defaults.

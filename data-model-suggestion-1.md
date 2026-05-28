# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: CI/CD Pipeline Optimizer · Created: 2026-05-29

## Philosophy

This model gives every CI/CD optimization concept its own table with explicit foreign keys. Repositories, pipelines, builds, jobs, tests, cache entries, and DORA metrics each have dedicated tables. Build execution is tracked at the job and test level, with flaky test detection via statistical analysis across runs. Predictive test selection results, build failure predictions, and cost data are stored as first-class analytical entities. The schema supports multiple CI platforms (GitHub Actions, GitLab CI, Jenkins, CircleCI, Buildkite) through a platform-agnostic pipeline model.

The schema aligns with DORA metrics definitions, SLSA provenance requirements, OpenTelemetry trace concepts for build spans, and IEEE 29119 test documentation standards.

**Best for:** Platform engineering teams requiring full SQL access to build performance data, cross-CI-platform pipeline analysis, auditable test selection decisions, and DORA metrics calculation with referential integrity across repositories, pipelines, builds, jobs, and tests.

**Trade-offs:**
- (+) Full referential integrity from repository → pipeline → build → job → test result
- (+) Cross-CI-platform analysis with consistent schema
- (+) DORA metrics computed from normalised build and deployment data
- (+) Flaky test detection via statistical queries on test result history
- (-) Higher table count for a domain with many entities
- (-) High-volume test results require partitioned tables
- (-) Adding new CI platform connectors requires adapter development

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| DORA Metrics | Deployment frequency, lead time, change failure rate, MTTR computed from builds/deployments |
| SLSA v1.0 | Build provenance attestation tracked per build |
| OpenTelemetry (OTLP) | Build and job spans modelled as trace-like entities |
| IEEE 29119-3 | Test result data structure aligned with test documentation standard |
| CycloneDX / SPDX | SBOM generation tracked as build artifacts |
| OpenAPI 3.2 | REST API documentation |
| ISO 8601 | All timestamps in ISO 8601 format |

---

## Repositories & Pipelines

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan            TEXT NOT NULL CHECK (plan IN ('free', 'team', 'enterprise')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           TEXT NOT NULL,
    name            TEXT NOT NULL,
    role            TEXT NOT NULL CHECK (role IN ('owner', 'admin', 'engineer', 'viewer', 'api_service')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);

CREATE TABLE repositories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    full_name       TEXT NOT NULL,
    vcs_provider    TEXT NOT NULL CHECK (vcs_provider IN ('github', 'gitlab', 'bitbucket', 'azure_devops')),
    default_branch  TEXT NOT NULL DEFAULT 'main',
    is_monorepo     BOOLEAN NOT NULL DEFAULT false,
    language_primary TEXT,
    build_system    TEXT CHECK (build_system IN ('gradle', 'maven', 'npm', 'yarn', 'pnpm', 'turborepo', 'nx', 'bazel', 'make', 'cargo', 'go', 'other')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, full_name)
);

CREATE INDEX idx_repositories_tenant ON repositories(tenant_id);

CREATE TABLE pipelines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    repository_id   UUID NOT NULL REFERENCES repositories(id),
    name            TEXT NOT NULL,
    ci_platform     TEXT NOT NULL CHECK (ci_platform IN ('github_actions', 'gitlab_ci', 'jenkins', 'circleci', 'buildkite', 'harness', 'other')),
    config_path     TEXT,
    trigger_type    TEXT CHECK (trigger_type IN ('push', 'pull_request', 'schedule', 'manual', 'tag')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    config_hash     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pipelines_repository ON pipelines(repository_id);
```

## Builds & Jobs

```sql
CREATE TABLE builds (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    pipeline_id     UUID NOT NULL REFERENCES pipelines(id),
    external_id     TEXT NOT NULL,
    branch          TEXT NOT NULL,
    commit_sha      TEXT NOT NULL,
    commit_message  TEXT,
    author          TEXT,
    trigger         TEXT CHECK (trigger IN ('push', 'pull_request', 'schedule', 'manual', 'tag', 'api')),
    status          TEXT NOT NULL CHECK (status IN ('queued', 'running', 'passed', 'failed', 'cancelled', 'timed_out')),
    started_at      TIMESTAMPTZ,
    finished_at     TIMESTAMPTZ,
    duration_ms     BIGINT,
    queue_time_ms   BIGINT,
    is_deployment   BOOLEAN NOT NULL DEFAULT false,
    deployment_env  TEXT,
    runner_type     TEXT,
    runner_os       TEXT,
    cost_microcents BIGINT,
    slsa_level      SMALLINT,
    provenance_uri  TEXT,
    failure_prediction JSONB,
    -- {"predicted_at_stage": 2, "confidence": 0.89, "risk_factors": ["flaky_test_x", "recent_commit_y"]}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (started_at);

CREATE INDEX idx_builds_pipeline ON builds(pipeline_id, started_at);
CREATE INDEX idx_builds_tenant ON builds(tenant_id, started_at);
CREATE INDEX idx_builds_status ON builds(tenant_id, status);
CREATE INDEX idx_builds_commit ON builds(commit_sha);

CREATE TABLE jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    build_id        UUID NOT NULL REFERENCES builds(id),
    name            TEXT NOT NULL,
    stage           TEXT,
    stage_order     SMALLINT,
    status          TEXT NOT NULL CHECK (status IN ('queued', 'running', 'passed', 'failed', 'cancelled', 'skipped')),
    started_at      TIMESTAMPTZ,
    finished_at     TIMESTAMPTZ,
    duration_ms     BIGINT,
    queue_time_ms   BIGINT,
    runner_type     TEXT,
    runner_os       TEXT,
    cost_microcents BIGINT,
    cache_hit       BOOLEAN,
    cache_key       TEXT,
    cache_size_bytes BIGINT,
    retry_count     SMALLINT NOT NULL DEFAULT 0,
    log_url         TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_jobs_build ON jobs(build_id);
CREATE INDEX idx_jobs_name ON jobs(name);
CREATE INDEX idx_jobs_cache ON jobs(cache_key) WHERE cache_hit;
```

## Tests & Flaky Detection

```sql
CREATE TABLE test_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    build_id        UUID NOT NULL REFERENCES builds(id),
    job_id          UUID NOT NULL REFERENCES jobs(id),
    test_suite      TEXT NOT NULL,
    test_name       TEXT NOT NULL,
    test_class      TEXT,
    status          TEXT NOT NULL CHECK (status IN ('passed', 'failed', 'skipped', 'error', 'flaky_passed')),
    duration_ms     BIGINT,
    failure_message TEXT,
    failure_type    TEXT,
    retry_number    SMALLINT NOT NULL DEFAULT 0,
    is_selected     BOOLEAN NOT NULL DEFAULT true,
    selection_model TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_test_results_build ON test_results(build_id);
CREATE INDEX idx_test_results_test ON test_results(tenant_id, test_suite, test_name);
CREATE INDEX idx_test_results_failed ON test_results(tenant_id) WHERE status IN ('failed', 'error');

CREATE TABLE flaky_tests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    repository_id   UUID NOT NULL REFERENCES repositories(id),
    test_suite      TEXT NOT NULL,
    test_name       TEXT NOT NULL,
    test_class      TEXT,
    flakiness_rate  NUMERIC(5,4) NOT NULL,
    total_runs      INT NOT NULL,
    failed_runs     INT NOT NULL,
    first_flaky_at  TIMESTAMPTZ NOT NULL,
    last_flaky_at   TIMESTAMPTZ NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'quarantined', 'fixed', 'ignored')),
    root_cause      TEXT CHECK (root_cause IN (
        'shared_state', 'timing_dependency', 'network_call', 'race_condition',
        'order_dependent', 'resource_leak', 'environment_specific', 'unknown'
    )),
    root_cause_confidence NUMERIC(5,4),
    suggested_fix   TEXT,
    model_version   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, repository_id, test_suite, test_name)
);

CREATE INDEX idx_flaky_tests_repo ON flaky_tests(repository_id);
CREATE INDEX idx_flaky_tests_rate ON flaky_tests(tenant_id, flakiness_rate DESC);
```

## Cache, Metrics & AI

```sql
CREATE TABLE cache_entries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    cache_key       TEXT NOT NULL,
    repository_id   UUID NOT NULL REFERENCES repositories(id),
    size_bytes      BIGINT NOT NULL,
    hit_count       INT NOT NULL DEFAULT 0,
    last_hit_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ,
    UNIQUE (tenant_id, cache_key)
);

CREATE INDEX idx_cache_entries_repo ON cache_entries(repository_id);

CREATE TABLE dora_metrics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    repository_id   UUID REFERENCES repositories(id),
    metric_date     DATE NOT NULL,
    deployment_frequency NUMERIC(10,4),
    lead_time_seconds BIGINT,
    change_failure_rate NUMERIC(5,4),
    mttr_seconds    BIGINT,
    total_builds    INT NOT NULL DEFAULT 0,
    total_deployments INT NOT NULL DEFAULT 0,
    avg_build_duration_ms BIGINT,
    cache_hit_rate  NUMERIC(5,4),
    test_pass_rate  NUMERIC(5,4),
    flaky_test_count INT NOT NULL DEFAULT 0,
    total_cost_microcents BIGINT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, repository_id, metric_date)
);

CREATE INDEX idx_dora_tenant ON dora_metrics(tenant_id, metric_date);

CREATE TABLE ai_suggestions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    suggestion_type TEXT NOT NULL CHECK (suggestion_type IN (
        'pipeline_optimization', 'test_selection', 'flaky_root_cause',
        'cache_improvement', 'parallelization', 'cost_saving',
        'failure_prediction', 'yaml_refactor', 'dora_insight',
        'build_anomaly'
    )),
    entity_type     TEXT,
    entity_id       UUID,
    severity        TEXT NOT NULL CHECK (severity IN ('info', 'warning', 'critical')),
    title           TEXT NOT NULL,
    description     TEXT NOT NULL,
    evidence        JSONB NOT NULL DEFAULT '{}',
    recommended_action TEXT,
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'accepted', 'dismissed', 'expired', 'auto_applied')),
    confidence      NUMERIC(5,4),
    model_version   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);

CREATE INDEX idx_ai_suggestions_tenant ON ai_suggestions(tenant_id);
CREATE INDEX idx_ai_suggestions_pending ON ai_suggestions(tenant_id) WHERE status = 'pending';

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    actor_type      TEXT NOT NULL CHECK (actor_type IN ('user', 'system', 'api_key', 'ci_platform', 'ai', 'scheduler')),
    actor_id        TEXT,
    action          TEXT NOT NULL,
    resource_type   TEXT NOT NULL,
    resource_id     UUID,
    changes         JSONB,
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_log_tenant ON audit_log(tenant_id, created_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Entity Management | 2 | tenants, users |
| Repositories & Pipelines | 2 | repositories, pipelines |
| Builds & Jobs | 2 | builds (partitioned), jobs |
| Tests & Flaky Detection | 2 | test_results (partitioned), flaky_tests |
| Cache, Metrics & AI | 4 | cache_entries, dora_metrics, ai_suggestions, audit_log (partitioned) |
| **Total** | **12** | 3 partitioned tables |

---

## Key Design Decisions

1. **CI-platform-agnostic pipeline model** — pipelines.ci_platform distinguishes GitHub Actions, GitLab CI, Jenkins, etc. while builds, jobs, and test results share a consistent schema regardless of source.

2. **Builds with failure prediction** — builds.failure_prediction stores the ML model's early warning with confidence and risk factors, enabling "predicted failure at stage 2" alerts before the build completes.

3. **Jobs with cache tracking** — jobs carry cache_hit, cache_key, and cache_size_bytes for cache efficiency analysis. The cache_entries table tracks the global cache state.

4. **Test results with selection tracking** — test_results.is_selected and selection_model record whether a test was run by the predictive test selection model or skipped, enabling selection accuracy validation.

5. **Flaky tests with root cause** — flaky_tests stores statistical flakiness rates alongside AI-attributed root causes (shared_state, timing, network, race_condition) and suggested fixes.

6. **DORA metrics pre-computed daily** — dora_metrics stores the four key metrics plus build performance, cache hit rate, and cost per day per repository, powering the DORA dashboard.

7. **Cost tracking at job level** — jobs.cost_microcents and builds.cost_microcents enable FinOps analysis per build, per repository, and per team.

8. **SLSA provenance on builds** — builds.slsa_level and provenance_uri track supply chain security attestation per build.

9. **Build system on repository** — repositories.build_system identifies the build toolchain (gradle, npm, turborepo, etc.) for cross-ecosystem pipeline analysis.

10. **Monorepo flag** — repositories.is_monorepo enables affected-change detection logic that differs between monorepo and single-package repositories.

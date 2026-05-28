# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: CI/CD Pipeline Optimizer · Created: 2026-05-29

## Philosophy

This model keeps builds and test results as relational tables (for time-series queries and DORA metrics calculation) while embedding pipeline configurations, job details, cache state, and flaky test analysis into JSONB on their parent records. Each build is a self-contained execution record with embedded job timeline, test summary, and cost breakdown. The tenant holds repository configurations, pipeline definitions, and DORA metric settings.

CI/CD optimization has two access patterns: time-series queries ("build duration trend for the last 250 builds") that need relational indexing; and build-level queries ("show me everything about this build including all jobs and test failures") that benefit from a single-document model.

**Best for:** Teams building a rapid MVP that needs to iterate on pipeline analysis features without migrations, while maintaining relational performance for build time-series and DORA metrics dashboards.

**Trade-offs:**
- (+) Build is a complete execution record — single fetch for build detail view
- (+) New job types or test frameworks added as JSONB without ALTER TABLE
- (+) Builds and test results remain relational for time-series queries
- (-) Cross-build job analysis requires JSONB operators
- (-) Flaky test detection across builds requires scanning embedded test data
- (-) Larger build rows for builds with many jobs and tests

---

## Core Tables

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,

    users_json      JSONB NOT NULL DEFAULT '[]',

    repositories_json JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "name": "frontend", "full_name": "org/frontend",
    --   "vcs_provider": "github", "default_branch": "main",
    --   "is_monorepo": true, "build_system": "turborepo",
    --   "pipelines": [{
    --     "id": "uuid", "name": "CI", "ci_platform": "github_actions",
    --     "config_path": ".github/workflows/ci.yml", "trigger_type": "pull_request"
    --   }]
    -- }]

    flaky_tests_json JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "repository_id": "uuid", "test_suite": "unit", "test_name": "test_user_create",
    --   "flakiness_rate": 0.08, "total_runs": 250, "failed_runs": 20,
    --   "status": "active", "root_cause": "shared_state",
    --   "suggested_fix": "Reset database state in setUp()", "confidence": 0.82
    -- }]

    cache_config_json JSONB NOT NULL DEFAULT '{}',
    -- {"max_size_gb": 50, "ttl_hours": 168, "entries": [
    --   {"key": "npm-abc123", "repo_id": "uuid", "size_bytes": 52428800, "hit_count": 45}
    -- ]}

    config_json     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "plan": "team",
    --   "dora": {"enabled": true, "deployment_branch": "main"},
    --   "test_selection": {"enabled": true, "model": "xgboost_v2", "confidence_threshold": 0.8},
    --   "cost_tracking": {"enabled": true, "currency": "USD"},
    --   "alerts": {"channels": ["slack"], "failure_prediction_threshold": 0.85}
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenants_slug ON tenants(slug);

CREATE TABLE builds (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    repository_id   UUID NOT NULL,
    pipeline_id     UUID NOT NULL,
    external_id     TEXT NOT NULL,
    branch          TEXT NOT NULL,
    commit_sha      TEXT NOT NULL,
    commit_message  TEXT,
    author          TEXT,
    trigger         TEXT,
    status          TEXT NOT NULL CHECK (status IN ('queued', 'running', 'passed', 'failed', 'cancelled', 'timed_out')),
    started_at      TIMESTAMPTZ,
    finished_at     TIMESTAMPTZ,
    duration_ms     BIGINT,
    queue_time_ms   BIGINT,
    is_deployment   BOOLEAN NOT NULL DEFAULT false,
    deployment_env  TEXT,
    cost_microcents BIGINT,

    -- Embedded jobs
    jobs_json       JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "name": "build", "stage": "build", "stage_order": 1,
    --   "status": "passed", "duration_ms": 45000, "queue_time_ms": 3000,
    --   "runner_type": "ubuntu-latest", "cost_microcents": 6000,
    --   "cache_hit": true, "cache_key": "npm-abc123", "cache_size_bytes": 52428800
    -- }, {
    --   "id": "uuid", "name": "test", "stage": "test", "stage_order": 2,
    --   "status": "failed", "duration_ms": 180000,
    --   "cache_hit": false, "retry_count": 1
    -- }]

    -- Test summary
    test_summary_json JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "total": 1250, "passed": 1240, "failed": 8, "skipped": 2, "flaky": 3,
    --   "duration_ms": 180000, "selected_by_model": 450, "selection_model": "xgboost_v2",
    --   "failures": [
    --     {"suite": "unit", "name": "test_user_create", "message": "AssertionError", "is_flaky": true}
    --   ]
    -- }

    -- Failure prediction
    prediction_json JSONB,
    -- {"predicted_at_stage": 2, "confidence": 0.89, "risk_factors": ["flaky_test_x", "recent_commit_y"]}

    -- SLSA provenance
    slsa_json       JSONB,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (started_at);

CREATE INDEX idx_builds_tenant ON builds(tenant_id, started_at);
CREATE INDEX idx_builds_repo ON builds(repository_id, started_at);
CREATE INDEX idx_builds_status ON builds(tenant_id, status);
CREATE INDEX idx_builds_commit ON builds(commit_sha);

CREATE TABLE test_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    build_id        UUID NOT NULL REFERENCES builds(id),
    test_suite      TEXT NOT NULL,
    test_name       TEXT NOT NULL,
    status          TEXT NOT NULL CHECK (status IN ('passed', 'failed', 'skipped', 'error', 'flaky_passed')),
    duration_ms     BIGINT,
    failure_message TEXT,
    is_selected     BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_test_results_build ON test_results(build_id);
CREATE INDEX idx_test_results_test ON test_results(tenant_id, test_suite, test_name);
CREATE INDEX idx_test_results_failed ON test_results(tenant_id) WHERE status IN ('failed', 'error');

CREATE TABLE ai_suggestions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    suggestion_type TEXT NOT NULL CHECK (suggestion_type IN (
        'pipeline_optimization', 'test_selection', 'flaky_root_cause',
        'cache_improvement', 'parallelization', 'cost_saving',
        'failure_prediction', 'yaml_refactor', 'dora_insight', 'build_anomaly'
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
| Configuration | 1 | tenants (embeds users, repositories, pipelines, flaky tests, cache, config) |
| Builds & Tests | 2 | builds (partitioned, embeds jobs/test summary/prediction), test_results (partitioned) |
| AI & Audit | 2 | ai_suggestions, audit_log (partitioned) |
| **Total** | **5** | 3 partitioned tables |

---

## Key Design Decisions

1. **Build as a complete execution record** — jobs, test summary, failure prediction, and SLSA provenance all embed on the build. The build detail view is a single-row fetch.

2. **Test results remain relational** — individual test results stay relational for flaky test detection via statistical queries across builds (GROUP BY test_name, COUNT failures).

3. **Repositories and pipelines on tenant** — the repository catalogue with pipeline definitions embeds on the tenant. Adding a new CI platform or repository is a JSONB update.

4. **Flaky tests on tenant** — tenants.flaky_tests_json stores the current flaky test inventory with root causes and suggested fixes, enabling the flaky test dashboard from a single tenant fetch.

5. **Cache state on tenant** — cache entries and configuration embed on the tenant for the cache management view.

6. **Test selection tracking on builds** — builds.test_summary_json records how many tests were selected by the ML model vs. total, enabling selection accuracy validation.

7. **DORA metrics computed from builds** — deployment frequency, lead time, change failure rate, and MTTR are computed from builds WHERE is_deployment = true rather than stored in a separate table.

8. **Cost tracking embedded on builds and jobs** — cost_microcents on builds and in jobs_json enables FinOps analysis without a separate cost table.

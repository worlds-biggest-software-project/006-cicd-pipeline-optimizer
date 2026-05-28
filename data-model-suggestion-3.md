# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: CI/CD Pipeline Optimizer · Created: 2026-05-29

## Philosophy

This model treats every build execution, job completion, test result, cache operation, failure prediction, and optimization action as an immutable event. The pipeline's performance state is not a mutable record — it is a projection computed by replaying build and test events. DORA metrics are projections from deployment events. Flaky test detection replays test result events to compute statistical variance. Predictive test selection and failure prediction are events with full model input traceability.

CI/CD optimization is naturally event-driven: builds start, jobs execute, tests pass or fail, caches hit or miss. An event-sourced model preserves the complete execution timeline, enables "as-of" DORA queries, and permanently records every AI prediction for validation against actual outcomes.

**Best for:** Platform engineering teams requiring full build execution audit trails, retroactive DORA metric recalculation, permanent recording of every AI prediction for accuracy validation, and temporal analysis of build performance evolution.

**Trade-offs:**
- (+) Complete build execution timeline for every pipeline run
- (+) AI predictions permanently recorded and validated against actual outcomes
- (+) DORA metrics recomputable with new definitions without backfilling
- (+) Flaky test detection from event replay with configurable windows
- (-) Read models must be maintained for dashboards
- (-) High test result event volume requires partition management
- (-) Current DORA metrics require read model, not direct query

---

## Event Infrastructure

```sql
CREATE TABLE event_store (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     TEXT NOT NULL CHECK (stream_type IN (
        'tenant', 'user', 'repository', 'pipeline',
        'build', 'job', 'test', 'cache',
        'prediction', 'optimization', 'ai', 'config'
    )),
    stream_id       UUID NOT NULL,
    version         BIGINT NOT NULL,
    event_type      TEXT NOT NULL,
    actor_type      TEXT NOT NULL CHECK (actor_type IN (
        'user', 'system', 'api_key', 'ci_platform', 'ai',
        'test_selector', 'failure_predictor', 'cache_engine',
        'scheduler', 'projection_engine'
    )),
    actor_id        TEXT,
    tenant_id       UUID,
    ce_source       TEXT NOT NULL,
    ce_type         TEXT NOT NULL,
    ce_time         TIMESTAMPTZ NOT NULL,
    ce_specversion  TEXT NOT NULL DEFAULT '1.0',
    data            JSONB NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_type, stream_id, version)
) PARTITION BY RANGE (ce_time);

CREATE INDEX idx_events_stream ON event_store(stream_type, stream_id, version);
CREATE INDEX idx_events_tenant ON event_store(tenant_id, ce_time);
CREATE INDEX idx_events_type ON event_store(event_type, ce_time);

CREATE TABLE stream_snapshots (
    stream_type     TEXT NOT NULL,
    stream_id       UUID NOT NULL,
    version         BIGINT NOT NULL,
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_type, stream_id, version)
);

CREATE TABLE projection_checkpoints (
    projection_name TEXT NOT NULL,
    partition_key   TEXT NOT NULL DEFAULT '_global',
    last_event_id   UUID NOT NULL,
    last_event_time TIMESTAMPTZ NOT NULL,
    state           JSONB NOT NULL DEFAULT '{}',
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (projection_name, partition_key)
);
```

## Event Taxonomy

### Build Lifecycle Events
- `build_queued` — {pipeline_id, branch, commit_sha, author, trigger}
- `build_started` — {runner_type, runner_os}
- `build_completed` — {status, duration_ms, cost_microcents}
- `build_deployed` — {environment, deployment_id}

### Job Events
- `job_started` — {build_id, name, stage, stage_order, runner_type}
- `job_completed` — {status, duration_ms, cost_microcents, retry_count}
- `job_cache_hit` — {cache_key, cache_size_bytes, time_saved_ms}
- `job_cache_miss` — {cache_key, expected_key}
- `job_cache_stored` — {cache_key, size_bytes}

### Test Events
- `test_executed` — {build_id, job_id, suite, name, class, status, duration_ms, failure_message, is_selected, selection_model}
- `test_retried` — {build_id, suite, name, retry_number, status}

### Prediction Events
- `failure_predicted` — {build_id, predicted_at_stage, confidence, risk_factors, model_version}
- `failure_prediction_validated` — {build_id, predicted, actual, was_correct}
- `test_selection_computed` — {build_id, total_tests, selected_tests, model_version, confidence}
- `test_selection_validated` — {build_id, selected_count, failures_caught, failures_missed}

### Flaky Test Events
- `flaky_test_detected` — {repository_id, suite, name, flakiness_rate, total_runs, failed_runs}
- `flaky_test_root_cause_attributed` — {suite, name, root_cause, confidence, suggested_fix, model_version}
- `flaky_test_quarantined` — {suite, name}
- `flaky_test_fixed` — {suite, name, fix_commit}

### Optimization Events
- `pipeline_optimization_suggested` — {pipeline_id, optimization_type, expected_savings_ms, description}
- `cache_optimization_suggested` — {repository_id, current_hit_rate, projected_hit_rate, recommendation}
- `cost_optimization_suggested` — {repository_id, current_cost, projected_cost, recommendation}
- `yaml_refactor_suggested` — {pipeline_id, changes, rationale}

### AI Events
- `ai_suggestion_generated` — {suggestion_type, entity_type, entity_id, title, description, evidence, confidence}
- `ai_suggestion_accepted` — {suggestion_id}
- `ai_suggestion_dismissed` — {suggestion_id, reason}

---

## Read Models

```sql
CREATE TABLE rm_build_dashboard (
    tenant_id       UUID NOT NULL,
    repository_id   UUID NOT NULL,
    metric_date     DATE NOT NULL,
    total_builds        INT NOT NULL DEFAULT 0,
    passed              INT NOT NULL DEFAULT 0,
    failed              INT NOT NULL DEFAULT 0,
    avg_duration_ms     BIGINT,
    p95_duration_ms     BIGINT,
    avg_queue_time_ms   BIGINT,
    cache_hit_rate      NUMERIC(5,4),
    test_pass_rate      NUMERIC(5,4),
    flaky_test_count    INT NOT NULL DEFAULT 0,
    total_cost_microcents BIGINT NOT NULL DEFAULT 0,
    prediction_accuracy NUMERIC(5,4),
    selection_accuracy  NUMERIC(5,4),
    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, repository_id, metric_date)
) PARTITION BY RANGE (metric_date);

CREATE TABLE rm_dora_metrics (
    tenant_id       UUID NOT NULL,
    repository_id   UUID,
    metric_date     DATE NOT NULL,
    deployment_frequency NUMERIC(10,4),
    lead_time_seconds    BIGINT,
    change_failure_rate  NUMERIC(5,4),
    mttr_seconds         BIGINT,
    performance_tier     TEXT CHECK (performance_tier IN ('elite', 'high', 'medium', 'low')),
    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, repository_id, metric_date)
);

CREATE TABLE rm_flaky_test_registry (
    tenant_id       UUID NOT NULL,
    repository_id   UUID NOT NULL,
    test_suite      TEXT NOT NULL,
    test_name       TEXT NOT NULL,
    flakiness_rate  NUMERIC(5,4) NOT NULL,
    total_runs      INT NOT NULL,
    status          TEXT NOT NULL,
    root_cause      TEXT,
    root_cause_confidence NUMERIC(5,4),
    suggested_fix   TEXT,
    first_flaky_at  TIMESTAMPTZ,
    last_flaky_at   TIMESTAMPTZ,
    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, repository_id, test_suite, test_name)
);

CREATE INDEX idx_rm_flaky_rate ON rm_flaky_test_registry(tenant_id, flakiness_rate DESC);

CREATE TABLE rm_pipeline_health (
    tenant_id       UUID NOT NULL,
    pipeline_id     UUID NOT NULL,
    name            TEXT NOT NULL,
    ci_platform     TEXT NOT NULL,
    repository_name TEXT NOT NULL,
    avg_duration_ms     BIGINT,
    cache_hit_rate      NUMERIC(5,4),
    failure_rate_7d     NUMERIC(5,4),
    cost_7d_microcents  BIGINT,
    optimization_suggestions JSONB NOT NULL DEFAULT '[]',
    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, pipeline_id)
);

CREATE TABLE rm_prediction_accuracy (
    tenant_id       UUID NOT NULL,
    model_type      TEXT NOT NULL,
    model_version   TEXT NOT NULL,
    metric_date     DATE NOT NULL,
    total_predictions   INT NOT NULL DEFAULT 0,
    correct_predictions INT NOT NULL DEFAULT 0,
    accuracy        NUMERIC(5,4),
    precision_score NUMERIC(5,4),
    recall_score    NUMERIC(5,4),
    f1_score        NUMERIC(5,4),
    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, model_type, model_version, metric_date)
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 3 | event_store (partitioned), stream_snapshots, projection_checkpoints |
| Read Models | 5 | rm_build_dashboard (partitioned), rm_dora_metrics, rm_flaky_test_registry, rm_pipeline_health, rm_prediction_accuracy |
| **Total** | **8** | 2 partitioned tables |

---

## Key Design Decisions

1. **Build lifecycle as events** — build_queued → build_started → build_completed → build_deployed captures the full execution timeline. DORA lead time is computed from the first commit event to deployment.

2. **Prediction validation as events** — failure_predicted and failure_prediction_validated events create a permanent accuracy record. rm_prediction_accuracy materialises model performance metrics.

3. **Test selection validation** — test_selection_computed and test_selection_validated events track which tests were selected and whether they caught all failures, enabling selection model improvement.

4. **Flaky test lifecycle as events** — flaky_test_detected → root_cause_attributed → quarantined → fixed creates a complete remediation audit trail.

5. **DORA metrics as a projection** — rm_dora_metrics computes the four key metrics from build and deployment events with performance tier classification (elite/high/medium/low).

6. **Cache operations as events** — job_cache_hit, job_cache_miss, and job_cache_stored events enable cache efficiency analysis and optimization suggestions.

7. **Cost tracking from events** — build and job completion events carry cost_microcents. rm_build_dashboard aggregates daily costs per repository.

8. **Pipeline health as a projection** — rm_pipeline_health provides per-pipeline summary including optimization suggestions from AI events.

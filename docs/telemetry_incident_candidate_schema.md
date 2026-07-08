# Telemetry Incident Candidate Contract

## 1. Purpose

This document defines the shared JSON contract used to exchange incident candidates between telemetry-specific detection pipelines and the later cross-signal correlation stage.

The first producer is [`notebooks/02_metrics_detection.ipynb`](../notebooks/02_metrics_detection.ipynb), which writes `data/outputs/metric_incident_candidates.json`. The log pipeline should write `log_incident_candidates.json` using the same document and candidate envelope. Trace candidates can adopt the contract later.

The contract separates two concerns:

- telemetry-specific pipelines detect and correlate evidence within one signal source;
- cross-signal correlation compares the resulting candidates using normalized time and service identity.

A metric candidate represents the metric evidence already correlated within a scenario. It is not a confirmed incident or root-cause conclusion.

## 2. Contract principles

1. All correlation timestamps are normalized to UTC and serialized as RFC 3339 strings ending in `Z`.
2. Every candidate represents an interval, not only a single timestamp.
3. `scenario_id` and canonical service keys are the primary non-temporal join keys.
4. Source-local evidence remains source-specific; metric and log detector scores must not be compared directly.
5. Public IDs must be deterministic across equivalent reruns.
6. Sequential notebook IDs such as `feature_episode_0013` are internal implementation details and must not appear in the public JSON contract.
7. A candidate contains enough semantic evidence to be understood after the notebook kernel and intermediate DataFrames no longer exist.

## 3. Shared document envelope

Both metric and log artifacts use this top-level structure:

```json
{
  "schema_name": "aiops.metric_incident_candidates",
  "schema_version": "1.0.0",
  "generated_at": "2026-07-08T07:09:01.192987Z",
  "producer": {},
  "time_contract": {},
  "provenance": {},
  "candidates": []
}
```

All fields in this table are required:

| Field | Type | Meaning |
|---|---|---|
| `schema_name` | string | Artifact schema, for example `aiops.metric_incident_candidates` or `aiops.log_incident_candidates` |
| `schema_version` | string | Semantic version of this source-specific contract |
| `generated_at` | RFC 3339 timestamp | Time at which the artifact was generated |
| `producer` | object | Pipeline and detector metadata |
| `time_contract` | object | Timestamp, interval, resolution, and tolerance semantics |
| `provenance` | object | Dataset processing and timestamp-normalization metadata |
| `candidates` | array | Zero or more incident candidates |

The artifact does not include `candidate_count`; consumers can derive it from `candidates.length` without risking an inconsistent duplicate value.

## 4. Time contract

Example metric time contract:

```json
{
  "normalized_timezone": "UTC",
  "timestamp_format": "RFC3339",
  "interval_semantics": "closed",
  "native_resolution_seconds": 30,
  "default_correlation_tolerance_seconds": 90
}
```

All fields in this table are required:

| Field | Rule |
|---|---|
| `normalized_timezone` | Must be `UTC` for cross-signal correlation |
| `timestamp_format` | Must be `RFC3339` |
| `interval_semantics` | Currently `closed`: both start and end are observed timestamps |
| `native_resolution_seconds` | Resolution of the producer's normalized evidence |
| `default_correlation_tolerance_seconds` | Default allowed temporal gap between candidates |

Log source timestamps in the current dataset appear to use UTC+8 local time. The log pipeline must normalize them to UTC before emitting candidates and preserve the assumption:

```json
{
  "timestamp_normalization": {
    "source_timezone": "UTC+08:00",
    "applied_offset_seconds": -28800,
    "assumption": true,
    "confidence": "medium"
  }
}
```

Metric timestamps are already UTC and therefore use an offset of zero with `assumption: false`.

## 5. Shared candidate envelope

Every candidate must contain:

```json
{
  "candidate_id": "mic_b541dfeaa3ebbeea",
  "signal_source": "metrics",
  "event_time": {},
  "scope": {},
  "correlation": {},
  "summary": {},
  "evidence": []
}
```

All fields in this table are required:

| Field | Type | Shared semantics |
|---|---|---|
| `candidate_id` | string | Deterministic candidate identifier |
| `signal_source` | enum | `metrics`, `logs`, or `traces` |
| `event_time` | object | Normalized occurrence interval |
| `scope` | object | Affected services and their roles |
| `correlation` | object | Keys consumed by cross-signal correlation |
| `summary` | object | Compact source-specific information for filtering and ranking |
| `evidence` | array | Source-specific evidence embedded in the candidate |

`summary` and `evidence` intentionally differ between metrics and logs. Their internal fields are not cross-source join keys.

## 6. Candidate identity

`candidate_id` must be derived from stable semantic content, such as:

```text
schema_version
 signal_source
 scenario_id
 event start/end
 scope
 sorted service_keys
 sorted affected signal names
```

The ID must not depend on DataFrame row order or sequential internal IDs. The current metric producer uses a SHA-256 digest with the prefix `mic_`.

Recommended source prefixes are:

| Source | Prefix |
|---|---|
| Metrics | `mic_` |
| Logs | `lic_` |
| Traces | `tic_` |
| Cross-signal incident | `cic_` |

## 7. Event time

```json
{
  "start": "2022-07-13T11:25:00Z",
  "end": "2022-07-13T11:33:00Z",
  "duration_seconds": 480.0
}
```

`start` is the first observed time in the correlated candidate, and `end` is the last observed time. `duration_seconds` is retained as an intentional convenience field for filtering and ranking, although it can be derived from the interval.

## 8. Service identity and scope

```json
{
  "level": "cross_entity",
  "entities": [
    {
      "role": "application",
      "service": {
        "namespace": "train-ticket",
        "name": "ts-auth-service"
      }
    },
    {
      "role": "database",
      "service": {
        "namespace": "train-ticket",
        "name": "ts-auth-mongo"
      }
    }
  ]
}
```

Allowed scope levels currently are:

- `entity_local`: one service/component is represented;
- `cross_entity`: more than one service/component is represented.

The role is descriptive. Correlation must use canonical service identity rather than aliases such as `application` and `mongo`.

Canonical mappings for the current scenario are:

| Internal entity | Role | Canonical service key |
|---|---|---|
| `application` | `application` | `train-ticket/ts-auth-service` |
| `mongo` | `database` | `train-ticket/ts-auth-mongo` |

The same `service.namespace` and `service.name` values must be emitted by the log pipeline.

## 9. Correlation keys

```json
{
  "scenario_id": "ts-auth-mongo_4.4.15_2022-07-13",
  "service_keys": [
    "train-ticket/ts-auth-service",
    "train-ticket/ts-auth-mongo"
  ],
  "time_tolerance_seconds": 90
}
```

Required correlation fields:

| Field | Meaning |
|---|---|
| `scenario_id` | Experiment or operational scenario partition |
| `service_keys` | Sorted canonical service identities |
| `time_tolerance_seconds` | Candidate-specific temporal tolerance |

Optional correlation field:

- `trace_ids`: strong join keys when the source actually contains trace context.

An empty `trace_ids` array must not be emitted. Metric candidates omit it because aggregate Prometheus metrics do not contain request trace context.

`service_keys` duplicates information available under `scope.entities` intentionally: it is the normalized, directly indexable join key used by the correlator.

## 10. Cross-signal matching

A metric and log candidate are eligible for correlation when:

```text
same scenario_id
AND time intervals overlap or fall within the configured tolerance
AND service_keys have a non-empty intersection
```

For intervals `A` and `B`, using tolerance `T`:

```text
A.start <= B.end + T
AND B.start <= A.end + T
```

Optional evidence can strengthen a match:

- a shared `trace_id`;
- a known service dependency;
- a domain-specific ordered temporal pattern.

Temporal proximity alone creates a correlation candidate, not proof of causality.

## 11. Metric-specific summary

```json
{
  "affected_metrics": [
    "container_cpu_usage_seconds_total",
    "container_memory_working_set_bytes"
  ],
  "anomaly_point_count": 17,
  "max_abs_score": 578.222786,
  "contributor_count_changed": true
}
```

| Field | Meaning |
|---|---|
| `affected_metrics` | Metrics represented by the candidate |
| `anomaly_point_count` | Detector points summarized by the candidate, not the number of incidents |
| `max_abs_score` | Largest source-local metric detector score |
| `contributor_count_changed` | Whether the number of metric contributors changed during evidence |

`max_abs_score` is meaningful only within the metric detector. It must not be compared numerically with a log detector score.

## 12. Metric-specific evidence

```json
{
  "entity_role": "application",
  "service": {
    "namespace": "train-ticket",
    "name": "ts-auth-service"
  },
  "event_time": {
    "start": "2022-07-13T11:25:30Z",
    "end": "2022-07-13T11:28:00Z"
  },
  "incident_type": "correlated_resource_event",
  "metric_evidence": [
    {
      "metric_name": "container_memory_working_set_bytes",
      "event_time": {
        "start": "2022-07-13T11:25:30Z",
        "end": "2022-07-13T11:28:00Z"
      },
      "feature_signals": [
        {"feature": "value_sum", "direction": "high"},
        {"feature": "value_mean", "direction": "low"}
      ],
      "pattern": "possible_composition_change",
      "anomaly_point_count": 8,
      "max_abs_score": 99.973386,
      "single_contributor": false,
      "contributor_count_changed": true
    }
  ]
}
```

The evidence is self-contained. It does not expose sequential notebook identifiers such as feature episode, metric event, entity incident, or scenario incident IDs.

## 13. Log-specific summary and evidence

The log producer must preserve the shared envelope but can define log-specific details:

```json
{
  "summary": {
    "log_count": 37,
    "warn_count": 25,
    "error_count": 12,
    "affected_template_count": 3,
    "max_detector_score": 8.4
  },
  "evidence": [
    {
      "entity_role": "application",
      "service": {
        "namespace": "train-ticket",
        "name": "ts-auth-service"
      },
      "event_time": {
        "start": "2022-07-13T11:25:10Z",
        "end": "2022-07-13T11:30:40Z"
      },
      "severity": {
        "text": "ERROR",
        "number": 17
      },
      "template_id": "template_0042",
      "occurrence_count": 12,
      "direction": "high",
      "trace_ids": ["optional-trace-id"]
    }
  ]
}
```

The log pipeline should follow these rules:

- normalize severity text and number consistently;
- retain `trace_id` only when present in the source record;
- preserve template identity and occurrence count;
- avoid embedding unrestricted raw log messages in the candidate contract;
- store source-local detector scores in the log summary or evidence only.

## 14. Optional normalized assessment

A future pipeline may add a cross-source-comparable assessment:

```json
{
  "assessment": {
    "severity": "high",
    "confidence": "medium",
    "normalized_score": 0.82,
    "method": "cross_signal_v1"
  }
}
```

This object is optional and must not be populated by directly comparing raw metric and log anomaly scores. It requires an explicit normalization method and version.

## 15. Versioning rules

- Additive optional fields require a minor schema-version increment.
- Removing fields, changing meanings, changing types, or making optional fields required requires a major increment.
- Producers and consumers must reject unsupported major versions.
- A JSON Schema should eventually enforce required fields, formats, enums, and `additionalProperties` behavior.

## 16. Current output locations

```text
data/outputs/
├── metric_incident_candidates.json
├── log_incident_candidates.json        # planned
└── correlated_incidents.json           # planned
```

`metric_incident_candidates.json` and `log_incident_candidates.json` are source-level candidate artifacts. `correlated_incidents.json` will contain the later cross-signal result and must use new `cic_` identifiers rather than reusing either source candidate ID.

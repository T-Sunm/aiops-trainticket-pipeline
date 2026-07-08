# Canonical Metric Series Selection

## Purpose

This document explains how canonical Prometheus time series are selected for the scenario `ts-auth-mongo_4.4.15_2022-07-13`. It covers only the selection stage implemented in [`notebooks/02_metrics_detection.ipynb`](../notebooks/02_metrics_detection.ipynb).

The objective is to transform an ambiguous set of cluster-wide metric series into a well-defined collection representing two runtime entities:

- The `ts-auth-service` application
- Its `ts-auth-mongo` database dependency

This stage does not perform anomaly detection. Its purpose is to establish which observations are valid inputs to later analysis.

## 1. Problem: a metric file is not a service metric

A common first assumption is that a metric file stored under a scenario folder contains only data for the service named by that scenario.

For example:

```text
Scenario folder:
ts-auth-mongo_4.4.15_2022-07-13

Metric file:
ts-auth-service_3_Mongo_4.4.15.json_container_cpu_usage_seconds_total.json
```

The prefix identifies the experiment in which the data was collected. It does not identify the owner of every time series inside the file.

The file is a Prometheus Matrix response for `container_cpu_usage_seconds_total`. Its `data.result` array contains thousands of labeled series collected from the cluster, including:

- `ts-auth-service`
- `ts-auth-mongo`
- Other Train Ticket services
- Kubernetes infrastructure
- Container-level series
- Pod-level cgroup series
- Pod sandbox series

Therefore, neither the scenario name nor the metric file name is sufficient for selecting application telemetry. Ownership must be inferred from the labels of each series.

## 2. Problem: one workload can have several metric views

Prometheus and cAdvisor expose Kubernetes resource usage at several hierarchy levels. The same workload can appear as an application container, a pod-level cgroup, and a pod sandbox.

Consider a simplified memory example:

```text
Application container   100 MiB
POD sandbox               5 MiB
Pod total               105 MiB
```

These values are not three independent resource consumers. The pod total already includes the application container and sandbox usage.

Adding them produces:

```text
100 + 5 + 105 = 210 MiB
```

This is hierarchical double counting. The workload actually uses approximately `105 MiB` at pod level, not `210 MiB`.

The same ambiguity applies to CPU. A broad filter that selects every series whose `pod` contains `ts-auth` may retain both the container measurement and the enclosing pod cgroup measurement.

## 3. Problem: application and database are different entities

The substring `ts-auth` matches two distinct workloads:

```text
ts-auth-service  -> authentication application
ts-auth-mongo    -> MongoDB dependency
```

Although they participate in the same request path, they represent different operational components. Their resource behavior must be attributed separately.

For example, increased MongoDB CPU may indicate expensive queries while application CPU remains stable. If both are summed into a single `ts-auth` signal, the result loses component attribution and may incorrectly suggest that the application itself is responsible for the change.

This leads to two normalized entities:

| Entity | Kubernetes workload | Interpretation |
|---|---|---|
| `application` | `ts-auth-service` | Authentication application resource usage |
| `mongo` | `ts-auth-mongo` | Database dependency resource usage |

## 4. Reading the Prometheus Matrix JSON

Canonical selection is applied to the structure returned by the Prometheus HTTP API. Each metric file in this scenario follows the Matrix result model:

```json
{
  "status": "success",
  "data": {
    "resultType": "matrix",
    "result": [
      {
        "metric": {
          "__name__": "<metric name>",
          "pod": "<pod name>",
          "container": "<container name>",
          "instance": "<scrape target>",
          "...": "other labels"
        },
        "values": [
          [<unix timestamp>, "<sample value>"],
          [<unix timestamp>, "<sample value>"]
        ]
      }
    ]
  }
}
```

The hierarchy has two important levels:

```text
data.result[]         one element per labeled time series
├── metric            label dictionary identifying the series
└── values[]          ordered timestamp-value samples for that series
```

`metric` does not contain a measurement value. It describes what the series represents. `values` contains the observations belonging to that identity. The sample value is encoded as a string by the Prometheus API and must be converted to a numeric type.

### 4.1 Labels used by canonical selection

| JSON path | Purpose in this project |
|---|---|
| `data.result[].metric.__name__` | Identifies CPU, memory, or network metric family |
| `data.result[].metric.pod` | Identifies the Kubernetes pod and workload prefix |
| `data.result[].metric.container` | Distinguishes workload-container, sandbox, and some hierarchy levels |
| `data.result[].metric.image` | Supports verification of the running application or MongoDB image |
| `data.result[].metric.instance` | Preserves the kubelet scrape target for provenance |
| `data.result[].metric.id` | Exposes the underlying cgroup path and hierarchy level |
| `data.result[].values` | Contains the timestamped samples retained after classification |

The Prometheus label `service` is commonly `kubelet` in these container files. It identifies the scrape job or endpoint rather than the Train Ticket application service, so it is not used as application ownership.

### 4.2 Example: canonical application CPU series

The following shortened object is taken from the selected CPU metric file:

```json
{
  "metric": {
    "__name__": "container_cpu_usage_seconds_total",
    "service": "kubelet",
    "pod": "ts-auth-service-96d95d474-bdfpt",
    "container": "ts-auth-service",
    "image": "docker.io/codewisdom/ts-auth-service-with-jaeger:v1",
    "instance": "10.252.1.21:10250",
    "cpu": "total"
  },
  "values": [
    [1657711577.09, "5.737832534"],
    [1657711607.016, "12.950618888"]
  ]
}
```

This series is retained because:

```text
metric name = CPU target metric
container   = ts-auth-service
entity      = application
level       = workload container
```

The two values are raw cumulative CPU-counter samples from the same labeled series. Canonical selection retains them without yet interpreting their change over time.

### 4.3 Example: overlapping pod-level CPU series

The same file contains another series for the same pod:

```json
{
  "metric": {
    "__name__": "container_cpu_usage_seconds_total",
    "service": "kubelet",
    "pod": "ts-auth-service-96d95d474-bdfpt",
    "instance": "10.252.1.21:10250",
    "id": "/kubepods.slice/.../kubepods-burstable-pod9e27ecde_....slice",
    "cpu": "total"
  },
  "values": [
    [1657711577.09, "7.363367252"],
    [1657711607.016, "13.022984171"]
  ]
}
```

The `pod` label still contains `ts-auth-service`, but the workload-container label is absent and the `id` ends at the pod cgroup. This is a pod-level view. It is excluded from canonical CPU selection because retaining it together with the application-container series would introduce hierarchical overlap.

This example shows why filtering only on `pod contains "ts-auth-service"` is insufficient.

### 4.4 Example: canonical application network series

The network file represents the same application pod differently:

```json
{
  "metric": {
    "__name__": "container_network_transmit_packets_total",
    "service": "kubelet",
    "pod": "ts-auth-service-96d95d474-bdfpt",
    "container": "POD",
    "instance": "10.252.1.21:10250",
    "interface": "eth0"
  },
  "values": [
    [1657711592.124, "9"],
    [1657711622.207, "10"]
  ]
}
```

Here, `container="POD"` is retained because the metric measures the shared pod network namespace. The `pod` prefix attributes the series to the application:

```text
metric name = network target metric
container   = POD
pod prefix  = ts-auth-service-
entity      = application
```

The comparison illustrates that the same label value cannot be accepted or rejected independently of metric semantics. `container="POD"` is excluded for CPU and memory but is canonical for network in the observed dataset.

### 4.5 From JSON object to selection decision

For every element in `data.result`, canonical preprocessing makes the following decision:

| Metric labels | Decision | Reason |
|---|---|---|
| CPU/memory + `container=ts-auth-service` | Retain as `application` | Correct workload-container level |
| CPU/memory + `container=ts-auth-mongo` | Retain as `mongo` | Correct workload-container level |
| CPU/memory + missing container or `container=POD` | Exclude | Overlapping or non-workload level |
| Network + `container=POD` + application pod prefix | Retain as `application` | Shared application pod network namespace |
| Network + `container=POD` + MongoDB pod prefix | Retain as `mongo` | Shared MongoDB pod network namespace |
| Unrelated workload labels | Exclude | Outside the entities in scope |

Only after this series-level decision is accepted are the entries in `values` expanded into rows in `raw_metrics`.

## 5. Motivation for canonical selection

An analytical metric must have an explicit observation unit. In this scenario, a usable series must answer both questions:

1. **Which entity does this measurement belong to?**
2. **At which Kubernetes hierarchy level is it measured?**

Without these definitions, a numeric value may be technically valid but semantically ambiguous.

Canonical selection is introduced to provide:

- **Entity attribution:** application and database remain distinguishable.
- **Non-overlapping measurement:** only one resource view is selected for a metric.
- **Reproducibility:** selection is expressed as deterministic label rules.
- **Auditability:** every retained sample preserves its source labels.
- **Construct validity:** the resulting series measures the component and resource concept it claims to represent.

This preprocessing step is consequently part of the measurement design, not merely a performance optimization or data-cleaning convenience.

## 6. Definition of a canonical series

In this project, a **canonical metric series** is the intentionally selected Prometheus series used as the authoritative representation of one metric for one normalized entity.

Canonical does not mean that the source series is globally unique or universally correct. It means that, for the analytical question and dataset schema currently in scope, the series has been selected according to an explicit and non-overlapping rule.

A canonical selection rule specifies:

```text
metric family
+ entity identity
+ Kubernetes measurement level
= retained series
```

For example:

```text
CPU
+ application
+ container-level measurement
= container == "ts-auth-service"
```

A series that does not satisfy a canonical rule is excluded from this analysis, even if its name contains `ts-auth`.

## 7. Why the selection rule depends on the metric

CPU, memory, and network are not represented identically by cAdvisor.

### 7.1 CPU and memory

The analysis asks how much CPU or memory the actual workload container uses. The canonical representation is therefore the container-level series:

```text
container = "ts-auth-service"
```

or:

```text
container = "ts-auth-mongo"
```

Series with `container="POD"` or a missing container label are not used for CPU and memory because they represent a different hierarchy level and may overlap with the selected workload-container value.

### 7.2 Network

Containers inside a Kubernetes pod share the pod network namespace. In the observed dataset, network transmit metrics are associated with:

```text
container = "POD"
```

The label `container="POD"` identifies the correct measurement level but does not identify whether the pod belongs to the application or MongoDB. The `pod` label supplies that ownership information:

```text
pod starts with "ts-auth-service-" -> application
pod starts with "ts-auth-mongo-"   -> mongo
```

The meaning of `container="POD"` is therefore metric-dependent. It is excluded for CPU and memory but retained for network in this dataset.

## 8. Canonical selection policy

The complete policy is:

| Metric | Canonical application rule | Canonical MongoDB rule | Excluded overlapping views |
|---|---|---|---|
| `container_cpu_usage_seconds_total` | `container == "ts-auth-service"` | `container == "ts-auth-mongo"` | `container="POD"`, missing container, pod cgroup |
| `container_memory_working_set_bytes` | `container == "ts-auth-service"` | `container == "ts-auth-mongo"` | `container="POD"`, missing container, pod cgroup |
| `container_network_transmit_packets_total` | `container == "POD"` and application pod prefix | `container == "POD"` and MongoDB pod prefix | Other pods and network interfaces outside the target entities |

This policy creates six logical groups:

```text
application + CPU
application + memory
application + network
mongo       + CPU
mongo       + memory
mongo       + network
```

The groups remain separate. Canonical selection does not combine application and database measurements into a single total.

## 9. Implementation

The notebook implements the policy in `classify_series(labels)`:

```python
def classify_series(labels):
    metric_name = labels.get("__name__")
    pod = str(labels.get("pod") or "")
    container = labels.get("container")

    if metric_name not in TARGET_METRICS:
        return None

    if metric_name in {CPU_METRIC, MEMORY_METRIC}:
        if container == "ts-auth-service":
            return "application"

        if container == "ts-auth-mongo":
            return "mongo"

        return None

    if metric_name == NETWORK_METRIC and container == "POD":
        if pod.startswith("ts-auth-service-"):
            return "application"

        if pod.startswith("ts-auth-mongo-"):
            return "mongo"

    return None
```

The return value defines the selection decision:

| Return value | Meaning |
|---|---|
| `application` | Retain the series and attribute it to `ts-auth-service` |
| `mongo` | Retain the series and attribute it to `ts-auth-mongo` |
| `None` | Exclude the series from the canonical dataset |

The function operates on series labels before individual timestamp-value pairs are expanded. This avoids loading unrelated cluster series into the canonical dataset.

## 10. Canonical data representation

Each retained timestamp-value pair becomes one row in `raw_metrics`:

| Field | Meaning |
|---|---|
| `timestamp` | Prometheus sample timestamp converted from Unix time to UTC |
| `value` | Raw Prometheus sample value |
| `series_id` | Identifier assigned to one source time series |
| `entity` | `application` or `mongo` |
| `metric_name` | Prometheus metric name |
| `pod` | Source Kubernetes pod |
| `container` | Source container or pod network namespace label |
| `image` | Source container image when available |
| `instance` | Prometheus scrape target |

The result uses long format. The six logical groups occupy rows distinguished by `entity` and `metric_name`; they are not merged into six measurement columns.

Two intermediate tables are retained:

- `series_inventory`: one row per canonical source series, used to inspect identity, labels, and sample coverage.
- `raw_metrics`: one row per retained timestamped sample, used as the canonical input dataset.

`series_id` preserves source-series boundaries. This is necessary because two pods that expose the same metric are still different time series.

## 11. Observed result for the selected scenario

Canonical selection reduces the cluster-wide metric responses to 15 source series and 751 samples:

| Entity | Metric | Canonical series | Samples |
|---|---|---:|---:|
| `application` | CPU | 2 | 130 |
| `application` | Memory | 2 | 132 |
| `application` | Network | 2 | 130 |
| `mongo` | CPU | 3 | 119 |
| `mongo` | Memory | 3 | 121 |
| `mongo` | Network | 3 | 119 |
| **Total** |  | **15** | **751** |

The presence of multiple series for an entity reflects multiple pod identities during the observation period. It does not imply that all of those pods were active concurrently for the entire hour.

This result should be interpreted as six canonical entity-metric groups backed by 15 lifecycle-specific source series.

## 12. Validation invariants

The selection is checked through explicit invariants:

1. CPU and memory series must have a non-empty workload-container label.
2. CPU and memory series must not use `container="POD"`.
3. Network series must use `container="POD"` under the observed dataset schema.
4. A `series_id` must belong to only one normalized entity.
5. Application and MongoDB series must remain distinguishable.

These checks are important because a rule can execute successfully while still selecting the wrong semantic level. Assertions make schema assumptions visible and cause the notebook to fail early when those assumptions are violated.

## 13. Scope and limitations

The canonical policy is derived from the labels observed in `ts-auth-mongo_4.4.15_2022-07-13`. It is appropriate for the current proof of concept but should not be treated as a universal Prometheus rule.

Potential variations include:

- Different exporters or cAdvisor versions
- Different container and pod naming conventions
- Additional sidecar containers
- Network metrics attached to another label combination
- Workloads whose names share ambiguous prefixes

For another dataset family, the label schema and hierarchy must be profiled before reusing these rules. A production implementation should express entity mappings and canonical policies through configuration rather than hard-coded service names.

## 14. Conclusion

Raw Prometheus data does not provide an analysis-ready definition of “application CPU,” “database memory,” or “service network.” That definition must be constructed from metric semantics, Kubernetes hierarchy, and workload identity.

Canonical selection provides this definition by:

- Separating `ts-auth-service` from `ts-auth-mongo`
- Selecting a single non-overlapping measurement level for each metric
- Preserving source labels and series identity
- Excluding unrelated cluster telemetry

The output is not an anomaly result. It is a semantically controlled measurement dataset on which later analysis can be based.

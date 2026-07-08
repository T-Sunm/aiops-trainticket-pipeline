# Metrics Detection Report: Signal Preparation and Observed Behavior

## Purpose and scope

This report records the metric-processing results observed in [`notebooks/02_metrics_detection.ipynb`](../notebooks/02_metrics_detection.ipynb) for scenario `ts-auth-mongo_4.4.15_2022-07-13`. It covers notebook stages 1–7: scenario setup, canonical-series loading, signal transformation, resampling, entity-level aggregation, timeline inspection, and detection-feature preparation.

Canonical selection is summarized here only where it affects the analysis. Its motivation, label semantics, and selection policy are documented separately in [Canonical Metric Series Selection](02_canonical_metric_series.md). Validation assertions are intentionally outside the scope of this report.

At this stage the notebook has not yet applied an anomaly detector. Terms such as *spike*, *drop*, and *transition* describe visible behavior relative to the surrounding observations; they are not detector-confirmed anomalies.

## 1. Scenario and analysis configuration

The notebook analyzes telemetry under:

```text
data/raw/ts-auth-mongo_4.4.15_2022-07-13
```

The analysis separates two operational entities:

- `application`: the `ts-auth-service` workload;
- `mongo`: the MongoDB workload used by the service.

For each entity, three Prometheus metrics are retained:

| Metric | Raw type | Analysis unit |
|---|---|---|
| `container_cpu_usage_seconds_total` | Counter | CPU cores |
| `container_memory_working_set_bytes` | Gauge | MiB |
| `container_network_transmit_packets_total` | Counter | Packets/second |

The common resampling interval is 30 seconds. Counter observations separated by more than 90 seconds are treated as an invalid rate interval.

## 2. Canonical metric input

Prometheus exports several labeled views of the same Kubernetes workload. The notebook therefore selects one intended representation for each entity and metric before expanding the samples:

- CPU and memory use the workload-container series;
- network uses the pod network-namespace series (`container="POD"`);
- application and MongoDB remain separate entities.

The full rationale is in [Canonical Metric Series Selection](02_canonical_metric_series.md).

The selected input contains **751 timestamped samples** across **15 lifecycle-specific source series**:

| Entity | Metric | Source series | Distinct pods | Samples |
|---|---|---:|---:|---:|
| application | CPU | 2 | 2 | 130 |
| application | Memory | 2 | 2 | 132 |
| application | Network transmit | 2 | 2 | 130 |
| mongo | CPU | 3 | 3 | 119 |
| mongo | Memory | 3 | 3 | 121 |
| mongo | Network transmit | 3 | 3 | 119 |

The inventory includes multiple pod identities over the observation period. This does not mean all listed pods were active simultaneously; later time-bucket counts describe which series were actually observed at each timestamp.

## 3. Transformation into analysis signals

Raw Prometheus values are not directly comparable across metric types. CPU and network are cumulative counters, while memory is an instantaneous gauge.

For each `series_id`, samples are ordered by timestamp and transformed independently. Keeping series boundaries prevents a delta from being calculated between two different pods or metric lifecycles.

For CPU and network:

```text
elapsed_seconds = current_timestamp - previous_timestamp
counter_delta   = current_value - previous_value
signal_value    = counter_delta / elapsed_seconds
```

This produces CPU cores for CPU usage and packets per second for network transmit activity. A rate is discarded when the counter decreases, the time interval is non-positive, the interval exceeds 90 seconds, or there is no previous observation.

For memory:

```text
signal_value = value_bytes / 1024²
```

This converts the gauge directly to MiB without differencing.

### Observed transformation quality

| Entity | Metric | Rows | Valid signals | Counter resets | Invalid gaps |
|---|---|---:|---:|---:|---:|
| application | CPU | 130 | 128 | 0 | 0 |
| application | Memory | 132 | 132 | 0 | 0 |
| application | Network transmit | 130 | 128 | 0 | 0 |
| mongo | CPU | 119 | 116 | 0 | 0 |
| mongo | Memory | 121 | 121 | 0 | 0 |
| mongo | Network transmit | 119 | 116 | 0 | 0 |

No negative counter delta or excessive timestamp gap was observed. CPU and network have one unavailable rate at the start of each source series because a first counter sample has no preceding value. This explains the differences of two samples for application and three for MongoDB. Memory remains valid from its first sample because it is a gauge.

## 4. Resampling and entity-level aggregation

Each transformed series is resampled independently into 30-second buckets. If a bucket contains multiple valid observations, their mean becomes the bucket value. Empty buckets are removed; the notebook does not interpolate missing values.

The resampled series are then grouped by:

```text
timestamp × entity × metric_name × unit
```

This changes the analytical level from individual source series to an entity-and-metric snapshot. The resulting features are:

| Feature | Interpretation |
|---|---|
| `value_sum` | Total observed resource use or activity across series |
| `value_mean` | Mean value per observed series |
| `value_median` | Typical series value, less sensitive to one high series |
| `value_max` | Highest single-series value |
| `active_pod_count` | Distinct pod labels contributing a valid value |
| `observed_series_count` | Distinct canonical series contributing a valid value |

The chart labels `value_mean` as “Mean per pod.” This interpretation is exact only when one canonical series represents each pod for a given metric. More generally, it is the mean per observed series. Likewise:

```text
value_sum = value_mean × observed_series_count
```

Consequently, a separation between sum and mean reveals a change in the number of contributing series, while a change in mean reflects a change in the workload carried by the average contributing series.

## 5. Observed normalized metric timelines

![Normalized application and MongoDB metric timelines](../assets/02_normalized_metric_timelines.png)

The figure contains two entities by three metrics. The blue line is total activity (`value_sum`); the orange line is mean activity per observed series (`value_mean`).

### 5.1 Application CPU

Application CPU stays near approximately 0.00–0.01 cores for most of the early period, then rises sharply around 11:26–11:31 UTC. During the elevated interval:

- total CPU reaches approximately 0.20–0.24 cores;
- mean CPU reaches approximately 0.09–0.12 cores;
- total is roughly twice the mean.

The simultaneous increase of both sum and mean shows that the elevation is not explained only by an additional contributing series: CPU use per observed series also increased substantially. The approximate 2:1 ratio indicates two contributing series during much of this interval.

After the interval, both CPU measures return close to their earlier level, apart from shorter isolated peaks.

### 5.2 Application memory

Application memory initially remains near 340 MiB, with sum and mean overlapping. Around 11:25 UTC, the two measures move in opposite directions:

- total memory rises toward approximately 580 MiB;
- mean memory initially falls to approximately 170 MiB and then increases toward 290 MiB;
- total becomes approximately twice the mean.

This is mathematically consistent with a composition change from one contributing series to two. A newly observed series with a lower initial working set can increase the total while reducing the mean. Later, the total falls to approximately 250–270 MiB and overlaps the mean again, consistent with a return to one contributing series.

The plot supports a lifecycle or topology transition, but does not identify its cause. Rollout, restart, rescheduling, or scaling are plausible hypotheses that require pod metadata or event/log evidence.

### 5.3 Application network transmit

Application network transmit is more variable than its CPU and memory baselines. Much of the series fluctuates around approximately 0.3–0.6 packets/second, with several isolated peaks. Activity becomes more volatile around and after 11:25 UTC, including peaks near 1.7 packets/second.

Sum and mean overlap for much of the timeline and separate during parts of the transition interval. This indicates that the number of contributing network series changes over time. The larger peaks describe increased transmit activity, but the plot alone cannot distinguish workload traffic from lifecycle-related communication.

### 5.4 MongoDB CPU

MongoDB CPU fluctuates primarily around approximately 0.003–0.010 cores, with a maximum visible peak near 0.015 cores around 11:30 UTC. Its sum and mean almost completely overlap, indicating one contributing canonical series at most timestamps.

MongoDB CPU does not show an elevation comparable in magnitude or duration to the application CPU interval. The strongest CPU change in this figure is therefore localized to the application side, although this observation does not establish the source of the workload.

### 5.5 MongoDB memory

MongoDB memory is initially stable near 112–114 MiB. Around 11:25 UTC it contains a one-bucket drop to approximately zero, followed by recovery to a lower range near 58–70 MiB and a gradual stepwise increase.

Because empty resampling buckets are removed with `dropna()`, the near-zero point is not simply an empty bucket converted to zero; at least one retained memory observation in that bucket was near zero. The abrupt drop and lower post-transition baseline are consistent with a MongoDB series or process lifecycle change, but restart and deployment evidence is needed before assigning a cause.

The sum and mean remain nearly coincident, so the visible level change is not explained by simultaneous aggregation of several MongoDB series.

### 5.6 MongoDB network transmit

MongoDB network transmit is generally near 0.07–0.13 packets/second, with short peaks around 11:29–11:35 UTC. The largest reaches approximately 0.6 packets/second. Sum and mean overlap almost entirely, again indicating one contributing series at most timestamps.

The MongoDB network peak occurs near the application CPU and network elevation. This temporal alignment is useful correlation evidence for later analysis, but it does not by itself prove that application activity caused the MongoDB traffic.

### 5.7 Cross-metric interpretation

The most prominent shared transition is concentrated around 11:25–11:31 UTC:

- application CPU rises sharply both in total and per-series terms;
- application memory changes composition and then settles at a lower level;
- application network activity becomes more volatile;
- MongoDB memory changes baseline;
- MongoDB network produces short peaks while MongoDB CPU increases only modestly.

This combination is compatible with a workload and deployment/lifecycle transition occurring in the same period. It is not yet sufficient to separate a fault from an intentional rollout or scaling event. That distinction belongs to later detection and cross-signal correlation stages.

## 6. Features retained for anomaly scoring

Although the entity-level table computes sum, mean, median, and maximum, the notebook selects three numerical views for independent scoring:

```text
value_sum
value_mean
value_max
```

They preserve complementary behavior:

- `value_sum` detects a change in total entity demand;
- `value_mean` detects a change in average demand per contributing series;
- `value_max` preserves a high single-series event that aggregation could dilute.

`active_pod_count` and `observed_series_count` remain attached as context columns. They are not melted into scored values at this stage, but are essential for explaining whether a sum change coincides with scaling, lifecycle turnover, or missing observations.

`value_median` is retained in the aggregate table but is not selected by `FEATURES_TO_SCORE`. It remains available for robust descriptive comparisons or a later detector design.

## 7. Detection-ready representation and findings

The aggregate table is converted from wide to long form. Each row represents one independently scoreable stream:

```text
timestamp
entity
metric_name
feature_name
observed_value
unit
active_pod_count
observed_series_count
```

With two entities, three metrics, and three selected feature types, the downstream detector receives 18 logical streams:

```text
2 entities × 3 metrics × 3 features = 18 streams
```

The preparation stages establish the following evidence for subsequent anomaly detection:

1. Canonical selection prevents overlapping Kubernetes resource views from being added together.
2. Counter rates are calculated within source-series boundaries and no reset or excessive gap was observed.
3. All metrics share 30-second analysis buckets without synthetic interpolation.
4. Entity-level totals and per-series statistics preserve both system-wide and localized behavior.
5. A clear multi-metric transition appears around 11:25–11:31 UTC, led by application CPU and accompanied by application composition changes and MongoDB memory/network changes.
6. Pod and series counts must remain part of the evidence used to interpret any alert raised during that transition.

The next notebook stage applies a leakage-aware rolling median and median absolute deviation detector. Detector thresholds, anomaly labels, and alert evaluation are intentionally not reported here because they occur after section 7.

# Log Preparation Report: Template Grouping and Time-Series Features

## Purpose and scope

This report records the log-processing results observed in [`notebooks/03_logs_detection.ipynb`](../notebooks/03_logs_detection.ipynb) for scenario `ts-auth-mongo_4.4.15_2022-07-13`. It covers the stages completed before log anomaly detection: loading the structured log artifacts, validating their relationship, normalizing timestamps, enriching the template catalog, preparing template text, grouping templates with TF-IDF and K-Means, mapping those groups back to individual log records, and aggregating log activity into 30-second time-series features.

Despite the notebook name, the current implementation does **not** score anomalies, create log episodes, or export log incident candidates. The terms *cluster* and *analysis group* in this report describe representation and aggregation only; they do not identify incidents or root causes.

## 1. Input data and configuration

The notebook reads two pre-parsed log artifacts under:

```text
data/raw/ts-auth-mongo_4.4.15_2022-07-13
```

| Input | Analytical role | Rows | Columns |
|---|---|---:|---:|
| `LOGS_ts-auth-service_3_Mongo_4.4.15.txt_structured.csv` | One row per observed log occurrence | 147,205 | 10 |
| `LOGS_ts-auth-service_3_Mongo_4.4.15.txt_templates.csv` | One row per structural log template | 1,182 | 3 |

The structured file already contains the output of an upstream log parser:

```text
LineId
Date
Time
Level
LoggingReporter
Content
EventId
EventTemplate
ParameterList
```

Consequently, the notebook does not run Drain or another template miner again. `EventId` is used as the stable join key between individual log occurrences and the template catalog.

The current analysis configuration is:

| Setting | Value |
|---|---|
| Scenario | `ts-auth-mongo_4.4.15_2022-07-13` |
| Assumed source timezone | `Asia/Shanghai` |
| Normalized timezone | UTC |
| Time bucket | 30 seconds |
| Default namespace | `train-ticket` |
| Default service | `ts-auth-service` |
| Default service key | `train-ticket/ts-auth-service` |

The timezone and service identity are current notebook assumptions rather than conclusions derived from the file content. Their implications are discussed in [Current limitations](#10-current-limitations).

## 2. Input validation

The notebook verifies the required schemas and compares the `EventId` sets in both input files.

| Validation | Result |
|---|---:|
| Structured log rows | 147,205 |
| Template catalog rows | 1,182 |
| Unique structured-log `EventId` values | 1,182 |
| Unique catalog `EventId` values | 1,182 |
| Duplicate `LineId` values | 0 |
| Duplicate catalog `EventId` values | 0 |
| Structured rows without `EventId` | 0 |
| Structured `EventId` values missing from catalog | 0 |
| Catalog `EventId` values unused by structured logs | 0 |

The two files therefore form a complete many-to-one relationship:

```text
many structured log occurrences
                |
                | EventId
                v
one template-catalog record per EventId
```

This relationship allows template-level semantic grouping to be computed once for 1,182 templates and then inherited by all 147,205 log occurrences without re-vectorizing repeated messages.

### Severity distribution

| Level | Log count | Share of all logs |
|---|---:|---:|
| INFO | 146,599 | 99.588% |
| ERROR | 458 | 0.311% |
| WARN | 148 | 0.101% |

The dataset is strongly dominated by INFO records. Severity therefore remains attached as explicit evidence rather than being inferred from semantic cluster membership.

## 3. Timestamp normalization

The notebook combines the `Date` and `Time` columns, parses them as local timestamps, localizes them to `Asia/Shanghai`, converts them to UTC, and sorts the resulting records chronologically.

All 147,205 timestamps are parsed successfully. The normalized observation range is:

```text
first: 2022-06-14 09:26:35.024 UTC
last:  2022-07-13 11:51:19.068 UTC
```

Sorting is necessary because input row order is not chronological: the original file begins with later July records and ends with earlier June records. `LineId` remains available for tracing any normalized record back to its source row.

The resulting `logs` table also adds:

```text
is_info
is_warn
is_error
service_namespace
service_name
service_key
```

## 4. Enriched template catalog

The three-column source catalog is enriched with statistics calculated from the structured log occurrences. For each `EventId`, the notebook records:

| Field | Meaning |
|---|---|
| `observed_occurrences` | Number of structured log rows using the template |
| `first_seen` / `last_seen` | First and last normalized UTC observations |
| `info_count` / `warn_count` / `error_count` | Severity distribution |
| `unique_reporter_count` | Number of distinct logging reporters |
| `representative_content` | One original message retained for inspection |
| `error_ratio` / `warn_ratio` | Severity count divided by observed occurrences |

The source `Occurrences` field matches the count reconstructed from the structured logs for all 1,182 templates:

| Catalog-quality check | Result |
|---|---:|
| Templates without observations | 0 |
| Occurrence-count mismatches | 0 |
| Templates with at least one ERROR | 39 |
| Templates with at least one WARN | 19 |

The most frequent template is `Span reported: <*> - <*>`, with 25,695 occurrences. Several other high-volume templates describe Spring request mappings, bean initialization, filter mappings, and other framework or lifecycle activity. This distribution is important because semantic similarity and occurrence frequency describe different properties: K-Means groups template text, while the catalog retains how often each template actually occurred.

## 5. Template-text preparation

Semantic grouping operates on `EventTemplate`, not on the 147,205 raw `Content` strings. This prevents frequently repeated messages from being vectorized thousands of times and keeps the unit of clustering equal to one structural template.

The cleaning function:

1. separates camelCase and PascalCase terms;
2. converts text to lowercase;
3. removes the parser wildcard `<*>`;
4. retains letters, digits, underscores, hyphens, and whitespace;
5. removes tokens composed only of digits;
6. normalizes repeated whitespace.

Operational vocabulary such as `failed`, `timeout`, `connection`, `mongo`, `database`, `exception`, `request`, `authentication`, and `retry` is intentionally retained.

### Non-clusterable template handling

Of the 1,182 templates, 1,181 retain text after cleaning and are eligible for TF-IDF and K-Means. One template contains only wildcards:

| EventId | EventTemplate | Occurrences | ERROR | WARN |
|---|---|---:|---:|---:|
| `dede275d` | `<*> <*>` | 345 | 0 | 0 |

After wildcard removal, this template has no semantic text. The notebook therefore does not fabricate a semantic cluster for it and does not discard its 345 log occurrences. Instead, it records:

```text
is_clusterable      = False
semantic_cluster_id = null
analysis_group      = event_dede275d
```

This preserves the original `EventId` as an independently observable stream for later analysis.

## 6. TF-IDF representation

TF-IDF is fitted on the 1,181 cleaned, clusterable templates with:

```text
ngram range       = (1, 2)
minimum document frequency = 2
maximum document frequency = 0.95
sublinear term frequency    = enabled
normalization               = L2
```

The resulting sparse matrix has shape:

```text
1,181 templates x 1,561 TF-IDF features
```

Each row describes the unigram and bigram vocabulary of one template. Occurrence counts and severity counts are not included in this vector, so K-Means groups textual structure rather than allowing high-volume or high-severity templates to dominate the semantic representation.

## 7. K-Means selection and cluster profiles

The notebook evaluates eight candidate values of K with cosine silhouette score as a comparative signal:

| K | Cosine silhouette | Smallest cluster | Largest cluster | Singleton clusters |
|---:|---:|---:|---:|---:|
| 8 | 0.376629 | 59 | 297 | 0 |
| 10 | 0.389686 | 18 | 297 | 0 |
| 12 | 0.394685 | 18 | 297 | 0 |
| 15 | 0.401682 | 17 | 297 | 0 |
| 20 | 0.411373 | 17 | 297 | 0 |
| 25 | 0.432489 | 7 | 297 | 0 |
| 30 | 0.440939 | 5 | 297 | 0 |
| 40 | **0.455844** | 4 | 297 | 0 |

The current notebook selects `K = 40` because it has the highest silhouette score among the tested candidates. This is a data-driven working choice, not proof that 40 is the operationally correct taxonomy. Silhouette is explicitly treated as a comparative diagnostic rather than the only validation criterion.

For each semantic cluster, `cluster_profile_df` retains:

```text
semantic_cluster_id
template_count
total_occurrences
error_count
warn_count
top_terms
representative_templates
```

`centroid_similarity` is also attached to each clusterable template. It measures how similar the TF-IDF vector is to its assigned cluster centroid and helps identify representative or weakly matched templates. Cluster IDs remain numerical; the current notebook does not assign manual semantic labels.

## 8. Unified analysis groups

The pipeline needs one aggregation key that works for both semantic clusters and templates that cannot be clustered. It therefore creates `analysis_group` according to:

```text
template has semantic_cluster_id
    -> semantic_cluster_<id>

template has no semantic_cluster_id
    -> event_<EventId>
```

Examples:

| EventId | Clusterable | Semantic cluster | Analysis group |
|---|---:|---:|---|
| A normal clustered template | Yes | 4 | `semantic_cluster_4` |
| `dede275d` | No | null | `event_dede275d` |

This produces 41 logical analysis groups in the current run: 40 semantic clusters and one independently retained EventId group. The design avoids two problematic alternatives:

- it does not place every unclusterable template into a shared synthetic cluster such as `-1`;
- it does not remove their log occurrences from later time-series analysis.

## 9. Mapping and time-series aggregation

The template-level mapping contains:

```text
EventId
is_clusterable
semantic_cluster_id
analysis_group
```

It is merged back into the structured logs with a validated many-to-one join on `EventId`. The merge preserves all 147,205 log occurrences, and every occurrence receives a non-null `analysis_group`.

The resulting analytical tables are:

| DataFrame | Shape | Unit of one row |
|---|---:|---|
| `template_catalog_df` | 1,182 x 19 | One `EventId` / structural template |
| `clustered_logs_df` | 147,205 x 23 | One observed log occurrence enriched with its analysis group |
| `cluster_timeseries_df` | 11,039 x 13 | One 30-second bucket x service x analysis group |

Each enriched log occurrence is assigned to a 30-second UTC bucket. Records are then grouped by:

```text
time_window
x service_namespace
x service_name
x service_key
x analysis_group
```

The aggregate retains:

| Feature | Interpretation |
|---|---|
| `log_count` | Total log occurrences in the group and bucket |
| `info_count` | INFO occurrences |
| `warn_count` | WARN occurrences |
| `error_count` | ERROR occurrences |
| `unique_template_count` | Distinct `EventId` values represented |
| `unique_reporter_count` | Distinct logging reporters represented |
| `error_ratio` | ERROR share of the bucket's group activity |
| `warn_ratio` | WARN share of the bucket's group activity |

This aggregation is the bridge between discrete log messages and later time-series analysis. It reduces 147,205 occurrence-level records to 11,039 analysis-group observations while retaining severity, template diversity, and reporter diversity. Because the bucket interval matches the 30-second metric resolution, future cross-signal work can align log activity and metric behavior on a shared UTC timeline.

No empty buckets are synthesized in the current representation. A missing `time_window x analysis_group` row means no log from that group was observed in that bucket; it has not yet been converted to an explicit zero.

## 10. Current limitations

The current output is a preparation-stage proof of concept and has several important limits:

1. **Timezone is assumed.** The source timestamps are treated as `Asia/Shanghai` and shifted to UTC, but the notebook has not validated this offset against deployment events or another telemetry source. An incorrect offset would prevent reliable correlation with metrics.
2. **Service identity is assigned globally.** Every row is currently labeled `train-ticket/ts-auth-service` based on the filename. However, the template vocabulary includes order, route, station, travel, consign, price, contacts, security, and other Train Ticket domains. The file scope or service-classification policy should be verified before treating `service_key` as canonical evidence.
3. **The observation span is broader than the named scenario date.** Normalized records run from June 14 to July 13, 2022. Later analysis must decide whether to use the entire history as baseline data or filter to a scenario-specific interval.
4. **K is selected statistically, not operationally.** `K = 40` has the best silhouette score among the tested values, but the clusters have not yet been reviewed for cohesion, duplication, or usefulness to operators.
5. **Template cleaning is intentionally light.** Some IDs, dates, UUID fragments, package names, or domain-specific values can remain in cleaned text and influence TF-IDF similarity.
6. **Cluster IDs have no intrinsic meaning.** `semantic_cluster_4` is an aggregation key, not a stable operational category. IDs can change if preprocessing, input data, K, or the random seed changes.
7. **No anomaly semantics exist yet.** High log volume, ERROR presence, a rare EventId, or a high ratio is descriptive evidence only at this stage.
8. **Missing group buckets are implicit.** A later detector must define whether absent observations become zero, remain missing, or are handled differently for rarity and novelty signals.

## 11. Detection-ready boundary and next step

The current notebook completes the following preparation flow:

```text
structured log occurrences + template catalog
-> input validation
-> UTC normalization
-> enriched template catalog
-> cleaned template text
-> TF-IDF vectors
-> 40 semantic clusters
-> unified analysis_group mapping
-> enriched occurrence-level logs
-> 30-second log time-series features
```

The result is ready for a separate anomaly-design stage, but that stage should begin only after validating timezone, service scope, cluster quality, and missing-bucket semantics. Future detection can then evaluate complementary evidence such as volume changes, ERROR/WARN bursts, rare or newly observed EventIds, and changes in template composition without altering the preparation contract documented here.

structured logs

    EventId

      │

      │ mapping theo EventId

      ▼

template catalog

    EventId + EventTemplate

      │

      │ TF-IDF + K-Means

      ▼

semantic_cluster_id

    analysis_group

      │

      │ mapping ngược theo EventId

      ▼

clustered_logs_df

    mỗi dòng vẫn là một log occurrence

    timestamp chính xác

    EventId

    analysis_group

      │

      │ floor timestamp về 30 giây

      ▼

time_window

    09:00:03 → 09:00:00

    09:00:18 → 09:00:00

    09:00:45 → 09:00:30

      │

      │ groupby

      ▼

cluster_timeseries_df

    time_window

    service_key

    analysis_group

    log_count

    info_count

    warn_count

    error_count

    ...

      │

      │ xác định active period

      │ zero-fill bucket thiếu

      ▼

detection_input_df

    timeline đều mỗi 30 giây

    09:00:00 → count 3

    09:00:30 → count 1

    09:01:00 → count 0

    09:01:30 → count 4

      │

      │ Rolling Median + MAD

      │ ERROR/WARN novelty

      │ ERROR/WARN burst

      ▼

anomaly_candidates_df

    các bucket × analysis_group bất thường

      │

      │ mapping ngược bằng

      │ time_window + service_key + analysis_group

      ▼

anomaly_logs_df

    LineId

    EventId

    EventTemplate

    Content
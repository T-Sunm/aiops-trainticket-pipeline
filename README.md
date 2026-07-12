# AIOps Train Ticket Pipeline

Pipeline chuyển metric và log của hệ thống Train Ticket thành các incident candidate có thể dùng cho bước correlation.

Dataset: [Anomalies in Microservice Train Ticket](https://zenodo.org/records/6979726).

## Workflow

```mermaid
flowchart LR
    A[Raw telemetry] --> B[Metric pipeline]
    A --> C[Log preprocessing]
    C --> D[30-second time series]
    D --> E[Anomaly detection]
    E --> F[Episode aggregation]
    B --> G[Metric candidates]
    F --> H[Log candidates]
    G --> I[Correlation / incident analysis]
    H --> I
```

### Log pipeline

```mermaid
flowchart LR
    A[147,205 logs] --> B[1,182 templates]
    B --> C[41 analysis groups]
    C --> D[30s buckets + zero-fill]
    D --> E[Median/MAD + novelty rules]
    E --> F[51 anomaly buckets]
    F --> G[51 incident candidates]
```

![Active periods và timeline 30 giây](assets/03_log_active_periods_zero_fill.png)

![Các tín hiệu anomaly và candidates](assets/03_log_anomaly_candidates.png)

## Tài liệu

- [Dataset exploration](docs/01_dataset_exploration_report.md)
- [Metric detection](docs/02_metrics_detection_report.md)
- [Log preprocessing, anomaly detection và episode aggregation](docs/03_logs_preparation_report.md)

## Chuẩn bị dữ liệu

```powershell
Invoke-WebRequest -Uri "https://zenodo.org/records/6979726/files/anomalies_microservice_trainticket_version_configurations.zip?download=1" -OutFile "dataset.zip"
Expand-Archive -Path "dataset.zip" -DestinationPath "data/raw"
```

Chạy lần lượt các notebook `03_1` → `03_2` → `03_3`. Đầu ra cuối của log pipeline là `data/outputs/log_incident_candidates.json`.

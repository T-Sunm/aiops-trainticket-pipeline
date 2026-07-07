# AIOps Train Ticket Pipeline

Dataset: [Anomalies in Microservice Train Ticket](https://zenodo.org/records/6979726).

Download and extract the dataset into `data/raw`:

```powershell
Invoke-WebRequest -Uri "https://zenodo.org/records/6979726/files/anomalies_microservice_trainticket_version_configurations.zip?download=1" -OutFile "dataset.zip"
Expand-Archive -Path "dataset.zip" -DestinationPath "data/raw"
```

## Documentation

- [Dataset exploration report](docs/01_dataset_exploration_report.md)
- [Canonical metric series selection](docs/02_metrics_detection_report.md)

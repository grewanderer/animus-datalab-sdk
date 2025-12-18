<p align="center">
  <img src="assets/banner.png" width="100%" alt="Animus Memory Core">
</p>

<h1 align="center">Animus DataPilot</h1>
<p align="center">
  <em>Deterministic control plane for enterprise ML datasets, experiments, and lineage</em>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/status-production--ready-0f172a?style=flat-square">
  <img src="https://img.shields.io/badge/deployment-on--prem--first-0f172a?style=flat-square">
  <img src="https://img.shields.io/badge/network-air--gapped--compatible-0f172a?style=flat-square">
  <img src="https://img.shields.io/badge/python-3.9+-0f172a?style=flat-square">
</p>

---

## Overview

This directory contains the **Python SDK** used by **CI systems and ML pipelines** to publish metadata to **Animus DataPilot**.

The SDK is intentionally minimal and deterministic, designed for:

* CI-driven experiment registration
* immutable experiment runs
* dataset version binding
* signed CI image attestation (git + image digest)
* live telemetry from training containers
* artifact registration from training and evaluation jobs

It is suitable for **on‑prem**, **air‑gapped**, and **non‑interactive** environments.

---

## Install

```bash
pip install animus-datalab
```

### Local development

```bash
pip install -e python
```

### Local development (from the Animus monorepo root)

```bash
pip install -e sdk/python
```

---

## Environment Variables

* `ANIMUS_GATEWAY_URL` (default: `http://localhost:8080`)
* `ANIMUS_AUTH_TOKEN` (optional, `Bearer` token for gateway auth)
* `ANIMUS_CI_WEBHOOK_SECRET` (optional; required if using `post_ci_webhook`)
* `DATAPILOT_URL` / `RUN_ID` / `TOKEN` (provided to training containers by Animus; used by `RunTelemetryLogger.from_env()`)

---

## Experiments Usage

```python
from animus_sdk import ExperimentsClient

client = ExperimentsClient(gateway_url="http://localhost:8080")

exp = client.create_experiment(
    name="baseline",
    description="Baseline training run",
    metadata={"team": "ml", "project": "fraud"},
)

run = client.create_run(
    experiment_id=exp["experiment_id"],
    dataset_version_id="YOUR_DATASET_VERSION_ID",
    status="succeeded",
    params={"lr": 1e-3},
    metrics={"accuracy": 0.91},
)

client.post_ci_webhook(
    payload={
        "run_id": run["run_id"],
        "provider": "github_actions",
        "context": {"workflow": "train.yml", "job": "train"},
    }
)
```

---

## CI image registration (signed)

Register a built image digest. This binds experiment runs to the **git commit** and **container image** used during execution.

```python
import os

from animus_sdk import ExperimentsClient

client = ExperimentsClient(
    gateway_url=os.environ.get("ANIMUS_GATEWAY_URL")
)

client.post_ci_report(
    payload={
        "image_digest": "sha256:...",
        "repo": "ghcr.io/acme/train",
        "commit_sha": "deadbeef...",
        "pipeline_id": "build-123",
        "provider": "github_actions",
    }
)
```

---

## Execute training run

```python
from animus_sdk import ExperimentsClient

client = ExperimentsClient(gateway_url="http://localhost:8080")

resp = client.execute_run(
    experiment_id="YOUR_EXPERIMENT_ID",
    dataset_version_id="YOUR_DATASET_VERSION_ID",
    image_ref="ghcr.io/acme/train@sha256:...",
)

print(resp["run_id"])
```

---

## Live Telemetry (training containers)

Training containers launched by Animus receive `DATAPILOT_URL`, `RUN_ID`, and `TOKEN` automatically.

Use `RunTelemetryLogger` to emit **append‑only metrics and events** without blocking training execution.

```python
from animus_sdk import RunTelemetryLogger

logger = RunTelemetryLogger.from_env(timeout_seconds=2.0)
logger.log_status(status="starting")

for step in range(100):
    loss = 1.0 / (step + 1)
    logger.log_metric(step=step, name="loss", value=loss)
    logger.log_progress(
        step=step,
        total_steps=100,
        percent=(step + 1) / 100.0,
    )

logger.log_status(status="finished")
logger.close(flush=True, timeout_seconds=5.0)
```

---

## Dataset download (training containers)

```python
import os

from animus_sdk import DatasetRegistryClient

datasets = DatasetRegistryClient(
    gateway_url=os.environ["DATAPILOT_URL"],
    auth_token=os.environ["TOKEN"],
)

datasets.download_dataset_version(
    dataset_version_id=os.environ["DATASET_VERSION_ID"],
    dest_path="/tmp/dataset.zip",
)
```

---

## Run artifacts (training / evaluation containers)

```python
import os

from animus_sdk import ExperimentsClient

exp = ExperimentsClient(
    gateway_url=os.environ["DATAPILOT_URL"],
    auth_token=os.environ["TOKEN"],
)

exp.upload_run_artifact(
    run_id=os.environ["RUN_ID"],
    kind="model",
    file_path="/tmp/model.json",
    name="model",
    metadata={"format": "json"},
)
```

---

## Design principles

* CI-first, non-interactive usage
* deterministic identifiers
* append-only telemetry
* explicit dataset and run binding
* compatible with on-prem and air-gapped deployments

---

## License

Apache-2.0

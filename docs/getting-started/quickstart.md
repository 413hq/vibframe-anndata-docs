# Quickstart

This page shows the two supported workflows: in-memory for moderate datasets and blockwise H5AD processing for datasets larger than RAM.

## In-memory workflow

```python
from vibframe_anndata import (
    add_features,
    import_raw,
    remove_features,
    write_h5ad,
)

adata = import_raw(
    "dataset.vibframe.zip",
    config={
        "version": 1,
        "ground_truth": {"enabled": True, "scope": "all", "on_missing": "error"},
    },
)

# Evaluation labels travel with the AnnData but remain outside X and obs.
truth = adata.obsm.get("ground_truth")  # absent for a DiagGT-only source

adata = add_features(
    adata,
    {
        "version": 1,
        "features": [
            {"name": "rms", "source": "waveform"},
            {"name": "kurtosis", "source": "waveform"},
        ],
    },
)

print(adata.shape)
print(adata.obs.head())
print(adata.var[["feature_name", "signal_source", "unit"]])

adata = remove_features(adata, ["kurtosis"])
write_h5ad(adata, "dataset_features.h5ad")
```

The original raw signals remain stored in the AnnData object while the variable axis changes.

## Large-file workflow

Use the on-disk API when the present raw signal payload does not fit comfortably in RAM:

```python
from vibframe_anndata import import_raw_to_h5ad, add_features_to_h5ad

base_path = import_raw_to_h5ad(
    "dataset.vibframe.zip",
    "dataset_raw.h5ad",
    config={
        "version": 1,
        "raw_import": {"on_missing_signal": "nan"},
        "ground_truth": {"enabled": True, "scope": "all", "on_missing": "error"},
        "output": {"dtype": "float32"},
    },
    block_size_mib=8,
)

feature_path = add_features_to_h5ad(
    base_path,
    {
        "version": 1,
        "output": {"dtype": "float32"},
        "feature_policy": {"on_error": "nan"},
        "features": [
            {"name": "rms", "source": "waveform"},
        ],
    },
    output="dataset_features.h5ad",
    block_rows=256,
)

print(feature_path)
```

The output file is written transactionally: the requested destination is promoted only after all blocks succeed.

## Read evaluation metadata or upgrade an existing H5AD

```python
from vibframe_anndata import (
    list_evaluation_files, get_snapshot_ground_truth, add_ground_truth_to_h5ad,
)

print(list_evaluation_files("dataset_features.h5ad"))
# When construction snapshot labels exist:
labels = get_snapshot_ground_truth("dataset_features.h5ad")

# Existing 0.2.x H5AD: preserve features/raw arrays and add complete source annotations.
add_ground_truth_to_h5ad(
    "older_features.h5ad", "dataset.vibframe.zip",
    output="complete_features.h5ad",
)
```

Evaluation import is opt-in; omit or disable `ground_truth` for an unlabelled source.
The retrofit reads annotations/source metadata and makes a transactional disk copy, not a second
raw-signal ingestion. Keep `source`, `machine` and `snap_t` in `obs`. See
[Ground truth and evaluation](../guides/ground-truth.md) for waveform labels and original DiagGT tables.

## Recalculate a feature

```python
from vibframe_anndata import recalculate_features

adata = recalculate_features(
    adata,
    {
        "version": 1,
        "features": [
            {"name": "rms", "source": "waveform"},
        ],
    },
)
```

The out-of-core equivalent is `recalculate_features_to_h5ad(...)`.

## Inspect available catalog metrics

After importing VibFrame catalog metadata:

```python
from vibframe_anndata import available_catalog_metrics, audit_metric_catalog

print(available_catalog_metrics(adata))

for status in audit_metric_catalog(adata):
    if not status.supported:
        print(status)
```

The audit is the authoritative runtime view of which persisted catalog definitions can be calculated with the signals present in that dataset.

## Next steps

- [Choosing a workflow](../guides/workflows.md)
- [Configuration](../guides/configuration.md)
- [AnnData data model](../concepts/data-model.md)
- [Python API](../reference/api.md)

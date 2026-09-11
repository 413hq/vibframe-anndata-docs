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

adata = import_raw("dataset.vibframe.zip")

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
print(adata.var[["name", "source"]])

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

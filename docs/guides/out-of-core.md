# Out-of-core workflow

The out-of-core API is designed for VibFrame datasets whose raw numerical payload should not be materialized fully in RAM.

## Phase 1: VibFrame → base H5AD

```python
from vibframe_anndata import import_raw_to_h5ad

base = import_raw_to_h5ad(
    "dataset.vibframe.zip",
    "dataset_raw.h5ad",
    config={
        "version": 1,
        "raw_import": {
            "waveforms": True,
            "spectra": True,
            "on_missing_signal": "nan",
        },
        "output": {"dtype": "float32"},
    },
    block_size_mib=8,
)
```

The writer scans the replayable VibFrame source, allocates the required HDF5/Awkward structures and writes numerical values in bounded blocks. The final destination is promoted only after structural validation succeeds.

## Phase 2: add features by observation blocks

```python
from vibframe_anndata import add_features_to_h5ad

output = add_features_to_h5ad(
    base,
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
```

For each observation block, only the raw Awkward slices reachable from those rows are reconstructed in memory. Feature values are then written directly into the on-disk `X` matrix.

## In-place atomic update

If `output` is omitted, the source path is replaced atomically after a successful calculation:

```python
add_features_to_h5ad(
    "dataset_raw.h5ad",
    config,
    block_rows=256,
)
```

Use an explicit `output=` while experimenting if you want to preserve the base H5AD.

## Recalculate features

```python
from vibframe_anndata import recalculate_features_to_h5ad

recalculate_features_to_h5ad(
    "dataset_features.h5ad",
    config,
    output="dataset_recalculated.h5ad",
    block_rows=256,
)
```

## Remove features

```python
from vibframe_anndata import remove_features_from_h5ad

remove_features_from_h5ad(
    "dataset_features.h5ad",
    ["rms"],
    output="dataset_without_rms.h5ad",
)
```

Removal rewrites the variable axis without loading the complete raw payload.

## Validate a large H5AD

```python
from vibframe_anndata import validate_streamed_h5ad

validate_streamed_h5ad("dataset_features.h5ad")
```

This performs structural checks directly against the file without materializing the multi-GiB raw arrays.

## Tuning `block_rows`

`block_rows` controls the number of observations reconstructed for a feature-calculation block.

- Smaller values reduce peak memory.
- Larger values may improve throughput by amortizing per-block overhead.
- The best value depends on signal lengths and how many raw channels are reachable from each observation.

The default is conservative. Start there unless you have measured a reason to change it.

## Representative scalability result

The 0.1.0 acceptance path converted a synthetic 180-day / 12-machine fleet into a 10.875 GiB float32 H5AD with a peak resident memory of about 1.060 GiB. A subsequent 36-column feature-enrichment run used about 0.321 GiB peak RSS.

These figures are validation evidence, not a universal resource guarantee: your raw signal lengths and feature mix determine actual memory usage.

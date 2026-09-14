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
        "output": {
            "dtype": "float32",
            "raw_compression": "none",
        },
    },
    block_size_mib=8,
)
```

The writer scans the replayable VibFrame source, allocates the required HDF5/Awkward structures and writes numerical values in bounded blocks. Temporary NumPy memmaps are explicitly closed before cleanup, which keeps the streamed import portable across POSIX and Windows. The final destination is promoted only after structural validation succeeds.

Starting in 0.2.1, waveform sample count is **not** part of logical channel identity. Repeated captures of the same physical channel therefore remain one ragged channel even when their sample lengths differ; exact per-snapshot lengths remain stored in the ragged companion metadata.

## Optional raw HDF5 compression

Release 0.2.1 supports lossless compression of the large ragged raw `node2-data` datasets:

```python
config = {
    "version": 1,
    "output": {
        "dtype": "float32",
        "raw_compression": "lzf",  # none | lzf | gzip
    },
}
```

For gzip, add `raw_compression_level` from 1 to 9; omitted gzip level defaults to 4. Compression is opt-in and `none` remains the default because the representative benchmark favored uncompressed I/O.

On the 155,520-observation variable-waveform acceptance fixture after the 0.2.1 channel-identity fix:

| Codec | H5AD size | Write time | Waveform RMS enrichment |
|---|---:|---:|---:|
| none | 13.854 GiB | 155.621 s | 75.297 s |
| LZF | 13.795 GiB | 192.416 s | 94.201 s |
| gzip-1 | 12.757 GiB | 555.066 s | 329.147 s |
| gzip-4 | 12.713 GiB | 626.528 s | 329.163 s |

All sampled raw digests matched the uncompressed baseline. LZF saved only 0.43%, while gzip saved about 8% at a much larger runtime cost, so compression is exposed as an explicit storage/performance trade-off rather than a new default.

The same representative dataset had previously produced approximately 98.35 GiB because variable waveform lengths were incorrectly treated as separate logical channels. With the corrected identity there are 36 waveform channels rather than 24,291, and the uncompressed file is 13.854 GiB while preserving the exact raw sample counts.

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

Release 0.2.0 introduced one immutable execution plan before observation-block iteration. Features are grouped by raw source/channel so each required channel is loaded once per block. Fixed-length spectra use dense NumPy workspaces with shared frequency/power intermediates, while compatible waveform statistics share reductions.

The planned streaming path writes calculated matrix blocks directly into on-disk `X`; it does **not** rebuild an enriched AnnData object for every observation block.

Before the first block is computed, the resolved target `var` and `uns` metadata are passed through the exact AnnData/HDF5 encoder inside the transactional partial file. Metadata that cannot be serialized therefore fails immediately rather than after a long-running calculation.

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

`block_rows` controls the number of observations processed for a feature-calculation block.

- Smaller values reduce peak working-set size.
- Larger values may improve throughput by amortizing per-block overhead.
- The best value depends on signal lengths, channel reachability and feature mix.

The default is conservative. Start there unless you have measured a reason to change it.

## Representative 0.2.0 feature-execution result

The accepted 180-day / 12-machine float32 H5AD feature benchmark used 155,520 observations, Python 3.12.14, `block_rows=256` and an 8 GiB macOS arm64 host.

| Workload | Added variables | Time | Peak RSS |
|---|---:|---:|---:|
| waveform-smoke | 36 | 32.705 s | 0.310 GiB |
| spectral-single | 1 | 28.261 s | 0.611 GiB |
| spectral-multi | 5 | 33.568 s | 0.754 GiB |
| EDA: 10 spectral + 3 waveform requests | 118 | 43.396 s | 0.860 GiB |

All workloads preserved raw observation/signal counts, passed structural and numerical sanity checks, and produced zero infinite feature values.

For comparison, the 0.1.0 36-column waveform control took 51.358 s at 0.321 GiB peak RSS. The 0.2.0 waveform-smoke result is therefore 36.3% faster on the same accepted feature dataset.

These figures are release evidence, not universal resource guarantees: your raw signal lengths, channel density and feature mix determine actual runtime and memory usage.

## Why execution remains serial

A measured 1/2/4/8-worker decision probe found the vectorized single-worker path fastest on the acceptance runner. The 0.2.x line therefore keeps serial execution rather than exposing a worker API that made the measured workload slower.

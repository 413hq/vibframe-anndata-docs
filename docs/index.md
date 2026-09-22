# vibframe-anndata

`vibframe-anndata` is a Python package for converting VibFrame vibration data into reusable [`AnnData`](https://anndata.readthedocs.io/) / `.h5ad` datasets and then deriving features from the raw signals already stored there.

The central idea is deliberately simple:

1. **Ingest VibFrame once** and preserve raw waveforms/spectra plus provenance in AnnData.
2. **Add, recalculate or remove features later** without reopening the original VibFrame.

```text
VibFrame
   │
   ▼
raw AnnData / H5AD
(obs + raw signals + provenance)
   │
   ├── add features
   ├── recalculate features
   └── remove features
   ▼
analysis-ready AnnData / H5AD
```

## Install

```bash
python -m pip install vibframe-anndata
```

The current release is **0.3.0** and supports Python 3.10+.

## Pick the right path

| Dataset | Recommended API | Memory model |
|---|---|---|
| Fits comfortably in RAM | `import_raw()` + `add_features()` | In-memory |
| Larger than RAM | `import_raw_to_h5ad()` + `add_features_to_h5ad()` | Bounded-memory / blockwise |

Both paths implement the same logical AnnData contract and the same numerical feature semantics.

!!! tip "Start here"
    If you are using the package for the first time, read the [Quickstart](getting-started/quickstart.md), then [Choosing a workflow](guides/workflows.md).

## What the package preserves

A package-generated dataset can contain:

- one `obs` row per vibration snapshot;
- calculated features in `X` / `var`;
- raw waveform and spectrum payloads in `obsm`;
- optional snapshot labels in `obsm['ground_truth']` and per-waveform annotations in `obsm['waveform_ground_truth']`;
- a byte-exact evaluation archive in `uns['vibframe_evaluation']`, including DiagGT, manifests and scenario context;
- snapshot/channel alignment information;
- VibFrame metric catalogs and feature descriptors;
- package configuration and provenance in `uns`.

This makes the H5AD a reusable analysis artifact rather than a one-shot export.

## Complete evaluation data

With `ground_truth.enabled: true`, release 0.3.0 preserves all regular files under
`evaluation/`, `ground-truth/` and `ground_truth/`, plus root JSON/YAML and machine JSON context.
Snapshot and waveform construction labels have explicit source/machine/time/channel alignment;
DiagGT originals retain their own schema and meaning. No diagnostic interval is silently converted
into a snapshot target.

```python
from vibframe_anndata import import_raw_to_h5ad, list_evaluation_files

path = import_raw_to_h5ad(
    "dataset.vibframe.zip", "dataset_raw.h5ad",
    config={
        "version": 1,
        "ground_truth": {"enabled": True, "scope": "all"},
        "output": {"dtype": "float32"},
    },
    block_size_mib=8,
)
print(list_evaluation_files(path))
```

Metadata accessors can read labels and individual original tables directly from an H5AD without
loading its waveforms. `add_ground_truth_to_h5ad()` can enrich existing 0.2.x files without
recalculating features. Evaluation remains opt-in and outside `X`/`obs`; `scope: snapshot`
preserves the former snapshot-only behavior. See [Ground truth and evaluation](guides/ground-truth.md)
for complete examples, missing-data rules, resource limits and migration.

## What changed in 0.2.1

Release 0.2.1 is a focused storage/correctness patch over 0.2.0:

- variable waveform sample count is no longer part of logical channel identity, so captures of the same physical channel with different lengths stay in one ragged channel;
- representative variable-length ingestion therefore uses 36 waveform channels rather than 24,291 fragmented identities;
- raw ragged HDF5 payload compression is configurable with `none`, `lzf` or `gzip`;
- `none` remains the default after representative benchmarking showed that LZF saved only 0.43% and gzip about 8% while materially increasing write and feature-read time.

On the representative 155,520-observation variable-waveform fixture, the corrected uncompressed H5AD is **13.854 GiB**, versus approximately **98.35 GiB** with the erroneous channel fragmentation. See [Out-of-core workflow](guides/out-of-core.md) and [Configuration](guides/configuration.md).

Release 0.2.0 introduced the underlying scalable feature engine: channel-centric planning, vectorized spectral workspaces, fused waveform reductions, matrix-only out-of-core enrichment and serialization preflight. Its accepted mixed EDA workload added 118 variables in **43.396 s** with **0.860 GiB** peak RSS.

## Metric support

Across the supplied metric catalogs, 76 of 104 unique metric names are reproducible from the available persisted signals. Unsupported definitions are rejected explicitly rather than approximated. Phase/cross-phase metrics still require an authoritative complex/phase representation, and two `cross_point_ratio` metrics still require an authoritative definition. See [Metric support](reference/metric-support.md).

## Versioned documentation

Use the version selector in the header when working with an older package release. `latest` follows the newest published release; numbered documentation remains available for reproducibility.

## License

The software package is distributed under the BSD 3-Clause License. External datasets, VibFrame captures and third-party material are not relicensed by the package documentation.

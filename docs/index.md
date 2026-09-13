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

The current release supports Python 3.10+.

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
- snapshot/channel alignment information;
- VibFrame metric catalogs and feature descriptors;
- package configuration and provenance in `uns`.

This makes the H5AD a reusable analysis artifact rather than a one-shot export.

## What changed in 0.2.0

Release 0.2.0 focuses on scalable feature execution and release hardening:

- channel-centric planning loads each required raw channel once per observation block;
- catalog-backed spectral families use vectorized shared workspaces;
- compatible waveform statistics share reductions;
- out-of-core enrichment writes feature matrices directly without rebuilding enriched AnnData objects for every block;
- mixed catalog/registered feature metadata is normalized and serialization-preflighted before long-running computation;
- release validation now includes explicit macOS and Windows smoke checks.

On the accepted 180-day / 12-machine H5AD, the representative mixed EDA workload added 118 variables in **43.396 s** with **0.860 GiB** peak RSS. The waveform control improved from **51.358 s** in 0.1.0 to **32.705 s** in 0.2.0.

See [Out-of-core workflow](guides/out-of-core.md) for the measured release-scale results.

## Metric support in 0.2.0

Across the supplied metric catalogs, 76 of 104 unique metric names are reproducible from the available persisted signals. Unsupported definitions are rejected explicitly rather than approximated. Phase/cross-phase metrics still require an authoritative complex/phase representation, and two `cross_point_ratio` metrics still require an authoritative definition. See [Metric support](reference/metric-support.md).

## Versioned documentation

Use the version selector in the header when working with an older package release. `latest` follows the newest published release; numbered documentation remains available for reproducibility.

## License

The software package is distributed under the BSD 3-Clause License. External datasets, VibFrame captures and third-party material are not relicensed by the package documentation.

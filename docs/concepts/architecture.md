# Architecture

`vibframe-anndata` separates source ingestion from feature engineering so a VibFrame source is read once and the resulting H5AD becomes the durable analysis artifact.

## Processing model

```text
Small / medium VibFrame
  → replayable VibFrame adapter
  → in-memory ragged AnnData base
  → NumPy / SciPy feature engine
  → enriched AnnData

Large VibFrame
  → replayable VibFrame adapter
  → bounded scan + direct HDF5/Awkward write
  → base H5AD on disk
  → observation-block raw slices
  → same NumPy / SciPy feature engine
  → blockwise X/var update
  → enriched H5AD
```

The important architectural boundary is that feature engineering consumes raw signals **from AnnData/H5AD**, not from the original VibFrame.

## VibFrame adapter boundary

The adapter is the only layer that understands the physical VibFrame representation, including dataset/machine metadata, Parquet waveform/spectrum tables and the metric catalog.

A replayable adapter allows the scalable writer to make one pass for exact shape/count discovery and a later pass for numerical writes without retaining the complete raw payload in memory.

## Ragged raw storage

New datasets use `awkward-ragged-v1` raw storage. Only signals that actually exist are stored.

This avoids a dense Cartesian allocation across:

- all snapshots;
- all machines;
- all channels;
- the longest signal length.

Companion matrices preserve channel positions, real lengths and speed metadata so asynchronous acquisition schedules remain explicit.

## Shared numerical engine

Both in-memory and out-of-core APIs use the same feature implementation. This is intentional: changing execution mode should not imply changing the numerical definition of a feature.

The feature engine contains:

- registered generic NumPy metrics such as RMS, peaks and kurtosis;
- catalog-backed calculations driven by persisted metric descriptors and reference semantics;
- explicit unsupported paths where required source information is unavailable.

## Out-of-core feature editing

For large H5AD files, feature calculation works approximately as follows:

1. validate the package-generated H5AD structure;
2. create a transactional output copy;
3. load small `obs`, `var` and `uns` metadata;
4. read only raw Awkward slices reachable from the current observation block;
5. reconstruct a block-sized temporary AnnData;
6. run the same feature service used in-memory;
7. write that block directly into the on-disk `X` dataset;
8. promote the final variable contract only after all blocks succeed.

Memory therefore scales mainly with the current observation block and reachable signals rather than the total raw H5AD size.

## Failure semantics

Unsupported feature definitions fail explicitly. Large writes use temporary files so an exception does not leave an incomplete file under the requested final name.

## Design principles

1. Follow native AnnData orientation: observations × variables.
2. Treat raw waveforms/spectra as durable inputs, not disposable intermediates.
3. Store only present raw samples.
4. Preserve provenance and catalog descriptors.
5. Use one numerical implementation across execution modes.
6. Prefer explicit unsupported behavior over undocumented approximation.
7. Keep production calculation independent of ground-truth trend tables used for regression testing.

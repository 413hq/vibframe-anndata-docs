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
  → immutable feature execution plan
  → observation-block raw channel batches
  → matrix-only feature executor
  → direct X/var update
  → enriched H5AD
```

The important architectural boundary is that feature engineering consumes raw signals **from AnnData/H5AD**, not from the original VibFrame.

## VibFrame adapter boundary

The adapter is the only layer that understands the physical VibFrame representation, including dataset/machine metadata, Parquet waveform/spectrum tables and the metric catalog.

A replayable adapter allows the scalable writer to make one pass for exact shape/count discovery and a later pass for numerical writes without retaining the complete raw payload in memory.

The base raw-signal layout is `twave-vibframe-parquet/0.2`. The default catalog adapter in 0.2.2
records `twave-vibframe-parquet/0.4`, adding optional metric-catalog and snapshot-truth discovery.
These identifiers describe implemented package behavior; they are not asserted to be official
producer/TWave schema versions.

## Ragged raw storage

New datasets use `awkward-ragged-v1` raw storage. Only signals that actually exist are stored.

This avoids a dense Cartesian allocation across:

- all snapshots;
- all machines;
- all channels;
- the longest signal length.

Companion matrices preserve channel positions, real lengths and speed metadata so asynchronous acquisition schedules remain explicit.

## Shared numerical semantics

In-memory and out-of-core APIs share the same feature definitions and validation rules. Changing execution mode must not change the numerical meaning of a feature.

The feature engine contains:

- registered generic NumPy metrics such as RMS, peaks and kurtosis;
- catalog-backed calculations driven by persisted metric descriptors and reference-derived semantics;
- explicit unsupported paths where required source information is unavailable.

## 0.2.0 execution planning

For large H5AD files, feature execution is planned once before observation-block iteration:

1. resolve requested features and construct one immutable execution plan;
2. group features by raw source/channel;
3. load each required raw channel once for the current observation block;
4. execute compatible feature families against shared numerical workspaces;
5. write the resulting matrix block directly into on-disk `X`.

Fixed-length spectra use dense NumPy workspaces that share frequency axes and cumulative/power intermediates across bands, peaks, harmonic families and sidebands. Compatible waveform statistics share RMS, extrema and moment reductions.

This replaces the older per-feature/per-block orchestration pattern and avoids rebuilding an enriched temporary AnnData object for every block.

## Metadata and transactional writes

Large feature edits operate on a transactional copy. Small static metadata and the raw HDF5 layout are discovered outside the hot block loop.

Mixed catalog-backed and registered feature rows are normalized to one string-safe `var` schema. Before any observation block is computed, the resolved target `var` and `uns` are passed through the exact AnnData/HDF5 serializer under disposable preflight keys in the partial file.

If metadata is not serializable, the operation fails before expensive feature computation begins. The final destination is promoted only after the entire operation validates successfully.

## Bounded-memory base import

The streamed base writer uses temporary memory-mapped indices while constructing final Awkward/HDF5 structures. Those memmaps, including waveform tacho companions, are explicitly closed before temporary-directory cleanup. This is required for Windows portability as well as clean resource ownership on POSIX systems.

## Parallelism decision

Release 0.2.0 deliberately keeps production feature execution serial. A reproducible 1/2/4/8-worker probe found the vectorized single-worker path fastest on the acceptance runner, so no worker API was added without evidence of benefit.

## Failure semantics

Unsupported feature definitions fail explicitly. Metadata serialization is preflighted before block execution, and large writes use temporary files so an exception does not leave an incomplete file under the requested final name.

## Design principles

1. Follow native AnnData orientation: observations × variables.
2. Treat raw waveforms/spectra as durable inputs, not disposable intermediates.
3. Store only present raw samples.
4. Preserve provenance and catalog descriptors.
5. Keep numerical semantics consistent across execution modes.
6. Reuse raw channels and numerical intermediates when multiple features share them.
7. Prefer explicit unsupported behavior over undocumented approximation.
8. Keep production calculation independent of ground-truth trend tables used for regression testing.

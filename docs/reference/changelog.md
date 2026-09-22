# Changelog

## 0.3.0 — 2026-09-22

### Added

- Complete opt-in evaluation preservation: all files below `evaluation/`, `ground-truth/`
  and `ground_truth/`, root JSON/YAML context and machine JSON documents are retained
  byte-for-byte in `uns['vibframe_evaluation']`, with source/path identities, sizes and SHA-256.
  This includes normative DiagGT documents, observations, consolidated observations,
  findings, manifests, generator configuration, waveform truth and paired-control metadata.
- Snapshot-aligned waveform annotations in `obsm['waveform_ground_truth']`, with explicit
  raw-channel bindings, missing/ambiguous diagnostics and strict microsecond timestamp handling.
- Public resident/H5AD accessors: `list_evaluation_files`, `read_evaluation_file`,
  `read_evaluation_json`, `read_evaluation_table`, `export_evaluation_files`,
  `get_snapshot_ground_truth`, `get_waveform_ground_truth`.
- `add_ground_truth_to_h5ad` enriches existing package H5AD files transactionally, without
  recalculating features or re-reading source raw sample payloads.
- Capture-time companion matrices preserve each raw row's `t` separately from `snap_t`;
  known/fallback flags distinguish an explicit acquisition timestamp from a legacy fallback.
- Configurable `ground_truth.scope` (`all` or legacy `snapshot`) and an explicit 512 MiB
  default encoded-sidecar budget; exceeding the budget fails instead of discarding truth.

### Changed

- When ground-truth ingestion is enabled, `all` is now the default scope. The feature remains
  disabled by default. Select `scope: snapshot` for the previous 0.2.2 footprint and behavior.
- The catalog adapter contract is `twave-vibframe-parquet/0.5`; original raw compression
  defaults and numerical feature definitions are unchanged.
- Same-named input datasets are disambiguated by source-path hashes; repeating one source
  path is rejected instead of merging observations or annotations accidentally.
- Out-of-core feature edits preserve embedded sidecar buffers through HDF5 links rather
  than decoding or rewriting them with each feature block.

### Fixed

- Floating-point timestamp ambiguities, null identities, conflicting `timestamp`/`snap_t`,
  missing waveform modes and duplicate channel bindings now fail explicitly or retain a
  documented unresolved status under `on_missing: ignore`.
- Streamed H5AD validation occurs before destination replacement so failures leave the
  previous file intact. Export rejects unsafe paths and refuses overwrites.

### Interpretation

Complete preservation is not automatic semantic projection of arbitrary diagnostic intervals.
Only declared snapshot/waveform identities are aligned; DiagGT records and future schemas
remain available exactly as authored. See [Ground truth and evaluation](../guides/ground-truth.md).


## 0.2.2 — 2026-09-15

Patch release 0.2.2 makes snapshot-level evaluation truth a first-class, opt-in part of the AnnData
artifact.

### Added

- ingestion of partitioned `evaluation/snapshot_truth/**/*.parquet` into the observation-aligned
  `obsm['ground_truth']` DataFrame for both in-memory and streamed H5AD creation;
- strict `(source, machine_id, snap_t)` alignment, duplicate detection and configurable missing
  truth handling;
- source-file paths, row counts and SHA-256 digests in package provenance;
- structural and round-trip validation plus preservation across feature editing;
- public `GroundTruthConfig` with `enabled` and `on_missing` settings.

Ground-truth ingestion is disabled by default. Labels remain outside `X` and `obs`, and
`trends.parquet` remains excluded from production ingestion.

## 0.2.1 — 2026-09-14

Patch release 0.2.1 fixes variable-waveform channel fragmentation and adds benchmarked opt-in compression for large ragged raw HDF5 payloads.

### Fixed

- waveform `n_samples` is no longer part of logical channel identity;
- repeated captures of the same physical waveform channel with different sample lengths now remain one ragged channel;
- exact per-snapshot sample lengths and raw payload values remain preserved;
- the representative 155,520-observation variable-waveform fixture now uses **36 waveform channels instead of 24,291**;
- the corrected uncompressed H5AD is **13.854 GiB**, versus approximately **98.35 GiB** with the fragmented channel identity.

### Added

- `output.raw_compression` with `none`, `lzf` and `gzip`;
- `output.raw_compression_level` for gzip levels 1–9, defaulting to 4 when gzip is selected;
- compression is applied only to the large ragged raw `node2-data` payloads; small dense companion matrices remain uncompressed;
- exact round-trip regression coverage for uncompressed, LZF, gzip-1 and gzip-4 storage;
- representative size/write/read/feature-enrichment benchmark evidence for the compression decision.

### Compression decision

The representative post-fix benchmark produced:

| Codec | H5AD size | Write time | Waveform RMS enrichment |
|---|---:|---:|---:|
| none | 13.854 GiB | 155.621 s | 75.297 s |
| LZF | 13.795 GiB | 192.416 s | 94.201 s |
| gzip-1 | 12.757 GiB | 555.066 s | 329.147 s |
| gzip-4 | 12.713 GiB | 626.528 s | 329.163 s |

All sampled raw digests matched the uncompressed baseline. `none` therefore remains the default: LZF saved only 0.43%, while gzip saved about 8% at a substantially higher write and feature-read cost. Compression remains available when storage pressure justifies the trade-off.

Release validation passed Linux CI on Python 3.10/3.12, the non-publishing release-candidate build/wheel smoke, and macOS 14 + Windows latest platform smoke before publication. The final workflow then published to PyPI, installed the exact `0.2.1` package back from PyPI, and created the GitHub Release.

## 0.2.0 — 2026-09-13

Release 0.2.0 focuses on scalable feature execution, serialization safety and release maturity.

Highlights:

- channel-centric feature planning so same-channel requests reuse one raw load per observation block;
- vectorized catalog-backed spectral workspaces for bands, peaks, harmonic families and sidebands;
- fused waveform RMS/extrema/moment reductions for compatible registered statistics;
- matrix-only out-of-core execution that avoids rebuilding enriched AnnData objects per block;
- fail-fast HDF5 metadata preflight and canonical string-safe `var` metadata for mixed catalog/registered features;
- explicit temporary-memmap cleanup for streamed import portability on Windows;
- package-side VibFrame contract constants pinning `snap_t` as Unix-epoch microseconds interpreted in UTC;
- public API stability policy and third-party dependency/licence audit;
- opt-in macOS 14 and Windows latest release-smoke validation;
- hardened release automation with built-wheel smoke testing, checksums and exact post-PyPI verification.

### Final 180-day acceptance benchmark

On the accepted 155,520-observation / 12-machine H5AD with `block_rows=256`:

| Workload | Added variables | Time | Peak RSS |
|---|---:|---:|---:|
| waveform-smoke | 36 | 32.705 s | 0.310 GiB |
| spectral-single | 1 | 28.261 s | 0.611 GiB |
| spectral-multi | 5 | 33.568 s | 0.754 GiB |
| EDA: 10 spectral + 3 waveform requests | 118 | 43.396 s | 0.860 GiB |

All four workloads passed structural and numerical validation with zero infinite feature values. The waveform control improved from 51.358 s in 0.1.0 to 32.705 s in 0.2.0, a 36.3% reduction on the accepted dataset.

### Intentionally deferred

The project does not approximate missing external semantics:

- phase/cross-phase support still requires an authoritative persisted complex/phase representation;
- `cross_point_ratio` still requires an authoritative definition/reference;
- producer-side migration guarantees and authoritative per-metric unit/domain/precondition/production-tolerance contracts remain external inputs.

See [Known limitations](limitations.md) and [Metric support](metric-support.md) for the current boundary.

## 0.1.0 — 2026-09-11

Initial public release of `vibframe-anndata`.

Highlights:

- staged VibFrame → AnnData/H5AD workflow;
- reusable raw waveform/spectrum storage in `obsm`;
- in-memory and bounded-memory out-of-core APIs;
- incremental add/recalculate/remove feature operations;
- catalog-backed metric calculation with explicit unsupported definitions;
- deterministic provenance and configuration tracking;
- structural validation for resident AnnData and large H5AD files;
- synthetic reference regression and large-data scalability validation;
- PyPI distribution under BSD-3-Clause.

Numbered documentation snapshots remain available through the version selector for reproducibility.

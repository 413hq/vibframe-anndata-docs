# Changelog

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

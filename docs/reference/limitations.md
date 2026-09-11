# Known limitations

These limitations are part of the public 0.1.0 contract so unsupported behavior is not mistaken for a numerical result.

## Phase and cross-phase metrics

The current VibFrame spectra used by the package persist magnitude rather than complex FFT values or equivalent phase information. `peak_phase_*` and `cross_phase_*` definitions are therefore unsupported.

The package raises a controlled error instead of estimating phase from insufficient data.

## Cross-point ratio metrics

`axial_radial_ratio_1X` and `vertical_horizontal_ratio_1X` use the statistic `cross_point_ratio`. The supplied reference metric implementation did not provide an authoritative calculation for that statistic, so these definitions remain unsupported.

## Catalog coverage

Across the supplied catalogs, 76 of 104 unique metric names are reproducible in 0.1.0. All 430 catalog descriptors classified as supported passed the full synthetic reference regression at `atol=1e-3`, `rtol=1e-3`.

## VibFrame producer compatibility

The adapter targets the VibFrame contract exercised during development and validation. Future producer/schema changes may require an adapter update; 0.1.0 does not promise compatibility with unknown future VibFrame revisions.

## Variable-axis AnnData structures

Feature editing is intentionally conservative around variable-axis auxiliary structures. Objects that already use structures such as `layers`, `varm`, `varp` or `raw` may be rejected when an operation would risk silently breaking variable alignment.

Package-generated base datasets are not affected by this restriction.

## Out-of-core performance

Large-file ingestion and feature enrichment are bounded-memory and acceptance-tested. The 0.1.0 implementation prioritizes correctness and memory bounds over aggressive multi-feature I/O coalescing; deeper throughput optimization is planned for later releases.

## Platform coverage

Continuous release validation targets Linux on Python 3.10 and 3.12. The project has also been exercised on macOS/Python 3.12 during full reference/scalability validation. Continuous Windows/macOS CI coverage is not part of the 0.1.0 gate.

## AnnData/Awkward warnings

AnnData may emit an `ExperimentalFeatureWarning` when serializing Awkward-backed structures. This is an upstream status warning rather than evidence that the package calculation failed. Users should still validate generated files with the package validation functions.

## External data licensing

The BSD-3-Clause package license applies to the software. External VibFrame captures, reference datasets and third-party materials are not automatically relicensed by the package.

# Known limitations

These limitations are part of the public 0.2.0 contract so unsupported behavior is not mistaken for a numerical result.

## Phase and cross-phase metrics

The current VibFrame spectra used by the package persist magnitude rather than complex FFT values or equivalent phase information. `peak_phase_*` and `cross_phase_*` definitions are therefore unsupported.

The package raises a controlled error instead of estimating phase from insufficient data. An authoritative complex/phase representation is required before these metrics can be implemented.

## Cross-point ratio metrics

`axial_radial_ratio_1X` and `vertical_horizontal_ratio_1X` use the statistic `cross_point_ratio`. The supplied reference metric implementation does not provide an authoritative calculation for that statistic, so these definitions remain unsupported.

## Catalog coverage and tolerance authority

Across the supplied catalogs, 76 of 104 unique metric names are reproducible. All 430 catalog descriptors classified as supported passed the complete synthetic reference regression at `atol=1e-3`, `rtol=1e-3`.

Those tolerances are repository regression thresholds, not asserted TWave production tolerances or rounding rules. Authoritative per-metric units, valid domains, preconditions and production tolerances remain external contract inputs.

## VibFrame producer compatibility

The implemented package-side adapter contract is `twave-vibframe-parquet/0.2`. The package interprets `snap_t` as Unix-epoch microseconds, preserves the integer value in `obs['snap_t']`, and derives `obs['timestamp']` in UTC.

This describes package behavior, not an official producer-version guarantee. Future producer/schema changes that alter field meanings, timestamp units, raw representations or metric semantics may require an explicit adapter/contract revision.

## Variable-axis AnnData structures

Feature editing is intentionally conservative around variable-axis auxiliary structures. Objects that already use structures such as `layers`, `varm`, `varp` or `raw` may be rejected when an operation would risk silently breaking variable alignment.

Package-generated base datasets are not affected by this restriction.

## Out-of-core resource use

Large-file ingestion and feature enrichment are bounded-memory and acceptance-tested, but the measured figures are not universal resource guarantees. Raw signal lengths, channel density, feature mix and `block_rows` affect runtime and peak memory.

On the accepted 180-day / 12-machine H5AD, release 0.2.0 completed:

- 36 waveform features in **32.705 s** at **0.310 GiB** peak RSS;
- 1 spectral feature in **28.261 s** at **0.611 GiB** peak RSS;
- 5 spectral features in **33.568 s** at **0.754 GiB** peak RSS;
- the representative 10-spectral + 3-waveform EDA workload, producing 118 variables, in **43.396 s** at **0.860 GiB** peak RSS.

All four workloads passed structural/numerical validation with zero infinite feature values.

## Parallel execution

Release 0.2.0 intentionally keeps feature execution serial. A reproducible 1/2/4/8-worker scaling probe found the vectorized serial path fastest on the acceptance runner, so the package does not expose speculative production worker concurrency.

## Platform coverage

Mandatory pull-request/release CI targets Linux on Python 3.10 and 3.12. Release 0.2.0 additionally passed the opt-in platform-smoke workflow on macOS 14 / Python 3.12 and Windows latest / Python 3.12.

The platform-smoke workflow is manual by design so hosted-runner quota is not consumed on every development PR.

## CI reference coverage

Mandatory CI validates a deterministic synthetic subset of the reference contract. The broader local acceptance layer remains 430 supported descriptors / 3,440 reference values, including demodulation paths.

## AnnData/Awkward warnings

AnnData may emit an `ExperimentalFeatureWarning` when serializing Awkward-backed structures. This is an upstream status warning rather than evidence that the package calculation failed. Users should still validate generated files with the package validation functions.

## External data licensing

The BSD-3-Clause package license applies to the software. Repository-authored synthetic fixtures are non-sensitive, but external VibFrame captures, TWave-provided/reference material and other third-party data are not automatically relicensed by the package.

## Release immutability

PyPI publication and release tags are effectively immutable release events. Release 0.2.0 therefore went through normal CI, a non-publishing release-candidate gate, macOS/Windows platform smoke, built-wheel validation, PyPI Trusted Publishing, installation of the exact published version back from PyPI, and only then GitHub Release creation.

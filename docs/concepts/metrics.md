# Metrics and provenance

`vibframe-anndata` supports two related feature mechanisms.

## Registered generic metrics

The generic registry exposes benchmark/general-purpose waveform metrics such as:

- positive peak (`pk_positive` / `pk_plus`);
- negative peak (`pk_negative` / `pk_minus`);
- peak-to-peak (`peak_to_peak` / `p2p`);
- RMS;
- crest factor;
- kurtosis.

These are selected through normal feature requests.

## Catalog-backed metrics

When a VibFrame contains a persisted metric catalog, catalog-backed metrics use the catalog descriptor as the effective calculation contract.

This matters because a catalog definition can encode details such as:

- source signal family;
- frequency/order targets;
- band boundaries;
- harmonic families;
- sideband definitions;
- demodulation parameters;
- unit codes.

The package preserves the descriptor in feature metadata rather than silently replacing it with ad-hoc user options.

## Implemented calculation families

The current implementation includes:

- waveform peak+/peak-/peak-to-peak/RMS;
- crest factor, kurtosis and skewness;
- spectrum overall RMS with reference Hann correction;
- inclusive Hz/order bands;
- peak lookup with one-bin tolerance;
- harmonic-family RMS;
- sideband RMS;
- envelope/demodulation processing;
- bearing-frequency families where the source descriptor provides the required definition.

## Unsupported definitions are explicit

Some catalog definitions cannot be reconstructed from the persisted source signals.

For 0.1.0:

- phase and cross-phase metrics need complex spectral information, while the available spectra persist magnitude only;
- two cross-point ratio metrics lack an authoritative supplied reference calculation.

The package rejects these definitions rather than fabricating values.

## Inspect support at runtime

```python
from vibframe_anndata import available_catalog_metrics, audit_metric_catalog

names = available_catalog_metrics(adata)
print(names)

statuses = audit_metric_catalog(adata)
for status in statuses:
    print(status)
```

`audit_metric_catalog()` is the best way to answer “can this exact persisted metric definition be calculated?” for a particular imported dataset.

## Numerical regression target

Reference regression for supported synthetic catalog descriptors uses:

```text
absolute tolerance = 1e-3
relative tolerance = 1e-3
```

This tolerance belongs to validation/testing. Production feature values are not rounded to that precision automatically.

## Provenance

When a feature is materialized, the package keeps enough metadata to trace the result back to its request/catalog definition and raw channel selection. This is particularly important when multiple machines, processing modes or similarly named features coexist in one AnnData object.

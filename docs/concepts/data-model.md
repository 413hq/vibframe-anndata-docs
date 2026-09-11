# AnnData data model

The package uses AnnData in its native `n_obs × n_vars` orientation.

| AnnData slot | Meaning in `vibframe-anndata` |
|---|---|
| `obs` | One row per snapshot |
| `X` | Numerical matrix: snapshots × materialized features |
| `var` | One row per materialized feature/metric |
| `obsm['raw_waveforms']` | Snapshot-aligned ragged waveform payload |
| `obsm['raw_spectra']` | Snapshot-aligned ragged spectrum payload |
| `uns['vibframe_anndata']` | Provenance, configuration, catalog and raw-channel metadata |

## Base dataset

Immediately after raw import, the dataset may intentionally contain no calculated feature:

```text
shape == (n_snapshots, 0)
len(obs) == n_snapshots
len(var) == 0
```

That zero-feature object/file is the reusable phase-1 state.

## `obs`: snapshot axis

`snapshot_id` is mandatory and aligned with the observation index. Depending on available source metadata, additional columns can include:

- timestamp / `snap_t`;
- source and machine identifiers;
- speed information;
- `has_waveform`;
- `has_spectrum`.

`has_waveform` and `has_spectrum` are useful when acquisition schedules differ. A spectrum-only timestamp remains a valid observation even if no waveform exists for that row.

## Raw signals in `obsm`

New imports use ragged Awkward arrays:

```text
raw_waveforms[snapshot] → [waveform_0, waveform_1, ...]
raw_spectra[snapshot]   → [spectrum_0, spectrum_1, ...]
```

Only present signals are stored.

Companion arrays record:

- real sample length (`0` means absent);
- signal position in the ragged snapshot list (`-1` means absent);
- signal-level speed metadata where available;
- waveform tacho data where present.

This representation avoids allocating dense NaN-padded sample blocks for missing channels/timestamps.

## `X` and `var`

Every calculated feature occupies:

- one column in `X`;
- one row in `var`.

Catalog-backed features retain their catalog identity and descriptor metadata for traceability. Registered features derive deterministic logical identifiers from their request and selected raw channel.

Typical `var` metadata can include:

- feature name / logical id;
- source signal family;
- unit code;
- machine / point / processing mode;
- algorithm or reference status;
- options/configuration hash;
- metric/catalog identifiers.

## `uns['vibframe_anndata']`

Package provenance can include:

- package version;
- VibFrame adapter contract;
- source metadata;
- ingestion configuration and hash;
- effective feature configuration;
- missing/error policy;
- raw channel/storage metadata;
- persisted metric catalog;
- generation/modification timestamps.

This information is intended to make derived datasets auditable and reproducible.

## H5AD at scale

For large files, the ragged numerical payload is represented in standard AnnData/Awkward HDF5 groups with offset arrays and flat sample data. The package can validate the resulting structure without loading the complete raw payload into RAM.

## Validation

For resident AnnData:

```python
from vibframe_anndata import validate_anndata_contract
validate_anndata_contract(adata)
```

For large files:

```python
from vibframe_anndata import validate_streamed_h5ad
validate_streamed_h5ad("dataset.h5ad")
```

These functions validate structural/package invariants; they are not substitutes for domain-level validation of your experiment.

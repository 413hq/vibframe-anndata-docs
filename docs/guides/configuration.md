# Configuration

Configuration can be supplied as YAML, a Python mapping or an already validated `PackageConfig`.

## Example YAML

```yaml
version: 1

input:
  source: dataset.vibframe.zip

raw_import:
  waveforms: true
  spectra: true
  on_missing_signal: error   # error | skip_snapshot | nan

ground_truth:
  enabled: false             # opt in to evaluation/snapshot_truth
  on_missing: error          # error | ignore

output:
  path: dataset_features.h5ad
  dtype: float64             # float32 | float64
  raw_compression: none      # none | lzf | gzip
  # raw_compression_level: 4 # gzip only; integer 1..9

feature_policy:
  on_conflict: error         # error | replace | variant
  on_error: error            # error | nan

features:
  - name: rms
    source: waveform
    select:
      point_id: P1
    options: {}

  - name: disp_overall_rms
    source: spectrum

obs:
  include:
    - snapshot_id
    - timestamp
    - machine
```

## Top-level keys

| Key | Meaning |
|---|---|
| `version` | Configuration schema version. Current value: `1` |
| `input` | Optional source path for convenience workflows |
| `raw_import` | Which raw families to ingest and missing-signal policy |
| `ground_truth` | Opt-in snapshot evaluation-truth ingestion and missing-data policy |
| `output` | Optional output path, numerical dtype and raw HDF5 compression policy |
| `features` | Ordered list of feature requests |
| `obs` | Requested observation columns |
| `feature_policy` | Conflict and calculation-error policies |

Unknown keys are rejected.

## Snapshot evaluation ground truth

```yaml
ground_truth:
  enabled: true
  on_missing: error
```

When enabled, partitioned Parquet files below `evaluation/snapshot_truth/` are aligned to the final
observation axis by `(source, machine_id, snap_t)` and stored as the DataFrame
`obsm['ground_truth']`. If `snap_t` is absent, numeric `timestamp` values are interpreted as
Unix-epoch microseconds.

The default is disabled so labels cannot enter an ordinary ingestion workflow accidentally.
`on_missing: error` rejects an absent sidecar or an observation with no matching truth row.
`on_missing: ignore` omits an absent sidecar and permits explicitly missing aligned rows. Duplicate
truth keys always fail.

Source paths, row counts and SHA-256 digests are recorded under
`uns['vibframe_anndata']['ground_truth']`. The data is not copied into `X` or `obs`, and
`trends.parquet` remains excluded.

## Feature requests

Each feature request supports:

| Field | Meaning |
|---|---|
| `name` | Required feature or catalog metric name |
| `source` | Optional explicit `waveform` / `spectrum` check |
| `alias` | Optional display/logical alias |
| `select` | Scalar-valued channel/catalog selector |
| `options` | Algorithm-specific options for registered metrics |

Catalog-backed metrics take their effective calculation parameters from the persisted metric catalog. Ad-hoc `options` are rejected for those metrics rather than silently overriding the reference descriptor.

## Conflict policies

### `error`

Fail when the requested logical feature already exists.

```yaml
feature_policy:
  on_conflict: error
```

### `replace`

Replace the existing logical feature.

```yaml
feature_policy:
  on_conflict: replace
```

### `variant`

Keep the existing feature and create a deterministic variant identifier based on the request/descriptor hash.

```yaml
feature_policy:
  on_conflict: variant
```

## Missing raw signals

### `error`

Abort ingestion when a required signal is missing.

### `skip_snapshot`

Omit an incomplete snapshot and record the exclusion.

### `nan`

Preserve the snapshot. Missing raw signals are represented structurally; no fake sample block is allocated. Derived features may become `NaN` when calculation policy allows it.

## Feature calculation errors

```yaml
feature_policy:
  on_error: error
```

raises a controlled feature error for unsupported/non-finite calculation paths.

```yaml
feature_policy:
  on_error: nan
```

preserves the observation and records the result as `NaN` where appropriate.

## Numerical dtype

```yaml
output:
  dtype: float32
```

is useful for large H5AD datasets when reduced storage is acceptable.

```yaml
output:
  dtype: float64
```

uses double precision for the materialized numerical matrices.

The package does not implicitly change the semantics of catalog units when choosing the storage dtype.

## Raw HDF5 compression

Release 0.2.1 adds optional lossless compression for the large ragged raw `node2-data` payloads written by the streamed H5AD importer.

```yaml
output:
  raw_compression: none
```

is the default and keeps the fastest measured write/read path.

```yaml
output:
  raw_compression: lzf
```

uses HDF5 LZF compression.

```yaml
output:
  raw_compression: gzip
  raw_compression_level: 4
```

uses gzip. `raw_compression_level` is valid only with gzip and must be an integer from 1 to 9; when omitted for gzip, the package uses level 4.

Compression applies to the large ragged raw sample payloads, not to the small dense companion matrices. It does not change sample values or waveform lengths.

The 0.2.1 representative benchmark kept `none` as the default: LZF reduced the full H5AD by only 0.43% while increasing write time by 23.6% and waveform-RMS enrichment by 25.1%; gzip levels 1 and 4 reduced size by about 8% but increased write time to 3.57–4.03× and RMS enrichment to about 4.37× the uncompressed baseline. Use compression when storage pressure justifies that trade-off.

## Load configuration explicitly

```python
from vibframe_anndata import load_config

cfg = load_config("config.yaml")
print(cfg)
```

Validation happens before expensive processing whenever the required information is already available. Descriptor-dependent checks are performed when a persisted catalog request is resolved.

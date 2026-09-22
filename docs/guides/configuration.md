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
  enabled: false             # opt in to evaluation metadata
  scope: all                 # all | snapshot (legacy footprint)
  on_missing: error          # error | ignore
  max_sidecar_mib: 512       # positive integer; original encoded sidecar bytes

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
    - snap_t
    - source
    - machine
```

## Top-level keys

| Key | Meaning |
|---|---|
| `version` | Configuration schema version. Current value: `1` |
| `input` | Optional source path for convenience workflows |
| `raw_import` | Which raw families to ingest and missing-signal policy |
| `ground_truth` | Opt-in complete evaluation preservation, scope, coverage policy and encoded-sidecar budget |
| `output` | Optional output path, numerical dtype and raw HDF5 compression policy |
| `features` | Ordered list of feature requests |
| `obs` | Requested observation columns |
| `feature_policy` | Conflict and calculation-error policies |

Unknown keys are rejected.

## Ground truth and evaluation metadata

```yaml
ground_truth:
  enabled: true
  scope: all
  on_missing: error
  max_sidecar_mib: 512
```

`enabled` defaults to false. When true, `scope` defaults to `all`: retain all regular files under
`evaluation/`, `ground-truth/`, `ground_truth/`, root JSON/YAML and machine JSON context. Explicit
construction snapshot and waveform tables also receive source/machine/time/channel-aligned views
in `obsm`. Original files, including DiagGT and unknown future sidecars, live in
`uns['vibframe_evaluation']` with source/path/size/SHA-256 provenance.

`scope: snapshot` opts into the previous 0.2.2 behavior and footprint. It does not create a complete
archive or waveform projection. Neither scope adds labels to `X` or `obs`.

`on_missing: error` rejects absent requested evaluation data and missing/ambiguous declared
construction labels. Full scope does not require a construction table from a source that only
provides DiagGT. Mixed sources may provide different truth families. `ignore` permits explicitly
missing/unresolved values, never guessed labels. Duplicate truth keys and duplicate bindings fail.
Spectra-only imports may retain waveform annotations as unbound.

`max_sidecar_mib` is a positive integer limit on the sum of original encoded sidecar bytes in full
scope. The default is 512 MiB; exceeding it raises an error rather than dropping annotations.
It is not a process-RAM limit: decoded tables, JSON and aligned projections consume additional
memory. Read large annotation tables individually with the metadata accessors.

Integer `timestamp`/`snap_t` values must be exact UTC microseconds. Retain `source`, `machine`
and `snap_t` in `obs` when an H5AD may later be enriched using `add_ground_truth_to_h5ad()`.
See [Ground truth and evaluation](ground-truth.md) for examples and complete alignment rules.

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

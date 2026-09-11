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

output:
  path: dataset_features.h5ad
  dtype: float64             # float32 | float64

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
| `output` | Optional output path and numerical dtype |
| `features` | Ordered list of feature requests |
| `obs` | Requested observation columns |
| `feature_policy` | Conflict and calculation-error policies |

Unknown keys are rejected.

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

## Load configuration explicitly

```python
from vibframe_anndata import load_config

cfg = load_config("config.yaml")
print(cfg)
```

Validation happens before expensive processing whenever the required information is already available. Descriptor-dependent checks are performed when a persisted catalog request is resolved.

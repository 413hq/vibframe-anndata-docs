# In-memory workflow

Use the in-memory API when the raw numerical payload fits comfortably in RAM.

## 1. Import raw VibFrame data

```python
from vibframe_anndata import import_raw

adata = import_raw(
    "dataset.vibframe.zip",
    config={
        "version": 1,
        "raw_import": {
            "waveforms": True,
            "spectra": True,
            "on_missing_signal": "nan",
        },
    },
)
```

The base object may intentionally have zero features:

```python
print(adata.shape)  # (n_snapshots, 0)
```

Raw signals are already preserved in `obsm`, while snapshot metadata and provenance are available through `obs` and `uns`.

## Optional evaluation metadata

Use `ground_truth={"enabled": True, "scope": "all"}` inside the import configuration to retain
complete annotations. `obsm['ground_truth']` exposes declared snapshot labels, and
`obsm['waveform_ground_truth']` exposes channel-bound waveform records. Original DiagGT files and
context are retained in `uns['vibframe_evaluation']`. Public accessors accept this AnnData directly:

```python
from vibframe_anndata import list_evaluation_files
print(list_evaluation_files(adata))
```

The archive and aligned labels require metadata memory in addition to the raw signals. Evaluation
never enters features implicitly; original byte archives are dataset-wide even after observation
slicing. See [Ground truth and evaluation](ground-truth.md).

## 2. Add features

```python
from vibframe_anndata import add_features

adata = add_features(
    adata,
    {
        "version": 1,
        "feature_policy": {
            "on_conflict": "error",
            "on_error": "nan",
        },
        "features": [
            {"name": "rms", "source": "waveform"},
            {"name": "kurtosis", "source": "waveform"},
        ],
    },
)
```

Each materialized feature becomes one column in `X` and one row in `var`.

## 3. Recalculate features

```python
from vibframe_anndata import recalculate_features

adata = recalculate_features(
    adata,
    {
        "version": 1,
        "features": [
            {"name": "rms", "source": "waveform"},
        ],
    },
)
```

Recalculation uses replacement semantics for the requested logical features.

## 4. Remove features

```python
from vibframe_anndata import remove_features

adata = remove_features(adata, ["kurtosis"])
```

Removing a feature changes `X` / `var` but does not remove raw waveform or spectrum data.

## 5. Persist the result

```python
from vibframe_anndata import write_h5ad

write_h5ad(adata, "dataset_features.h5ad")
```

## Convenience conversion

For small datasets, `convert(...)` chains import, configured feature calculation and optional writing:

```python
from vibframe_anndata import convert

adata = convert(
    "dataset.vibframe.zip",
    config={
        "version": 1,
        "features": [{"name": "rms", "source": "waveform"}],
        "output": {"path": "dataset_features.h5ad"},
    },
)
```

For datasets of uncertain size, prefer the explicit out-of-core path instead of `convert(...)`.

## Validate the contract

```python
from vibframe_anndata import validate_anndata_contract

validate_anndata_contract(adata)
```

Validation checks alignment and provenance invariants rather than numerical accuracy of every feature value.

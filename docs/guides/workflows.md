# Choosing a workflow

The package exposes two execution modes with the same logical output contract.

## Use the in-memory API when

- the present waveform and spectrum payload fits comfortably in RAM;
- you want an ordinary `AnnData` object for interactive work;
- notebooks or exploratory analysis are the primary use case;
- you do not need to process multi-GiB raw datasets on constrained hardware.

Typical path:

```text
import_raw()
   ↓
AnnData in memory
   ↓
add_features() / recalculate_features() / remove_features()
   ↓
write_h5ad()
```

## Use the out-of-core API when

- the raw VibFrame payload is larger than available RAM;
- you want to create a base `.h5ad` directly on disk;
- feature engineering must operate in bounded observation blocks;
- preserving the source H5AD while creating a new enriched copy is useful.

Typical path:

```text
import_raw_to_h5ad()
   ↓
base H5AD on disk
   ↓
add_features_to_h5ad()
   ↓
recalculate_features_to_h5ad() / remove_features_from_h5ad()
```

## Both paths preserve the same model

Regardless of execution mode:

- `obs` represents snapshots;
- `X` represents calculated features;
- `var` describes those features;
- raw waveforms and spectra remain available through `obsm`;
- provenance and configuration live under `uns['vibframe_anndata']`.

The feature calculation implementation is shared between in-memory and out-of-core processing.

## Memory sizing rule

Do **not** estimate memory from the compressed `.vibframe.zip` size alone. Decompressed numerical arrays, channel alignment structures and Python/AnnData overhead can make the resident representation significantly larger.

For large or uncertain datasets, prefer `import_raw_to_h5ad()` from the start.

## Missing asynchronous signals

Vibration acquisition schedules may differ by signal family. For example, spectra may exist at many more timestamps than waveforms. With:

```yaml
raw_import:
  on_missing_signal: nan
```

the package retains the union snapshot axis and represents unavailable raw signals structurally rather than allocating fake sample vectors. Derived features can then become `NaN` when `feature_policy.on_error: nan` is enabled.

## Transactional behavior

Large-file operations use temporary/partial files and promote the requested output only after validation. This avoids leaving an apparently valid final H5AD after an interrupted calculation.

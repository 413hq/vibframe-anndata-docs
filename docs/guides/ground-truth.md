# Complete ground truth and evaluation metadata

## Data contract

Ground truth remains **opt-in** and never enters `X`, feature computation or `obs` implicitly.
The native AnnData orientation remains snapshots × features. Version 0.3.0 adds an independent,
lossless evaluation archive and explicit per-waveform annotations to the existing snapshot view.

| Location | Meaning |
|---|---|
| `obsm['ground_truth']` | Snapshot labels in the exact `obs_names` order, when a snapshot table exists |
| `obsm['waveform_ground_truth']` | One `records_json` cell per snapshot; zero or more waveform records with channel bindings |
| `uns['vibframe_evaluation']` | Original sidecar byte buffers and a source/path/SHA-256 manifest |
| `uns['vibframe_anndata']['ground_truth']` | Snapshot alignment diagnostics and provenance |
| `uns['vibframe_anndata']['waveform_ground_truth']` | Waveform alignment diagnostics and provenance |
| `obsm['raw_waveforms_capture_t']` / `raw_spectra_capture_t` | Actual capture `t` in integer microseconds UTC |
| Corresponding `*_capture_t_known` matrices | True for explicitly supplied `t`; false for legacy fallback/missing captures |

The original archive uses `vibframe-evaluation-bytes/1`; aligned waveform records use
`snapshot-aligned-json-records/1`. Numeric capture times use the same channel columns as
`raw_*_lengths`. An absent capture has length zero and a false known flag. Do not interpret its
integer sentinel as a date. A present legacy capture without `t` falls back to `snap_t`, with
`known == False`.

## What is preserved

All regular files recursively below **`evaluation/`, `ground-truth/`, `ground_truth/`** are
preserved, not only a hard-coded subset of columns. Root `.json`, `.yaml`, `.yml` documents and
`machine=*/*.json` context are preserved too. This covers construction truth, waveform duration,
offset, quantization and hashes; machine parameters; paired controls; scenario/spectral
configuration; normative `*.diaggt.json`; `observations.parquet`,
`observations_consolidated.parquet`, `findings.parquet`; and materialization manifests.

Original bytes retain nulls, nested Arrow structures, large integer values, JSON number spelling,
unknown future fields, schema metadata and file hashes. Convenience aligned views may stringify
nested/object columns for H5AD compatibility. Use `read_evaluation_table` for the original Arrow
schema or `read_evaluation_file` for exact bytes. Files are never executed. Symlinks, unsafe paths
and unfinished `.inprogress`, `.partial`, `.tmp` files are rejected or excluded as documented.

This is not a second raw-signal archive. Raw `waves.parquet`, `spectra.parquet` and
`trends.parquet` outside these annotation roots are not copied into it. Trend values remain a
numerical-regression oracle, never a source of calculated features.

## Import a complete dataset

```python
from vibframe_anndata import import_raw_to_h5ad

base = import_raw_to_h5ad(
    "DRMHB-compact.vibframe.zip",
    "DRMHB-raw.h5ad",
    config={
        "version": 1,
        "raw_import": {"on_missing_signal": "nan"},
        "ground_truth": {
            "enabled": True,
            "scope": "all",
            "on_missing": "error",
            "max_sidecar_mib": 512,
        },
        "output": {"dtype": "float32"},
    },
    block_size_mib=8,
)
```

The same configuration works with `import_raw()` when signals and annotations fit comfortably
in memory. `scope: snapshot` selects the previous snapshot-only footprint. With `enabled: false`
(the default), no evaluation archive or aligned labels are imported.

`all` supports a source that has DiagGT but no construction snapshot table: its diagnostic files
are preserved without fabricating `obsm['ground_truth']`. A mixture of sources may expose different
truth families. If a source declares a snapshot/waveform table, strict mode checks its coverage;
it does not require an unrelated source to provide a table it never declared.

## Exact alignment rules

Snapshots are joined by **`(source, machine_id, snap_t)`**, never by row order. Integer times are
Unix-epoch microseconds UTC. Fractional numeric timestamps, ambiguous floating-point integers,
null identities and conflicting `timestamp`/`snap_t` fields fail. Datetime-typed values are
explicitly normalized to microseconds. No seconds/milliseconds heuristic is applied.

Waveforms additionally match `point_id` and any supplied `proc_mode_id`, `mode_definition_id` or
`config_id` against the actual raw channels present at that snapshot. Omitted mode identifiers
are accepted only when the match is unambiguous. Two annotations cannot bind the same raw capture.
**Capture `t` is not `snap_t`:** a cropped waveform starts later than its parent snapshot.

`on_missing: error` rejects missing declared labels or unresolved/ambiguous waveform bindings.
`ignore` retains an explicit unresolved status; it does not guess the channel. A spectra-only
import may still retain waveform annotations as unbound because the waveforms were deliberately
not imported. Extra annotation rows remain in the original archive and alignment diagnostics.

Same-named source datasets are disambiguated with hashes of resolved input paths. Record the
source paths/set for reproducibility and for retrofitting. A repeated identical source path fails.

## Read labels without loading multi-GiB waveforms

```python
from vibframe_anndata import (
    get_snapshot_ground_truth, get_waveform_ground_truth,
    list_evaluation_files, read_evaluation_table, read_evaluation_json,
)

inventory = list_evaluation_files("DRMHB-raw.h5ad")
snapshot = get_snapshot_ground_truth("DRMHB-raw.h5ad")
waveform = get_waveform_ground_truth(
    "DRMHB-raw.h5ad", point_id="pump_DE_H", proc_mode_id="ACC_10K",
)
# Inspect the inventory: choose the exact source/path present in your dataset.
entry = inventory.loc[inventory["path"].str.endswith("observations.parquet")].iloc[0]
observations = read_evaluation_table(
    "DRMHB-raw.h5ad", entry["path"], source=entry["source"],
)
```

The accessors also accept an in-memory AnnData. Passing a file path opens only annotation and
manifest groups, not raw waveform/spectrum arrays. `get_waveform_ground_truth` returns a convenient
long DataFrame with `vfta_snapshot_id`, `vfta_source`, `vfta_channel_id`, `vfta_proc_mode_id`,
`vfta_alignment_status` and `vfta_path`. Original fields are kept alongside these binding fields.
If the source itself uses a reserved `vfta_*` name, the convenience view fails instead of
silently overwriting it; use the original table accessor.

`read_evaluation_file(data, path, source=...)` returns byte-exact content and verifies SHA-256 by
default. `read_evaluation_json` parses a selected JSON document. A path present in several sources
requires an explicit `source`. `export_evaluation_files(data, destination)` writes the original
files into source-hash subdirectories and refuses overwrites.

## Upgrade an existing 0.2.x H5AD

```python
from vibframe_anndata import add_ground_truth_to_h5ad

add_ground_truth_to_h5ad(
    "DRMHB-features.h5ad",
    "DRMHB-compact.vibframe.zip",
    output="DRMHB-features-complete.h5ad",
)
```

This operation reads source metadata/annotations, not source raw sample buffers. It copies the
existing H5AD transactionally on disk, retains `X`, `var` and raw arrays, adds the evaluation
archive/projections, validates, then promotes the output. `output=None` atomically replaces the
input. Additional temporary disk space is approximately one H5AD copy. Keep `obs.source`,
`obs.machine` and `obs.snap_t`: the operation fails if those identities were removed or if sources
cannot be matched. It does not retrospectively recover capture-time companion matrices on legacy
H5AD files; those are populated by new raw imports. Snapshot-only refresh preserves an existing
complete archive and waveform projection.

## Features, slicing and experimental controls

Adding, recalculating or removing features preserves ground truth. Out-of-core feature edits do
not decode embedded sidecar buffers. Aligned `obsm` rows follow AnnData slicing, whereas the
original archive is deliberately **dataset-wide** and remains unchanged after slicing. Exporting
it exports the original campaign, not a fabricated per-subset DiagGT manifest.

Controls are not silently merged into the longitudinal benchmark. Keep separate source files or
explicitly select their source/pair metadata. Operational-state oracle labels must be copied to a
model input only under a declared experimental protocol. Severity, physical-state labels, fault
identities, seeds, OOD flags and future labels are not automatic features.

## Preservation versus interpretation

Complete retention does not assign arbitrary DiagGT diagnoses or time intervals to every snapshot.
For example, consolidated diagnoses have their own validity/deduplication conventions. Researchers
must use the original document, table and `materialization.json` semantics when deriving a target;
the package does not substitute nearest-timestamp matching or assume a missing diagnosis means
healthy. Snapshot/waveform construction tables have explicit supported alignment rules above.

The reader preserves existing annotations; it does not certify the generator's physical validity,
validate all future DiagGT schemas, or infer missing labels. Invalid known alignment is reported;
unknown sidecars remain accessible losslessly rather than being dropped.

## Storage and memory

No VibFrame ZIP or raw-signal compression setting is modified. The H5AD grows by the sum of
original sidecar byte sizes plus the manifest, aligned label views and capture-time matrices.
`list_evaluation_files` reports exact archived sizes per source. The default 512 MiB limit covers
encoded original sidecars and can be raised explicitly; it is **not** a total process-RAM limit.
Decoded Parquet/JSON tables and aligned labels consume additional metadata memory. Raw streaming
remains blockwise. A single accessor call materializes its selected sidecar/table; large annotation
archives should be accessed one table at a time. In-memory AnnData naturally materializes its
archive as well as its raw signals. No compression benchmark is repeated by this release.

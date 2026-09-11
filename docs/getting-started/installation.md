# Installation

## Requirements

`vibframe-anndata` requires Python 3.10 or newer.

A clean virtual environment is recommended:

=== "venv"

    ```bash
    python -m venv .venv
    source .venv/bin/activate
    python -m pip install --upgrade pip
    python -m pip install vibframe-anndata
    ```

=== "uv"

    ```bash
    uv venv --python 3.12
    source .venv/bin/activate
    uv pip install vibframe-anndata
    ```

On Windows PowerShell, activate a `venv` with:

```powershell
.venv\Scripts\Activate.ps1
```

## Verify the installation

```bash
python - <<'PY'
import vibframe_anndata as vfta

print(vfta.__version__)
print(vfta.import_raw)
print(vfta.import_raw_to_h5ad)
PY
```

For release 0.1.0 the first line should print:

```text
0.1.0
```

## Core dependencies

The package builds on established scientific Python components including AnnData, Awkward Array, NumPy, pandas, PyArrow, SciPy and PyYAML.

No compiler or native extension build is required for the package itself.

## Pinning for reproducible research

For a paper, notebook archive or experiment that must remain reproducible, pin the package version explicitly:

```bash
python -m pip install "vibframe-anndata==0.1.0"
```

Then use the `0.1.0` documentation version rather than `latest`.

## Upgrading

```bash
python -m pip install --upgrade vibframe-anndata
```

Before upgrading an existing analysis pipeline, check the versioned changelog and known limitations. H5AD files preserve package provenance so the producer version can be inspected later.

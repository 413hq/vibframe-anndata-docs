# vibframe-anndata documentation

Public, versioned technical documentation for the [`vibframe-anndata`](https://pypi.org/project/vibframe-anndata/) Python package.

The package converts VibFrame vibration data into reusable `AnnData` / `.h5ad` datasets and supports incremental feature engineering without reopening the original VibFrame source.

## Documentation site

The site is built with MkDocs Material and versioned with Mike. Each published package release receives its own immutable documentation snapshot, while `latest` points to the newest release.

Expected site URL:

`https://413hq.github.io/vibframe-anndata-docs/`

## Local preview

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
python -m pip install vibframe-anndata
mkdocs serve
```

## Release synchronization

`.github/workflows/publish-docs.yml` can be run manually, via `repository_dispatch`, and on a schedule. It resolves a package version from PyPI, installs that exact version, builds the documentation against it and publishes a versioned snapshot to the `gh-pages` branch.

Documentation source is public here; the application repository does not need to be shared with package users.

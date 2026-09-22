# vibframe-anndata documentation

Public, versioned technical documentation for the [`vibframe-anndata`](https://pypi.org/project/vibframe-anndata/) Python package.

The package converts VibFrame vibration data into reusable `AnnData` / `.h5ad` datasets and supports incremental feature engineering without reopening the original VibFrame source.

## Documentation site

The site is built with MkDocs Material and versioned with Mike. Each published package release receives its own immutable documentation snapshot, while `latest` points to the newest release.

Site URL:

`https://413hq.github.io/vibframe-anndata-docs/`

## One-time GitHub Pages setup

The generated site is already published to the `gh-pages` branch. GitHub Pages only needs to be enabled once for this repository:

1. Open **Settings → Pages**.
2. Under **Build and deployment**, choose **Deploy from a branch**.
3. Select branch **`gh-pages`** and folder **`/(root)`**.
4. Save.

Equivalent GitHub CLI call for an administrator:

```bash
gh api --method POST repos/413hq/vibframe-anndata-docs/pages --input - <<'JSON'
{"build_type":"legacy","source":{"branch":"gh-pages","path":"/"}}
JSON
```

If Pages was already configured, use the repository settings rather than recreating the site.

## Local preview

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
python -m pip install vibframe-anndata
mkdocs serve
```

## Release synchronization

`.github/workflows/publish-docs.yml` runs when documentation changes land on `main`, checks PyPI hourly as a fallback, can be run manually, and accepts a `repository_dispatch` event for package-release integration. When it finds a package version that has not yet been documented it:

1. installs that exact `vibframe-anndata` release from PyPI;
2. builds the API reference against the installed package;
3. snapshots the current public guides under that semantic version;
4. moves the `latest` alias to the new version;
5. publishes the generated site to `gh-pages`.

`docs-version.txt` pins the package version represented by the source guides. Validation installs
that exact PyPI release. The scheduled publisher refuses to snapshot a different release from
stale guides, so new versions are published only after their documentation source is updated.

Existing numbered documentation snapshots are preserved unless a maintainer explicitly runs the workflow with `force=true`.

The push trigger means the normal release workflow is now: publish the package, update/merge the public documentation source, and let that `main` update publish the new versioned snapshot automatically. The hourly schedule remains a safety net if no explicit documentation-source update occurs.

`.github/workflows/validate-docs.yml` builds the site with `mkdocs build --strict` for every documentation change, without modifying released snapshots.

Documentation source is public here; the private development repository does not need to be shared with package users.

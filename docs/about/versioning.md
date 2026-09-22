# Documentation versioning

The documentation site is versioned independently from the repository branch layout.

## Numbered versions

Each published package release gets a documentation snapshot with the same semantic version:

```text
0.1.0
0.2.0
0.3.0
...
```

Use a numbered version when reproducing an analysis or when your environment pins a specific package release.

## `latest`

`latest` is an alias that points to the newest published documentation version. It is appropriate for new projects that install the current package release.

## How publishing works

The public documentation repository uses MkDocs Material and Mike. A documentation publication run:

1. resolves a package version from PyPI (or receives one explicitly);
2. installs that exact `vibframe-anndata` version;
3. builds the API reference against the installed package;
4. snapshots the current public guides under the matching documentation version;
5. updates the `latest` alias;
6. pushes the built versioned site to the `gh-pages` branch.

The source marker `docs-version.txt` pins the exact package documented by the guides and API.
The workflow checks this marker against the resolved package version and does not publish a new
release from stale guides. Update the public source and marker after the package release is ready;
the main-branch push publishes its versioned snapshot. Scheduled synchronization is a fallback
only when source and package versions agree.

Existing numbered snapshots are treated as immutable by default. A maintainer can explicitly republish a version with `force=true` when correcting documentation for a just-released package without changing the package artifact itself.

## Reproducible research

If your environment contains:

```text
vibframe-anndata==0.2.0
```

select documentation version `0.2.0`, not `latest`, when you need the documentation to remain pinned to that package version.

The package also stores producer/version provenance inside generated AnnData/H5AD artifacts, which can help identify the documentation version relevant to an existing dataset.

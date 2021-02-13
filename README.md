# Ligato Docs [![Documentation Status](https://readthedocs.org/projects/ligato/badge/?version=latest)](https://docs.ligato.io/en/latest/?badge=latest)

The Ligato Docs repository contains the documentation source text and figures for the Ligato project. You can locate the documentation website at [docs.ligato.io](https://docs.ligato.io/).

The documentation is built, versioned and hosted with the help of [ReadTheDocs.org](https://readthedocs.org/). It uses the [MkDocs](https://www.mkdocs.org/) static generator. 

## Updating Documentation

All commited changes to this repository will be reflected to the docs website after a build process is completed.  The build process can take several minutes before the updates appear on the docs website. You can watch the build process on [ReadTheDocs Ligato Builds](https://readthedocs.org/projects/ligato/builds/).If the build fails for some reason, the changes will not appear on the docs website.

## Running Locally

You should verify changes locally before pushing to the main docs repository to ensure that everything looks okay. You can fix broken links, or other formatting issues, before you deploy to the docs website. To do so, you need to install `mkdocs` locally.

### 1. Install MkDocs

Look over [install mkdocs](https://www.mkdocs.org/#installation)

Please make sure you install version 1.0.4, otherwise some elements may not format properly.

```bash
$ mkdocs --version
mkdocs, version 1.0.4 from /usr/local/lib/python3.6/dist-packages/mkdocs (Python 3.6)
```

### 2. Clone Repository

```bash
$ git clone https://github.com/ligato/docs
``` 

```
$ cd docs
```

### 3. Start Server

To start the local server, use following command in the root directory:

```bash
$ mkdocs serve
```

The documentation site should be accessible at: `http://localhost:8000/`. The server watches for and lists any changes in a log.

Note: The mkdocs rtd-dropdown theme is used (configured inside the mkdocs.yml file). If not present, you might need to install it. Instructions [here](https://github.com/cjsheets/mkdocs-rtd-dropdown).

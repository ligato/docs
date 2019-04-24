# Ligato Docs

The Ligato Docs repository contains main source of documentation for Ligato. It's available at: [docs.ligato.io](https://docs.ligato.io/).

The documentation is build, versioned and hosted with the help of [ReadTheDocs.org](https://readthedocs.org/) and currently uses static site generator [MkDocs](https://www.mkdocs.org/). In the future, we might migrate to use [Sphinx](http://www.sphinx-doc.org/en/master/).

## Updating Documentation

Every change in the repository will be reflected on the main page after a while since there si a build process which must be completed first. The build can be watched on [ReadTheDocs Ligato Builds](https://readthedocs.org/projects/ligato/builds/) and should only take 1-2 minutes. If the build fails for some reason, the changes will not appear on the site.

## Running Locally

It can be a good thing to check changes locally before pushing to the main repository to ensure that everything looks like required. It may prevent broken links or any other formatting issues which may become live. To do so, you need to install `mkdocs` locally.

### 1. Install MkDocs

This site should help: [install mkdocs](https://www.mkdocs.org/#installation)

Please make sure to install version 1.0.4, otherwise some elements may not be formatted properly.

```bash
$ mkdocs --version
mkdocs, version 1.0.4 from /usr/local/lib/python3.6/dist-packages/mkdocs (Python 3.6)
```

### 2. Clone Repository

```bash
$ git clone https://github.com/ligato/docs
``` 

### 3. Start Server

To start server, run following command in the root directory:

```bash
$ mkdocs serve
```

The documentation site should be accessible at: http://localhost:8000/. The server also watches all changes which can be immediately seen in the browser.

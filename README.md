# Ligato VPP-Agent documentation page

Site: [ligato.readthedocs.io](https://ligato.readthedocs.io/en/latest/) (will be moved to ligato.io soon)

The documentation is build, versioned and hosted with the help of [ReadTheDoc](https://readthedocs.org/) and currently uses static site generator [MkDocs](https://www.mkdocs.org/). In the future, we plan to transfer to [Sphinx](http://www.sphinx-doc.org/en/master/).

### How to update the documentation

Every change in the repository will be reflected on the main page after a while since there si a build process which must be completed first. The build can be watched on [ReadTheDocs Ligato builds](https://readthedocs.org/projects/ligato/builds/) and should only take 1-2 minutes. If the build fails for some reason, the changes will not appear on the site.

It can be a good thing to check changes locally before pushing to the main repository to ensure that everything looks like required. It may prevent broken links or any other formatting issues which may become live. To do so, you need to install `mkdocs` locally.

**1. Install MkDocs**

This site should help: [install mkdocs](https://www.mkdocs.org/#installation)

Please make sure to install version 1.0.4, otherwise some elements may not be formatted properly.
```bash
$ mkdocs --version
mkdocs, version 1.0.4 from /usr/local/lib/python3.6/dist-packages/mkdocs (Python 3.6)
```

**2. Get the docs.ligato.io repository**

```bash
git clone https://github.com/ligato/docs.ligato.io
``` 

**3. Start the MkDocs server**

In the `docs.ligato.io` directory:

```bash
mkdocs serve
```

The documentation site can be accessed at `http://localhost:8000/`. The server also watches all changes which can be immediately seen in the browser.








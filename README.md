[![Website](https://img.shields.io/badge/dmtn--352-lsst.io-brightgreen.svg)](https://dmtn-352.lsst.io)
[![CI](https://github.com/lsst-dm/dmtn-352/actions/workflows/ci.yaml/badge.svg)](https://github.com/lsst-dm/dmtn-352/actions/workflows/ci.yaml)

# Enhancing the Portability of the Butler

## DMTN-352

The LSST Science Pipelines Data Butler and associated middleware, has been developed almost entirely focused on the requirements associated with processing LSST data. Our main acknowledgment of the wider community was a deliberate decision early on to make the software installable standalone from PyPI, thereby theoretically enabling other observatories to use it. This has enabled SPHEREx to adopt the Butler but some early Butler design decisions do not make it as easy as it should be.

This document will explore ways in which we an adjust the Butler and other packages to make it more extensible and reusable.

**Links:**

- Publication URL: https://dmtn-352.lsst.io
- Alternative editions: https://dmtn-352.lsst.io/v
- GitHub repository: https://github.com/lsst-dm/dmtn-352
- Build system: https://github.com/lsst-dm/dmtn-352/actions/


## Build this technical note

You can clone this repository and build the technote locally if your system has Python 3.11 or later:

```sh
git clone https://github.com/lsst-dm/dmtn-352
cd dmtn-352
make init
make html
```

Repeat the `make html` command to rebuild the technote after making changes.
If you need to delete any intermediate files for a clean build, run `make clean`.

The built technote is located at `_build/html/index.html`.

## Publishing changes to the web

This technote is published to https://dmtn-352.lsst.io whenever you push changes to the `main` branch on GitHub.
When you push changes to a another branch, a preview of the technote is published to https://dmtn-352.lsst.io/v.

## Editing this technical note

The main content of this technote is in `index.md` (a Markdown file parsed as [CommonMark/MyST](https://myst-parser.readthedocs.io/en/latest/index.html)).
Metadata and configuration is in the `technote.toml` file.
For guidance on creating content and information about specifying metadata and configuration, see the Documenteer documentation: https://documenteer.lsst.io/technotes.

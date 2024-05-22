.. image:: https://img.shields.io/badge/sitcomtn--128-lsst.io-brightgreen.svg
   :target: https://sitcomtn-128.lsst.io
.. image:: https://github.com/lsst-sitcom/sitcomtn-128/workflows/CI/badge.svg
   :target: https://github.com/lsst-sitcom/sitcomtn-128/actions/

#############################################
Unrecognized Blends in Operations Rehearsal 3
#############################################

SITCOMTN-128
============

Unrecognized blends are blended objects that are mistakenly identified as a single object, usually due to a high degree of overlap caused by ground based seeing. Using a space based catalog we can attempt to match objects between the two and identify any unrecognized blends. In this technote we use the truth catalogs as a proxy and create a simple matching algorithm between truth and observation to label recognized and unrecognized blends. We then see how the rate varies with object properties such as i-mag and local density.

**Links:**

- Publication URL: https://sitcomtn-128.lsst.io
- Alternative editions: https://sitcomtn-128.lsst.io/v
- GitHub repository: https://github.com/lsst-sitcom/sitcomtn-128
- Build system: https://github.com/lsst-sitcom/sitcomtn-128/actions/


Build this technical note
=========================

You can clone this repository and build the technote locally if your system has Python 3.11 or later:

.. code-block:: bash

   git clone https://github.com/lsst-sitcom/sitcomtn-128
   cd sitcomtn-128
   make init
   make html

Repeat the ``make html`` command to rebuild the technote after making changes.
If you need to delete any intermediate files for a clean build, run ``make clean``.

The built technote is located at ``_build/html/index.html``.

Publishing changes to the web
=============================

This technote is published to https://sitcomtn-128.lsst.io whenever you push changes to the ``main`` branch on GitHub.
When you push changes to a another branch, a preview of the technote is published to https://sitcomtn-128.lsst.io/v.

Editing this technical note
===========================

The main content of this technote is in ``index.rst`` (a reStructuredText file).
Metadata and configuration is in the ``technote.toml`` file.
For guidance on creating content and information about specifying metadata and configuration, see the Documenteer documentation: https://documenteer.lsst.io/technotes.

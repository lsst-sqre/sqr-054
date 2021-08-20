..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

We have typically used pip to install supplementary software into the
"stack" environment, and maintained a separate "system" environment.
This has grown increasingly fragile, and now that the rubinenv work has
landed in the stack containers, we can use conda and avoid a great deal
of pollution of the stack.

.. Add content here.

The Status Quo
==============

Historically, the ``conda`` tool has been unable to solve the Science
Pipelines stack environment.  This has meant that we have had no choice
but to add packages to the stack with ``pip``.  Further, it has
historically been the case that the enviroment available in the stack
has often been far behind the current state of affairs, which in past
years caused us trouble with getting JupyterLab and its requisites
working.

The New ``rubinenv`` World
==========================

However, as of late January 2021, the ``rubinenv`` conda environment has
landed and is available in weekly builds of the stack.  It does
(eventually) solve with conda.  Therefore, since the Science Pipelines
stack environment *can* be extended with conda, we *should* be doing as
much addition to it as possible with conda, reserving pip for items that
are not packaged in conda at all (or, for instance, if we temporarily
need something ahead of a released version but available on GitHub).

One environment or two?
-----------------------

There are arguments to be made that we should preserve the stack
environment intact, and instead of adding anything to it, clone the
environment and add the Jupyter and Jupyter-adjacent packages to the
clone.

This works fine.  However, because the container uses an overlay
filesystem to build additional layers, all the hardlinks that conda
installs to save space turn into copies, so adding a clone of the
scipipe-env environment increases the container size by about 60%.  This
may not be an acceptable trade-off, given that it is not at all clear
that the interactive analysis environment *should* have the pipeline
batch environment available.

Goodbye to "system" python
--------------------------

However, once we have moved to using conda for maintenance of the
stack-based environment, it really makes no sense to continue with the
same system python setup we had.

In order to make system python work at all, we needed to duplicate a
huge amount of functionality from within the stack environment.  Most
notably that included a C compiler, Cmake, a modern git, and git-lfs.

To achieve parity between the environments, we would typically do (in
the container build), something like:

``pip install <stuff>; activate stack environment; pip install <stuff>``

This obviously does not work if the "system" python uses pip and the
"stack" python uses conda.

There are also issues, such as one found on 2 February 2021, where the
"system" python is at 3.6.8; there was, as of that date, no 3.8+ Python
for Centos 7 available via EPEL.  In order to get an IPython current
enough to support current jedi, we would therefore have had to retool
the build to use Python 3.8 from an SCL.  Thus, doing nothing and
maintaining the status quo is really not an option: it will require at
least as much effort as switching to the proposed solution below.

The Proposal
============

The obvious solution is to get rid of the "system" python.  At present,
all of the additional components we require work quite nicely with the
Python environment within the stack.  The only actual change (rather
than supplementation) we have to do to the things packaged with the
stack is a downgrade of dask, and that is simply because dask 2021.1
does not play well with the Kubernetes cluster driver, but 2020.12
does.  Once dask addresses this (as surely they must), then we can
remove that downgrade.

What if the Rubin environment stops working?
--------------------------------------------

If we do run into an issue where something we require means that we
are in conflict with major parts of the Science Pipelines stack, then
all we need to do is go with the "two environments" solution proposed
above.  While this comes with an increase in the container size, it
gives us the flexibility to either start from a clone of the stack
environment or start from a new environment and add just what we need.
The second was essentially what we were doing in the ``pip`` world.

In any event
------------

We cannot feasibly maintain two python environments built around two
different packaging systems.  While the author is not a fan of conda,
it's what the stack uses, and the ``rubinenv`` environment actually
manages to track both conda-installed packages and those installed into
the conda environment with ``pip`` reasonably well (indeed, except for
things installed from pip via git repositories, conda itself can install
all of the pip packages, which is something we take advantage of in our
pinned builds).

User-installed software
-----------------------

Allowing user-installed software will functionally be the same as it is
in the current regime: ``pip install --user`` will continue to work, but
users will also have the option to use ``conda env create`` or
``conda env clone``; their installed packages will live under their home
directories.  This can of course create issues as users have packages
installed that are incompatible with later versions of the stack; this
is why we offer the ``clear .local`` checkbox on the JupyterHub spawner
form.

Pinned "prod" builds
====================

In the past, our "bleed" builds have been floating, using the newest
versions of packages available, while, because the stack has been pinned
to specific versions, we also produce a separate "prod" build--which is
what is used for daily, weekly, and release images.  This requires a
manual reconciliation every so often of pins from bleed into prod.

That has not been an insane amount of work, but it has been getting
harder over time, and indeed has not been possible (at least not without
significant effort) since December 2020.

The existence of ``rubinenv`` means, however, that the stack itself is
not fully pinned.  See, for instance,
https://lsstc.slack.com/archives/C4RKBLK33/p1612202947357200?thread_ts=1612202910.356800&cid=C4RKBLK33
, specifically Eli Rykoff's statement "In the new conda unpinned env, it
depends on what the solver hands us.  And that can change from day to
day as new versions are released."

If that is the case, perhaps it's time to drop the "prod" builds
entirely, and just use conda to install our additional packages, since
there does not seem to me to be any reason to be *more* zealous about
exact pins than our upstream input container.

Rebuilding Historic Images
==========================

In general, we *can* currently overlay our own packages onto older
versions of the stack.  We don't do that very often, because the time
investment to do so is nontrivial and it hasn't been requested much.  In
general, the stack and JupyterLab UI should work, but things like Dask
and Bokeh, which are fairly tightly coupled to the underlying Python,
may or may not.

In general, however, our practice has been to treat ancient images as
immutable.  This of course presents a problem in that, for instance,
Release 13 will no longer work with our modern spawning and
authentication framework.  It may be worthwhile, as we approach
operations, to determine how far back we want to support stack releases
and spend some time modernizing the containers for release builds only,
so that they all run with whatever the current launching framework at
the beginning of operations is.  Clearly this effort should not be made
for weekly or daily builds; however, this is almost self-correcting in
that daily builds are purged after about a month, and weekly builds
after a year and a half.  Only release builds persist forever.

When we adopt the move to the conda-based world, we will lose the
ability to rebuild images older than ``rubinenv`` since the conda solve
will not work.  I propose we tag the final pip-based version and use
that if at some point we have to rebuild a version from before
``rubinenv`` landed.

Note that we will not be rebuilding historic images for newly-discovered
security vulnerabilities in the stack packages.  The RSP by design
provides its users with arbitrary code execution in the Notebook Aspect,
so the rest of the infrastructure already needs to be secured against the
notebook.  Notebook environments will be run with restricted capabilities
and privileges to limit their ability to attack the hosting
infrastructure.

The scope of a security vulnerability in a historic image is therefore
mostly limited to compromising the user's notebook itself.  Given the
types of operations users are likely to perform with historic images
(reproducing old results with a fixed version of the stack, not talking to
malicious Internet sites or installing new, possibly-compromised
software), this is an acceptable security risk given the important
scientific objective of reproducibility of old results, which requires not
upgrading software that's part of the scientific stack.

Choosing a recommended
======================

The spawner page for the RSP always provides a suggestion for which image to use when starting a container.
We refer to this special image as the "Recommended Image."
When choosing a recommended image, it's important to make sure that all of the various services and example notebooks work for that particular image.
The process is described in detail `here <https://jira.lsstcorp.org/browse/DM-30240>`_.
At a high level, the steps are:

#. Select a candidate weekly build
#. Owners of any notebooks distributed with the RSP should check that they run without error to completion on all deployments where they will be supplied.
#. The primary stakeholders then all sign off.

   - Frossie as manager of the RSP
   - Gregory as product owner of the RSP
   - Leanne for CET
   - Yusra for Science Pipelines

#. During a maintenance window, e.g. "Patch Thursday", devs will ensure the proposed recommended is pulled to nodes at all deployments and advance the recommended tag to the approved weekly image.

Conclusion
==========

We don't have much choice here.  Modifying individual pins has become
fraught with danger, and the environment in the RSP is continuing to
diverge from the upstream stack.  This will only get worse.

It makes no sense to try to construct a stack-equivalent Python
environment with ``pip``; if the stack uses ``conda`` then the "system"
Python, if any, should too.

At the moment, we can install our additional packages quite cleanly onto
the stack with ``conda`` and therefore a single-environment container
built with ``conda`` is *still* much closer to the input Science
Pipelines stack environment than what we *currently* get by installing
our packages with ``pip``.

Thus, it's my contention that we should collapse everything to just
using the stack Python.  If we run into something in the future where we
need to separate the environment that runs Jupyterlab from the stack
environment, we can clone the stack environment, or build a new conda
environment from scratch, and run Jupyterlab in that environment.  Right
now that does not appear to be necessary, and the more nimble stack
environment, combined with the slowdown of churn in Jupyterlab as it has
matured, makes me hopeful that it will never be necessary.

.. Do not include the document title (it's automatically added from metadata.yaml).

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa

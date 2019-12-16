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

.. note::

   **This technote is not yet published.**

   In order to better support agility and management of third-party dependencies, reduce conflicts when compiling the stack on new and older systems, simplify distribution releases and installation of builds, we propose switching to conda-forge for managing third-party dependencies and adopting the conda compiler, the adoption of several new conda recipes, and some small changes to eups and the lsst-build system.


Switching to conda-forge for third parties and compilers
========================================================

We need to switch to conda-forge for third parties. There's multiple
reasons for this, including these significant ones:

-  New system libraries on newer versions of linux come with different
   symbols (e.g. Ubuntu 19.10)
-  Needing a pyarrow from conda-forge, which also wants a boost from
   conda-forge
-  conda default channel lags releases from conda-forge; no community
   mechanism for contributing to the default to update these - these are
   for the Anaconda distribution
-  GCC 5.1 ABI changes

Switching Compilers
-------------------

Switching to conda-forge dependencies means an end to devtoolset
compilers and system dependencies for centos7 deployments, due to ABI
breakage - centos7+devtoolset requires the pre-5.1 GCC ABI. So the
options, for centos7 for example, are to either require a user to
install system dependencies and a compiler themselves or get them from
conda-forge, or through other non-standard channels. Retrieving those
dependencies from conda-forge would be the most user friendly way.

Matthew Becker merged conda compiler support into sconsUtils, so using
conda compilers should work just fine, although they are not battle
tested. I've been testing with conda compilers on linux since July, and
things tend to work just fine with that. I had issues with Mac compilers
but I think those may be resolved now. *I will be testing this some more
in the coming weeks*

Currently, If a user is using a newer system with the native compilers,
or newer compilers on an older system, there may still be issues around
using those compilers even with the conda-forge third parties. In any
case - conda compiler or newer compiler on top of conda - **additional
testing will still be required to ensure that builds work in all cases,
including when compilers newer than those of conda are installed on the
system outside conda**.

Installing the stack and environment
====================================

eups and conda are still underpinning the use cases for both developers
and users, how those environments are installed and managed is
different. Importantly, in both cases, we rely on activation scripts
(scripts in ``activate.d``) in both cases to properly setup the terminal
when a conda environment is activated. In both cases, we also will have
``scipipe_conda_env`` transformed into an eups package, which ``base``
will require.

End users consume releases with Stackvana
-----------------------------------------

.. code:: bash

   # conda already installed and conda-forge already configured
   $ conda create -n lsst-20.1.0 stackvana=20.1.0
   # lsst_distrib already activated
   (lsst-20.1)$ eups list

Matthew Becker has created the
```stackvana`` <https://github.com/beckermr/stackvana>`__ ecosystem, a
collection of conda recipes, for building and distributing the stack.
This ecosystem is oriented around a modified ``eups distrib install``
workflow that consumes stack **releases**, *ideally within the
conda-forge CI system*. Users, in fact, get an extremely simplified
installation, as the stack is distributed similar to other conda-forge
package. The target for this use case, at least within the context of
conda-forge, is really for releases only, which probably does not
include weekly or daily builds, as these are not LSST releases, and
conda-forge CI resources are relatively precious. It's important to note
that conda-forge states that is not suitable as a target for rapid
development. As is stated in their orgnazation guidelines:

   Publishing a package to conda-forge signals it is suitable for users
   not involved with development. However, publishing does not always
   happen error-free. Multiple commits are acceptable when debugging
   issues with the release process itself.

As such, it would probably be prudent to discuss release cadence of the
stack to meet the needs of the users consuming stackvana installs, or
rely on another channel [1].

Details
~~~~~~~

   **Note**: Stackvana implementation details are subject to change. In
   particular, it's conceivable that Stackvana may utilize eups for the
   build but rely on copying files to ``$PREFIX`` during the build and
   not package eups.

The first package in the stackvana ecosystem, ``stackvana-core``,
installs conda third parties (with semantic pinning, rather than exact
build pinning as ``scipipe_conda_env`` does currently), as well as eups
(eups db and setup script to ``activate.d``), sconsUtils, and eups remap
files for the third parties, and the conda compilers - everything needed
to build the stack. Thanks to added setup scripts to the environment's
``activate.d``, when the conda environment is activated, it will also
source the eups setup script, activating eups at the same time, which
provides a simplified user experience. The other package in the
ecosystem,\ ``stackvana``, relies on ``stackvana-core`` for the build
environment, it proceeds to build the ``lsst_distrib`` tag, and also
relies on ``activate.d`` to setup that\ ``lsst_distrib`` tag on
environment activation.

Stackvana uses semantic pinning derived from exact pinning in
``scipipe_conda_env`` in it's build recipe to install dependencies. This
is a slight departure from how ``scipipe_conda_env`` has worked to date,
which uses exact pinning. Stackvana version numbers track lsst_distrib
version numbers.

Currently, it's remapping many of the dependencies. As those
dependencies move to conda-forge and support is added to sconsUtils for
setting those up in the build environment, those remaps should mostly go
away.

Some splitting up some of lsst_distrib into individual stackvana
components (e.g. ``stackvana-afw``) may be required to keep compliation
times down fore the conda-forge CI system, although we are able to
finish the build within the current limits.

Developers start with a toolset environment
-------------------------------------------

**Installing a daily build:**

.. code:: bash

   # conda installed and conda-forge already configured, conda activated
   # lsst_toolset_env is metapackage - eups, lsst_build, repos, scons
   (base)$ conda create -n lsst-toolset lsst_toolset_env
   (base)$ conda activate lsst-toolset
   # If the previous command is the first activation, activate.d scripts
   # could run `lsst-build config init`, which can setup repos/versiondb
   (lsst-toolset)$ eups distrib install -t d_latest lsst_distrib
   # A new conda environment was created as part of installing
   # scipipe_conda_env.
   (lsst-toolset)$ setup lsst_distrib
   (scipipe-conda-env-1234abcde)$ 

**Development**:

.. code:: bash

   (base)$ conda create -n lsst-toolset lsst_toolset_env
   # lsst_toolset_env's activate.d scripts executed
   (base)$ conda activate lsst-toolset
   (lsst-toolset)$ cd ~/workspace/lsstsw
   (lsst-toolset)$ rebuild afw
   # internally - rebuild uses lsst-build to `prepare` and `build`
   # scipipe_conda_env is prepped, config, installed, declared first
   # Following that is the lsst-build build script for base, like so:
   # > (lsst-toolset)$ eupspkg PRODUCT=base ... prep
   # > # Note: scipipe_conda_env is setup via _build.tags
   # > (lsst-toolset)$ setup --vro=_build.tags -r .
   # > (scipipe-conda-env-1234abcde)$ eupspkg PRODUCT=base ... config
   # ... the rest of lsst-build's build script for base is executed

This workflow is optimized around both eups and lsst-build workflows.
This includes tasks such as the installation of weekly or daily builds
via ``eups distrib install``, and local development via lsstsw and
``rebuild``.

Developers start with a toolset environment that packages eups,
lsst-build, git, git-lfs, compilers, and scripts to configure data for
lsst/repos and lsst/versiondb. The installed toolset *environment* also
contains the eups database, with the installed eups managing conda
environments with the help of `environment
stacking. <#conda-environment-stacking>`__ The conda environment is an
eups package based on Nate Lust's ``scipipe_conda`` package and
``scipipe_conda_env`` repo. Nate's code originally went one step further
in relying on another ``miniconda`` eups package, so conda itself was
installed with eups. I believe we should not do that because we still
need a suitable python environment for eups itself, and for path length
reasons explained in detail in [2]. One thing to note that running
``setup -r .`` on the directory will not activate a complete environment
- the environment is always installed as part of the install phase of
eupspkg.

Nate's code had relied on manipulating environment variables in the
table file as conda does, by setting ``CONDA_PREFIX``, appending paths,
etc... However, when conda is activated, ``conda`` is actually a shell
function intended to be a wrapper over the actual conda CLI, modifying
the environment as necessary when switching between conda environments
or installing dependencies, as ``setup`` does as well, so this breaks or
may have some funny side effects (bad ``$PS1``) when not directly using
``conda activate`` or ``conda deactivate``. Notably, if conda is not
activated but the paths have been modified so that you can find the
conda binary - running a successive ``conda activate`` will fail,
notifying you with an error:

   ``CommandNotFoundError: Your shell has not been properly configured to use 'conda activate'.``

To address this, we can add new actions for an eups table file:

-  ``activateCondaEnv(...)``
-  ``activateCondaEnvStacked(...)``

This actions will emit ``conda activate {env_name}`` and
``conda deactivate`` statements which get evaluated by running
``setup``. The assumption here is that conda has been activated, which
it should have been if you can use eups. The second action,
``activateCondaEnvStacked`` is the one we actually need, but I'm
proposing adding both to eups, since the semantics are slightly
different.

The lsst_toolset_env package will be derived from the stackvana-core
build recipe, which packages up eups.

With scipipe_conda_env as an eups package, stackvana will possibly need
an eups remap file for that package.

Relatedly, in both the stackvana case and the developer case, there is
one important caveat to note by having ``base`` depend on
``scipipe_conda_env``. If a product, such as ``qserv`` or sims, wishes
to define it's own environment but relies a dependency which relies on
``base``, they will likely need to either remap ``scipipe_conda_env``
with their own environment, or produce their own version of
``scipipe_conda_env`` and rely on version restrictions in their table
file on ``scipipe_conda_env``. There may be some other solution that
hasn't been identified. This isn't any different than the current
situation with ``scipipe_conda_env``.

Conda Environment Stacking
~~~~~~~~~~~~~~~~~~~~~~~~~~

`Conda has an environment stacking
feature <https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html#nested-activation>`__.
We will use this to overlay the ``scipipe_conda_env`` environments over
the toolset environment, which allows us keep eups, lsst-build, and even
compilers after activating those environments.

Stacking can be used to improve lsst-build when executing a build in a
different conda environment, which is a CI and ``rebuild`` use case,
similar to how conda-build works I believe. Here is an example script
executing commands in different stacked environments:

.. code:: bash

   #!/bin/bash
   # re-init conda functions (conda/conda#7753)
   CONDA_EXE_ROOT=$(dirname $(dirname $CONDA_EXE))
   source $CONDA_EXE_ROOT/etc/profile.d/conda.sh
   # The following will pick up the system git
   echo "git in scipipe-env environment (conda run isolated environment)..."
   conda run -n scipipe-env git --version
   # The following will pick up git from the lsst-toolset environment
   echo "git from scipipe-env environment (stacked environments)..."
   conda activate --stack scipipe-env
   git --version

With that ``stacking.sh`` example, we can execute it:

.. code:: bash

   $ git --version
   git version 2.20.1 (Apple Git-117)
   $ source /opt/conda/bin/activate lsst-toolset
   (lsst-toolset)$ ./stacking.sh
   Stacking example
   git in scipipe-env environment (conda run isolated environment)...
   git version 2.20.1 (Apple Git-117)
   git from scipipe-env environment (stacked environments)...
   git version 2.23.0

.. _eups-lsst-build-lsstsw-newinstall-repos-versiondb-sconsutils-loadlsstbash:

eups, lsst-build, lsstsw, newinstall, repos, versiondb, sconsUtils, loadLSST.bash
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

By starting with conda environment and leveraging ``activate.d``
scripts, I think it's possible to encapsulate the execution of
lsstsw/bin/deploy, newinstall.sh, and loadLSST.bash so that parts of
those scripts are ran at environment activation time. The interface is
``conda activate``.

Going a step further, moving eups-related commands in lsstsw into
lsst-build (``rebuild``, ``mass-tag``) is desirable, as well as tools to
manage lsst/repos and lsst/versiondb, along with config files for
lsst-build itself. By delineating configuration commands for lsst-build,
with initialization and update actions to manage build-related data like
lsst/repos and lsst/versiondb, we reduce the complexity down to one
tool, lsst-build.

.. code:: bash

   # write new config file. Stored at $CONDA_PREFIX/etc/lsst-build/
   (lsst-toolset)$ lsst-build config init 
   # sync config - download lsst/repos/etc/repos.yaml, lsst/versiondb
   (lsst-toolset)$ lsst-build config sync
   # Switch repos. Equivalent to:
   # cd $CONDA_PREFIX/share/lsst/repos; git checkout tickets/DM-98765
   (lsst-toolset)$ lsst-build config write repos-ref tickets/DM-98765
   (lsst-toolset)$ lsst-build config sync

Should this not be desirable for some reason, the user is always free to
setup a new toolset environment.

Other considerations
~~~~~~~~~~~~~~~~~~~~

From an lsstsw/lsst-build standpoint, the stacked environment is a bit
kinder with CI than the stackvana approach, due to replication of large
product repos (e.g. git-lfs+afwdata).

Conclusion
~~~~~~~~~~

In conclusion, these are some thoughts and directions we are going.
Stackvana can drastically simplify user installation in the case of
consuming official releases. The toolset approach can provide developers
flexibility they need with eups and play nicely with CI. We must support
both.

Footnotes
^^^^^^^^^

**[1] Beyond conda-forge**

There's a few different avenues which could be investigated to improve
the experience of compiling, CI, and integration with conda-forge
further.

-  If the 2-4 releases per year of the stack is not adequate for the
   user base consuming stackvana, investigate more frequent releases.

   -  Alternatively, produce conda-forge compatible monthly or weekly
      stackvana *builds* to an lsst conda channel

-  Reduce proliferation repos to several core product repos and apply
   semantic versioning on it

   -  Aligns closer to product tree

   -  Map product repos to conda-forge packages with semantic versions

   -  Makes Stackvana work better with limited conda-forge CI resources

-  Investigate replacing lsst-build+scons+sconsUtils+eups with bazel

   -  Optimized for monorepo projects, but has support for external git
      repos

   -  Support for multiple langauges out of the box (see Tensorflow).

   -  Several projects to support remote build caching and build
      clusters. Shared build cache works nicely with ephemeral
      workers/cloud

   -  Shared build cache could be backed by nginx WebDAV, which we
      deploy on the LSP

   -  No built-in pytest support

-  Investigate replacing lsst-build+scons+sconsUtils+eups with CMake +
   ExternalProject

   -  More standardized than Bazel

   -  No native remote build cache. We could keep the shared filesystem
      way, but that might not help users outside of Jenkins/CI

   -  No built-in pytest support

**[2] Issues with installing miniconda with eups**

One issue around using conda as an eups dependency is linux path length.
With eups, the executables can be buried down a bit. To illustrate this,
assume eups is installed under a ``pipelines`` directory on CVMFS, and a
user is testing a new version of miniconda from a tickets branch. We
assume that the activated conda environment was named
``scipipe_conda_env``, which may be smaller than the normal path length.
We would likely end up with a path close of 129 characters:

``#!/cvmfs/sw.lsst.eu/pipelines/eups/Linux64/miniconda/tickets.DM-29999-g123456789+0123456789/envs/scipipe_conda_env/bin/python3.7``

Because of this, it's my recommendation to not package miniconda with
eups, as the path length and rely on the user to setup conda before
activating eups. I would recommend we tell users which conda version is
preferred and forward them to instructions on where to acquire conda.

   
.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa

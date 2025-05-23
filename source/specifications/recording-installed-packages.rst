.. highlight:: text

.. _recording-installed-packages:

============================
Recording installed projects
============================

This document specifies a common format of recording information
about Python :term:`projects <Project>` installed in an environment.
A common metadata format allows tools to query, manage or uninstall projects,
regardless of how they were installed.

Almost all information is optional.
This allows tools outside the Python ecosystem, such as Linux package managers,
to integrate with Python tooling as much as possible.
For example, even if an installer cannot easily provide a list of installed
files in a format specific to Python tooling, it should still record the name
and version of the installed project.


The .dist-info directory
========================

Each project installed from a distribution must, in addition to files,
install a "``.dist-info``" directory located alongside importable modules and
packages (commonly, the ``site-packages`` directory).

This directory is named as ``{name}-{version}.dist-info``, with ``name`` and
``version`` fields corresponding to :ref:`core-metadata`. Both fields must be
normalized (see the :ref:`name normalization specification <name-normalization>`
and the :ref:`version normalization specification <version-specifiers-normalization>`),
and replace dash (``-``) characters with underscore (``_``) characters,
so the ``.dist-info`` directory always has exactly one dash (``-``) character in
its stem, separating the ``name`` and ``version`` fields.

Historically, tools have failed to replace dot characters or normalize case in
the ``name`` field, or not perform normalization in the ``version`` field.
Tools consuming ``.dist-info`` directories should expect those fields to be
unnormalized, and treat them as equivalent to their normalized counterparts.
New tools that write ``.dist-info`` directories MUST normalize both ``name``
and ``version`` fields using the rules described above, and existing tools are
encouraged to start normalizing those fields.

.. note::

    The ``.dist-info`` directory's name is formatted to unambiguously represent
    a distribution as a filesystem path. Tools presenting a distribution name
    to a user should avoid using the normalized name, and instead present the
    specified name (when needed prior to resolution to an installed package),
    or read the respective fields in Core Metadata, since values listed there
    are unescaped and accurately reflect the distribution. Libraries should
    provide API for such tools to consume, so tools can have access to the
    unnormalized name when displaying distribution information.

This ``.dist-info`` directory may contain the following files, described in
detail below:

* ``METADATA``: contains project metadata
* ``RECORD``: records the list of installed files.
* ``INSTALLER``: records the name of the tool used to install the project.
* ``entry_points.txt``: see :ref:`entry-points` for details
* ``direct_url.json``: see :ref:`direct-url` for details

The ``METADATA`` file is mandatory.
All other files may be omitted at the installing tool's discretion.
Additional installer-specific files may be present.

This :file:`.dist-info/` directory may contain the following directories, described in
detail below:

* :file:`licenses/`: contains license files.
* :file:`sboms/`: contains Software Bill-of-Materials files (SBOMs).

.. note::

   The :ref:`binary-distribution-format` specification describes additional
   files that may appear in the ``.dist-info`` directory of a :term:`Wheel`.
   Such files may be copied to the ``.dist-info`` directory of an
   installed project.

The previous versions of this specification also specified a ``REQUESTED``
file. This file is now considered a tool-specific extension, but may be
standardized again in the future. See `PEP 376 <https://www.python.org/dev/peps/pep-0376/#requested>`_
for its original meaning.


The METADATA file
=================

The ``METADATA`` file contains metadata as described in the :ref:`core-metadata`
specification, version 1.1 or greater.

The ``METADATA`` file is mandatory.
If it cannot be created, or if required core metadata is not available,
installers must report an error and fail to install the project.


The RECORD file
===============

The ``RECORD`` file holds the list of installed files.
It is a CSV file containing one record (line) per installed file.

The CSV dialect must be readable with the default ``reader`` of Python's
``csv`` module:

* field delimiter: ``,`` (comma),
* quoting char: ``"`` (straight double quote),
* line terminator: either ``\r\n`` or ``\n``.

Each record is composed of three elements: the file's **path**, the **hash**
of the contents, and its **size**.

The *path* may be either absolute, or relative to the directory containing
the ``.dist-info`` directory (commonly, the ``site-packages`` directory).
On Windows, directories may be separated either by forward- or backslashes
(``/`` or ``\``).

The *hash* is either an empty string or the name of a hash algorithm from
:py:data:`hashlib.algorithms_guaranteed`, followed by the equals character ``=`` and
the digest of the file's contents, encoded with the urlsafe-base64-nopad
encoding (:py:func:`base64.urlsafe_b64encode(digest) <base64.urlsafe_b64encode()>` with trailing ``=`` removed).

The *size* is either the empty string, or file's size in bytes,
as a base 10 integer.

For any file, either or both of the *hash* and *size* fields may be left empty.
Commonly, entries for ``.pyc`` files and the ``RECORD`` file itself have empty
*hash* and *size*.
For other files, leaving the information out is discouraged, as it
prevents verifying the integrity of the installed project.

If the ``RECORD`` file is present, it must list all installed files of the
project, except ``.pyc`` files corresponding to ``.py`` files listed in
``RECORD``, which are optional.
Notably, the contents of the ``.dist-info`` directory (including the ``RECORD``
file itself) must be listed.
Directories should not be listed.

To completely uninstall a package, a tool needs to remove all
files listed in ``RECORD``, all ``.pyc`` files (of all optimization levels)
corresponding to removed ``.py`` files, and any directories emptied by
the uninstallation.

Here is an example snippet of a possible ``RECORD`` file::

    /usr/bin/black,sha256=iFlOnL32lIa-RKk-MDihcbJ37wxmRbE4xk6eVYVTTeU,220
    ../../../bin/blackd,sha256=lCadt4mcU-B67O1gkQVh7-vsKgLpx6ny1le34Jz6UVo,221
    __pycache__/black.cpython-38.pyc,,
    __pycache__/blackd.cpython-38.pyc,,
    black-19.10b0.dist-info/INSTALLER,sha256=zuuue4knoyJ-UwPPXg8fezS7VCrXJQrAP7zeNuwvFQg,4
    black-19.10b0.dist-info/licenses/LICENSE,sha256=nAQo8MO0d5hQz1vZbhGqqK_HLUqG1KNiI9erouWNbgA,1080
    black-19.10b0.dist-info/METADATA,sha256=UN40nGoVVTSpvLrTBwNsXgZdZIwoKFSrrDDHP6B7-A0,58841
    black-19.10b0.dist-info/RECORD,,
    black.py,sha256=45IF72OgNfF8WpeNHnxV2QGfbCLubV5Xjl55cI65kYs,140161
    blackd.py,sha256=JCxaK4hLkMRwVfZMj8FRpRRYC0172-juKqbN22bISLE,6672
    blib2to3/__init__.py,sha256=9_8wL9Scv8_Cs8HJyJHGvx1vwXErsuvlsAqNZLcJQR0,8
    blib2to3/__pycache__/__init__.cpython-38.pyc,,
    blib2to3/__pycache__/pygram.cpython-38.pyc,sha256=zpXgX4FHDuoeIQKO_v0sRsB-RzQFsuoKoBYvraAdoJw,1512
    blib2to3/__pycache__/pytree.cpython-38.pyc,sha256=LYLplXtG578ZjaFeoVuoX8rmxHn-BMAamCOsJMU1b9I,24910
    blib2to3/pygram.py,sha256=mXpQPqHcamFwch0RkyJsb92Wd0kUP3TW7d-u9dWhCGY,2085
    blib2to3/pytree.py,sha256=RWj3IL4U-Ljhkn4laN0C3p7IRdfvT3aIRjTV-x9hK1c,28530

If the ``RECORD`` file is missing, tools that rely on ``.dist-info`` must not
attempt to uninstall or upgrade the package.
(This restriction does not apply to tools that rely on other sources of information,
such as system package managers in Linux distros.)

.. note::

   It is *strongly discouraged* for an installed package to modify itself
   (e.g., store cache files under its namespace in ``site-packages``).
   Changes inside ``site-packages`` should be left to specialized installer
   tools such as pip. If a package is nevertheless modified in this way,
   then the ``RECORD`` must be updated, otherwise uninstalling the package
   will leave unlisted files in place (possibly resulting in a zombie
   namespace package).

The INSTALLER file
==================

If present, ``INSTALLER`` is a single-line text file naming the tool used to
install the project.
If the installer is executable from the command line, ``INSTALLER``
should contain the command name.
Otherwise, it should contain a printable ASCII string.

The file can be terminated by zero or more ASCII whitespace characters.

Here are examples of two possible ``INSTALLER`` files::

    pip

::

    MegaCorp Cloud Install-O-Matic

This value should be used for informational purposes only.
For example, if a tool is asked to uninstall a project but finds no ``RECORD``
file, it may suggest that the tool named in ``INSTALLER`` may be able to do the
uninstallation.


The entry_points.txt file
=========================

This file MAY be created by installers to indicate when packages contain
components intended for discovery and use by other code, including console
scripts and other applications that the installer has made available for
execution.

Its detailed specification is at :ref:`entry-points`.


The direct_url.json file
========================

This file MUST be created by installers when installing a distribution from a
requirement specifying a direct URL reference (including a VCS URL).

This file MUST NOT be created when installing a distribution from an other type
of requirement (i.e. name plus version specifier).

Its detailed specification is at :ref:`direct-url`.


The :file:`licenses/` subdirectory
==================================

If the metadata version is 2.4 or greater and one or more ``License-File``
fields is specified, the :file:`.dist-info/` directory MUST contain a :file:`licenses/`
subdirectory which MUST contain the files listed in the ``License-File`` fields in
the :file:`METADATA` file at their respective paths relative to the
:file:`licenses/` directory.
Any files in this directory MUST be copied from wheels by the install tools.


The :file:`sboms/` subdirectory
==================================

All files contained within the :file:`.dist-info/sboms/` directory MUST
be Software Bill-of-Materials (SBOM) files that describe software contained
within the installed package.
Any files in this directory MUST be copied from wheels by the install tools.


Intentionally preventing changes to installed packages
======================================================

In some cases (such as when needing to manage external dependencies in addition
to Python ecosystem dependencies), it is desirable for a tool that installs
packages into a Python environment to ensure that other tools are not used to
uninstall or otherwise modify that installed package, as doing so may cause
compatibility problems with the wider environment.

To achieve this, affected tools should take the following steps:

* Rename or remove the ``RECORD`` file to prevent changes via other tools (e.g.
  appending a suffix to create a non-standard ``RECORD.tool`` file if the tool
  itself needs the information, or omitting the file entirely if the package
  contents are tracked and managed via other means)
* Write an ``INSTALLER`` file indicating the name of the tool that should be used
  to manage the package (this allows ``RECORD``-aware tools to provide better
  error notices when asked to modify affected packages)

Python runtime providers may also prevent inadvertent modification of platform
provided packages by modifying the default Python package installation scheme
to use a location other than that used by platform provided packages (while also
ensuring both locations appear on the default Python import path).

In some circumstances, it may be desirable to block even installation of
additional packages via Python-specific tools. For these cases refer to
:ref:`externally-managed-environments`


History
=======

- June 2009: The original version of this specification was approved through
  :pep:`376`.  At the time, it was known as the *Database of Installed Python
  Distributions*.
- March 2020: The specification of the ``direct_url.json`` file was approved
  through :pep:`610`. It is only mentioned on this page; see :ref:`direct-url`
  for the full definition.
- September 2020: Various amendments and clarifications were approved through
  :pep:`627`.
- December 2024: The :file:`.dist-info/licenses/` directory was specified through
  :pep:`639`.

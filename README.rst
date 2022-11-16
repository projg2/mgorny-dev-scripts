==================
mgorny-dev-scripts
==================

This is a bunch of handy scripts that I use for ebuild development.
Please note that those scripts are generally very simple and have very
limited precondition checks and error handling.  Most of the time, they
terminate on any error (relying on external tools providing error
messages) and behave unpredictably e.g. when run with incorrect
arguments or in wrong directory.


Package committing helpers
==========================

copybump
--------
Sed wrapper that updates copyright notices in specified files.
Typical usage::

    # ('git diff --name-only' gives paths valid for top-level directory)
    # update all currently modified files
    copybump $(git diff --name-only)
    # update files modified by all commits since origin
    copybump $(git diff --name-only origin)

pkgcommit
---------
Dependencies: git

Commits changes, prepending package name determined from the current
directory.  Accepts same arguments as ``git commit``, handles ``-m``
argument specially.  Typical usage::

    # (in package directory)
    # commit current dir, open EDITOR for commit message
    pkgcommit -sS .
    # use provided commit message (prepends package)
    pkgcommit -sS -m 'Bump to 1.2.3' .

git-fixup
---------
Dependencies: git

Commits changes to the specified files (or current directory, if no
argument specified) as a fixup to the latest commit on those files.
Typical usage::

    # (in package directory)
    # initial commit
    pkgcommit -sS -m 'Bump to 1.2.3'
    # do some fixes
    git-fixup
    # rebase squashes fixup into parent commit
    git rebase -i -S origin


Package maintenance helpers
===========================

check-revdep
------------
Dependecies: pkgcore, portage-utils

Runs a visibility (dependency) check for all reverse dependencies
of the specified packages (or the current package, if none specified).
This can be used to verify that the cleanup of given package does not
break any revdeps (e.g. using ``<`` deps or old USE flags).  Typical
usage::

    cd foo/bar
    git rm bar-1.ebuild
    check-revdep
    pkgdev manifest
    pkgcommit -sS . -m 'Remove old'

Uses rdep_ command.  You may want to call ``rdep-fetch-cache`` if you
plan on using it multiple times.

pkgdiff
-------
Dependencies: portage

Calls ``ebuild(1)`` to extract archives for two specified ebuilds,
and then diffs the result.  Does not handle ebuilds unpacking multiple
directories into workdir or package.env PORTAGE_TMPDIR overrides.
Typical usage::

    pkgdiff foo-1.0.ebuild foo-1.1.ebuild

The data is extracted into ``${TMPDIR:-/tmp}/mgorny-dev-scripts/portage``
to avoid hitting collisions with other Portage builds.

With the ``--build-system``/``-b`` argument, it will attempt to show a diff of
only the build system files::

    pkgdiff --build-system foo-1.0.ebuild foo-1.1.ebuild

pkgbump
-------
Dependencies: portage, gentoolkit (ekeyword), git

Copies ebuild for a version bump, dropping keywords, updating Manifest
and running pkgdiff_ to compare archives.  Typical usage::

    pkgbump foo-1.0.ebuild foo-1.1.ebuild

mkpatchset
----------
Dependencies: git, scp

Create a patchset from fork-style repository, and upload it
to dev.gentoo.org.  Typical usage::

    #          tag          archive-name                upstream dgo-subdir
    mkpatchset gentoo-3.9.8 python-gentoo-patches-3.9.8 v3.9.8   python/


Package tree iteration helpers
==============================

foreach-pkg
-----------
Takes package cat/pns as arguments, and runs bash in each directory
specified.  Typical usage::

    # (inside repo)
    foreach-pkg app-foo/bar app-foo/frobnicate
    # (runs bash in app-foo/bar)
    ...
    # (runs bash in app-foo/frobnicate)
    ...

foreach-pkg-maint
-----------------
Takes maintainer e-mail as first argument, and optionally command
as the remaining arguments.  Finds all packages with maintainer present
in ``metadata.xml`` (cheap grep), and runs the specified command
in their directories.  If no command is specified, just runs bash
for further interaction.  Typical usage::

    # (inside repo)
    # runs bash in all packages maintained by foo@gentoo.org
    foreach-pkg-maint foo@gentoo.org
    # runs eshowkw in all xfce@ packages that have more than one version
    foreach-pkg-maint xfce@gentoo.org if-multiple-versions eshowkw -C |& less

llvm-foreach-pkg & llvm-foreach-pkg-rev
---------------------------------------
Runs the specified command in directories of all LLVM packages.
The non-suffixed variant iterates over them in dependency first order
(e.g. suitable for bumps), while -rev uses the reverse order
(e.g. suitable for cleanups).  Note that the command is not undergoing
bash expansions.

Typical usage::

    llvm-foreach-pkg sh -c 'x=( *14.0.0.9999* ); cp ${x} ${x/.9999}'
    git add -A
    pkgdev manifest
    llvm-foreach-pkg pkgcommit -sS . -m "Bump to 14.0.0"

    llvm-foreach-pkg sh -c 'git rm *14.0.0_rc4*'
    pkgdev manifest
    llvm-foreach-pkg-rev pkgcommit -sS . -m "Remove 14.0.0_rc4"

if-multiple-versions
--------------------
Wrapper that runs the specified command if the current directory
contains more than one ebuild file.  Live ebuilds (``*-9999.ebuild``)
are ignored.  See example above.

rdep
----
Dependencies: wget

Accepts one or more cat/pns and prints their reverse dependencies.
The data is fetched from qa-reports.g.o.  Typical usage::

    rdep app-foo/bar app-foo/frobnicate

If you plan to use it on a larger number of packages, you can prefetch
all data and have it put into ``${TMPDIR:-/tmp}/mgorny-dev-scripts``::

    rdep-fetch-cache


Bugzilla helpers
================

file-pkgcheck
-------------
Dependencies: pkgcheck, xdg-utils or exo (from xfce), perl

Run pkgcheck on specified packages, and open bug templates for each
result set.  Typical usage::

    file-pkgcheck app-foo/bar

file-rekeywordreq
-----------------
Dependencies: xdg-utils, perl

Runs a web browser with pre-filled Bugzilla template for requesting
rekeywording of the package specified as the first argument.  Typical
usage::

    file-rekeywordreq app-foo/bar

file-stablereq
--------------
Dependencies: xdg-utils, perl

Runs a web browser with pre-filled Bugzilla template for requesting
stabilization of package specified as the first argument.  Typical
usage::

    file-stablereq app-foo/bar-1.2.3

file-kernel-stablereq
---------------------
Dependencies: xdg-utils, perl

Runs a web browser with pre-filled Bugzilla template for requesting
stabilization of dist-kernel versions specified as arguments.  Typical
usage::

    file-kernel-stablereq 5.10.96 5.4.176

find-pkg-bugs
-------------
Dependencies: xdg-utils, perl

Runs a web browser with Bugzilla search for bugs referring to any
of the packages listed on command-line.  Typical usage::

    find-pkg-bugs app-bar/foo app-foo/bar


Lastriting helpers
==================

lr-file-bug
-----------
Dependencies: xdg-utils, perl

Opens a web browser with pre-filled bug template for removing a package
specified as the first argument, after 30 days.  Typical usage::

    lr-file-bug $(pkg)

lr-add-pmask
------------
Dependencies: git

Add a package.mask template entry for removal of package specified
as the first argument, optionally mentioning bug specified as the second
argument.  Typical usage::

    # without bug no
    lr-add-pmask app-foo/bar
    # with bug no
    lr-add-pmask app-foo/frobnicate 123456
    # edit package.mask afterwards
    vim profiles/package.mask

lr-commit-pmask
---------------
Dependencies: git

Attempts to determine package and bug list from package.mask entry
in ``git diff``, and commits it.  Typical usage::

    # add your package.mask entry
    vim profiles/package.mask
    # commit it
    lr-commit-pmask

lr-mail-pmask
-------------
Dependencies: git, xdg-utils, perl

Attempts to determine package and bug list from package.mask entry
in ``git diff``, and spawns e-mail client in order to send last rites
mail.  Typical usage::

    # add your package.mask entry
    vim profiles/package.mask
    # prepare mail
    lr-mail-pmask

lr-remove
---------
Dependencies: git, portage, xdg-utils or exo (from xfce), perl

Takes a package name as the first argument, and bug numbers as remaining
arguments.  Removes the specified package and commits it as lastrited
package removal.  Opens a web browser on all specified bugs + search
for package name.  Greps profiles for stale package references.  This
presumes you remove package.mask entry prior to running it.  Typical
usage::

    # find package to remove, remove its entry
    vim profiles/package.mask
    # remove the package
    lr-remove app-foo/bar 123456
    # (review the bugs, verify output for stale profile entries)
    # if additional profile entries were removed
    git commit -a --amend -S
    # if package should not be removed after all
    git reset --hard HEAD^


Stabilization helpers
=====================

stablereq-eshowkw
-----------------
Dependencies: pkgcheck, gentoolkit, pager

Find stabilization candidates and pipe them into eshowkw.  The script
accepts pkgcheck arguments.  Typical usage::

    stablereq-eshowkw 'dev-python/*'


stablereq-find-candidates
-------------------------
Dependencies: pkgcheck

Find stabilization candidates for a given maintainer. Typical usage::

    stablereq-find-candidates x11@gentoo.org


stablereq-find-pkg-bugs
-----------------------
Dependencies: pkgcheck, xdg-utils, perl

Find stabilization candidates and open a Bugzilla search in the web
browser for the relevant packages.  The script accepts pkgcheck
arguments.  Typical usage::

    stablereq-find-pkg-bugs 'dev-python/*'


stablereq-make-list
---------------------
Dependencies: pkgcheck, editor

Find stabilization candidates and pipe a list of file-stablereq calls
into an editor for editing and then running.  The script accepts
pkgcheck arguments.  Typical usage::

    stablereq-make-list 'dev-python/*'


Generic git repository helpers
==============================

git-foreach-repo
----------------
Runs the specified command in all git repositories found in current
directory and below.  Typical usage::

    git-foreach-repo git gc --prune --aggressive

git-make-empty
--------------
Dependencies: git

Creates an ``empty`` branch in the git repository that is detached from
history and contains no files, and checks it out.  The main idea is to
save space by cleanly emptying unused repositories while preserving
``.git`` directory.  Typical usage::

    git-make-empty


Package bumping helpers
=======================
Common dependencies: same as pkgbump + pkgcommit

bump-boto
---------
Bump ``dev-python/botocore``, ``dev-python/boto3`` and ``app-admin/awscli``
in lockstep.  Takes the old and new values for the last version
component (for botocore and boto3).  Typical usage::

    bump-boto 18 19

bump-kernels
------------
Bump dist-kernel packages.  Takes one or more pairs of <old-version>
and <new-version>.  Typical usage::

    bump-kernels 5.16.14 5.16.15 5.15.28 5.15.29 5.10.105 5.10.106

After the bumps, writes a diff from git origin into
``${BINPKG_DOCKER}/local.diff``.  ``BINPKG_DOCKER`` defaults to
``~/git/binpkg-docker`` and should be a checkout of binpkg-docker_ repo.
It should be used to build binary kernel packages, and then
bump-kernels-bin_ should be called.

Read `Distribution_Kernel/Bumping_kernels`_ on the wiki for more details.

.. _binpkg-docker: https://github.com/mgorny/binpkg-docker/
.. _Distribution_Kernel/Bumping_kernels: https://wiki.gentoo.org/wiki/Project:Distribution_Kernel/Bumping_kernels

bump-kernels-bin
----------------
Bump binary dist-kernel packages.  Takes one or more pairs
of <old-version> and <new-version>.  Typical usage::

    bump-kernels-bin 5.16.14 5.16.15 5.15.28 5.15.29 5.10.105 5.10.106

The package expects binary kernel .xpaks to be present in ``${BINPKG}``
subdirectories corresponding to architectures.  ``BINPKG`` defaults
to ``~/binpkg``.  The kernels are copied into ``DISTDIR``.


Patchset generation helpers
===========================
Common dependencies: same as mkpatchset_

python-patchset
---------------
Makes the ``dev-lang/python`` patchset.  Typical usage::

    python-patchset 3.10.2

Run it in `fork/cpython`_ checkout.  Remember to push the tags
afterwards.

.. _fork/cpython: https://gitweb.gentoo.org/fork/cpython.git/

pypy-patchset
---------------
Makes the ``dev-python/pypy3`` patchset.  Typical usage::

    #             branch  version
    pypy-patchset 3.9     7.3.9

Run it in `fork/pypy`_ checkout.  Note that the upstream for this
is the unofficial git mirror `mozillazg/pypy`_.  Remember to push
the tags afterwards.

.. _fork/pypy: https://gitweb.gentoo.org/fork/pypy.git/
.. _mozillazg/pypy: https://github.com/mozillazg/pypy/

llvm-patchset
-------------
Makes the ``sys-devel/llvm`` & co. patchset.  Typical usage::

    llvm-patchset 14.0.0

Run it in `fork/llvm-project`_ checkout.  Remember to push the tags
afterwards.  Read `LLVM/Releases`_ on the wiki for more details.

.. _fork/llvm-project: https://gitweb.gentoo.org/fork/llvm-project.git/
.. _LLVM/Releases: https://wiki.gentoo.org/wiki/Project:LLVM/Releases

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

pkgdiff
-------
Dependencies: portage

Calls `ebuild(1)` to extract archives for two specified ebuilds,
and then diffs the result.  Does not handle ebuilds unpacking multiple
directories into workdir or package.env PORTAGE_TMPDIR overrides.
Typical usage::

    pkgdiff foo-1.0.ebuild foo-1.1.ebuild

pkgbump
-------
Dependencies: portage, gentoolkit (ekeyword), git

Copies ebuild for a version bump, dropping keywords, updating Manifest
and running ``pkgdiff`` to compare archives.  Typical usage::

    pkgbump foo-1.0.ebuild foo-1.1.ebuild


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
all data and have it put into ``/tmp``::

    rdep-fetch-cache


Bugzilla helpers
================

file-pkgcheck
-------------
Dependencies: pkgcheck, xdg-utils or exo (from xfce), perl

Run pkgcheck on specified packages, and open bug templates for each
result set.  Typical usage::

    file-pkgcheck app-foo/bar

file-stablereq
--------------
Dependencies: xdg-utils, perl

Runs a web browser with pre-filled Bugzilla template for requesting
stabilization of package specified as the first argument.  Typical
usage::

    file-stablereq app-foo/bar-1.2.3

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

.. _git-archive:

Supporting ``git archive`` builds
---------------------------------

By default ``setuptools-git-versioning`` reads version information by running
``git`` against the project's ``.git`` directory. When the project is built
from a ``git archive`` tarball (for example, GitHub's "Download ZIP", or a
manual ``git archive HEAD -o release.tar``), no ``.git`` directory exists and
``git`` cannot be invoked.

To make ``git archive`` builds work, add a ``.git_archival.txt`` file to your
repository whose contents will be rewritten by git at archive time. The
project will read the rewritten file when building from the archive.

Setup
~~~~~

1. Create ``.git_archival.txt`` in the repository root:

   .. code-block:: text
       :caption: .git_archival.txt

       node: $Format:%H$
       describe-name: $Format:%(describe:tags=true,match=*[0-9]*)$

2. Tell git to substitute the ``$Format:...$`` placeholders by adding the
   following line to ``.gitattributes`` in the repository root (creating the
   file if it does not exist):

   .. code-block:: text
       :caption: .gitattributes

       .git_archival.txt  export-subst

3. Commit both files:

   .. code-block:: bash

       git add .git_archival.txt .gitattributes
       git commit -m "add git archive support"

When ``git archive`` runs, the placeholders are expanded into the actual
commit SHA and ``git describe`` output for the archived commit. When the
package is later built from the extracted archive,
``setuptools-git-versioning`` reads the file and resolves the version using
the configured ``template`` / ``dev_template`` / ``dirty_template``.

The same file format is used by ``setuptools-scm``, so a single
``.git_archival.txt`` works with both tools.

Optional: include branch information
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If your templates reference ``{branch}``, also add a ``ref-names`` line:

.. code-block:: text
    :caption: .git_archival.txt (with branch info)

    node: $Format:%H$
    describe-name: $Format:%(describe:tags=true,match=*[0-9]*)$
    ref-names: $Format:%D$

.. warning::

    Including ``ref-names`` causes the archive's contents to change every
    time a new ref points at the archived commit (for example, when a new
    branch is created). This breaks archive checksum stability across
    re-archivals of the same commit. Only opt in if you actually need
    ``{branch}`` substitution.

If ``ref-names`` is not present (or is present but indicates a detached
``HEAD``) and a template references ``{branch}``, the literal string
``HEAD`` is substituted - matching the output of
``git rev-parse --abbrev-ref HEAD`` in detached-HEAD state.

Priority and interaction with other schemas
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The version source is selected in this order:

1. ``PKG-INFO`` (sdist install) - wins whenever present.
2. ``.git_archival.txt`` - used when the file exists and its placeholders
   have been substituted.
3. The normal flow: ``version_callback``, ``version_file``, live ``git``
   commands, ``starting_version``.

This means ``.git_archival.txt`` only takes effect when there is no
``PKG-INFO`` (so a normal sdist install still wins) and is opportunistic in
working checkouts: a stray un-substituted file logs a warning and is
ignored, falling through to the live ``git`` flow.

Limitations
~~~~~~~~~~~

- ``tag_filter``, ``tag_formatter``, and ``sort_by`` have no effect on
  archive builds. The tag is whatever ``git describe`` chose at archive
  time.
- ``count_commits_from_version_file`` and ``version_file`` are not consulted
  in the archive flow.
- Older git versions (<2.32) do not understand the ``%(describe...)``
  placeholder. In that case the file is left with the literal text
  ``%(describe...)`` and ``setuptools-git-versioning`` will warn and fall
  back to the ``ref-names`` field for the tag (which only succeeds when
  ``HEAD`` is exactly on a tag).

Importing a Git Submodule
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

How to import commits and files from one repo's root to another's that
has it as a submodule, with the path prefix prepended, without losing
history.

.. contents::


Why not subtree
------------------------------------------------------------------------------

Why not use ``git-subtree add``? Because it uses a merge commit to do
its work, and the prefix change won't survive later rebases.  The
committed trees will end up appearing in the root, unprefixed.  Rebase
normally throws out merge commits:

  By default, a rebase will simply drop merge commits from the todo
  list, and put the rebased commits into a single, linear branch.

Supposedly, this can be rectified:

  With ``--rebase-merges``, the rebase will instead try to preserve the
  branching structure within the commits that are to be rebased, by
  recreating the merge commits. Any resolved merge conflicts or manual
  amendments in these merge commits will have to be resolved/re-applied
  manually.

This has been tried::

  cd ~/src/destrepo
  git co -b subtree
  git remote add srcrepo ~/src/srcrepo
  git fetch srcrepo
  git subtree add --prefix=path/to/prefix srcrepo master
  git rebase --rebase-merges --strategy subtree master
  git co master
  git merge --ff-only subtree

However, after a subsequent rebase, the prefix unfortunately is still
missing.  After many attempts at variations, was not able to figure out
how to do it.  In https://stackoverflow.com/questions/12858199 there are
claims that this method works, but it does not for me.


Why not read-tree
------------------------------------------------------------------------------

Also tried using ``read-tree``::

  cd ~/src/destrepo
  git co -b readtree
  git remote add srcrepo ~/src/srcrepo
  git fetch srcrepo
  git merge -s ours --no-commit --allow-unrelated-histories srcrepo/master
  git read-tree --prefix=path/to/prefix -u srcrepo/master
  git commit -m import-srcrepo

However that resulted in the same issue that the prefix is missing from
the imported files after a rebase.


Cherry-pick gets the job done
------------------------------------------------------------------------------

We can instead use a manual process to make a new history by removing
submodule paths and ``.gitmodules`` changes in the destination repo,
pre-altering the prefix point in the source repo, and then picking all
the commits therefrom

First, make backups before starting::

  cp -a ~/src/srcrepo ~/src/srcrepo.bak
  cp -a ~/src/destrepo ~/src/destrepo.bak

Enter the destination repository::

  cd ~/src/destrepo

Remove the deployed files and gitconfig for the submodule::

  git submodule deinit path/to/prefix

Remove all traces in the history::

  git filter-repo --force --partial --invert-paths --path path/to/prefix

Find the first commit that adds a reference to path/to/prefix.  Keep
around to note all the commits after it that will need modification::

  git log --oneline .gitmodules | less -+F

Mark the .gitmodules addition as a drop, and each subsequent one for
edit so as to hand-remove the .gitmodules reference::

  git rebase -i $firstcommit^
  git rebase --continue

Now enter the source repository::

  cd ~/src/srcrepo

Move the commits into the prefix ahead of time, so they won't need
"changing" in destination repository::

  git filter-repo --force --path-rename :path/to/prefix/ # trailing slash!

Also add new summary line prefixes to log messages here if necessary,
using either ``git-filter-repo --replace-message`` expressions for a
large number of them, or with an interactive rebase to mark some of them
and do an amendment by hand.  Example adds ``initpgp:`` prefix to
commits that are written in my feature/chore/bug/nocode/deletion
single-character prefix scheme (ie, first two chars preserved)::

  git filter-repo --replace-message <(echo 'regex:^(..)==>\1initpgp: ')

Next, identify the commit range we want to pick, which is all commits in
source repository.  Assuming we are at master branch tip, and the repo has
only one root commit (has not been subtree merged before)::

  firstcommit=$(git rev-list --max-parents=0 @)
  lastcommit=$(git rev-list @ | head -1)

Now import the commits to the destination and pick them::

  cd ~/src/destrepo
  git remote add srcrepo ~/src/srcrepo
  git fetch --no-tags srcrepo
  git cherry-pick $(git rev-list $firstcommit $lastcommit)

**Note the new commit hash at the top** after the first pick, as this is
the start of the new imported history and we need it for the next step::

  firstpick=<hash>

Since these picked commits will all be topologically at the end of the
history, but we actually want to have them show up in the git log at
their rightful place in history chronologically (adjacent to other
commits that happened at the time, rather than appended onto the end),
we can copy the author date to the committer date and reorder the
commits.

The ``filter-repo`` tool has a dictionary with fields from the commit,
which can be shown like so::

  git filter-repo --force \
    --commit-callback $'from pprint import pprint\npprint(commit.__dict__)'

We can modify these by assignment.  In this case we will fix up the
committer date to match the author date::

  git filter-repo --force --partial --commit-callback \
    'commit.committer_date = commit.author_date' \
    --refs $firstpick^

  git log --format='%H %at %s' \
  | sort -nk2,2 | field 1,3- | sed 's,^,pick ,' \
  >| rebase-todo

Now we do an interactive rebase, but edit the todo list after initiating
(this also gives us a chance to do any hand-swaps of position or drop
commits, or mark for rewording to add prefixes, etc)::

  git rebase -i --root
  <delete todo list>
  <read in file rebase-todo>


Conclusion
------------------------------------------------------------------------------

At the end of this manual technique, the history should be linear,
including both projects, ordered chronologically by author date, and
include the prefix-pathed source repository files, with no more
submodule, and no merge commit pollution.

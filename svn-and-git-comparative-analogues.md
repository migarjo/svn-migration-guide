# Subversion (SVN) and Git: comparative analogues
- [Subversion (SVN) and Git: comparative analogues](#subversion--svn--and-git--comparative-analogues)
  * [What is this document?](#what-is-this-document-)
  * [Subversion: a technical deep-dive](#subversion--a-technical-deep-dive)
    + [What about _Merge Tracking_?](#what-about--merge-tracking--)
  * [Branching: the logical flow](#branching--the-logical-flow)
    + [What about the `SQLite` database?](#what-about-the--sqlite--database-)
    + [The SVN Database: `wc.db` layout](#the-svn-database---wcdb--layout)
      - [Sample `wc.db` entry](#sample--wcdb--entry)
  * [Dealing with nested repositories](#dealing-with-nested-repositories)
    + [What about `svn:externals`?](#what-about--svn-externals--)
  * [SVN and Git: comparative analogues](#svn-and-git--comparative-analogues)
    + [Snapshots vs Diffs](#snapshots-vs-diffs)
      - [Snapshots](#snapshots)
      - [Diffs](#diffs)
    + [Merge operations](#merge-operations)
    + [Migrating history: revision vs commit](#migrating-history--revision-vs-commit)
      - [Commits](#commits)
      - [Revisions](#revisions)
      - [History](#history)
  * [Conclusion](#conclusion)
      - [References](#references)

## What is this document?
The purpose of this document is to detail how Subversion handles branching, tagging, merging and change tracking under the covers in order to address some common misunderstandings when dealing with migrations. The overall course of this document will show that `branches` and `tags` are significant because teams _treat_ them as such, and not because subversion tracks them as such. We will also discuss `snapshots` and `diff`, as well as the _merging_ process. Subversion is fundamentally different from how Git works, which creates `Refspecs` for tags and branches, signifying that they are tracked differently than normal files in the existing folder. Git is always working on a branch, even if that branch is _master_, so your changes are always tracked in relation to a branch.

## Subversion: a technical deep-dive
While it is true that you can track branches programmatically while developing with subversion, this is something that needs to be done _externally to subversion_, either manually by teams or by using a platform, such as [FishEye](https://www.atlassian.com/software/fisheye) or [TeamForge](http://blogs.collab.net/subversion/what-subversion=).

[**FishEye Quote**](https://confluence.atlassian.com/fisheye041/svn-tag-and-branch-structure-847745810.html):
>Since tags and branches are implemented via directory copies in Subversion, they are not really first-class concepts. This means that FishEye has to determine branch and tag information by examining the paths involved in Subversion operations and matching these against branch and tag conventions used in the repository. Since these conventions are not fixed, you may need to tell FishEye what conventions you use in your repository. By default FishEye has some inbuilt rules which handle the most common conventions typically used in most Subversion sites. If, however, you've decided to use a custom convention, you can define custom rules to describe what your tag/branch structure looks like. These settings can be edited on the 'Add Repository' or 'Edit Repository' pages in the FishEye Administration pages.

### What about _Merge Tracking_?
Subversion certainly _must_ track the changes _somewhere_ if there's going to be any rolling back. This is where the `svn:mergeinfo` property comes in. As of Subversion 1.5, merge tracking is a useful feature for tracking the merge history. During each merge operation, the `svn:mergeinfo` property is changed to record the merge. The `svn:mergeinfo` property will live on the target in this case. Unfortunately, subversion only records where changes are _merged from_, not where they are _merged to_. This presents a challenge when attempting to analyze a repository without having a defined convention.

Another challenge regarding merge tracking is the fact that it only updates the property after you merge. This means that we may have countless branches that are never merged, and therefor would never be recorded as having been so.

With the flexibility of having `trunk` or not, or making some other folder serve as the `trunk` with a different name, having `branches` that aren't called "branches", or `tags` that aren't called "tags", branches that are never merged, and so forth, limitless possibilities exist wherein the developers using the system must track it themselves, or else an external system.

## Branching: the logical flow
1. The user checks out a repository
2. The user creates a remote branch using `svn copy <source> <target>`
    - This copies the files with their history
    - This creates a new revision.
    - There are no references to the original files or folders, but only log entries tracking the creation of the new file and marks the change as `A` in the history to denote _added_ files.
3. The may also switch to an existing branch (`svn switch https://www.example.com/svn/branches/v1p2p3`)
    - This triggers a series of _cheap copies_ to be created. A _cheap copy_ is essentially a _link_ to the original file, and it updates the `symlink_target` field in the _local_ `wc.db`.
    - This action affects only the local _working copy_, and targets a remote directory for committing the changes to. No remote references are created at this point.
    - The local `wc.db`, or _working copy database_ is updated with entries to be tracked, but they are tracked as normal files and folders.
4. The user makes changes
    - This causes the changed files to be saved as _new files_, and no longer _cheap copies_.
5. The user checks in the changes
    - This causes the files to be pushed to the targeted remote directory, _as normal files and folders_, with the checked-out revision number.
    - There are no references to the branch itself, outside of the local _working copy_
6. The user merges the `branch` to the `trunk` (this can also happen in the working copy)
    - This triggers _merge tracking_ on the _target_ to update the `svn:mergeinfo` property for that revision.

This flow seems logical enough, but it makes a few assumptions.

**Assumptions:**
- We're merging to `trunk` (this may not be true, and may be branch to branch)
- We're merging from a `branch` (this can be _from_ any directory)
- We're keeping our branches in the same parent folder (this may not be the case)
- We're merging a folder and not individual files (we'll discuss this further down)

Consider the following layout:
```bash
repository/
  - project1/
    - folder1/
      - branch1/
      - subfolder2/
        - branch2/
        - subfolder3/
          - branch3/
```

In this example, we may have _mergeinfo_ for each of the branches, but there is no way to map `branches` in our migration. This is because `git-svn` can look at a single directory per repository for branches. Is `branch3` a branch of `subfolder3`, or is it a branch of some other directory? `svn copy` doesn't leave a trail of where it was copied _from_, only that it was newly added to the repository. If we treat each of these as branch folders then we need 3 separate repositories, as `git-svn` cannot multi-map.

Another point to note is that `svn copy` does not _have_ to be used for branching and tagging. It can be used for other purposes, and very often is used for such.

Finally, the `svn:mergeinfo` property is only available _if_ there is a merge. If you do not merge, there is no merge tracking.

### What about the `SQLite` database?
Merge history is not stored in the SQLite database. This is documented in [Apache's documentation](https://svn.apache.org/repos/asf/subversion/trunk/notes/merge-tracking/index.html)

### The SVN Database: `wc.db` layout

| Name | Data type | Primary Key | Foreign Key | Unique | Check | Not NULL | Collate | Default Value |
| :--- | :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| `wc_id` | INTEGER | :key: | :link: | | | :white_check_mark: | | _NULL_ |
| `local_relpath` | TEXT | :key: | | | | :white_check_mark: | | _NULL_ |
| `op_depth` | INTEGER | :key: | | | | :white_check_mark: | | _NULL_ |
| `parent_relpath` | TEXT | | | | | | | _NULL_ |
| `repos_id` | INTEGER | | :link: | | | | | _NULL_ |
| `repos_path` | TEXT | | | | | | | _NULL_ |
| `revision` | INTEGER | | | | | | | _NULL_ |
| `presence` | TEXT | | | | | :white_check_mark: | | _NULL_ |
| `moved_here` | INTEGER | | | | | | | _NULL_ |
| `moved_to` | TEXT | | | | | | | _NULL_ |
| `kind` | TEXT | | | | | :white_check_mark: | | _NULL_ |
| `properties` | BLOB | | | | | | | _NULL_ |
| `depth` | TEXT | | | | | | | _NULL_ |
| `checksum` | TEXT | | :link: | | | | | _NULL_ |
| `symlink_target` | TEXT | | | | | | | _NULL_ |
| `changed_revision` | INTEGER | | | | | | | _NULL_ |
| `changed_date` | INTEGER | | | | | | | _NULL_ |
| `changed_author` | TEXT | | | | | | | _NULL_ |
| `translated_size` | INTEGER | | | | | | | _NULL_ |
| `last_mod_time` | INTEGER | | | | | | | _NULL_ |
| `dev_cache` | BLOB | | | | | | | _NULL_ |
| `file_external` | INTEGER | | | | | | | _NULL_ |
| `inherited_props` | BLOB | | | | | | | _NULL_ |

#### Sample `wc.db` entry

```sql
root@41901aed26d3:~# sqlite3 wc.db
SQLite version 3.11.0 2016-02-15 17:29:24
Enter ".help" for usage hints.
sqlite> .headers on
sqlite> .mode column
sqlite> SELECT * FROM nodes WHERE op_depth = 0 LIMIT 1;
wc_id       local_relpath                     op_depth    parent_relpath         repos_id    repos_path                        revision    presence    moved_here  moved_to    kind        properties  depth       checksum                                        symlink_target  changed_revision  changed_date      changed_author  translated_size  last_mod_time     dav_cache                                                                                      file_external  inherited_props
----------  --------------------------------  ----------  ---------------------  ----------  --------------------------------  ----------  ----------  ----------  ----------  ----------  ----------  ----------  ----------------------------------------------  --------------  ----------------  ----------------  --------------  ---------------  ----------------  ---------------------------------------------------------------------------------------------  -------------  ---------------
1           branches/1.0.1/Covers/Covers.ini  0           branches/1.0.1/Covers  1           branches/1.0.1/Covers/Covers.ini  3196        normal                              file        ()                      $sha1$f44ea39e4e713f487fcb56864368794935c8f526                  568               1194096505109862  canni2007       587              1522348090868849  (svn:wc:ra_dav:version-url 62 /svn/ultrastardx/!svn/rvr/568/branches/1.0.1/Covers/Covers.ini)
```

Our initial thought here is to use either the `symlink_target` field, or the `properties` field to determine what this branch points to. Unfortunately, as we can see from the database info above, there are no references to this being a branch, other than the folder name itself. The `symlink_target` field is updated _in the working copy_ when you use `svn switch`. This triggers a `cheap copy` to be created, but this means that files are created upon the commit. When they are created they are no longer _symlinks_, and are not tracked as such. This leaves us unable to find the target _outside of our local working copy_.

## Dealing with nested repositories

It is fairly common to have repositories with multiple projects in them. The normal structure for this sort of project layout is as follows:

```bash
repository/
  - project1/
    - branches/
      - branch1/
      - branch2/
      - branch3/
    - tags/
      - 1.0/
      - 1.2/
    - trunk/
  - project2/
    - branches/
      - branch1/
      - branch2/
    - tags/
      - 2.0.1-beta/
    - trunk/
  - project3/
    - branches/
      - branch1/
    - tags/
      - 1.0.1-alpha/
    - trunk/
```

If all projects in Subversion were managed with this structure then migrating them to Git would be a pretty straight-forward process. The reality is, however, there are _many_ projects where `branches`, `tags` and `trunk` are not consistently structured. Consider the following layout:

```bash
repository/
  - myfile.txt
  - docs/
  - trunk/
  - branches/
    - branch1/
    - branch2/
  - folder1/
    - trunk/
  - folder2/
    - myotherfile.txt
    - subfolder1/
    - subfolder2/
      - branches/
        - branch1/
        - branch2/
      - tags/
        - 1.0.1-alpha/
      - trunk/
  - folder3/
    - tags/
      - 2.1/
    - myfolder/
```

In this example we have files in the root of the directory, and a separate `trunk` folder. The migration of this repository to git requires that _either_ the repository root _or_ `trunk` be mapped to a git `master`. They cannot both map. If we map `trunk` to `master` then we will be ignoring the files and folders at the root level. This is usually not the desired effect. A solution would be to move the files in the root of the repository into the `trunk` folder prior to migrating, or else treat the root of the repo as `trunk`.

Another challenge with this structure is the fact that we have `branches`, `tags` and `trunk` in nested folders, but not following any particular convention. As discussed in the `svn:mergeinfo` section, these folders will track where they were merged _from_. If we have `repository/folder2/subfolder2/branches/branch1/`, we cannot tell where it was merged _to_ without analyzing each file in the repository at each revision. This seems logical enough, but if we have this branch merged into multiple places then we lose the ability to determine the target. Consider the following as an example:

```bash
svn merge folder2/subfolder2/branches/branch1/ trunk/
svn merge folder2/subfolder2/branches/branch1/ folder2/subfolder2/trunk/
```

In the example above we have multiple target directories, which would lead us to determine that `folder2/subfolder2/branches/branch1/` is a branch, but does not tell us _what it is a branch of_. Is it a branch of `trunk`? Is it a branch of `folder2/subfolder2/trunk/`?

We could then take a look at the `svn log` to see where `svn copy` originally created the branch. This, again, seems like the logical next step. The challenge with this, however, also lies in consistency. Subversion allows for any folder to be merged into any folder, which means that you do not _have_ to create a branch using `svn copy`, and this often proves to be the practice. Additionally, since there is no difference between how subversion treats `tags` and `branches`, there would be no way to programmatically discover a repository layout without the administrator providing parameters by which their teams abide.

### What about `svn:externals`?
Subversion provides the ability to set [`svn:externals`](http://svnbook.red-bean.com/en/1.7/svn.advanced.externals.html) when working with multiple revisions or repositories at once. This really seems like a perfect mapping to _git submodules_, but they differ in many respects. The challenge with using `svn:externals` is the same challenge we face with other potentials: namely, inconsistent use and formatting. It is somewhat uncommon for teams using nested projects to treat them as externals, so relying on this property would leave us with no information in most cases. Another challenge is that the property for `svn:externals` is applied to a particular revision, and not maintained across revisions. This means we can have a different set of externals applied to a folder for each revision, and it cannot provide a definitive list of URL's to use as git submodules.

## SVN and Git: comparative analogues
When considering a migration from subversion to git, there are a few technical and idealogical differences that are generally misunderstood. This usually leads to a significant expenditure of resources trying to accomplish what can't reasonably be expected to succeed.

### Snapshots vs Diffs
Many teams will ask about their binaries when migrating, and argue that subversion has no issue tracking changes in binaries. What is generally misunderstood here is that subversion maintains _snapshots_, whereas git tracks _diffs_. There is a fundamental difference between the two which should be properly understood when comparing the two systems.

#### Snapshots
A **_snapshot_** is a _point in time_ reference to block-level data. When there is a change made, a _delta_, or incremental difference is recorded of the _block-level_ state, and nothing is or can be recorded about the data itself, just the block changes. If we revert to an older state then the blocks are changed back. The consequence is that our data (or contents) will match the old data, but the subversion server is not aware of the contents, only the block changes in the successive versions of files.

Subversion _does_ have a command to view the diffs (`svn diff`), but this is fundamentally different than that of git. In the case of subversion, it will read the block data from the old revision, then read the block data from the new revision and output the difference. This difference is read in realtime and not stored as a difference in contents.

A challenge commonly faced with the _snapshot_ approach is that you cannot have two people working on the same file at once. This often causes merge conflicts because the changes are tracked at the block-level and not tracked in the contents.

![snapshots](https://user-images.githubusercontent.com/865381/38507376-5f2a62fe-3bea-11e8-84bf-11cdf3d3d9fe.png)

_Snapshots record only the block-level changes since the last snapshot_.

#### Diffs
A **_diff_** is comparison of changes _in the contents_ between two files. For text files, this means that it will look at deleted lines, added lines and changed lines, and store a record of the differences. For binary files, this means that it cannot view them, and stores the entire file as a new copy. When using _diff_, there is no attempt to view the block-level differences. The resulting outcome is that you will end up with multiple _full_ copies of the file in your repository _and history_.

![diff](https://user-images.githubusercontent.com/865381/38505680-c4fd9e2a-3be5-11e8-9c0f-9ad9cfe49957.png)

_Diff does not track the block-level differences, only contents_.

### Merge operations
Subversion allows for very flexible merging, and accommodates several scenarios that Git does not. One such scenario is the ability to merge individual files between branches or folders. In git we can merge branches to branches, but not individual files, and it must be a merge between branches. With subversion, however, you can merge individual files between any folder, which allows a user to be very granular in their merge operations.

On the surface this seems like an oversight in git, or a significant shortcoming. In practice, however, having the flexibility to merge like this has historically prevented teams for having a unified workflow, and hampers collaboration. Teams and individuals have limitless flexibility to practice bad habits and you lose definitive controls around workflow. This also presents a problem when attempting to dynamically discover a repository layout based on the repository history.

### Migrating history: revision vs commit
In subversion, revisions and commits are inseparable _in practice_, but in git they are distinctly differentiated in theory and in practice.

#### Commits
A _commit_, in its simplest form, is a record of the changes made, along with data around the changes. In subversion, this commit will include an optional comment, the timestamp and the author's username. In git, this commit will include the author's name, email, SHA1 hash, optional comment, and timestamp. When working with subversion, committing to a repo checks in the files to the remote repository. In git, a commit happens locally, and changes aren't pushed to the remote until the `git push` command is issued. There is a bit more context around this in the section on revisions.

#### Revisions
A _revision_, or version, is any change in form. In SVN, a revision is the state at a point in time of the entire tree in the repository.

Quote from [Subversion in Action](http://structure.usc.edu/svn/svn.basic.in-action.html#svn.basic.in-action.revs):
>Unlike those of many other version control systems, Subversion's revision numbers apply to entire trees, not individual files. Each revision number selects an entire tree, a particular state of the repository after some committed change. Another way to think about it is that revision N represents the state of the repository filesystem after the Nth commit. When Subversion users talk about “revision 5 of foo.c”, they really mean “foo.c as it appears in revision 5.” Notice that in general, revisions N and M of a file do not necessarily differ!

In effect, this means that a commit and revision within subversion are handled the same. Since the commits can only happen when changes are pushed to the subversion server, the revision is updated when the commit is created. Creating a new commit will create a new revision. Subversion's central focus is revisions.

In git, commits update the _commit objects_, but revisions are changed and tracked for each file, not for the entire tree. This provides flexibility with undoing changes at a granular level, and helps to avoid merge conflicts.

#### History
The consequence of these differences becomes more apparent when we're mapping subversion history to git. The level of data tracked by subversion in the commits does not map directly to git, and requires significant modification to retain and map. The lack of a one-to-one means that the migrated history becomes more of a "best effort storytelling" than actual history. It may be mostly accurate, but it won't paint the picture we'd want.

**Subversion:**
```bash
$ svn log --with-all-revprops --xml lib/crit_bits.c
<?xml version="1.0"?>
<log>
  <logentry revision="912">
    <author>jane</author>
    <date>2018-07-29T14:47:41.169894Z</date>
    <msg>Fix up the last remaining known regression bug.</msg>
    <revprops>
      <property name="testresults">all passing</property>
    </revprops>
  </logentry>
</log>
```

**Git:**
```bash
sha1(
    commit message  => "second commit"
    commiter        => Jane Doe <jane.doe@example.com>
    commit date     => Sat Nov 8 11:13:49 2018 +0100
    author          => Jane Doe <jane.doe@example.com>
    author date     => Sat Nov 8 11:13:49 2018 +0100
    tree            => 9c435a86e664be00db0d973e981425e4a3ef3f8d
    parents         => [0d973e9c4353ef3f8ddb98a86e664be001425e4a]
)
```

Given the data above we can see that subversion logs significantly less data in each revision than what we see in our git commits. Mapping this info means adding data that was never there, and this inherently disqualifies any history from being usable in an audit. We will also note that a revision in subversion applies to all files in the tree, whereas in git we will have a revision for each file.

## Conclusion
Subversion is fundamentally different in how it operates from Git. Much of the inherent functionality in Git is _assumed_ to be present in subversion, and vice-versa, but instead subversion relies heavily on human behavior and has some flexibility that was intentionally avoided in git. Branches in subversion are branches because they are _treated_ like branches. Tags in subversion are tags because they're _treated_ like tags. Nested repositories are nested repositories because they're _treated_ like nested repositories. Subversion can merge any file to any other file, and any directory to any directory. Subversion tracks changes, but not with the granular `metadata` that we often want. Third-party platforms, such as [FishEye](https://www.atlassian.com/software/fisheye), [TeamForge](http://blogs.collab.net/subversion/what-subversion=) and [GitHub](https://github.com) provide _some_ this metadata through the use of their own database and cataloging, but there is still a need for teams to define their own behaviors for these platforms.

#### References
- [Subversion Merge Tracking](https://svn.apache.org/repos/asf/subversion/trunk/notes/merge-tracking/index.html)
- [Subversion Tags](http://svnbook.red-bean.com/en/1.6/svn.branchmerge.tags.html)
- [Subversion Branches](http://svnbook.red-bean.com/en/1.6/svn.branchmerge.switchwc.html)
- [Subversion Externals](http://svnbook.red-bean.com/en/1.7/svn.advanced.externals.html)
- [Subversion Advanced Properties](http://svnbook.red-bean.com/en/1.7/svn.advanced.props.html)
- [Pro Git](https://git-scm.com/book/en/v2)
- [Git SVN and using Git with other VCS's](https://git-scm.com/book/en/v2/Git-and-Other-Systems-Git-as-a-Client)
- [SVN Tag and Branch Structure](https://confluence.atlassian.com/fisheye041/svn-tag-and-branch-structure-847745810.html)
- [What is Subversion?](http://blogs.collab.net/subversion/what-subversion=)
- [Anatomy of a Git Commit](https://blog.thoughtram.io/git/2014/11/18/the-anatomy-of-a-git-commit.html)

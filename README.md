## Subversion to Git: Best Practices
There is a lot to account for when considering or planning a change from Subversion to Git. To ensure a healthy, happy move, we recommend a tiered approach:
- [Assess current workflow and estimate toolchain impact](#assess-current-workflow-and-estimate-toolchain-impact)
- [Determine what to migrate](#determine-what-to-migrate)
- [Preliminary steps and preparations](#preliminary-steps-and-preparations)
- [Migrate source code](#migrate-source-code)
- [Revisit workflow to measure progress](#revisit-workflow-to-measure-progress)

### Assess current workflow and estimate toolchain impact
Migrating to GitHub is more than just moving source code. A successful migration includes a planned effort around cultural transformation, workflow modernization, and improved collaboration. It's important to:
- Set expectations
  - Decide how and when you'll switch to Git. This could be gradually (by team or repository), or in a single organization-wide move.
  - Determine how much history _actually_ needs to be migrated to Git. 
    - Migrating history actually means _re-creating_ that history via new commits, so be aware that this new history is unlikely to meet audit or compliance requirements.
    - Commit history for trunk may be more useful than the more ephemeral branches used during development. Many teams opt to migrate the history of just their default branch when switching between version control systems.
- Determine changes to workflow
  - How will you adapt workflows to take full advantage of Git's distributed nature?
  - Do your current repositories exhibit qualities that can negatively impact Git performance? These can include large binaries, a monolithic multi-project structure, and long-running or divergent branches.
  - How will you communicate these recommended workflow changes to your teams?
  - What other tools in your current workflow (build systems, CI, deployment) will be impacted by the change to Git?
- Enable everyone with training
  - Provide training so that your developer teams are comfortable and confident with Git and GitHub. This provides a positive experience for developers, avoids misconceptions, and eliminates confusion.
  - Document and collaborate on your updated workflows in a shared space.

#### How Git is different
Going hand-in-hand with the cultural shift, you need to be aware of how using Git differs:
- Forks and Remotes mean you can push code to different locations
- SHA hashes identify individual commits
- Git/GitHub use a Staging Area
- Workflows will need to adapt 
  - Branches are easy to make and are intended to be ephemeral (merged back into a default/`master` branch when ready).
  - Pull Requests are used to have conversations and collaborate, then are merged when ready.
  - The GitHub UI is very user-friendly, but can take some adjustment, particularly for teams not used to communicating & collaborating in writing.

#### Pre-migration considerations
Before you migrate, it is ideal to stop working in Subversion until you perform the migration, and then switch completely to Git. This avoids "drift" where code changes could occur in multiple authoritative locations.
  - If this isn't possible, utilize `cron` jobs to update repositories every week.
  - It's a good idea to have Git and GitHub subject matter experts to whom team members can address their questions and concerns.
  - Having a wiki or README with basic information or a Q&A can be helpful.
  - Training is always helpful and increases comfort levels and confidence. [
  - Knowledge will start to transfer organically, but can be accelerated by shared spaces for information exchange.

### Determine what to migrate
#### The challenge
The biggest challenge when migrating from Subversion to Git is preserving large histories. Other issues to consider include dependencies, binary files, and superfluous branches. 

#### The solution
1. Do you need to bring over a full history? If you can place everything into read-only mode and leave it in Subversion, this is the simplest solution.
  - Scrutinize what, if any, files should be migrated and be highly selective.

2. If you're unable to leave history in Subversion and start fresh in Git, determine what the essential items are, and omit transferring anything non-essential.
  - Can you initialize with your current working tree?
  - Can you bring over a single branch and move forward from there?
  - Determine methods for bringing over branches and history that represent the essential, minimal amount of information you need.

3. Examine the size of the source repository. While there is no real "cut off" regarding the size, a good sanity-check point would be to try to keep repositories under 1GB.
  - If a repository is over 1GB in size, can you determine why? Are there large binaries or compiled binary artifacts checked into version control?
    - These binary artifacts could be moved into an artifact management system, GitHub's [Releases](https://help.github.com/articles/about-releases/) feature, or [Git LFS](https://git-lfs.github.com/), as appropriate.
  - Is the repository monolithic, combining several related projects together? Could it be separated into smaller component repositories?

4. Take the time to clean up and restructure repositories before migrating to alleviate time, confusion, and clean-up post-migration. Branch history can get complicated and may not line up with how Git thinks about things. 
  - Restructuring after the fact can a bigger undertaking than doing this step in advance.
  - Do you need to bring over all branches?
    
### Preliminary steps and preparations
We stress and suggest the following steps to ensure all necessary items transfer into Git. Non-essential items, such as superfluous branches, should be removed before the actual process of migration.

1. Ready your environment for migration. 
  - If you plan to include binaries in your repository 
  - Ensure that you have at least Git 2.6 installed to avoid issues with case sensitivity

### Migrate source code
- If you have determined to perform a clean-cutover, follow the directions [in this playbook](./svn-to-git-clean-cutover-playbook.md) to migrate.

- If you have decided to migrate history from your subversion repository, clone the [svn2git](https://github.com/nirvdrum/svn2git) repository and follow the directions in the README to migrate based on your use case.
  - If you have found large files that you can remove from your repository, you will need to remove them from the entire history as well. Git provides an [out-of-the-box option](https://git-scm.com/docs/git-filter-branch) for rewriting history, including removing specified files. [BFG Repo-Cleaner](https://rtyley.github.io/bfg-repo-cleaner/) is an optimized solution that can scan your repositories for potentially problematic files and help you control what to remove.


__Note:__ 
An issue with this is history: you can look into the `p4-history/Release` branch for more details since all the changes were squashed into one commit.

### Revisit workflow to measure progress
Cultural transformations do not often happen overnight, so it's vital to have timely check-ins for:
- Developer happiness & productivity
  - Does everyone understand how to use Git and GitHub effectively?
  - Do they know who can answer any questions?
  - Are there any clarifications that need to be made?
- Workflow consistency
  - Is the updated workflow effective?
  - Is the workflow being followed consistently?
  - Are there any adaptations that need to be made?

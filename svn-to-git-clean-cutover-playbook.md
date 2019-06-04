# Moving repository from SVN server to GitHub Enterprise server.
1. Create a local directory with the same name that you want to give your repository (your SVN client may do it for you).
1. Checkout the repository that you want to migrate from SVN into your previously created local directory.
1. Run the `git init` command from your local directory.
1. Create [.gitignore](https://help.github.com/articles/ignoring-files) file in local directory and add **.svn** as the only text in the file unless your project has files that you need to ignore, in which case list them here.
1. Run the git add command to make git aware of all the files in the local directory that will be migrated.
  - `git add .`
1. Run git status to confirm.
  - `git status`
1. Commit the files to git.
  - `git commit -m "Initial migration from SVN for <repo-name>"`
1. Start adding branches. Skip to step 16 if not needed.
1. Run `git checkout -b <desired-branch-name>`.
1. Remove all files except for .gitignore file and .git folder (this will allow you to replace the folder contents with the new branch's contents, which is required by SVN).
1. Repeat step 2 for this branch.
1. Add any undesired files or folders to .gitignore that are in the new branch.
1. Run `git add .`.
1. Run `git status` to confirm everything looks OK.
1. Commit the files to git.
  - `git commit -m "Initial migration from SVN for <branch-name>"`
1. Your project is now ready to be uploaded to GitHub Enterprise. Create an empty repository on GitHub Enterprise:
  - Select 'org_name' as the Owner.
  - Give new repository desired name.
  - Enter new repository description. It should describe what the repository is for internally.
  - Select 'Public'.
  - Do not initialize with a README.
  - Do not select .gitignore or license templates.
  - Click 'Create repository'.
1. Add repository remote origin to your local git settings.
  - `git remote add origin git@git.example.com:org_name/<new-repo>.git`
1. Make sure your public key is uploaded to your GitHub Enterprise local account settings (e.g. `https://git.example.com/settings/ssh`).
1. Push your new repository to GitHub Enterprise.
  - `git push --set-upstream origin master` (do this each time for every branch you have).
1. Per our team policy, every repository should have a README.md with details on how to onboard new employees with this project. Please add one.
1. Delete SVN's tracking folder (.svn/) from your local directory.

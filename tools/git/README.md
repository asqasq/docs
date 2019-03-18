# Tips and tricks around using git

## Add submodule
    git submodule add git@github.com:someusername/somerepo.git

## Deleting branches, locally and remotely
Deleting local branches does not automatically delete remote branches.
To delete a remote branch, one can kind of push the delete of the local branch
remotely by using this command:

         git push <remotename> :<branchname>

For example:

        git branch -d mybranch
        git push origin :mybranch



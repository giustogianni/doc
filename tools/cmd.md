# Commands Wrapped

## Unix
| cmd         | def  |
|-------------|---|
| crontab -e  | Open the crontab configuration file for the current user. User-specific.  |
| curl -k -X  | For transferring data with URLs. `-k` skip certification. `-X` or `--request` send custom request to the server, usually GET, HEAD and POST.  |
| curl ifconfig.me  | Get host IP address.  |
| dig \<url\>   | Perform a DNS lookup.  |
| grep -r \<str\> .  | Look recursively for string str in current folder.  |
| head -n \<n\> \<filename\>   | Displays the specified number of lines from the top of the file. Default (without tag): 10.  |
| rsynch | Synchronizing files and directories between two locations over a remote shell. Fast because incremental, meaning only the differences between the source and the destination are transferred. |
| scp | Securely copy files and directories between two locations through ssh. Syntax: `scp [OPTION] [user@]SRC_HOST:]file1 [user@]DEST_HOST:]file2`.  |
|   |   |

## Git
| cmd  | def  |
|------|------|
| git commit --amend --no-edit  | Amends a commit without changing its commit message. `--amend` replaces the most recent commit entirely, meaning the amended commit will be a new entity with its own ref. `--no-edit` use the selected commit message without launching an editor. |
| git log --oneline --decorate --graph --all  | Display commit graph.  |
| git pull --ff-only  | Only update to the new history if there is no divergent local history.   |
| git pull --rebase   | Rebase the current branch on top of the upstream branch after fetching.   |
| git push --set-upstream origin \<branch\>  | Push when the current branch has no upstream branch.  |

## Docker
| cmd  | def  |
|---|---|
| docker exec -it <container-name> /bin/bash  | Execute `/bin/bash` command inside the running container.  |
| docker image ls   | List images.  |
| docker load --input \<filename.tar\>  | Load image from local `.tar` file.  |
| docker rm -vf $(docker ps -aq)   | Remove all container and its volume.  |
| docker rmi -f $(docker images -aq)  | Delete all images.  |
| docker save -o \<filename.tar\> \<docker-image\>  | Save docker image, *e.g.* ghcr.io/cloudquery/cloudquery, locally to a `.tar` archive. |

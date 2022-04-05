# git introduction workshop
This repo contains an introductory walkthrough of common git operations,
including:
  - Setting up global git config
  - Setting up ssh to github
  - Forking and cloning repos
  - Creating new branches
  - Commit and push
  - Rebasing dev branches with production updates
  - Resolving merge conflicts
  - Merging back to the main branch
  - Cleaning up merged branches
  - Creating release tags

## Setting up global git config
When you try to commit code for the first time, you might be prompted to setup your git configuration user details.
Your git config is a file located at `~/.gitconfig`, and essentially contains the name and email
address that is used in commit messages. An example git config file looks like this:

```
# This is Git's per-user configuration file.
[user]
	name = Your Name
	email = your-email@parallelworks.com
```

While you can create the `.gitconfig` with any text editor, it is recommended to use git commands to
do this. A simple `~/.gitconfig` file can be made with the following commands:

```
git config --global user.name "Matt Long"
git config --global user.email mlong@parallelworks.com
```

Aside from user details, you can also configure the default behavior of commands such as `git pull`. Newer git clients
may print informational messages prompting you to setup these configurations in repos you work in.
The `~/.gitconfig` file can be considered a global config file that applies to all repos you work with.
git repos should also include a hidden directory called `.git` at the top level, and will include a `config` file with additional  configurable parameters specific to that repo:

```
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
	ignorecase = true
	precomposeunicode = true
[remote "origin"]
	url = git@github.com:parallelworks/git-workshop.git
	fetch = +refs/heads/*:refs/remotes/origin/*
[branch "main"]
	remote = origin
	merge = refs/heads/main
[pull]
	rebase = false
```  

## Setting up ssh to github
git repositories are typically cloned using https or ssh. I prefer to use ssh when working from a terminal, especially if the repo I am trying to clone is private. Before using git over ssh, you will need to create an ssh key and add it to your github profile.

Create an ssh key:
```
ssh-keygen
# this creates an RSA ssh key by default.
# You will be prompted to enter a name for the key file as well as a passphrase.
```

If you created your ssh key with a non-default name, you will need to tell your ssh client
to use that key when connecting to `github.com`. One way to do this is with an ssh config file.
To do this, create a new file at `~/.ssh/config` and provide the following details
(replacing the IdentifyFile with the name of your key):

```
Host github.com 
    Hostname github.com 
    User git
    IdentityFile ~/.ssh/github
```

After creating your ssh key, you will need to add it to your github account. Instructions for this
are available at github's website here: https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account

Once you've added your key to your account,
you can test ssh with the following command:
```
ssh -T git@github.com
```

If the connection is successful, you should get a response back that looks like this:
```
Hi mattlongt! You've successfully authenticated, but GitHub does not provide shell access.
```

## Forking and cloning repos
Now that ssh to github has been setup, we can clone a repo and begin making contributions. 

To proceed with the rest of the tutorial, fork this repo to create your own base version of
the repository. This will provide you with a lab to practice git operations. 

1. Create a new empty private repo for the fork
  - Follow documentation here to create a new empty repo: https://docs.github.com/en/get-started/quickstart/create-a-repo
  - Create the repo under your personal github account, not the Parallel Works organization
  - Don't initialize the repo (leave all init boxes unchecked)
  - Name your repo `git-workshop`

2. Clone your new repo 

```
git clone git@github.com:<github username>/git-workshop.git
```

3. Add the original repo as an upstream remote, and push

```
cd git-workshop

git config pull.rebase false

git remote add upstream git@github.com:parallelworks/git-workshop.git

git pull upstream main 

git push origin master # empty github repos use `master` as the default branch if you didn't rename it already.
```

4. Reset your new repo as the upstream

```
git remote set-url --add upstream git@github.com:<github username>/git-workshop.git
git remote set-url --delete upstream git@github.com:parallelworks/git-workshop.git
```

5. Rename the `master` branch to `main`  
The documentation provided here will explain the reasoning behind renaming the `master` branch,
and how to do it.

https://github.com/github/renaming

```
# after renaming the `master` branch to `main`...
git branch -m master main
git fetch origin
git branch -u origin/main main
git remote set-head origin -a
```

## Creating new branches
For this exercise, we will be creating two separate branches
  - Note: it is a good idea to prefix your branches with a user identifier, and potentially a ticket/issue number such as:

```
git checkout -b mlong/1/update_pod_tag
```
  - the following examples use very simple branch names instead.

```
git checkout main
git pull
git checkout -b update_pod_tag
git push --set-upstream origin update_pod_tag

git checkout main
git pull
git checkout -b add_svc
git push --set-upstream origin add_svc
```

## Commit and push

### First branch
The first branch we will work on is `update_pod_tag`.

```
git checkout update_pod_tag
git pull
```

We will be making a simple change on this branch. Edit the `pod.yaml` file
and update the pod image tag from `11.14.2` to `latest`. The full file should now look like:

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app.kubernetes.io/name: proxy
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
      - containerPort: 80
```

Add the file to the commit manifest:

```
git add pod.yaml
```

Commit and push the change to your branch:
```
git commit -m 'updated nginx image to use latest tag'

git push
```

### Second branch
Switch to the second development branch you created

```
git checkout add_svc

git pull
```

Copy the `svc.yaml` file from the `resources` directory to the top level of the repo

```
cp resources/svc.yaml ./ 
```

Edit `pod.yaml` and add a name to the port. Edit the tag to be `11.14.3`. `pod.yaml` should look like:

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app.kubernetes.io/name: proxy
spec:
  containers:
  - name: nginx
    image: nginx:11.14.3
    ports:
      - containerPort: 80
        name: http-web-svc
```

Add the files to the commit manifest:

```
git add pod.yaml svc.yaml
```

Commit and push the change to your branch:

```
git commit -m 'added service'
git push
```

Merge `add_svc` to main:

```
git checkout main
git pull
git merge add_svc
git push
```

## Rebasing dev branches with production updates
Switch back to the first branch you made

```
git checkout update_pod_tag
```

Begin a rebase to pull in changes from `add_svc`

```
git rebase main
```

## Resolving merge conflicts
Rebasing at this point should show a merge conflict in `pod.yaml` that looks like:

```
<<<<<<< HEAD
    image: nginx:11.14.3
=======
    image: nginx:latest
>>>>>>> a10e999 (test)
```

To resolve the conflict, remove the markers and keep the image tag you want
The only thing remaining from the merge conflict snippet above should be:

```
image: nginx:latest
```

Finish the rebase
```
git add pod.yaml
git rebase --continue

# This will drop you into an interactive text editor so you can add a commit message. For this exercise, simply write and exit the file (:wq if using vim, which should be the default)

git pull

# Simply write quit out of this interactive editor prompt as well

git push
```

## Merging back to the main branch
```
git checkout main
git pull
git diff --color-words main..update_pod_tag
git merge update_pod_tag
git push
```

## Cleaning up merged branches
```
git branch -d update_pod_tag
git push origin --delete update_pod_tag

git branch -d add_svc
git push origin --delete add_svc
```

## Creating release tags
github documentation for creating release tags: https://docs.github.com/en/repositories/releasing-projects-on-github/managing-releases-in-a-repository

You can also add a tag from the command line:

```
git checkout main
git pull
git tag -a v1.0 -m "my version 1.0"
git push origin v1.0
```

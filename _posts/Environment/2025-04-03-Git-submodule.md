---
title: git method to add submodules
date: 2025-04-03
categories: [git]
tags: [git]     # TAG names should always be lowercase
---

> Problem: I want to sychronize two modules: mA and mB(gitlab), mA(github) contained in mB
> Reason is because they are in the different environments

# Method 1: submodule

## Add the repo
1. Add GitHub Repo as a Submodule in GitLab 
```shell
cd your-gitlab-repo/
git submodule add [repo url] qemu
git commit -m "Add QEMU 9.0.0 (patched) as submodule"
git push origin main
```

2. Update submodule later

when the github repo git new commits

```shell
cd qemu
git pull origin main  # Pull latest changes from GitHub
cd ..
git add qemu
git commit -m "Update QEMU submodule to <commit-hash>"
git push origin main
```
## remove the submodule
However, it is not what we want because we need to manually pulled and seems they are seperate repos

1. deinit the sub module
```shell
# Remove the submodule from .gitmodules and .git/config
git submodule deinit -f -- qemu       # Deinitialize the submodule
git rm -f qemu                        # Remove the submodule directory
rm -rf .git/modules/qemu              # Delete the submodule's git metadata
```

2. Commit the changes
```shell
git add .gitmodules             # Stage the .gitmodules change (if it still exists)
git commit -m "Remove QEMU submodule"
git push origin main            # Push to GitLab
```

# Method 2: Git Subtree method

Git Subtree is a method to embed one repository (e.g., your QEMU fork on GitHub) inside another repository (e.g., GitLab project) while preserving commit history. Unlike submodules, subtrees merge the external repo’s files directly into your main repo, making it act like a normal directory.

### Key Differences: Subtree vs. Submodule
![subtree and submodule diff](/commons/images/environment/gitsubmodulediff.png)

When to Use Subtree?
- You want QEMU’s files to be part of your main repo (not a separate reference).

- You need to modify the embedded code and push changes back to the original repo.

- You dislike submodules’ complexity (e.g., forgetting to init/update them).

## steps
1. Add the GitHub Repo as a Remote

```shell
git remote add qemu-remote [repo url]
git fetch qemu-remote  # Download QEMU’s commits (but don’t merge yet)
```

2. Merge the repo into a subdirectory

- `--prefix=qemu`: Puts QEMU’s files in a `qemu/` directory.
- `qemu-remote main`: Uses the `main` branch of the GitHub repo.
- `--squash`: Combines all QEMU commits into `one single commit` in your GitLab repo (keeps history cleaner).

```shell
git subtree add --prefix=qemu qemu-remote main --squash
```

3. Push to GitLab

```shell
git push origin main  # Now QEMU is part of your repo!
```

### How to Update the Subtree Later

When QEMU’s GitHub repo gets new commits:

```shell
git fetch qemu-remote  # Get latest changes
git subtree pull --prefix=qemu qemu-remote main --squash
```
This merges updates into your qemu/ directory.

### How to Push Changes Back to GitHub

If you modify files in `qemu/` and want to contribute back:

```shell
git subtree push --prefix=qemu qemu-remote your-branch-name
```

This pushes your local qemu/ changes to the GitHub repo’s branch.

## Example Workflow

1. Initial Setup:
```shell
git remote add qemu-remote https://github.com/your-username/qemu-9.0.0-ubuntu18.git
git subtree add --prefix=qemu qemu-remote main --squash
```

2. Daily Use:
Edit files in `qemu/` like normal (they’re part of your repo now!).

3. Sync Updates from GitHub
```shell
git subtree pull --prefix=qemu qemu-remote main --squash
```
4. Push Your Changes to GitHub:
```shell
git subtree push --prefix=qemu qemu-ubuntu18-fixes
```


## Conflict may need be resolved later as a normal merge
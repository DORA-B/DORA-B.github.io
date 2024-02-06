---
title: Cycle Dependency in Apt when encountering Space Limit
date: 2023-04-14
categories: [Linux]
tags: [linux]     # TAG names should always be lowercase
---
Happens in the process when installing cuda-toolkit-12-3
===
# Problem
Install cuda-toolkit-12-3, but there is no space, so use ctrl-C to terminate the install process.
```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2004/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt-get update
sudo apt-get -y install cuda-toolkit-12-3
```
and every apt update reports the error that there is a dependency to the unfinished package, need to "apt -f install", but when executing this command, it will install the infinished dependency packages, which leads to no-space error.

> sudo apt-get --fix-broken install

![no enought space cycle](/commons/images/3141911-20240105010708660-501527966.png)

Here are commands that are not useful:

```bash
sudo apt-get clean
sudo apt-get autoclean
sudo apt-get -f install
```
Try to reslove depencencies:

```bash
sudo dpkg --configure -a
sudo apt-get -f install
```
> dpkg -l | grep cuda-toolkit
```bash
iU  cuda-toolkit                               12.3.2-1                                                       amd64        CUDA Toolkit meta-package
iU  cuda-toolkit-12-3                          12.3.2-1                                                       amd64        CUDA Toolkit 12.3 meta-package
ii  cuda-toolkit-12-3-config-common            12.3.101-1                                                     all          Common config package for CUDA Toolkit 12.3.
ii  cuda-toolkit-12-config-common              12.3.101-1                                                     all          Common config package for CUDA Toolkit 12.
ii  cuda-toolkit-config-common                 12.3.101-1                                                     all          Common config package for CUDA Toolkit.
```
and these packages have dependencies problems that are showed in the "dpkg --configure -a"

# Solve method_1 remove old-version kernels to free space

> dpkg --list 'linux-image*'

Or: 

```bash
dpkg -l 'linux-*' | sed '/^ii/!d;/'"$(uname -r | sed "s/\(.*\)-\([^0-9]\+\)/\1/")"'/d;s/^[^ ]* [^ ]* \([^ ]*\).*/\1/;/[0-9]/!d'
```
TO remove:

```bash
dpkg -l 'linux-*' | sed '/^ii/!d;/'"$(uname -r | sed "s/\(.*\)-\([^0-9]\+\)/\1/")"'/d;s/^[^ ]* [^ ]* \([^ ]*\).*/\1/;/[0-9]/!d' | xargs dpkg --remove
```

[Source](https://nicolasbouliane.com/blog/ubuntu-apt-get-f-fails-no-space-left-on-device-apt-get-autoremove-doesnt-work)

## What is the point here
 To use `dpkg` instead of `apt-get purge`.
 And in the same way, we can use `aptitude` sometimes to replace `apt` to solve lock problem.

# Solve method_2 `--force`

```bash
# remove packages in dpkg
dpkg --purge --force-all cuda-toolkit
# and force remove in apt
sudo apt-get -f autoremove
```
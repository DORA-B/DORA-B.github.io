---
title: how to connect the vscode ssh remote to a deprecated os
date: 2025-08-25
categories: [Environment]
tags: [vim]     # TAG names should always be lowercase
published: true
---

Problem: I want to use vsocde (1.103) remote pack to connect my remote host server (x86_64, 18.04, glibc 2.27), but that is not supported by VS Code (OSes that don't have glibc >= 2.28 and libstdc++ >= 3.4.25) via the Remote - SSH extension. 

References:
- [stack overflow1](https://stackoverflow.com/questions/79561735/using-custom-sysroot-for-vscode-remote-ssh-server-does-not-work-in-version-1-99)
- [sysroot build](https://github.com/ursetto/vscode-sysroot)

# Process

Over view of the whole idea
```
Your System:           VS Code Sysroot:
/lib/libc.so.6         ~/.vscode-server/sysroot/lib/libc.so.6
(glibc 2.27)          (glibc 2.28)
     ↑                        ↑
Used by system apps    Used only by VS Code
```
## build toolchain for the sysroot
- clone the repo and better to have a docker env to execute them.
```shell
# Clone the repository
git clone https://github.com/ursetto/vscode-sysroot.git
cd vscode-sysroot
# Build the sysroot (this takes 15-30 minutes)
make
# we'll get a toolchain/vscode-sysroot-x86_64-linux-gnu.tgz under toolchain directory
```
- unzip it and copy it to the vscode-server directory

```shell
# Create the directory
mkdir -p ~/.vscode-server
# Extract the sysroot
tar zxf vscode-sysroot-x86_64-linux-gnu.tgz -C ~/.vscode-server
# Copy the environment script
cp sysroot.sh ~/.vscode-server/
# better to clear all the files under the directory before *.log, code*, ...
```
- VS Code server uses patchelf during the installation process to consume the required libraries from the sysroot, so download it simply.

```shell
wget https://github.com/NixOS/patchelf/releases/download/0.18.0/patchelf-0.18.0-x86_64.tar.gz
# tar it and there is an executable bin in `bin/`
# chmod it
# put it to the system level `/usr` dir
sudo cp bin/patchelf /usr/local/bin/
sudo chmod +x /usr/local/bin/patchelf
```

## Put environmetal global vars related to vscode installation
Create the following 3 environment variables:
```
VSCODE_SERVER_CUSTOM_GLIBC_LINKER path to the dynamic linker in the sysroot (used for --set-interpreter option with patchelf)
VSCODE_SERVER_CUSTOM_GLIBC_PATH path to the library locations in the sysroot (used as --set-rpath option with patchelf)
VSCODE_SERVER_PATCHELF_PATH path to the patchelf binary on the remote host
```
- we need put them in the system level config, rather than the current user's `bashrc` --> tried but failed.
```shell
# Create the environment file
sudo tee /etc/profile.d/vscode-sysroot.sh << 'EOF'
VSCODE_SERVER_PATCHELF_PATH=/usr/local/bin/patchelf
VSCODE_SERVER_CUSTOM_GLIBC_LINKER=/home/qingchen/.vscode-server/sysroot/lib/ld-linux-x86-64.so.2
VSCODE_SERVER_CUSTOM_GLIBC_PATH=/home/qingchen/.vscode-server/sysroot/lib:/home/qingchen/.vscode-server/sysroot/usr/lib

export VSCODE_SERVER_PATCHELF_PATH
export VSCODE_SERVER_CUSTOM_GLIBC_LINKER
export VSCODE_SERVER_CUSTOM_GLIBC_PATH
EOF
```

## reconnect with remote ssh

Got a warning popup (and notice bar) that the target OS was not supported, but then everything worked.





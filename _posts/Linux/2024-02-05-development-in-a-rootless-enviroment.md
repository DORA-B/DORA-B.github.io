---
title: Development in a rootless environment
date: 2024-02-05
categories: [Linux]
tags: [linux, development]     # TAG names should always be lowercase
---
# How to install packages

1. Using brew, Homebrew/Linuxbrew
2. Download Github Repository Homebrew from [Link](https://docs.brew.sh/Homebrew-on-Linux)
3. Install it under `~/.homebrew`
4. Write the file location to the `bashrc` file `export PATH="[home path]/.homebrew/bin:$PATH"`
5. Install like:

> brew install [package name]

# Create Python venv for development
1. Install python and python3-venv Package:
	Use the following command to install the python3-venv package:
	```bash
	sudo apt-get install python3.8-venv
	```
2. Recreate the Virtual Environment:
	Once you've installed the necessary package, try creating your virtual environment again:
	```bash
	python3.8 -m venv ml-isa-env
	[or] 
	pip3 install virtualenv
	virtualenv -p /usr/bin/python3.6 venv
	```
3. Activate the Virtual Environment:
	After successfully creating the virtual environment, you can activate it using:
	```bash
	Copy code
	source ml-isa-env/bin/activate
	```
	Your command prompt should change to indicate that you are now working inside the `ml-isa-env` virtual environment.
4. Deactivate Virtual Environment:
	Once you're done working in the virtual environment and want to go back to the global Python, you can deactivate it by simply typing:
	```bash
	deactivate
	```
	
5. if in the system there is no global python is installed, create a local python version to create the virtual env

```shell
[home name]@lance:~/PyEnv/ml-isa-env$ pyenv install 3.10
Downloading Python-3.10.13.tar.xz...
-> https://www.python.org/ftp/python/3.10.13/Python-3.10.13.tar.xz
Installing Python-3.10.13...
Installed Python-3.10.13 to /home/[home name]/.pyenv/versions/3.10.13
```

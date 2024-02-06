# differences between `--user` and '-U' in pip command

-U: the same as '--upgrade', if the requirement has been downlaoded, upgrade to the lastest version

-h: this command `pip install -h` can be used to refer different paramaters in pip command

--user: python download module packages to user packages  

# A way for scripting language to directly call C/C++ 

SWIG: Simplified Wrapper and Interface Generator. A packaging and interface generator tool that can generate calling interfaces of various cripting languages for C/C++ programs, so that programs written in C/C++ can be directly called through scripting languages.

# show all files and their size in KB/MB/GB
ls -lh
du [option] [file] 

option: h means human readable

# Sometimes using docker may encounter problem 

> System has not been booted with systemd as init system (PID 1). Can't operate 

+ In WSL
+ In autodl platform(AutoDL customer service, and the customer service said that the pytorch virtual environment instance created by AutoDL is a nividia-docker image, not a virtual machine, so you cannot install docker in docker (that is, you can't nest dolls), nor can you deploy K8s on that server.)

```cmd
root@autodl-container-0d2d11b33c-8b34c4ed:/home# sudo systemctl start docker
System has not been booted with systemd as init system (PID 1). Can't operate.
```
then change to 
```cmd
root@autodl-container-0d2d11b33c-8b34c4ed:/home# sudo /etc/init.d/docker start
* Starting Docker: docker
```
![docker in autodl failed](/commons/images/3141911-20230426171319198-961774676.png)

# Make a docker image of Linux system in use

1. pack the whole system
```shell
tar --numeric-owner --exclude=/proc --exclude=/sys -zcvpf [/data/centos7-svr.tar.gz] /
--numeric-owner: execute follow choices
--exclude: excude  
-zcvf: zip and put in a absolute path
```
2. import the image in a new machine

```shell
docker import [centos7-base.tar]  
```

3. test and run

```shell
docker run -it  --name  [DOCKER NAME] /bin/bash
```

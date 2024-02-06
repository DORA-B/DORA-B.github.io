NVIDIA-Driver in Linux Ubuntu
=============================

# Before all

DKMS stands for Dynamic Kernel Module Support. It is a framework that allows third-party kernel modules, such as the NVIDIA driver, to be automatically rebuilt when a new kernel is installed [1](https://docs.nvidia.com/datacenter/tesla/tesla-installation-notes/index.html). This way, the NVIDIA driver can remain compatible with the kernel and provide hardware acceleration for OpenGL and Vulkan applications [2](https://wiki.debian.org/NvidiaGraphicsDrivers).

The NVIDIA driver can be installed in different ways on Linux systems, such as using runfile installers, package managers, or containerized drivers [3](https://wiki.debian.org/NvidiaGraphicsDrivers). Some Linux distributions, such as Debian [4](https://confluence.slac.stanford.edu/display/SCSPub/nvidia-automatic-builds-via-dkms), Ubuntu [5] (https://wiki.debian.org/NvidiaGraphicsDrivers), and Arch Linux [6](https://www.jianshu.com/p/e562752cdbee), provide DKMS packages for the NVIDIA driver that can be installed using the system's package manager. This makes it easier to update the driver and the kernel without breaking the functionality of the NVIDIA graphics card [7](https://wiki.archlinux.org/title/NVIDIA).

If you want to install the NVIDIA driver on your Linux system, you should first identify your GPU model and the recommended driver version for it [8](https://wiki.debian.org/NvidiaGraphicsDrivers). You can use the nvidia-detect tool or the lspci command to do that [9](https://wiki.debian.org/NvidiaGraphicsDrivers). Then, you can follow the instructions for your specific Linux distribution and installation method [10](https://wiki.debian.org/NvidiaGraphicsDrivers). You may also need to install some prerequisites, such as kernel headers, build tools, and 32-bit libraries [11](https://wiki.debian.org/NvidiaGraphicsDrivers).

# Problem

Here is a problem when using `nvidia-smi `command in Ubuntu

```plaintext
nvidia-smi
NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver. Make sure that the latest NVIDIA driver is installed and running.
```

And usign lspci and lshw command:

```powershell
lspci | grep -i nvidia
03:00.0 VGA compatible controller: NVIDIA Corporation GM204GL [Quadro M5000] (rev a1)
03:00.1 Audio device: NVIDIA Corporation GM204 High Definition Audio Controller (rev a1)

lshw -C display
WARNING: you should run this program as super-user.
  *-display UNCLAIMED   
       description: VGA compatible controller
       product: GM204GL [Quadro M5000]
       vendor: NVIDIA Corporation
       physical id: 0
       bus info: pci@0000:03:00.0
       version: a1
       width: 64 bits
       clock: 33MHz
       capabilities: vga_controller bus_master cap_list
       configuration: latency=0
       resources: memory:f6000000-f6ffffff memory:e0000000-efffffff memory:f0000000-f1ffffff ioport:e000(size=128) memory:c0000-dffff
WARNING: output may be incomplete or inaccurate, you should run this program as super-user.
```

Try to reinstalling the kernel headers

```powershell
> sudo apt install linux-headers-$(uname -r)
linux-headers-5.15.0-86-generic is already the newest version (5.15.0-86.96~20.04.1).
linux-headers-5.15.0-86-generic set to manually installed.
> dkms status
nvidia, 450.119.03, 5.8.0-53-generic, x86_64: installed
```

Check kernels that have been installed:

```powershell
 dpkg --get-selections |grep linux-image
```

Check for drivers that have been already installed locally

```powershell
ls /usr/src | grep nvidia
nvidia-450.119.03
```

And

```powershell
sudo dkms install -m nvidia -v 450.119.03

Kernel preparation unnecessary for this kernel.  Skipping...
applying patch disable_fstack-clash-protection_fcf-protection.patch...patching file Kbuild
Hunk #1 succeeded at 84 (offset 13 lines).


Building module:
cleaning build area...
unset ARCH; [ ! -h /usr/bin/cc ] && export CC=/usr/bin/gcc; env NV_VERBOSE=1 'make' -j16 NV_EXCLUDE_BUILD_MODULES='' KERNEL_UNAME=5.15.0-86-generic IGNORE_XEN_PRESENCE=1 IGNORE_CC_MISMATCH=1 SYSSRC=/lib/modules/5.15.0-86-generic/build LD=/usr/bin/ld.bfd modules......(bad exit status: 2)
Error! Bad return status for module build on kernel: 5.15.0-86-generic (x86_64)
Consult /var/lib/dkms/nvidia/450.119.03/build/make.log for more information.
```

reboot, Problem still exists, amd according to the answers, the error below seems a common situation.

![dkms nvidia build wrong](/commons/images/3141911-20240104120319329-1996713054.png)

> dpkg -l | grep nvidia

List the software installed on the current system and the status of the software packages
![dpkg list to see status](/commons/images/3141911-20240104121355019-1625341005.png)
![see nvidia-related packages](/commons/images/3141911-20240104121623666-2064261766.png)

it seems there is no incompatibility.

> ubuntu-drivers devices

```cmd
== /sys/devices/pci0000:00/0000:00:02.0/0000:03:00.0 ==
modalias : pci:v000010DEd000013F0sv000010DEsd00001152bc03sc00i00
vendor   : NVIDIA Corporation
model    : GM204GL [Quadro M5000]
driver   : nvidia-driver-525-server - distro non-free
driver   : nvidia-driver-525 - distro non-free
driver   : nvidia-driver-450-server - distro non-free
driver   : nvidia-driver-535 - distro non-free recommended
driver   : nvidia-driver-418-server - distro non-free
driver   : nvidia-driver-470-server - distro non-free
driver   : nvidia-driver-535-server - distro non-free
driver   : nvidia-driver-390 - distro non-free
driver   : nvidia-driver-470 - distro non-free
driver   : xserver-xorg-video-nouveau - distro free builtin
```

> sudo ubuntu-drivers autoinstall

```cmd
Reading package lists... Done
Building dependency tree     
Reading state information... Done
Some packages could not be installed. This may mean that you have
requested an impossible situation or if you are using the unstable
distribution that some required packages have not yet been created
or been moved out of Incoming.
The following information may help to resolve the situation:

The following packages have unmet dependencies:
 nvidia-driver-535 : Depends: libnvidia-gl-535 (= 535.129.03-0ubuntu0.20.04.1) but it is not going to be installed
                     Depends: nvidia-kernel-source-535 (= 535.129.03-0ubuntu0.20.04.1) but it is not going to be installed
                     Depends: libnvidia-compute-535 (= 535.129.03-0ubuntu0.20.04.1) but it is not going to be installed
                     Depends: libnvidia-extra-535 (= 535.129.03-0ubuntu0.20.04.1) but it is not going to be installed
                     Depends: nvidia-compute-utils-535 (= 535.129.03-0ubuntu0.20.04.1) but it is not going to be installed
                     Depends: libnvidia-decode-535 (= 535.129.03-0ubuntu0.20.04.1) but it is not going to be installed
                     Depends: libnvidia-encode-535 (= 535.129.03-0ubuntu0.20.04.1) but it is not going to be installed
                     Depends: nvidia-utils-535 (= 535.129.03-0ubuntu0.20.04.1) but it is not going to be installed
                     Depends: xserver-xorg-video-nvidia-535 (= 535.129.03-0ubuntu0.20.04.1) but it is not going to be installed
                     Depends: libnvidia-cfg1-535 (= 535.129.03-0ubuntu0.20.04.1) but it is not going to be installed
                     Depends: libnvidia-fbc1-535 (= 535.129.03-0ubuntu0.20.04.1) but it is not going to be installed
                     Recommends: libnvidia-compute-535:i386 (= 535.129.03-0ubuntu0.20.04.1)
                     Recommends: libnvidia-decode-535:i386 (= 535.129.03-0ubuntu0.20.04.1)
                     Recommends: libnvidia-encode-535:i386 (= 535.129.03-0ubuntu0.20.04.1)
                     Recommends: libnvidia-fbc1-535:i386 (= 535.129.03-0ubuntu0.20.04.1)
                     Recommends: libnvidia-gl-535:i386 (= 535.129.03-0ubuntu0.20.04.1)
E: Unable to correct problems, you have held broken packages.
```


# Reinstall
sudo apt --purge -y remove "cuda*"
![apt list show](/commons/images/3141911-20240205114050311-1873117247.png)


remove all the versions and dependencies of nvidia

```
sudo apt-get remove --purge '^nvidia-.*'
sudo apt-get remove --purge '^libnvidia-.*'
sudo apt-get remove --purge '^cuda-.*'
```
After cleanup of the nvidia, show dpkg -l, and make sure they are all cleaned, finally reinstalled.

```cmd
sudo apt-get autoremove
sudo apt-get autoclean
sudo apt-get update
sudo ubuntu-drivers autoinstall  # This will install the recommended driver
```
reboot and after that:

![nvidia-smi command show](/commons/images/3141911-20240205114113626-1289358686.png)


and show nvidia from lsmod:

```cmd
lsmod | grep nvidia
nvidia_uvm           1548288  0
nvidia_drm             94208  3
nvidia_modeset       1327104  5 nvidia_drm
nvidia              56180736  63 nvidia_uvm,nvidia_modeset
drm_kms_helper        307200  1 nvidia_drm
drm                   618496  7 drm_kms_helper,nvidia,nvidia_drm
```

Conclusion: low version of nvidia driver and not compatible with kernel, so need to reinstall all the lib*, cuda*, and nvidia-driver*.
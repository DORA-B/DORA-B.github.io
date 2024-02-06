<center><font size="5">1. Why WASM containerzation in HPC systems recommended in the paper in the "privilege aspect"</font></center>
<center><font size="5">2. How functions in Wasm be implemented"</font></center>

---

[TOC]

---

paper can be accessed here: 
> https://dl.acm.org/doi/10.1145/3572848.3577436

# Abstract from paper

Several HPC-focused containerization solutions, such as **Charliecloud [72], Shifter [49], Singularity [58], Podman [48], and Sarus [33]** have been introduced.

In contrast to previous approaches, this paper investigates using a new novel technology called WebAssembly (Wasm) [52], dubbed as an **alternative** to Linux containers [81], for packaging and distributing HPC applications

While HPC-focused containerization solutions such as Singularity [58] and Podman [48] support **rootless-containers** through fakeroot [61], their current implementations do not support distributed filesystems such as GPFS commonly found on HPC systems [70, 80].

Moreover, as argued by [71], building Open Container Initiative (OCI) [69] compliant container images on HPC resources by unprivileged (normal) users where the applications will eventually run is significantly hard and requires support from the supercomputing center. 

This is because most container building solutions such as Docker [43] also require root privileges. As a result, most users use their own local systems for building/developing their application container images and then transfer the built image to a login/front-end node of an HPC system for execution.

## Why choose WASM as an alternative in the privilege aspect

==In this section, only the Privilege advantages of using WASM have been extracted, that means using MPIWasm as an embedder has a better solution to the privilege problem==

# What is fakeroot

Fakeroot is a feature that allows an unprivileged user to run a container as a “fake root” user by leveraging user namespace UID/GID mapping. This means that the user can have almost the same administrative rights as root but only inside the container and the requested namespaces


The implementation of fakeroot in Singularity and Podman allows users to run containers as a "fake root" user without requiring actual root privileges on the host system, by utilizing user namespace UID/GID mapping. Here's a detailed explanation:

## Singularity:
+ Command Configuration:
  + A command `singularity config fakeroot` was introduced in Singularity version 3.5 that allows the configuration of `/etc/subuid` and `/etc/subgid` mappings from the Singularity command line. However, to use this command, one must have root access or use sudo since the mapping files are security-sensitive​​.
+ Rootless Mode:
  + Referred to as "rootless mode" or "fakeroot feature", this feature allows an unprivileged user to run a container as a "fake root" user by leveraging user namespace UID/GID mapping. This means a "fake root" user has almost the same administrative rights as root but only within the container​​.
+ User Namespace Requirements:
  + When `--fakeroot` is used, it impersonates a root user when building or running a container. For unprivileged creation of user namespaces, a kernel version of >=3.8 is required, with >=3.18 being recommended due to security fixes for user namespaces​4​.

## Podman:
+ Rootless Mode:

  + Similar to Singularity, Podman also implements a rootless mode which requires user namespaces to be enabled. This mode provides a complete emulation but requires administrator setup​.

+ Fakeroot Feature:
  + The `--fakeroot` option is available on many functions, allowing an unprivileged user to run a container with the appearance of running as root. This feature operates in multiple different ways depending on what is available on the system​.

+ RootlessKit:
  + RootlessKit is used for the fakeroot implementation in Podman to support rootless mode. By default, RootlessKit uses the builtin port forwarding driver, which does not propagate source IP addresses​​.
+ User Namespace Support:
  + Podman offers User Namespace support, allowing running containers without requiring root privileges. This is a part of the mechanism that enables the fakeroot functionality​.

## Shortcomings:

1. Dependency on System Configuration:
  + Both Singularity and Podman depend on system configurations like enabling user namespaces, which might require administrator privileges.

2. Limited Scope of Privileges:
+ The "fake root" user has almost the same administrative rights as root but only within the container, which could be a limitation based on the use case.
3. Kernel Version Dependency:
+ In Singularity, enabling the --fakeroot option requires specific kernel versions due to security considerations.
These implementations help in enhancing security by not requiring actual root privileges on the host system while still allowing necessary operations within the container environment.

# Materials

+ For podman: Unsupported file systems in rootless mode
> https://docs.podman.io/en/latest/markdown/podman.1.html#rootless-mode

+ For Sigularity: Fakeroot / (sub)uid/gid mapping

> https://apptainer.org/admin-docs/master/installation.html 

# Differences between WASM and C in Function aspect

In the assembly produced by C programs, where a function call is expressed as a jump instruction to the address of the function’s first instruction, a typical exploit is to change this address to take control of the program’s control flow. However, such exploits are not possible with Wasm since it features control flow integrity by enforcing structured program control flow. This is because of two reasons. 

+ First, in Wasm, a function is represented as an index in a table which adds an additional level of indirection to express the function address. 

+ Second, the Wasm specification prevents constructing arbitrary memory addresses [42] and the separation of the embedder and the module’s memory prevents overwriting function instructions.

![function call in wasm](/commons/images/3141911-20231006040836962-653564373.png)

+ You can find how Webassmebly's exported functions ==[here](https://developer.mozilla.org/en-US/docs/WebAssembly/Exported_functions)==

+ You can find how Webassmebly's functions are called ==[here](https://coinexsmartchain.medium.com/wasm-introduction-part-6-table-indirect-call-65ad0404b003)==


# Some References
[72] Reid Priedhorsky and Tim Randles. 2017. Charliecloud: Unprivileged containers for user-defined software stacks in hpc. In Proceedings of the international conference for high performance computing, networking, storage and analysis. 1–10. https://doi.org/10.1145/3126908.3126925

[49] Lisa Gerhardt, Wahid Bhimji, Shane Canon, Markus Fasel, Doug Jacobsen, Mustafa Mustafa, Jeff Porter, and Vakho Tsulaia. 2017. Shifter: Containers for hpc. In Journal of physics: Conference series, Vol. 898. IOP Publishing, 082021. https://doi.org/10.1088/17426596/898/8/082021

[58] Gregory M Kurtzer, Vanessa Sochat, and Michael W Bauer. 2017. Singularity: Scientific containers for mobility of compute. PloS one 12, 5 (2017), e0177459. https://doi.org/10.1371/journal.pone.0177459

[48] Holger Gantikow, Steffen Walter, and Christoph Reich. 2020. Rootless Containers with Podman for HPC. In International Conference on High Performance Computing. Springer, 343–354. https://doi.org/10.1007/ 978-3-030-59851-8_23

[33] Lucas Benedicic, Felipe A Cruz, Alberto Madonna, and Kean Mariotti. 2019. Sarus: Highly scalable docker containers for hpc systems. In International Conference on High Performance Computing. Springer, 46–60. https://doi.org/10.1007/978-3-030-34356-9_5

[61] Linux. [n.d.]. Fakeroot. https://wiki.debian.org/FakeRoot. Accessed 09/27/2021

[71] Podman. [n.d.]. Filesystem Considerations in Rootless Mode. https: //tinyurl.com/2p8efntt. Accessed 09/27/2021.

[80] Singularity. [n.d.]. Filesystem Considerations in Rootless Mode. https://apptainer.org/admin-docs/master/installation.html. Accessed 09/27/2021.

[69] Open Container Initiative (OCI). [n.d.]. https://opencontainers.org/. Accessed 07/07/2022.

[81] Solomon Hykes. [n.d.]. Wasm+WASI: An alternative to Linux Containers. https://twitter.com/solomonstre/status/1111004913222324225? lang=en. Accessed 09/27/2021.

[52] Andreas Haas, Andreas Rossberg, Derek L. Schuff, Ben L. Titzer, Michael Holman, Dan Gohman, Luke Wagner, Alon Zakai, and JF Bastien. 2017. Bringing the Web up to Speed with WebAssembly. SIGPLAN Not. 52, 6 (June 2017), 185–200. https://doi.org/10.1145/ 3140587.3062363

[42] Frank Denis. [n.d.]. Memory management in WebAssembly: guide for C and Rust programmers. https://www.fastly.com/blog/webassemblymemory-management-guide-for-c-rust-programmers
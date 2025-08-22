---
title: Qemu introduction
date: 2025-03-27
categories: [Qemu]
tags: [qemu]     # TAG names should always be lowercase
published: false
---

> Problems: Have used qemu for a long time, but need to implement the code in it so this should be a supplement for finishing this project

# Materials
- [Emulation in QEMU](https://www.qemu.org/docs/master/about/emulation.html)
  - SemiHosting
  - TCCG Plugins
- [User Mode Emulation](https://www.qemu.org/docs/master/user/main.html)
- Developer Information
  - [Internal QEMU APIs](https://www.qemu.org/docs/master/devel/index-api.html)
  - [TCG Emulation](https://www.qemu.org/docs/master/devel/index-tcg.html)
  - [Code base](https://www.qemu.org/docs/master/devel/codebase.html)

# Differences between LLVM and QEMU

## Core Architecture differences 

LLVM is a compiler infrastructure framework and QEMU is a machine emulator and virtualization for running binaries for different architectures (emulation).



| Feature | **LLVM** | **QEMU** |
| --- | --- | --- |
| **Frontend** | Parses high-level code (e.g., C/C++) → LLVM IR | Loads binary (already compiled code) |
| **IR / Intermediate** | Uses LLVM IR (typed SSA-based intermediate representation) | Uses TCG (Tiny Code Generator) IR or guest instruction representation |
| **Backend** | Translates LLVM IR → target machine code | Translates guest machine code → host machine code |
| **JIT Support** | Yes (e.g., LLVM JIT or MCJIT) | Yes (QEMU uses dynamic binary translation, a form of JIT) |

## Differences between LLVM-IR (SSA-based) and Qemu's TCG IR


### Design Philosophy


| Feature | **LLVM IR** | **QEMU TCG IR** |
| --- | --- | --- |
| **Design Goal** | Optimize source-level code at compile-time | Translate guest binary to host binary at runtime (DBT) |
| **Level** | **High-level**, close to source-level but target-aware | **Low-level**, very close to machine instructions |
| **Typing** | **Strongly typed** (SSA-based, with i8, i32, float, etc.) | **Weakly typed or untyped**, mostly word-sized (host-reg-sized) |
| **SSA** | Full SSA form with registers as versioned variables | Partial or no SSA, variables are temporary registers |



### Expressiveness and Abstractions

| Feature | **LLVM IR** | **QEMU TCG IR** |
| --- | --- | --- |
| **Control Flow** | Full support (branches, loops, dominators, phi-nodes) | Very basic (jumps, conditional jumps) |
| **Data Types** | Rich: integers, floats, vectors, pointers | Primitive: mostly integers (machine-word sized), no float/vector types |
| **Memory Model** | Abstract (alias analysis, load/store analysis) | Concrete (explicit guest memory load/store instructions) |
| **Function Calls** | Yes, with argument passing and return values | No real concept of calls — it’s linear translation of instructions |
| **Optimization Support** | Designed for aggressive passes (e.g., GVN, loop unrolling, inlining, etc.) | Minimal; TCG optimizes for speed of **translation**, not execution |


### Performance and Use Case Differences

| Aspect | **LLVM IR** | **TCG IR (QEMU)** |
| --- | --- | --- |
| **Targeted for** | AOT/JIT compilers, static analyzers | Runtime dynamic binary translation |
| **Speed of Translation** | Slower (lots of analysis, global info) | **Fast**, linear pass; can translate millions of guest instructions quickly |
| **Speed of Execution** | High, especially after heavy optimization | Decent; but TCG focuses more on correctness and fast translation |
| **Debug/Instrumentation** | Rich, can insert high-level debug info | Harder — very low-level and machine-oriented |



# Compile Qemu docs

- Env preparation (meson, ninja, )
```shell
pip3 install sphinxcontrib-blockdiag sphinxcontrib-seqdiag
```
- When doing configuration, enable docs
```shell
mkdir buildocs && cd buildocs
../configure --enable-docs
make html
``` 

# About QEMU (concise)


QEMU is consist of System Emulation and User Mode Emulation

## Differences between System Emulation and User Mode Emulation
| Feature | **User-mode emulation** | **System-mode emulation** |
| --- | --- | --- |
| Emulates | Only a user-level binary (e.g., an ARM executable) | An entire machine/system: CPU + RAM + devices + BIOS |
| Guest OS | ❌ No guest OS — it uses the **host OS kernel** | ✅ Yes — boots a full guest OS (e.g., Linux kernel, Windows) |
| Use case | Running foreign-arch binaries on native host (e.g., `arm` on `x86`) | Full VM emulation or virtualization |
| Kernel emulation | ❌ No — syscalls are **translated to host syscalls** | ✅ Yes — guest kernel and drivers are fully emulated |
| Speed | Faster for individual programs | Slower, more flexible |
| Example | `qemu-arm ./hello_arm` | `qemu-system-arm -kernel zImage ...` |


### Understanding of emulation

> Cannot run an ARM binary inside an x86-32 system emulation (on an x86_64 host), even with a TCG. Here's the precise technical breakdown:

1. Architecture Isolation in System Emulation
- `qemu-system-i386` emulates x86 hardware, which:
  - Only understands x86 instructions
  - Has no capability to execute ARM binaries, even with TCG
- TCG translates guest→host instructions, but:
  - The guest OS (x86-32) lacks ARM ELF loader/ABI support
  - The emulated x86 CPU has no ARM execution mode

- **Scenario**: x86_64 host → x86-32 guest → ARM binary

|Feature|User-Mode (qemu-arm)|System-Mode (qemu-system-arm)|
| --- | --- | --- |
|Cross-arch execution | Yes (ARM binary on x86_64 host) | No (ARM guest OS required)|
|TCG Usage | Translates ARM→x86_64 | Translates ARM→x86_64 for entire VM|
|Binary Compatibility | Runs ARM binaries directly | Requires ARM kernel + userspace|


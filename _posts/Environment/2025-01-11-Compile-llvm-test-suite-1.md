---
title: Compile-llvm-test-suit-story-1
date: 2025-01-11
categories: [Environment]
tags: [linux]     # TAG names should always be lowercase
published: false
---

> Problem1: How to compile the compile llvm test suite using the self-defined methods
> Problem2: When use arm-linux-gnueabihf to compile, there exists `vectoerized` instructions, which is **not** good for translation
> Problem3: How to use musl to compile the benchmark? (in order to get a smaller library)

- Compiler: arm-linux-gnueabihf/arm-linux-gnueabi

## Resources
- [test-suite Guide, Configuration](https://llvm.org/docs/TestSuiteGuide.html)
- [Github link](https://github.com/llvm/llvm-test-suite.git)

## Instructions to build a single benchmark
```bash

mkdir ../llvm-test-suite-build
cd ../llvm-test-suite-build

--sysroot=/usr/arm-linux-gnueabi

# Fine-Grained control
cmake \
  -DCMAKE_C_COMPILER=arm-linux-gnueabihf-gcc \
  -DCMAKE_CXX_COMPILER=arm-linux-gnueabihf-g++ \
  -DCMAKE_C_FLAGS="--sysroot=[/path/to/sysroot] -fno-if-conversion -fno-if-conversion2 -fno-tree-loop-if-convert -ffunction-sections -fdata-sections" \
  -DCMAKE_CXX_FLAGS="--sysroot=[/path/to/sysroot] -fno-if-conversion -fno-if-conversion2 -fno-tree-loop-if-convert -ffunction-sections -fdata-sections" \
  -DTEST_SUITE_SUBDIRS="SingleSource/Benchmarks/McGill" \
  ../llvm-test-suite

```

## Benchmarks introduction 

However, the gotten benchmarks in the directory include all the different benchmarks already, cannot get one subbenchmark out.
```bash
├── Bitcode
├── build_gnueabi
├── CMakeCache.txt
├── CMakeFiles
├── cmake_install.cmake
├── External
├── lit.cfg
├── lit.site.cfg
├── litsupport
├── Makefile
├── MicroBenchmarks 
├── MultiSource
├── SingleSource
└── tools
```

- `SingleSource/`
  Contains test programs that are only a single source file in size. A subdirectory may contain several programs.

- `MultiSource/`
  Contains subdirectories which entire programs with multiple source files. Large benchmarks and whole applications go here.

- `MicroBenchmarks/`
  Programs using the google-benchmark library. The programs define functions that are run multiple times until the measurement results are statistically significant.

- `External/`
  Contains descriptions and test data for code that cannot be directly distributed with the test-suite. The most prominent members of this directory are the SPEC CPU benchmark suites. See External Suites.

- `Bitcode/`
  These tests are mostly written in LLVM bitcode.

- `CTMark/`
  Contains symbolic links to other benchmarks forming a representative sample for compilation performance measurements.



## qemu 

```bash
~/qemu-8.0.0/build_arm32/qemu-arm -d in_asm,out_asm -L /usr/arm-linux-gnueabihf queens > queens.qlog 2>&1
~/qemu-8.0.0/build_arm32/qemu-arm -d in_asm,out_asm -L /usr/arm-linux-gnueabi helloarm > helloarm.qlog 2>&1

~/qemu-8.0.0/build_arm32/qemu-arm -d in_asm,out_asm queens > queens.qlog 2>&1
```

cmake \
  -DCMAKE_C_COMPILER=/usr/local/musl-arm/bin/musl-gcc \
  -DCMAKE_C_FLAGS="--sysroot=/usr/local/musl-arm/ -fno-if-conversion -fno-if-conversion2 -fno-tree-loop-if-convert -ffunction-sections -fdata-sections" \
  -DTEST_SUITE_SUBDIRS="SingleSource/Benchmarks/McGill" \
  ../llvm-test-suite

cmake \
  -DCMAKE_C_COMPILER=/usr/local/musl-arm/bin/musl-gcc \
  -DCMAKE_C_FLAGS="--sysroot=/usr/local/musl-arm/ -fno-if-conversion -fno-if-conversion2 -fno-tree-loop-if-convert -ffunction-sections -fdata-sections" \
  -DCMAKE_EXE_LINKER_FLAGS="-static" \
  -DTEST_SUITE_SUBDIRS="SingleSource/Benchmarks/McGill" \
  ../llvm-test-suite

cmake \
  -DCMAKE_C_COMPILER=arm-linux-gnueabi-gcc \
  -DCMAKE_C_FLAGS="--sysroot=/usr/arm-linux-gnueabi/ -fno-if-conversion -fno-if-conversion2 -fno-tree-loop-if-convert -ffunction-sections -fdata-sections" \
  -DCMAKE_EXE_LINKER_FLAGS="-static" \
  -DTEST_SUITE_SUBDIRS="SingleSource/Benchmarks/McGill" \
  ../llvm-test-suite
  
cmake \
  -DCMAKE_C_COMPILER=arm-linux-gnueabi-gcc \
  -DCMAKE_C_FLAGS="--sysroot=/usr/arm-linux-gnueabi/ -fno-if-conversion -fno-if-conversion2 -fno-tree-loop-if-convert -Werror=conditionally-supported -ffunction-sections -fdata-sections" \
  -DCMAKE_EXE_LINKER_FLAGS="-static" \
  -DTEST_SUITE_SUBDIRS="SingleSource/Benchmarks/McGill" \
  ../llvm-test-suite

cmake \
  -DCMAKE_C_COMPILER=gcc \
  -DCMAKE_C_FLAGS="-m32 -fno-if-conversion -fno-if-conversion2 -fno-tree-loop-if-convert -ffunction-sections -fdata-sections" \
  -DCMAKE_EXE_LINKER_FLAGS="-static" \
  -DTEST_SUITE_SUBDIRS="SingleSource/Benchmarks/McGill" \
  ../llvm-test-suite

cmake \
  -DCMAKE_C_COMPILER=gcc \
  -DCMAKE_C_FLAGS="-m32 -fno-if-conversion -fno-if-conversion2 -fno-tree-loop-if-convert -ffunction-sections -fdata-sections --sysroot=/usr/include/i386-linux-gnu" \
  -DCMAKE_EXE_LINKER_FLAGS="-static -m32" \
  -DTEST_SUITE_SUBDIRS="SingleSource/Benchmarks/McGill" \
  ../llvm-test-suite

## glibc file used by one compiler



>  [**_NOTE:_**]
> ar [OPTIONS] archive_name member_files
Here, OPTIONS are flags that define the action want to perform, 'archive_name' is the name of the archive file want to create or modify, and 'member_files' are the individual files wish to include in the archive.


- the libc.a used by compilation option `-m32`
```shell
cc -m32 -print-file-name=libc.a
/usr/lib/gcc/x86_64-linux-gnu/7/../../../../lib32/libc.a
# Extract memcpy's object file (usually in memcpy.o or memcpy.S.o)
ar x $(gcc -m32 -print-file-name=libc.a) memcpy.o
```
- the libc.a used by cross compilation option `gnueabi-gcc` 
```shell
(ml-isa-bard) qingchen@lance:~$ arm-linux-gnueabi-gcc -print-file-name=libc.a
/usr/lib/gcc-cross/arm-linux-gnueabi/7/../../../../arm-linux-gnueabi/lib/../lib/libc.a

# find the memcpy 
(ml-isa-bard) qingchen@lance:~$ ar t $(arm-linux-gnueabi-gcc -print-file-name=libc.a) | grep memcpy
aeabi_memcpy.o
memcpy.o
wmemcpy.o
memcpy_chk.o
wmemcpy_chk.o
ar x $(arm-linux-gnueabi-gcc -print-file-name=libc.a) memcpy.o
```

### Try to find the memcpy func assembled in the libc.so

```shell
qingchen@lance:~/Bench/glibc-2.27$ arm-linux-gnueabi-readelf -s /usr/arm-linux-gnueabi/lib/libc.so.6 | grep memcpy
   348: 00082e5c     8 FUNC    WEAK   DEFAULT   12 wmemcpy@@GLIBC_2.4
  1048: 000eb49c    24 FUNC    GLOBAL DEFAULT   12 __wmemcpy_chk@@GLIBC_2.4
  1168: 0007bda0   668 FUNC    GLOBAL DEFAULT   12 memcpy@@GLIBC_2.4
  1306: 00017474     4 FUNC    GLOBAL DEFAULT   12 __aeabi_memcpy4@@GLIBC_2.4
  1314: 00017474     4 FUNC    GLOBAL DEFAULT   12 __aeabi_memcpy8@@GLIBC_2.4
  1686: 000e99b8    20 FUNC    GLOBAL DEFAULT   12 __memcpy_chk@@GLIBC_2.4
  1734: 00017474     4 FUNC    GLOBAL DEFAULT   12 __aeabi_memcpy@@GLIBC_2.4

qingchen@lance:~/Bench/glibc-2.27$ arm-linux-gnueabi-objdump -d --start-address=0x7bda0 --stop-address=0x7C03C /usr/arm-linux-gnueabi/lib/libc.so.6 | less

# the glibc version used by gnueabi-gcc
qingchen@lance:~/Bench$ strings /usr/arm-linux-gnueabi/lib/libc.so.6 | grep "GNU C Library"
GNU C Library (Ubuntu GLIBC 2.27-3ubuntu1) stable release version 2.27.
```

- The same as what can be found in gnueabi, tthe glibc in gcc 32 bit is like
```shell
(ml-isa-bard) qingchen@lance:~$ gcc -m32 -print-file-name=libc.so.6
/lib/i386-linux-gnu/libc.so.6
(ml-isa-bard) qingchen@lance:~$ readelf -s /lib/i386-linux-gnu/libc | grep memcpy
libc-2.27.so      libcidn-2.27.so   libcidn.so.1      libcrypt-2.27.so  libcrypt.so.1     libc.so.6     

# i386 lib need to be installed: `sudo apt install linux-libc-dev:i386`
(ml-isa-bard) qingchen@lance:~$ readelf -s /lib/i386-linux-gnu/libc.so.6 | grep memcpy
   378: 0009a260    41 FUNC    WEAK   DEFAULT   13 wmemcpy@@GLIBC_2.0
   383: 00085ee0    37 FUNC    GLOBAL DEFAULT   13 __memcpy_by2@GLIBC_2.1.1
   392: 00085ee0    37 FUNC    GLOBAL DEFAULT   13 __memcpy_by4@GLIBC_2.1.1
   906: 00085ee0    37 FUNC    GLOBAL DEFAULT   13 __memcpy_c@GLIBC_2.1.1
   922: 00085ee0    37 FUNC    GLOBAL DEFAULT   13 __memcpy_g@GLIBC_2.1.1
  1136: 00108750    61 FUNC    GLOBAL DEFAULT   13 __wmemcpy_chk@@GLIBC_2.4
  1267: 0007fd10    67 IFUNC   GLOBAL DEFAULT   13 memcpy@@GLIBC_2.0
  1832: 00107260    67 IFUNC   GLOBAL DEFAULT   13 __memcpy_chk@@GLIBC_2.3.4
```

## Files comparison
- Files that compiled from glibc:
```bash
(ml-isa-bard) qingchen@lance:~/Bench/single-mcgill-build/build_gnueabi/int$ strings chomp | grep "libc"
/lib/ld-linux.so.3
libc.so.6
__stack_chk_fail
calloc
malloc
__libc_start_main
ld-linux.so.3
  value = %d
Mode : 1 -> multiple first moves
 Selection : 
Enter number of Columns : 
player %d plays at (%d,%d)
player %d loses
/usr/lib/gcc-cross/arm-linux-gnueabi/7/../../../../arm-linux-gnueabi/lib/../lib/crt1.o
/usr/lib/gcc-cross/arm-linux-gnueabi/7/../../../../arm-linux-gnueabi/lib/../lib/crti.o
call_weak_fn
/usr/lib/gcc-cross/arm-linux-gnueabi/7/../../../../arm-linux-gnueabi/lib/../lib/crtn.o
dump_play.part.1
deregister_tm_clones
__do_global_dtors_aux
completed.9929
__do_global_dtors_aux_fini_array_entry
elf-init.oS
__libc_csu_fini
calloc@@GLIBC_2.4
show_list
__stack_chk_fail@@GLIBC_2.4
make_play
make_list
malloc@@GLIBC_2.4
__libc_start_main@@GLIBC_2.4
dump_play
__dso_handle
equal_data
dump_list
__libc_csu_init
show_play
ncol
get_value
melt_data
get_real_move
valid_data
.note.gnu.build-id
.rel.dyn
.rel.plt
.data.rel.ro
```
- Compiled from musl
```bash
(ml-isa-bard) qingchen@lance:~/Bench/musl-mcgill/SingleSource/Benchmarks/McGill$ strings chomp | grep "libc"
__libc_start_main.c
libc.c
__init_libc
__libc
__libc_start_init
__libc_exit_fini
__libc_start_main
```



Seems there are more distinct functions and basic blocks in a statically linked **musl** binary, even though the overall file size is smaller compared to a **glibc**-linked binary. In other words, “musl results in smaller total bytes on disk but a _larger_ function/block count” is not a contradiction—it stems from the way musl organizes its code, how the compiler identifies function boundaries, and how your tools are enumerating them.

* * *

1) Musl vs. Glibc Linking Style
-------------------------------

1. **Musl** is often written as a collection of many small, straightforward C routines that call each other.
    
    * So internally, musl might implement certain math or string routines in multiple short helper functions.
    * This can show up as _more named or discovered subroutines_ in the final binary.
2. **Glibc** has some large, monolithic (or assembly-optimized) implementations.
    
    * Some “helper” logic may be inlined or placed in hidden static sections that are not discovered as separate functions.
    * So you end up with fewer top-level function symbols, but each might be bigger or contain more logic internally.

Hence, the static linking with musl might result in smaller total `.text` size on disk (because musl is typically smaller and more straightforward), but that code is arranged in more small pieces. Meanwhile glibc may have fewer visible function “entry points” in your final binary.

* * *

2) Symbol Discovery and Basic Block Counting
--------------------------------------------

**Angr** or your disassembler will pick out “function boundaries” by looking for typical function prologues, cross‐references, or symbol tables. Musl’s implementation style can produce:

* Many small subroutines for floating-point conversions: e.g. `__aeabi_d2iz`, `__aeabi_dadd`, `__aeabi_cdcmpeq`, etc.
* Routines for memory or string operations each broken up into specialized helpers.
* Extra “internal” or “helper” routines that do `futex()` calls, memory expansions, etc.

That means your tool sees more function entry points and enumerates them as separate functions or separate blocks. Conversely, in glibc:

* Some routine might be all in one large `.o` or combine multiple logic paths behind fewer exported function names.
* Or it might do a lot of inlining (especially in the hand‐optimized assembly routines), so your analysis sees one single bigger block.

**Hence** the BFS or function-lister ends up with more total “function” records in the musl binary, and each function can also be further split into more small basic blocks.

* * *

3) Why the Binary Size Can Still Shrink
---------------------------------------

1. **Musl** tries to keep each subcomponent fairly minimal.
2. **Glibc** can contain big extension points, versioned symbols, locale expansions, all sorts of fallback paths, etc.
3. Even though musl’s approach may define _more subroutines_, each routine might be smaller or more specialized. Summed up, the total `.text + .data` sections can still be smaller.

Hence you get:

> * **Smaller total file size**: Because the code itself is overall simpler/lighter.
> * **Greater function/block count**: Because the code is chunked into many little routines rather than fewer, bigger lumps.

* * *

4) Typical Patterns That Increase Musl’s Function Count
-------------------------------------------------------

* **Floating-point “AEABI” routines** on ARM (e.g. `__aeabi_d2iz`, `__aeabi_dadd`, etc.):  
    Glibc sometimes merges these or provides them via a single piece of hand‐rolled assembly, so the disassembler identifies fewer “function” boundaries. Musl might expose them as multiple separate symbols.
* **Basic I/O** (like `printf_core`, `vfprintf`, etc.):**  
    Musl’s `printf` can be quite modular, so you see “helpers” for integer printing (`fmt_u`), floating printing (`decfloat`), “pop_arg`, etc.
* **Memory routines** (`malloc`, `free`, `realloc`)**  
    Musl’s allocator calls smaller sub-helpers (`alloc_fwd`, `alloc_rev`, `__expand_heap`, etc.) that show up as distinct functions.
* **Locking and concurrency** (like `futex`, `__vm_wait`, `__a_cas`, etc.)**  
    Musl tries to do minimal locking but in multiple small helper files. Glibc might inline or unify them.

* * *

5) Conclusion
-------------

Seeing **more** “function: block number: ...” entries in your log does not mean musl is “bigger” in the sense of code footprint or memory usage. It simply means the _analysis sees more separate subroutines_ because of the _coding style and linker symbol arrangement._ In reality, you likely still see a smaller overall ELF file with musl. That is perfectly normal and expected when switching from glibc to musl for static linking.

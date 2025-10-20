---
title: Binary Format and Analysis with linking information
date: 2025-10-17
categories: [Compiler]
tags: [arm, compiler]     # TAG names should always be lowercase
published: false
---


> Reason: In analyzing the qemu's (in,out asm) log, and the process from assembly code (PIC) to object code (Relocatable), I want to know how these mapped information is kept for compiler to get them, because I need to create a mapping among them.

# A description of the assembling process

- Assembly source is often written as position-independent code (PIC/PIE-friendly); the produced object file (.o) is a relocatable object that contains relocations (not fixed addresses). The final PIE executable is linked to be position-independent and may use GOT/PLT and PC-relative relocations.
- Labels in assembly (basic-block labels, local/global symbols, data labels) are translated by the assembler into symbol table entries and relocation records when necessary. References to those labels become either immediate offsets (if fully resolvable by the assembler) or relocations that the linker/loader will fix up, often as PC-relative references for PIC/PIE.

## Short contract 

- Inputs: assembly source with labels and references.
- Outputs: object (.o) with symbol table and relocations; final PIE/shared or non-PIE executable with resolved addresses.
- Error modes: assembler can resolve intra-section forward/backward label references without relocations; external/absolute addresses produce relocations; link-time/loader errors if symbols unresolved.

## step-by-step

1. Assembly -> object (.o) by the assembler:

- The assembler converts each label into a symbol and records its value relative to the section (e.g., .text offset 0x20).
- For a reference to a symbol:
    - If it's in the same section and the assembler can compute the offset (e.g., a forward/backward branch using PC-relative encodings that fit), the assembler may encode the immediate offset directly — sometimes creating a relocation if it can't finalize the value for linking or if the reference needs a specific relocation type.
    - If the reference targets an external symbol, a different section, or requires a relocation type (e.g., 32-bit absolute), the assembler emits a relocation entry in the .rel/.rela section pointing to the symbol.
- The resulting .o is relocatable: it contains sections, symbols, and relocation entries. It has no single fixed virtual addresses (only section-relative offsets).
2. Linker -> final binary (executable, PIE, or shared object):
- The linker lays out sections into memory addresses and processes relocations. For a PIE or shared object, the layout is chosen so the final code can be loaded at arbitrary base addresses. That means:
    - Many references will be kept PC-relative (RIP-relative on x86-64) so that the code doesn't require absolute fixups at load time.
    - For references that need absolute addresses (e.g., accessing absolute data), the linker may use the GOT/PLT mechanism: small entries in the Global Offset Table are populated at load or lazy-bind time; code fetches addresses via GOT entries (which are PC-relative accesses to GOT addresses).
- For non-PIE executables, the linker might resolve addresses to absolute virtual addresses when it produces an executable with a fixed base.
3. Loader at runtime (for PIE/DSO):

- For dynamic/shared objects and PIE, the loader performs relocations that are not PC-relative (e.g., R_X86_64_32, R_X86_64_GLOB_DAT) to fix absolute addresses, or sets up the GOT for indirect accesses. PC-relative relocations (R_X86_64_PC32/PCREL32/RIP-relative) typically don't need loader fixups if the final addresses remained as relative offsets in the binary (depending on relocation type).

## Examples (concise)
- Local basic-block label used by a short branch:
    - Assembly: .Lloop: ; jmp .Lloop
    - Assembler commonly emits a PC-relative branch with an encoded offset — no relocation needed if the offset fits and is within the same section; otherwise a relocation may be emitted.
- Reference to global data:
  - mov rax, offset_of(global_var)
  - Assembler emits a relocation (e.g., R_X86_64_64 or R_X86_64_PC32 depending on syntax/assembler and whether code is PIC). For PIE/PIC, assembler/linker prefer PC-relative (R_X86_64_PC32 or RIP-relative MOV/MOVABS variants) or use GOT indirection (R_X86_64_GOTPCREL).
- External function call:
  - call printf
  - Assembler emits a relocation against printf. For a shared object / PIE, the call may go through PLT (Procedure Linkage Table) and the relocation type will instruct the dynamic linker to resolve it and patch the GOT/PLT entry.

## Typical relocation types and PC-related forms (x86-64 examples)
- PC-relative: R_X86_64_PC32, R_X86_64_32 (PC relative variants), R_X86_64_GOTPCREL — used so the code remains position-independent. On x86-64, RIP-relative addressing is common (e.g., mov rax, [rip + offset]).
- Absolute: R_X86_64_64 (64-bit absolute address) — requires runtime/ link-time fixups, not position-independent.
- GOT/PLT: R_X86_64_GOTPCREL fills a PC-relative reference to the GOT entry; R_X86_64_JUMP_SLOT and R_X86_64_GLOB_DAT are used by the dynamic loader.

## Why assembly is said to be PIE-friendly but object is relocatable
- "Assembly is PIE" isn't strictly correct as a general statement. The assembly source may be written to use PC-relative addressing and PIC-friendly patterns (no absolute addresses), making it suitable for PIE. The assembler produces a relocatable object file (.o) that records relocations rather than final absolute addresses. The final linked PIE binary is arranged to be position-independent, relying heavily on PC-relative relocations and GOT/PLT indirection.
- So it's more precise to say: "The assembly is written as position-independent (PIC/PIE-friendly). The assembler produces a relocatable object (.o) containing relocations. The linker produces a PIE executable using PC-relative relocations and GOT/PLT as necessary."

## Edge cases and gotchas
- Some instructions require absolute addresses or large immediates that can't be encoded PC-relative — assembler/linker may need to use GOT or generate relocation types that force runtime fixups.
- Small-range branches can be encoded by the assembler without relocation; if the size changes during link, the assembler might have emitted a relocation or used a long-form jump created by the linker.
- On some platforms, the assembler resolves some local symbol references at assembly time (no relocation), but the linker could still adjust section layout and change actual runtime distances — this is OK for PC-relative encodings if computed relative to section start and later fixed by linker, but if absolute addresses were assumed, it breaks.
- Differences between architectures: arm, aarch64, riscv, etc., have different relocation types and conventions (GOT/PLT equivalents, TLS handling). Principles are similar but details differ.
## How to inspect (practical commands)
Use objdump/readelf to see symbols and relocations:
  - objdump -d file.o # disassemble and show PC-relative forms
  - readelf -r file.o # show relocation entries in object
  - readelf -s file.o # show symbol table
  - objdump -R executable # show dynamic relocations

## Other inspections
1. `readelf -h`: ET_EXEC vs ET_DYN (PIE), entry, machine, flags.    
2. `readelf -l`: find the **PT_LOAD** mapping (file ↔ VA), `PT_GNU_RELRO`, `INTERP`.
3. `readelf -S`: where **.text/.rodata/.data/.bss** live (VAs and offsets).
4. `readelf -d`: dynamic tags (neededs, PLTGOT, reloc tables).    
5. `readelf -r`: relocations (look for JUMP_SLOT/GLOB_DAT).
6. `readelf -s` & `nm -n`: symbols by address.    
7. `objdump -D -M reg-names-raw`: confirm PLT stubs, `ldr pc` veneers, and literal pools.
    


```shell
arm-linux-gnueabi-readelf -r Shootout-nestedloop.combined.o # the object file

Relocation section '.rel.text' at offset 0x908 contains 3 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
00000028  0000151c R_ARM_CALL        00000000   strtol
0000004c  0000141c R_ARM_CALL        00000000   printf
00000058  00001103 R_ARM_REL32       00000000   .L.str

Relocation section '.rel.ARM.exidx' at offset 0x920 contains 1 entry:
 Offset     Info    Type            Sym.Value  Sym. Name
00000000  0000012a R_ARM_PREL31      00000000   .text

Relocation section '.rel.debug_info' at offset 0x928 contains 40 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
00000006  00000902 R_ARM_ABS32       00000000   .debug_abbrev
0000000c  00000c02 R_ARM_ABS32       00000000   .debug_str
00000012  00000c02 R_ARM_ABS32       00000000   .debug_str
00000016  00000a02 R_ARM_ABS32       00000000   .debug_line
0000001a  00000c02 R_ARM_ABS32       00000000   .debug_str
0000001e  00000102 R_ARM_ABS32       00000000   .text
0000002f  00000202 R_ARM_ABS32       00000000   .rodata.str1.1
00000040  00000c02 R_ARM_ABS32       00000000   .debug_str
00000047  00000c02 R_ARM_ABS32       00000000   .debug_str
0000004e  00000c02 R_ARM_ABS32       00000000   .debug_str
00000060  00000c02 R_ARM_ABS32       00000000   .debug_str
0000006d  00000c02 R_ARM_ABS32       00000000   .debug_str
00000084  00000102 R_ARM_ABS32       00000000   .text
0000008e  00000c02 R_ARM_ABS32       00000000   .debug_str
00000099  00000d02 R_ARM_ABS32       00000000   .debug_loc
0000009d  00000c02 R_ARM_ABS32       00000000   .debug_str
000000a8  00000d02 R_ARM_ABS32       00000000   .debug_loc
000000ac  00000c02 R_ARM_ABS32       00000000   .debug_str
000000b7  00000d02 R_ARM_ABS32       00000000   .debug_loc
000000bb  00000c02 R_ARM_ABS32       00000000   .debug_str
000000c6  00000d02 R_ARM_ABS32       00000000   .debug_loc
000000ca  00000c02 R_ARM_ABS32       00000000   .debug_str
000000d5  00000d02 R_ARM_ABS32       00000000   .debug_loc
000000d9  00000c02 R_ARM_ABS32       00000000   .debug_str
000000e4  00000d02 R_ARM_ABS32       00000000   .debug_loc
000000e8  00000c02 R_ARM_ABS32       00000000   .debug_str
000000f3  00000d02 R_ARM_ABS32       00000000   .debug_loc
000000f7  00000c02 R_ARM_ABS32       00000000   .debug_str
00000102  00000d02 R_ARM_ABS32       00000000   .debug_loc
00000106  00000c02 R_ARM_ABS32       00000000   .debug_str
00000111  00000d02 R_ARM_ABS32       00000000   .debug_loc
00000115  00000c02 R_ARM_ABS32       00000000   .debug_str
00000120  00000d02 R_ARM_ABS32       00000000   .debug_loc
00000124  00000c02 R_ARM_ABS32       00000000   .debug_str
00000133  00000102 R_ARM_ABS32       00000000   .text
0000014b  00000102 R_ARM_ABS32       00000000   .text
00000154  00000102 R_ARM_ABS32       00000000   .text
00000161  00000c02 R_ARM_ABS32       00000000   .debug_str
0000017c  00000c02 R_ARM_ABS32       00000000   .debug_str
0000018d  00000c02 R_ARM_ABS32       00000000   .debug_str

Relocation section '.rel.debug_line' at offset 0xa68 contains 1 entry:
 Offset     Info    Type            Sym.Value  Sym. Name
000000e2  00000102 R_ARM_ABS32       00000000   .text

Relocation section '.rel.debug_frame' at offset 0xa70 contains 2 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
00000018  00000b02 R_ARM_ABS32       00000000   .debug_frame
0000001c  00000102 R_ARM_ABS32       00000000   .text

## check the relocation of the executable file
arm-linux-gnueabi-readelf -r Shootout-nestedloop # executable file

Relocation section '.rel.dyn' at offset 0x2b4 contains 1 entry:
 Offset     Info    Type            Sym.Value  Sym. Name
00021020  00000115 R_ARM_GLOB_DAT    00000000   __gmon_start__

Relocation section '.rel.plt' at offset 0x2bc contains 5 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
0002100c  00000216 R_ARM_JUMP_SLOT   00000000   strtol@GLIBC_2.4
00021010  00000316 R_ARM_JUMP_SLOT   00000000   printf@GLIBC_2.4
00021014  00000516 R_ARM_JUMP_SLOT   00000000   __libc_start_main@GLIBC_2.4
00021018  00000116 R_ARM_JUMP_SLOT   00000000   __gmon_start__
0002101c  00000416 R_ARM_JUMP_SLOT   00000000   abort@GLIBC_2.4

## use -R option of objdump to check the executable file
arm-linux-gnueabi-objdump -R Shootout-nestedloop # check the executable file

Shootout-nestedloop:     file format elf32-littlearm

DYNAMIC RELOCATION RECORDS
OFFSET   TYPE              VALUE 
00021020 R_ARM_GLOB_DAT    __gmon_start__
0002100c R_ARM_JUMP_SLOT   strtol@GLIBC_2.4
00021010 R_ARM_JUMP_SLOT   printf@GLIBC_2.4
00021014 R_ARM_JUMP_SLOT   __libc_start_main@GLIBC_2.4
00021018 R_ARM_JUMP_SLOT   __gmon_start__
0002101c R_ARM_JUMP_SLOT   abort@GLIBC_2.4

```



# File types 

1. The Three Main ELF “Types”
--------------------------------

The ELF header’s `e_type` field can have values such as:

| ELF Type | Name | Typical Use | Example Binary |
| --- | --- | --- | --- |
| `ET_EXEC` | Executable file | Classic static address executable | `/bin/busybox`, `a.out` |
| `ET_DYN` | Shared object file | Shared libraries, PIE executables | `/lib/x86_64-linux-gnu/libc.so.6`, `/usr/bin/bash` (PIE) |
| `ET_REL` | Relocatable file | Object file (`.o`) | `foo.o` before linking |

2. What Is PIE (Position-Independent Executable)?
----------------------------------------------------

A **PIE executable** is an ELF of type **`ET_DYN`**, **not** `ET_EXEC`, but it **can be executed**.

It is compiled so that **its code and data can be loaded at any memory address** — just like a `.so` shared library.  
This allows **ASLR (Address Space Layout Randomization)** to randomize its base address at runtime.

```bash
# PIE binary
gcc -fPIE -pie main.c -o pie_bin
```

**Inspect with readelf:**


```shell
# file type of the object code
arm-linux-gnueabi-readelf -h Shootout-nestedloop.combined.o 
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              REL (Relocatable file) # relocable file
  Machine:                           ARM
  Version:                           0x1
  Entry point address:               0x0
  Start of program headers:          0 (bytes into file)
  Start of section headers:          2896 (bytes into file)
  Flags:                             0x5000000, Version5 EABI
  Size of this header:               52 (bytes)
  Size of program headers:           0 (bytes)
  Number of program headers:         0
  Size of section headers:           40 (bytes)
  Number of section headers:         22
  Section header string table index: 21
```

```shell
# file type of the executable file
arm-linux-gnueabi-readelf -h Shootout-nestedloop 
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           ARM
  Version:                           0x1
  Entry point address:               0x10340
  Start of program headers:          52 (bytes into file)
  Start of section headers:          8784 (bytes into file)
  Flags:                             0x5000200, Version5 EABI, soft-float ABI
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         9
  Size of section headers:           40 (bytes)
  Number of section headers:         35
  Section header string table index: 34
```

**And check base address at runtime:**

```bash
ldd ./pie_bin
    linux-vdso.so.1 (0x00007ffc51dfd000)
    libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f8c64a00000)
```

Then each time run it:

```bash
cat /proc/$(pidof pie_bin)/maps | head
```

Executable mapped at **different random base addresses** (like `0x55cc5f000000` → `0x55cc60100000`).

3. Non-PIE (default for most compilers (up to Distributions (Debian, Ubuntu, Fedora, etc.)))

However, if it is compiled as Non-PIE Executable (`ET_EXEC`)
compile **without PIE**, the binary has **fixed addresses** for code and data:

```bash
gcc -no-pie main.c -o exec_bin
```

```bash
readelf -h exec_bin | grep Type
```

Output:

```
Type:                              EXEC (Executable file)
```

Then check memory mapping:

```bash
cat /proc/$(pidof exec_bin)/maps | head
```

```
00400000-00401000 r--p 00000000 08:01 123456 /home/user/exec_bin
00401000-00402000 r-xp 00001000 08:01 123456 /home/user/exec_bin
```

→ **Always loads at the same base (0x00400000)**.  
ASLR **cannot randomize** its load address (though shared libs and stack/heap are still randomized).

4. Shared Library (`ET_DYN`)
-------------------------------

Shared objects are also `ET_DYN`, but they are **not entry points** (they export symbols):

```bash
gcc -fPIC -shared libfoo.c -o libfoo.so
```

```bash
readelf -h libfoo.so | grep Type
```

Output:

```
Type:                              DYN (Shared object file)
```

They are **loaded by the dynamic linker (`ld-linux.so`)** at randomized base addresses.  
A PIE executable behaves _exactly like this_, except it has an entry point (`_start`).
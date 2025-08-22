---
title: x86-assembly-combination
date: 2025-02-17
categories: [Assembly]
tags: [x86]     # TAG names should always be lowercase
published: false
---

> Problem: When want to combine the translated arm blocks into x86 assembly code, need to understand the layout of the assembly code of x86

# Translation

```json
    {
        "arm": {
            "instructions": [
                "mov r3, #0",
                "mov r0, r3",
                "pop {fp, pc}"
            ]
        },
        "x86": {
            "instructions": [
                "movl $0, %ebx",
                "movl %ebx, %eax",
                "popl %edi",
                "ret"
            ]
        }
    },
    {
        "arm": {
            "instructions": [
                "push {fp, lr}",
                "add fp, sp, #4",
                "ldr r0, [pc, #0xc]",
                "bl #0x102dc"
            ]
        },
        "x86": {
            "instructions": [
                "pushl %ebx",
                "pushl %ebp",
                "movl %esp, %ebp",
                "addl $4, %ebp",
                "movl 0xc(%edi), %eax",
                "call 0x102dc"
            ]
        }
    },
```

- arm

```arm
	.arch armv5t
	.eabi_attribute 20, 1
	.eabi_attribute 21, 1
	.eabi_attribute 23, 3
	.eabi_attribute 24, 1
	.eabi_attribute 25, 1
	.eabi_attribute 26, 2
	.eabi_attribute 30, 6
	.eabi_attribute 34, 0
	.eabi_attribute 18, 4
	.file	"hello.c"
	.text
	.section	.rodata
	.align	2
.LC0:
	.ascii	"Hello, World!\000"
	.text
	.align	2
	.global	main
	.syntax unified
	.arm
	.fpu softvfp
	.type	main, %function
main:
	@ args = 0, pretend = 0, frame = 0
	@ frame_needed = 1, uses_anonymous_args = 0
	push	{fp, lr}
	add	fp, sp, #4
	ldr	r0, .L3
	bl	puts
	mov	r3, #0
	mov	r0, r3
	pop	{fp, pc}
.L4:
	.align	2
.L3:
	.word	.LC0
	.size	main, .-main
	.ident	"GCC: (Ubuntu/Linaro 7.5.0-3ubuntu1~18.04) 7.5.0"
	.section	.note.GNU-stack,"",%progbits
```

- x86

```x86
# combx86.s (AT&T syntax)
    .section .rodata
LC0:
    .string "Hello, World!"

    .text
    .globl main
    .extern puts

main:
    pushl %ebp
    movl  %esp, %ebp

    movl  $LC0, %eax
    pushl %eax
    call  puts
    addl  $4, %esp

    xorl  %eax, %eax

    leave
    ret
```

### How to assemble & link

1. **Assemble** with NASM for 32-bit output:
    
    ```bash
    nasm -f elf32 combx86.asm -o combx86.o
    ```
    
2. **Link** with the 32-bit C libraries (so that calls to puts will work):
    
    ```bash
    gcc -m32 combx86.o -o combx86
    ```
    
    This ensures 32-bit libc is linked, and puts is resolved. Then `./combx86` should print `Hello, World!` without segmentation fault.

3. Use gnu directly assemble
```bash
gcc -m32 combx86.s -o combx86
```

Same reasoning: that links with the right 32-bit libc so `puts` is resolved.

### Summary

To avoid the segmentation fault:

1. Mark `puts` as an external symbol.
2. In the assembly code, push the string address and `call puts`.
3. Link **with** the 32-bit C library by using `gcc -m32 combx86.o -o combx86`.
4. Confirm that not mixing 64-bit and 32-bit code incorrectly.

Doing so ensures a valid address for `puts` is used at runtime, preventing the segfault.


# compare to the assembly code directly gotten from the hello.c:
```x86
	.file	"hello.c"
	.text
	.section	.rodata
.LC0:
	.string	"Hello, World!"
	.text
	.globl	main
	.type	main, @function
main:
.LFB0:
	.cfi_startproc
	leal	4(%esp), %ecx
	.cfi_def_cfa 1, 0
	andl	$-16, %esp
	pushl	-4(%ecx)
	pushl	%ebp
	.cfi_escape 0x10,0x5,0x2,0x75,0
	movl	%esp, %ebp
	pushl	%ebx
	pushl	%ecx
	.cfi_escape 0xf,0x3,0x75,0x78,0x6
	.cfi_escape 0x10,0x3,0x2,0x75,0x7c
	call	__x86.get_pc_thunk.ax
	addl	$_GLOBAL_OFFSET_TABLE_, %eax
	subl	$12, %esp
	leal	.LC0@GOTOFF(%eax), %edx
	pushl	%edx
	movl	%eax, %ebx
	call	puts@PLT
	addl	$16, %esp
	movl	$0, %eax
	leal	-8(%ebp), %esp
	popl	%ecx
	.cfi_restore 1
	.cfi_def_cfa 1, 0
	popl	%ebx
	.cfi_restore 3
	popl	%ebp
	.cfi_restore 5
	leal	-4(%ecx), %esp
	.cfi_def_cfa 4, 4
	ret
	.cfi_endproc
.LFE0:
	.size	main, .-main
	.section	.text.__x86.get_pc_thunk.ax,"axG",@progbits,__x86.get_pc_thunk.ax,comdat
	.globl	__x86.get_pc_thunk.ax
	.hidden	__x86.get_pc_thunk.ax
	.type	__x86.get_pc_thunk.ax, @function
__x86.get_pc_thunk.ax:
.LFB1:
	.cfi_startproc
	movl	(%esp), %eax
	ret
	.cfi_endproc
.LFE1:
	.ident	"GCC: (Ubuntu 7.5.0-3ubuntu1~18.04) 7.5.0"
	.section	.note.GNU-stack,"",@progbits
```

The “directly compiled” assembly from gcc includes extra boilerplate and infrastructure (e.g. prologue/epilogue for alignment, PIC thunk, etc.) that a typical gcc build chain generates by default, while your “combined” hand-written assembly is a minimal snippet that omits those GCC niceties. That’s why the gcc output has more code and labels (e.g. `.LFB0`, `_GLOBAL_OFFSET_TABLE_`, `__x86.get_pc_thunk.ax`) while your version is more concise.


1. GCC’s Extra Code / Labels
----------------------------

When compiling with `gcc`, you get:

1. **Function Prologue/Epilogue** for **Stack Realignment**:
    
    * `andl $-16, %esp` or `subl $12, %esp` ensures the stack is 16-byte aligned (for SSE instructions).
    * The labeling like `.LFB0` / `.cfi_startproc` is for debugging (DWARF CFI directives) and frame info.
2. **PIC or Position-Independent Code** “Thunk”:
    
    * `_GLOBAL_OFFSET_TABLE_`, `__x86.get_pc_thunk.ax`, etc. are used in **Position Independent Executables** (PIE) or shared libraries. They let the code find addresses for global data (like your “Hello, World!” string) relative to `%ebx` or `%eax`.
3. **PLT / GOT** references:
    
    * `call puts@PLT` instructs the code to call “puts” through the Procedure Linkage Table if it’s dynamically linked.
    * That uses `_GLOBAL_OFFSET_TABLE_` to find the function’s real address at runtime.
4. **Debug / CFI** directives:
    
    * `.cfi_*` lines instruct debuggers how to track registers or how to unwind the stack.

Hence the direct code from `gcc` is a typical example of _fully relocatable, position-independent, debug-friendly code_.

* * *

2. Minimal “Hand-Rolled” or “Combined” Code
-------------------------------------------

By contrast, your “combined” x86 snippet might just:

```asm
section .rodata
LC0:
  db "Hello, World!",0

section .text
global main
extern puts

main:
  push ebp
  mov ebp, esp
  mov eax, LC0
  call puts
  xor eax, eax
  pop ebp
  ret
```

**No** extra labels like `.LFB0` or `__x86.get_pc_thunk.ax`, no alignment or PIC logic. This is enough to do “main => puts => return 0” but it lacks dynamic relocations for a fully PIC build, and it’s not storing debug info.

* * *

3. Why Fewer Lines & Labels?
----------------------------

* **You** wrote a minimal code that does “push, call puts, pop, ret,” so it has fewer instructions.
* **GCC** adds the standard function prologue, potential realignment, debug CFI, position-independent references, etc.
* The difference is purely “extra feature overhead” that gcc injects by default for real-world compatibility.

* * *

Conclusion
----------

Yes, both assemblies implement the same “print Hello, World! then return,” but:

* The **gcc** version is a **fully general** piece of code with PIC addressing, alignment, debug info, and standard function epilogue.
* The **hand-rolled** or minimal snippet is a **bare-bones** version that does the essential steps only—hence it’s smaller with fewer labels.

Both are valid, but the gcc version is more robust for real-world compilation, while your combined snippet is simpler.

# [GDB combined assembly](https://sourceware.org/gdb/current/onlinedocs/gdb.html/index.html#SEC_Contents)

1. **Assembling** and **linking** the 32-bit code with debugging symbols,
2. **Using a debugger** such as GDB to run the program, set breakpoints, step through each instruction, and display register contents after each step.

* * *

1. Build / Assemble with Debug Symbols
--------------------------------------

To debug properly, you want to include debugging information in your assembly object file and final executable. The exact method depends on which assembler and build flow you use:

### a) Using GAS (GNU As) + GCC

If your assembly is in AT&T syntax (e.g., `.s` file) and you typically do:

```bash
gcc -m32 -g combx86.s -o combx86
```

* `-g` includes debug symbols.
* `-m32` ensures it’s a 32-bit build.
* The resulting `combx86` is an ELF with debug info, so GDB can step instruction by instruction.

### b) Using NASM + LD / GCC

If your assembly is in NASM (Intel syntax):

1. **Assemble** with NASM, adding `-g -F dwarf` for debug info:
    
    ```bash
    nasm -f elf32 -g -F dwarf combx86.asm -o combx86.o
    ```
    
    Here, `-g -F dwarf` instructs NASM to produce DWARF debug info.
    
2. **Link** with `ld` or `gcc`, ensuring 32-bit mode:
    
    ```bash
    ld -m elf_i386 combx86.o -o combx86
    ```
    
    or
    
    ```bash
    gcc -m32 combx86.o -o combx86
    ```
    

Now, `combx86` has debug symbols so GDB can see your labels, line info, etc.

* * *

2. Use GDB to Step Instruction by Instruction
---------------------------------------------

1. **Launch GDB**:
    
    ```bash
    gdb ./combx86
    ```
    
2. **Set a Breakpoint** at `main` or `_start`, or anywhere in your code:
    
    ```gdb
    (gdb) break main
    ```
    
    or if your code doesn’t have a “main” label, break at your label’s name:
    
    ```gdb
    (gdb) break blockA
    ```
    
3. **Run**:
    
    ```gdb
    (gdb) run
    ```
    
    GDB stops at your breakpoint.
    
4. **Step** Through Each Instruction:
    
    ```gdb
    (gdb) stepi
    ```
    
    or `si` for short. This steps exactly one machine instruction at a time (rather than C-level statements).
    
5. **Examine Register Contents**:
    
    ```gdb
    (gdb) info registers
    ```
    
    or
    
    ```gdb
    (gdb) info all-registers
    ```
    
    This shows EAX, EBX, ECX, EDX, ESI, EDI, EBP, ESP, EIP, flags, etc. after each instruction.  
    You can also do `display /r $eax` or so for real-time register display after each step.
    
6. **Automate** Display of Registers
    
    If you want GDB to automatically show certain registers after every instruction, do:
    
    ```gdb
    (gdb) display/i $pc          # show the current instruction
    (gdb) display $eax
    (gdb) display $ebx
    (gdb) display $ecx
    ...
    (gdb) stepi
    ```
    
    Now every time you do a `stepi`, GDB prints the new instruction and those registers automatically.
    

* * *

3. Example GDB Session
----------------------

```bash
$ gdb ./combx86
(gdb) break main
Breakpoint 1 at 0x80480a0
(gdb) run
Starting program: /path/to/combx86

Breakpoint 1, main ()
    at combx86.asm:?  # or some line info
(gdb) stepi
=> 0x080480a0:  push ebp
(gdb) info registers
eax            0x0
ebx            0x0
ecx            0x0
edx            0x0
esi            0x0
edi            0x0
ebp            0xffffcfcc
esp            0xffffcfcc
... 
(gdb) stepi
=> 0x080480a1:  mov ebp,esp
...
```

* * *

4. Summary
----------

1. **Assemble** your code with debug info: `-g -F dwarf` (NASM) or `-g` (GAS + GCC).
2. **Link** as 32-bit: `-m32`.
3. **Load** it into GDB, set a breakpoint, and **step** instruction by instruction with `stepi`.
4. **Check** registers with `info registers` or use `display` to show them automatically.

This way, you see each instruction’s effect on every register in real-time, which is perfect for debugging your “combined x86 assembly code.”

Below are some approaches to do that, from the simplest (stepping one instruction at a time) to more automated methods.

* * *

1. Use `stepi` (or `si`) and Display the Current Instruction
------------------------------------------------------------

1. **Set a breakpoint** at the start (e.g. your main label).
2. **Run** until that breakpoint.
3. **Use** the `stepi` (or `si`) command to advance exactly one machine instruction at a time.
4. **After each step**, GDB will show your new location. If you also do `display/i $pc`, GDB automatically prints the current instruction.

**Example**:

```gdb
(gdb) break main
(gdb) run
(gdb) display/i $pc       # always show the current instruction
(gdb) stepi               # step one instruction
(gdb) stepi
(gdb) stepi
...
```

Every time you type `stepi`, GDB shows the next instruction at `$pc`. This is the most common way to see **each** instruction as it executes.

### Show Registers Too

If you also want register dumps each time, you can do:

```gdb
(gdb) display/i $pc
(gdb) display/2x $esp      # or `display $eax`, etc.
(gdb) stepi
```

Now GDB prints the current instruction and the contents of `$esp` (or other registers) after each instruction.

* * *

2. Use TUI Mode with `layout asm`
---------------------------------

GDB has a “Text User Interface” (TUI) that can show the assembly in a split window:

```gdb
(gdb) layout asm
```

Then you can still do `stepi`, and GDB highlights the current instruction in the TUI. This is more interactive, but you still use `stepi` repeatedly.

* * *

3. Record and Replay (Reverse Debugging)
----------------------------------------

If your GDB and target environment supports process record/replay:

```gdb
(gdb) target record-full
(gdb) stepi
(gdb) stepi
...
```

You can then “rewind” instructions. This is advanced usage. Typically not required just to see each instruction once.

* * *

4. Script a Loop
----------------

If you literally want to see _all_ instructions run from start to finish, it can be huge. You might do something like:

```gdb
(gdb) set pagination off
(gdb) break *0x<start_addr>
(gdb) commands
> silent
> while ($pc < 0x<end_addr>)
>   stepi
>   info registers
>   x/i $pc
> end
> end
```

This macro runs `stepi` until `pc` reaches some address, printing registers and the instruction each time. This is a bit hacky, but it’s fully automated.

* * *

5. Summarizing
--------------

* **`stepi`** (or `si`) is the main approach: single-step the instructions, letting you see each instruction as it executes.
* **`display/i $pc`** automatically prints the current instruction each time GDB stops.
* If you want to see register changes, also do `display $eax`, `display $ebx`, etc.
* For a large code flow, a custom GDB script or TUI mode can help.

That’s how you show every instruction GDB actually executes.

## Debug Combined Assembly x86
![debug hello world](/commons/images/asm/debug_helloworld.png)

Those extra instructions (the repeated `xchg %ax, %ax`) are compiler/assembler‐inserted **no-ops** or “alignment padding” that don’t show up in your original source. In other words, they’re “filler” instructions placed after the function’s real code to align subsequent data or code to a particular boundary, or simply to pad out the section so the next symbol starts on an alignment boundary. Here’s the short explanation:

1. **`xchg %ax, %ax`** is effectively a 2-byte no-op on x86.
    * While `nop` is a 1-byte instruction, compilers and assemblers sometimes use multi-byte no-ops (`xchg %ax, %ax`, `movl %eax,%eax`, etc.) to achieve alignment on 2-byte or 16-byte boundaries.
2. **The compiler/assembler** inserted these instructions to ensure that the next function or data is properly aligned, or to pad out the end of the `main` function so the next symbol starts on a boundary the toolchain desires.
3. **No effect**: Those instructions do nothing to your program’s logic. They’re purely structural. That’s why they don’t appear in your original assembly code snippet you wrote or expected; the compiler generated them as post‐function padding during final code generation.


# [Calling Conventions of x86 32 bit](https://www.cs.virginia.edu/~evans/cs216/guides/x86.html)
- how to call and return from routines.
- the definition of a subroutine to determine how parameters should be passed to that subroutine. 
- Given a set of calling convention rules, high-level language compilers can be made to follow the rules, thus allowing hand-coded assembly language routines and high-level language routines to call one another.
- In practice, many calling conventions are possible. We will use the widely used C language calling convention. 
- The C calling convention is based heavily on the use of the hardware-supported stack. It is based on the push, pop, call, and ret instructions. Subroutine parameters are passed on the stack. Registers are saved on the stack, and local variables used by subroutines are placed in memory on the stack. The vast majority of high-level procedural languages implemented on most processors have used similar calling conventions.

![Stack during Subroutine Call](/commons/images/asm/stack-convention.png)

The cells depicted in the stack are 32-bit wide memory locations, thus the memory addresses of the cells are 4 bytes apart. The first parameter resides at an offset of 8 bytes from the base pointer. Above the parameters on the stack (and below the base pointer), the call instruction placed the return address, thus leading to an extra 4 bytes of offset from the base pointer to the first parameter. When the ret instruction is used to return from the subroutine, it will jump to the return address stored on the stack.

## Caller Rules

To make a subrouting call, the caller should:
1. Before calling a subroutine, the caller should save the contents of certain registers that are designated caller-saved. The caller-saved registers are `EAX, ECX, EDX`. Since the called subroutine is allowed to modify these registers, if the caller relies on their values after the subroutine returns, the caller must push the values in these registers onto the stack (so they can be restore after the subroutine returns.)
2. To pass parameters to the subroutine, push them onto the stack before the call. The parameters should be pushed in inverted order (i.e. last parameter first). Since the stack grows down, the first parameter will be stored at the lowest address (this inversion of parameters was historically used to allow functions to be passed a variable number of parameters).
3. To call the subroutine, use the call instruction. This instruction places the return address on top of the parameters on the stack, and branches to the subroutine code. This invokes the subroutine, which should follow the callee rules below.

After the subroutine returns (immediately following the call instruction), the caller can expect to find the return value of the subroutine in the register EAX. To restore the machine state, the caller should:
1. Remove the parameters from stack. This restores the stack to its state before the call was performed.
2. Restore the contents of caller-saved registers (EAX, ECX, EDX) by popping them off of the stack. The caller can assume that no other registers were modified by the subroutine.

### Example

The code below shows a function call that follows the caller rules. The caller is calling a function _myFunc that takes three integer parameters. First parameter is in EAX, the second parameter is the constant 216; the third parameter is in memory location var.

```bash
push [var] ; Push last parameter first
push 216   ; Push the second parameter
push eax   ; Push first parameter last
call _myFunc ; Call the function (assume C naming)
add esp, 12
```
Note that after the call returns, the caller cleans up the stack using the add instruction. We have 12 bytes (3 parameters * 4 bytes each) on the stack, and the stack grows down. Thus, to get rid of the parameters, we can simply add 12 to the stack pointer.
The result produced by _myFunc is now available for use in the register EAX. The values of the caller-saved registers (ECX and EDX), may have been changed. If the caller uses them after the call, it would have needed to save them on the stack before the call and restore them after it.


## Callee Rules

The definition of the subroutine should adhere to the following rules at the beginning of the subroutine:
   1. Push the value of EBP onto the stack, and then copy the value of ESP into EBP using the following instructions:
   ```bash
    push ebp
    mov  ebp, esp
   ```
   This initial action maintains the base pointer, EBP. The base pointer is used by convention as a point of reference for finding parameters and local variables on the stack. When a subroutine is executing, the base pointer holds a copy of the stack pointer value from when the subroutine started executing. Parameters and local variables will always be located at known, constant offsets away from the base pointer value. We push the old base pointer value at the beginning of the subroutine so that we can later restore the appropriate base pointer value for the caller when the subroutine returns. Remember, the caller is not expecting the subroutine to change the value of the base pointer. We then move the stack pointer into EBP to obtain our point of reference for accessing parameters and local variables.
   2. Next, allocate local variables by making space on the stack. Recall, the stack grows down, so to make space on the top of the stack, the stack pointer should be decremented. The amount by which the stack pointer is decremented depends on the number and size of local variables needed. For example, if 3 local integers (4 bytes each) were required, the stack pointer would need to be decremented by 12 to make space for these local variables (i.e., sub esp, 12). As with parameters, local variables will be located at known offsets from the base pointer.
   3. Next, save the values of the callee-saved registers that will be used by the function. To save registers, push them onto the stack. The callee-saved registers are EBX, EDI, and ESI (ESP and EBP will also be preserved by the calling convention, but need not be pushed on the stack during this step).
After these three actions are performed, the body of the subroutine may proceed. When the subroutine is returns, it must follow these steps:
   1. Leave the return value in EAX.
   2. Restore the old values of any callee-saved registers (EDI and ESI) that were modified. The register contents are restored by popping them from the stack. The registers should be popped in the inverse order that they were pushed.
   3. Deallocate local variables. The obvious way to do this might be to add the appropriate value to the stack pointer (since the space was allocated by subtracting the needed amount from the stack pointer). In practice, a less error-prone way to deallocate the variables is to move the value in the base pointer into the stack pointer: mov esp, ebp. This works because the base pointer always contains the value that the stack pointer contained immediately prior to the allocation of the local variables.
   4. Immediately before returning, restore the caller's base pointer value by popping EBP off the stack. Recall that the first thing we did on entry to the subroutine was to push the base pointer to save its old value.
   5. Finally, return to the caller by executing a ret instruction. This instruction will find and remove the appropriate return address from the stack.

Note that the callee's rules fall cleanly into two halves that are basically mirror images of one another. The first half of the rules apply to the beginning of the function, and are commonly said to define the prologue to the function. The latter half of the rules apply to the end of the function, and are thus commonly said to define the epilogue of the function.

### Example
Here is an example function definition that follows the callee rules:
```bash
.486
.MODEL FLAT
.CODE
PUBLIC _myFunc
_myFunc PROC
  ; Subroutine Prologue
  push ebp     ; Save the old base pointer value.
  mov ebp, esp ; Set the new base pointer value.
  sub esp, 4   ; Make room for one 4-byte local variable.
  push edi     ; Save the values of registers that the function
  push esi     ; will modify. This function uses EDI and ESI.
  ; (no need to save EBX, EBP, or ESP)

  ; Subroutine Body
  mov eax, [ebp+8]   ; Move value of parameter 1 into EAX
  mov esi, [ebp+12]  ; Move value of parameter 2 into ESI
  mov edi, [ebp+16]  ; Move value of parameter 3 into EDI

  mov [ebp-4], edi   ; Move EDI into the local variable
  add [ebp-4], esi   ; Add ESI into the local variable
  add eax, [ebp-4]   ; Add the contents of the local variable
                     ; into EAX (final result)

  ; Subroutine Epilogue 
  pop esi      ; Recover register values
  pop  edi
  mov esp, ebp ; Deallocate local variables
  pop ebp ; Restore the caller's base pointer value
  ret
_myFunc ENDP
END
```

The subroutine prologue performs the standard actions of saving a snapshot of the stack pointer in EBP (the base pointer), allocating local variables by decrementing the stack pointer, and saving register values on the stack.
In the body of the subroutine we can see the use of the base pointer. Both parameters and local variables are located at constant offsets from the base pointer for the duration of the subroutines execution. In particular, we notice that since parameters were placed onto the stack before the subroutine was called, they are always located below the base pointer (i.e. at higher addresses) on the stack. The first parameter to the subroutine can always be found at memory location EBP + 8, the second at EBP + 12, the third at EBP + 16. Similarly, since local variables are allocated after the base pointer is set, they always reside above the base pointer (i.e. at lower addresses) on the stack. In particular, the first local variable is always located at EBP - 4, the second at EBP - 8, and so on. This conventional use of the base pointer allows us to quickly identify the use of local variables and parameters within a function body.

The function epilogue is basically a mirror image of the function prologue. The caller's register values are recovered from the stack, the local variables are deallocated by resetting the stack pointer, the caller's base pointer value is recovered, and the ret instruction is used to return to the appropriate code location in the caller.

# [ARM Architectuer Procedure Call Standard (AAPCS)](https://developer.arm.com/documentation/dui0041/c/ARM-Procedure-Call-Standard?lang=en)


About the ARM Procedure Call Standard
The ARM Procedure Call Standard (APCS) is a set of rules that regulates and facilitates calls between separately compiled or assembled program fragments.

The APCS defines:

- constraints on the use of registers
- stack conventions
- passing of machine-level arguments and the return of machine-level results at externally visible function or procedure calls.

Because ARM processors are used in a wide variety of systems, the APCS is not a single standard but a consistent family of standards. (See [APCS variants](https://developer.arm.com/documentation/dui0041/c/ARM-Procedure-Call-Standard/About-the-ARM-Procedure-Call-Standard/APCS-variants?lang=en) below for details of the variants in the family.) When you implement systems such as runtime systems, operating systems, embedded control monitors, you must choose the variant or variants most appropriate to your requirements.

The different members of the APCS family are not binary compatible. If you are concerned with long-term binary compatibility you must choose your options carefully.

Throughout this chapter, the term function is used to mean function, procedure, or subroutine.

Call frames linked through a frame pointer register:

- No frame pointer register used. Call frames not linked.

- Frame pointer register used. Call frames linked.


## General purpose registers

The 16 integer registers are divided into 3 sets:

- four argument registers that can also be used as scratch registers or as caller-saved register variables
- five callee-saved registers, conventionally used as register variables
- seven registers that have a dedicated role, at least some of the time, in at least one variant of the APCS (see APCS variants).

The registers sp, lr and pc have dedicated roles in all variants of the APCS.

The ip register has a dedicated role only during function call. At other times it may be used as a scratch register. Conventionally, ip is used by compiler code generators as a local code generator temporary register.

There are dedicated roles for sb, fp, and sl in some variants of the APCS. In other variants they may be used as callee-saved registers.

The APCS permits lr to be used as a register variable when it is not in use during a function call. It further permits an ARM system specification to forbid such use in some, or all, non-user ARM processor modes.


| **Register** | **APCS Name** | **APCS Role** |
|-------------|--------------|--------------|
| r0  | a1  | argument 1 / scratch register / result |
| r1  | a2  | argument 2 / scratch register / result |
| r2  | a3  | argument 3 / scratch register / result |
| r3  | a4  | argument 4 / scratch register / result |
| r4  | v1  | register variable |
| r5  | v2  | register variable |
| r6  | v3  | register variable |
| r7  | v4  | register variable |
| r8  | v5  | register variable |
| r9  | sb / v6  | static base / register variable |
| r10 | sl / v7  | stack limit / stack chunk handle / register variable |
| r11 | fp / v8  | frame pointer / register variable |
| r12 | ip  | scratch register / new-sb in inter-link-unit calls |
| r13 | sp  | lower end of the current stack frame |
| r14 | lr  | link register / scratch register |
| r15 | pc  | program counter |

## About Registers

- Input parameters in R0, R1, R2, R3
- If more than 4, rest go on stack
- Output parameter in R0 if needed
- R0-R3, R12 can be used freely
- If use R4-R11, then save/restore on stack
- Push/pop an even number of register (wired one, but follow)
- 
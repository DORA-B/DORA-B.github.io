---
title: Stack implementation using LDM and STM
date: 2025-07-31
categories: [Assembly]
tags: [arm]     # TAG names should always be lowercase
published: true
---

> Reason: I need to mark different opcode for different behaviors of stm/ldm instrs.


The load and store multiple instructions can update the base register. For stack operations, the base register is usually the stack pointer, SP. This means that you can use these instructions to implement push and pop operations for any number of registers in a single instruction.

The load and store multiple instructions can be used with several types of stack:

### Descending or ascending
The stack grows downwards, starting with a high address and progressing to a lower one (a descending stack), or upwards, starting from a low address and progressing to a higher address (an ascending stack).

### Full or empty
The stack pointer can either point to the last item in the stack (a full stack), or the next free space on the stack (an empty stack).

To make it easier for the programmer, stack-oriented suffixes can be used instead of the increment or decrement, and before or after suffixes. Table 13 shows the stack-oriented suffixes and their equivalent addressing mode suffixes for load and store instructions.

![Table 13. Stack-oriented suffixes and equivalent addressing mode suffixes](/commons/images/asm/stack-oriented-suffix.png)

Table 14 shows the load and store multiple instructions with the stack-oriented suffixes for the various stack types.

![Table 14. Suffixes for load and store multiple instructions](/commons/images/asm/load_store_multiple_instruction.png)

### Stack Mnemonics (for readability only)

| Stack Type | Store (STM...) | Load (LDM...) |
| --- | --- | --- |
| Full Descending (-4, ld/st …) | `STMFD` = `STMDB` | `LDMFD` = `LDMIA` |
| Full Ascending (+4, ld/st …) | `STMFA` = `STMIB` | `LDMFA` = `LDMDA` |
| Empty Descending (ld/st, -4 …) | `STMED` = `STMDA` | `LDMED` = `LDMIB` |
| Empty Ascending (ld/st, +4 …) | `STMEA` = `STMIA` | `LDMEA` = `LDMDB` |


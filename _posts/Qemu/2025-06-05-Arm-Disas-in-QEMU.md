---
title: Arm arch disassembling process
date: 2025-06-05
categories: [Qemu]
tags: [qemu]     # TAG names should always be lowercase
published: false
---

[TOC]

> Problem: How an Arm instruction is disassembled in QEMU?

# Resources
- [ARM instruction set encoding](https://developer.arm.com/documentation/ddi0406/c/Application-Level-Architecture/ARM-Instruction-Set-Encoding/ARM-instruction-set-encoding?lang=en)
- [Unconditional Instructions](https://developer.arm.com/documentation/ddi0406/c/Application-Level-Architecture/ARM-Instruction-Set-Encoding/Unconditional-instructions?lang=en)


# [Arm Instruction Overview](https://developer.arm.com/documentation/ddi0406/c/Application-Level-Architecture/ARM-Instruction-Set-Encoding/ARM-instruction-set-encoding?lang=en)


The ARM instruction stream is a sequence of word-aligned words. Each ARM instruction is a single 32-bit word in that stream. The encoding of an ARM instruction is:

![major subdivisions of ARM isnsn set](/commons/images/qemu/arm-insn-svg.png)

This field contains one of the values `0b0000-0b1110`, as shown in Table. Most instruction mnemonics can be extended with the letters defined in the mnemonic extension column of this table.

![arm insn overview](/commons/images/qemu/arm-insn-overview.png)

## [The condition code field](https://developer.arm.com/documentation/ddi0406/c/Application-Level-Architecture/ARM-Instruction-Set-Encoding/ARM-instruction-set-encoding/The-condition-code-field?lang=en)

If the always (`AL`) condition is specified, the instruction is executed irrespective of the value of the condition flags. The absence of a condition code on an instruction mnemonic implies the `AL` condition code.

Table shows the major subdivisions of the ARM instruction set, determined by `bits[31:25, 4]`.
Most ARM instructions can be conditional, with a condition determined by `bits[31:28]` of the instruction, the `cond` field. This applies to all instructions except those with the `cond` field equal to `0b1111`.


## [Unconditional Instructions](https://developer.arm.com/documentation/ddi0406/c/Application-Level-Architecture/ARM-Instruction-Set-Encoding/Unconditional-instructions?lang=en)

In older ARM versions **(v3/v4)**, `cond=0xF` was defined as `NV` (Never), and any instruction with this was considered unpredictable behavior.

But starting from **ARMv5**:

* The `cond=0xF` space was **repurposed** for a new class of **unconditional instructions**, such as:
    * **VFP (Vector Floating Point)**
    * **NEON SIMD**
    * **iWMMXt**
    * **miscellaneous instructions**

In these instructions:

* The **condition codes (NZCV)** are **ignored**.
* Execution **always proceeds**, regardless of flags.
    

![unconditional insn overview](/commons/images/qemu/unconditional-insn-overview.png)
Other encodings in this space are undefined in ARMv5 and above.
All encodings in this space are unpredictable in ARMv4 and ARMv4T.
![unconditional insn encodings](/commons/images/qemu/unconditional-insn-layout.png)

## Other instruction decoding (Data-processing, memory-transferring)
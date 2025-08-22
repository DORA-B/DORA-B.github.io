---
title: Condition-Code-in-arm-isa
date: 2025-02-17
categories: [Assembly]
tags: [arm]     # TAG names should always be lowercase
published: false
---

> Problem: if condition code need to be eliminated, the below question need to be solved:
    - 1. what kind of condition code are there in arm?
    - 2. How condition codes are used?
    - 3. how codition codes are changed in arm? by what kind of instructions?
    - 4. how one block is splitted? 
    - 5. removal of condition code references? 
    - 6. and reconstructed a block without condition code?
  
# Condition Code Types

Capstone assigns **integer values (0-15)** to ARM **condition codes** based on the ARM architecture encoding. To interpret them, you need to **map these values to their corresponding mnemonics**.

## **Mapping Condition Codes (`cc`) to Meaning**

```python
from capstone.arm.arm_const import *
ARM_CC_MAP = {
    ARM_CC_INVALID: "INVALID",
    ARM_CC_EQ : "EQ (equal, Z == 1)",
    ARM_CC_NE : "NE (not equal, Z == 0)",
    ARM_CC_HS : "HS (unsigned higher or same, C == 1)",
    ARM_CC_LO : "LO (unsigned lower, C == 0)",
    ARM_CC_MI : "MI (negative, N == 1)",
    ARM_CC_PL : "PL (positive or zero, N == 0)",
    ARM_CC_VS : "VS (overflow, V == 1)",
    ARM_CC_VC : "VC (no overflow, V == 0)",
    ARM_CC_HI : "HI (unsigned higher, C == 1 and Z == 0)",
    ARM_CC_LS : "LS (unsigned lower or same, C == 0 or Z == 1)",
    ARM_CC_GE : "GE (greater or equal, N == V)",
    ARM_CC_LT : "LT (less than, N != V)",
    ARM_CC_GT : "GT (greater than, Z == 0 and N == V)",
    ARM_CC_LE : "LE (less or equal, Z == 1 or N != V)",
    ARM_CC_AL : "AL (always)"
}
```
## Explanation of Capstone `Insn` Object Fields

In Capstone, the `Insn` (instruction) object provides detailed information about each disassembled instruction. Here are the key fields observed in the images:

1. **Mnemonic & Operands**
    
    * `mnemonic`: The name of the instruction (`sub`, `cmp`, `ldrls`).
    * `op_str`: The full operand string for the instruction.
2. **Addressing & Size**
    
    * `address`: Memory address where the instruction is located.
    * `size`: Instruction size in bytes (ARM typically uses 4 bytes).
3. **Condition Code (CC)**
    
    * `cc`: The condition code associated with the instruction (e.g., 10 for `ls`, 15 for `al`).
    * This tells whether an instruction has a conditional suffix (like `ls` in `ldrls`).
    * The cc value follows **ARM condition encoding**, e.g.:
        * `EQ (Z == 1)`: 0
        * `NE (Z == 0)`: 1
        * `LS (C == 0 OR Z == 1)`: 10
        * `AL (Always)`: 15
4. **Flag Updates**
    * `update_flags`: Whether the instruction updates flags (like `cmp` does).
    * `cps_flag`: Whether the instruction modifies CPSR (current program status register).
    * `cps_mode`: The CPSR mode if applicable.
5. **Operands Information**
    
    * `operands`: A list of `ArmOp` objects, each representing an operand.
    * Each operand may contain:
        * `post_index`: Indicates whether the addressing mode involves post-incrementing.
        * `size`: The size of data being manipulated.
        * `writeback`: Whether the operand involves a writeback operation.
        * `regs_read`: Registers read.
        * `regs_write`: Registers modified by the instruction.

### What Is the `CPSR` Register?**

The **Current Program Status Register (CPSR)** is a **special system register** in ARM that **holds condition flags, control bits, and status bits**.

* **Condition Flags (NZCV)**
    
    * **N (Negative)**: Set when the result is negative.
    * **Z (Zero)**: Set when the result is zero.
    * **C (Carry)**: Set when an addition or shift results in a carry.
    * **V (Overflow)**: Set when signed arithmetic overflows.
* **Control Bits**
    
    * **Mode**: Indicates the processor mode (User, FIQ, IRQ, etc.).
    * **T**: Thumb mode bit (set if in Thumb state).

# How to check codition code in capstone?
1. Use **`insn.update_flags`** to check if flags are updated.
2. Use **`insn.mnemonic`** to determine which specific flags (NZCV) are affected.
3. Use **`insn.cc`** to extract the **conditional code** used in conditional execution.

```py
 for ins in block.capstone.insns:
     
     if ins.cc != ARM_CC_AL:
         print_insn_msg(ins)
         print(f"\tNeed Condition Code: {ARM_CC_MAP[ins.cc]}")
         print('\n')
     
     if ins.update_flags:
         print_insn_msg(ins)
         print("\tThis instruction updated the flags")
         (_, regs_write) = ins.regs_access()
         if len(regs_write) > 0:
             print("\tRegisters Explicitly modified:", end="")
             for r in regs_write:
                 print(" %s" %(ins.reg_name(r)),end="")
             print()
         print("\tRegisters Implicitly modified:", end="")
         for regname in {ins.reg_name(r) for r in ins.regs_write}:
             print(" %s" %(regname),end="")
         print('\n')
```
## Results are like:

```bash
Instruction <4 cmp r3, #0x1e>
	This instruction updated the flags
	Registers Explicitly modified: cpsr
	Registers Implicitly modified: cpsr

Instruction <8 ldrls pc, [pc, r3, lsl #2]>
	Need Condition Code: LS (unsigned lower or same, C == 0 or Z == 1)


Instruction <0 subs r3, r3, r2>
	This instruction updated the flags
	Registers Explicitly modified: r3
	Registers Implicitly modified:

Instruction <4 movne r3, #1>
	Need Condition Code: NE (not equal, Z == 0)


Instruction <8 cmp r2, #0>
	This instruction updated the flags
	Registers Explicitly modified: cpsr
	Registers Implicitly modified: cpsr

Instruction <12 movne r2, r3>
	Need Condition Code: NE (not equal, Z == 0)


Instruction <16 moveq r2, #1>
	Need Condition Code: EQ (equal, Z == 1)


Instruction <20 cmp r2, #0>
	This instruction updated the flags
	Registers Explicitly modified: cpsr
	Registers Implicitly modified: cpsr

Instruction <24 bne #0x4c>
	Need Condition Code: NE (not equal, Z == 0)


Instruction <0 cmp r4, #0>
	This instruction updated the flags
	Registers Explicitly modified: cpsr
	Registers Implicitly modified: cpsr
```
> However, cannot directly know what kind of condition code is affacted by the instruction --> may can know from IR

## what kind of condition code is changed


Neither Capstone nor Angr/pyvex directly give “which specific condition flags (N, Z, C, V) are updated.” Instead, they only indicate whether an instruction “updates flags” (i.e. writes to CPSR) and whether it has a condition code ≠ AL. 
Exactly which bits of APSR/CPSR (N, Z, C, V) are modified?

1. **Consult ARM documentation** or a known instruction table to see which flags that instruction sets or clears, or
2. **Parse the VEX IR** in detail, searching for which bits of `cc_dep1`, `cc_dep2`, etc. are changed.
In the VEX IR, code like:
```c
PUT(cc_op) = 0x2; // ARMG_CC_OP_SUB
...
t0 = Sub32(r3, r2)
```
and the IR might call `armg_calculate_condition(0x2, r3, r2, ???)`. Then you see how that function sets the flags in the VEX code. You can interpret “this is a sub => sets N, Z, C, V.” That’s again basically a manual approach.

3. Write small reference that given an ARM opcode + suffix (e.g. `subs`, `adds`, `cmp`, `tst`) tells you which flags are updated.
The most straightforward approach is to keep a small reference that says, for each ARM opcode that sets flags (like `subs`, `adds`, `cmp`, `movs`, etc.):

* e.g. `adds` sets `N, Z, C, V`
* `subs` sets `N, Z, C, V`
* `cmp` is effectively `subs` but the result is thrown away
* `movs` sets `N, Z` (but not `C` unless it’s a shift form).


## What Are `cc_op`, `cc_dep1`, `cc_dep2`, `cc_ndep`?

* **`cc_op`**: The “operation” that produced the condition flags. For instance, `ARMG_CC_OP_ADD` (an add instruction sets flags) or `ARMG_CC_OP_SUB`, etc. It’s typically a small integer or constant that tells VEX’s internal logic “which kind of arithmetic just ran, so we know how to interpret cc_dep1/cc_dep2.”
* **`cc_dep1`**: Usually the first operand (or result) used in computing flags. For a SUB (like `subs r3, r3, #1`), VEX might store the old value of `r3` or the final arithmetic result in `cc_dep1`.
* **`cc_dep2`**: Usually the second operand. If you do `sub r3, r5, r2`, it might store the old `r5` or `r2` in `cc_dep2`.
* **`cc_ndep`**: Sometimes used for certain instructions or storing additional bits needed to calculate flags (like carry or extended bits). Often 0 or used for partial merges.

**Essentially**, each ARM instruction that sets flags updates `PUT(cc_op) = ...`, `PUT(cc_dep1) = ...`, `PUT(cc_dep2) = ...` so that:

1. The result is in a normal register (like `r3`),
2. The **flags** are stashed in these 3 or 4 fields for later usage.

* * *

2. How Condition Codes Are Calculated
-------------------------------------

When a subsequent instruction or block checks condition codes (e.g., a conditional branch or a predicated instruction), the VEX IR calls something like `armg_calculate_condition(cc_op, cc_dep1, cc_dep2, cc_ndep)` to produce a boolean result for eq/ne/gt, etc. That function uses the stored operation type (`cc_op`) plus the old operands to _recompute_ or interpret the N, Z, C, and V flags.

For example:

1. **`subs r3, r3, #1`** might do:
    
    ```c
    PUT(cc_op) = ARMG_CC_OP_SUB;
    PUT(cc_dep1) = old_r3;
    PUT(cc_dep2) = 1;
    // r3 = old_r3 - 1
    ```
    
2. **Later** a conditional `bne` checks the Z flag. VEX IR calls `armg_calculate_condition(ARMG_CC_OP_SUB, old_r3, 1, cc_ndep)` internally to see if `Z=0` or not.

Thus, there’s no direct “cc register = {N,Z,C,V}.” Instead, the logic is “**store** the operation type and operands that set flags,” then **when needed** recalculate or interpret them with a small function. This is how VEX’s ARM front‐end avoids storing all flags explicitly for each instruction.

* * *

3. Why This Approach?
---------------------

ARM instructions can set or not set flags, can use or not use them, and the flags are determined by the type of arithmetic. Instead of storing separate bits (N=..., Z=..., C=..., V=...), VEX just stashes the old data (operands) plus an operation code (`cc_op`) so it can compute them on demand for a conditional check. This design is consistent with VEX’s generic model for “condition codes.”

* * *

4. Practical Implications
-------------------------

* If you want to see “which flags are set after `subs`,” you check the VEX IR instructions that do `PUT(cc_op)=ARMG_CC_OP_SUB; PUT(cc_dep1)=..., PUT(cc_dep2)=...`. That means the next conditional instruction referencing `cc_op` sees a sub.
* If you want to see if `Z=1`, you see how `armg_calculate_condition(...)` interprets those fields.
* Tools like angr or the VEX IR do not store “Z=1, N=0,” etc. directly in a single “cpsr” location. They store a combination of `cc_op`, `cc_depX`, etc.

* * *

Conclusion
----------

**`cc_op`, `cc_dep1`, `cc_dep2`, `cc_ndep`** are the VEX IR mechanism to track the “type of last flags-setting instruction” and “operands.” The actual bits (N,Z,C,V) are computed on‐demand for a subsequent conditional check. This is how VEX (and thus angr) handle ARM’s condition codes internally without explicitly storing each flag bit in a single “cpsr.”


# **Steps to Identify Which Condition Flags Are Affected**

1. **Check `update_flags`**
    
    * If `insn.update_flags` is `True`, then the instruction modifies condition flags.
2. **Check Instruction Mnemonic**
    
    * Different types of instructions affect different flags:
        * **Arithmetic Instructions (`ADD`, `SUB`, `RSB`, `CMP`, `CMN`, `TST`, `TEQ`)**
            * These modify **Negative (N), Zero (Z), Carry (C), and Overflow (V)** flags.
        * **Logical Instructions (`AND`, `EOR`, `ORR`, `BIC`, `TEQ`, `TST`)**
            * These modify **Negative (N), Zero (Z)** flags but **do not** affect Carry (C) and Overflow (V) unless explicitly shifted.
        * **Multiplication (`MUL`, `MLA`)**
            * Do **not** modify flags unless `S` (Set Flags) suffix is used.
        * **Comparison Instructions (`CMP`, `CMN`, `TST`, `TEQ`)**
            * Update **NZCV** flags but **do not store a result**.
3. **Determine Which Flags Are Changed**
    
    * Extract the **mnemonic** (`insn.mnemonic`) and match it against known flag-modifying operations.
    * Analyze the **operands** (`insn.operands`) to check for shifts, which might influence the **Carry (C) flag**.



#### **1. Instructions That Affect Flags (Explicitly or Implicitly)**

##### **A. Data Processing Instructions (With `S` Suffix)**

These **affect condition flags if they have an `S` suffix** (e.g., `ADDS`, `SUBS`, `ANDS`):

| **Instruction** | **Description** | **Flags Affected** |
| --- | --- | --- |
| `ADDS` | Add with carry | N, Z, C, V |
| `SUBS` | Subtract | N, Z, C, V |
| `RSBS` | Reverse subtract | N, Z, C, V |
| `ANDS` | Bitwise AND | N, Z |
| `EORS` | Bitwise XOR | N, Z |
| `ORRS` | Bitwise OR | N, Z |
| `MVNS` | Bitwise NOT | N, Z |
| `TEQ` | Test equality (`EOR` but no result stored) | N, Z |
| `TST` | Test bits (`AND` but no result stored) | N, Z |
| `CMN` | Compare negative (`ADD` but no result stored) | N, Z, C, V |
| `CMP` | Compare (`SUB` but no result stored) | N, Z, C, V |

##### **B. Memory Access Instructions**

Some **load/store** instructions update flags in special cases.

| **Instruction** | **Description** | **Flags Affected** |
| --- | --- | --- |
| `LDRS` | Load register and sign-extend | N, Z |
| `LDRSB` | Load signed byte | N, Z |
| `LDRSH` | Load signed halfword | N, Z |

##### **C. Multiply and Divide**

Some **multiplication-related** instructions affect condition flags.

| **Instruction** | **Description** | **Flags Affected** |
| --- | --- | --- |
| `MULS` | Multiply | N, Z |
| `MLAS` | Multiply and accumulate | N, Z |
| `UMULLS` | Unsigned Multiply Long | N, Z |
| `SMULLS` | Signed Multiply Long | N, Z |

##### **D. Shift and Rotate Instructions**

Some **shift/rotate** instructions update the carry flag (`C`).

| **Instruction** | **Description** | **Flags Affected** |
| --- | --- | --- |
| `LSLS` | Logical shift left | N, Z, C |
| `LSRS` | Logical shift right | N, Z, C |
| `ASRS` | Arithmetic shift right | N, Z, C |
| `RORS` | Rotate right | N, Z, C |

##### **E. Special Control Instructions**

Some instructions **explicitly change flags**.

| **Instruction** | **Description** | **Flags Affected** |
| --- | --- | --- |
| `MSR CPSR_f, #imm` | Modify status register | N, Z, C, V |
| `MRS Rd, CPSR` | Read status register | None (reads flags) |
| `BKPT` | Breakpoint | N, Z, C, V |

* * *

### **2. Condition Codes and Flags: Are They Both Used and Written?**

Yes, there **are cases where a conditional instruction both reads and writes flags**.

#### **Example 1: Conditional Execution with Flags Updating**

```assembly
ADDEQ R1, R2, R3  ; If Z == 1 (equal), perform ADD
```

* **Reads**: Zero flag (`Z`) to check if condition holds.
* **Writes**: If executed, **modifies flags** based on `ADD` behavior.

#### **Example 2: CMP and a Conditional Branch**

```assembly
CMP R0, R1    ; Updates N, Z, C, V flags
BLT Target    ; Uses N and V flags to determine if R0 < R1
```

* **`CMP` writes condition flags**.
* **`BLT` reads condition flags** (`N != V` for less than).

#### **Example 3: Multiply with Conditional Execution**

```assembly
MLSNE R1, R2, R3, R4  ; Multiply-subtract, executed only if Z == 0 (not equal)
```

* **Reads Z flag** for execution.
* **Writes N and Z flags** after execution.


### special cases


ARM does not have conditionally executed load instructions that also set flags.
> Why No LDR{S}{cond}?
- ARM’s design keeps memory operations and flag-setting separate for:=
- Pipeline efficiency (memory ops are slow, flags should depend on computation).
- Simpler exception handling (memory faults shouldn’t corrupt flags).
- Consistency (flags are only set by computational ops, not loads/stores).
---
title: x86-arch-eflags-register
date: 2025-03-17
categories: [Assembly]
tags: [x86]     # TAG names should always be lowercase
published: true
---
> References: [x86 Intel Architecture Software Manual](https://www.cs.cornell.edu/courses/cs412/2000SP/resources/Intel%20Architecture%20Vol.%201.PDF)

![eflags introduction in manual](/commons/images/asm/eflags-reg-intro.png)

![eflags register in x86](/commons/images/asm/x86-eflags-register.png)

### **The EFLAGS Register in x86 32-bit**

The **EFLAGS** register in **x86 (32-bit)** is a **32-bit status register** that stores **condition flags, control flags, and system flags**. These flags are updated by arithmetic, logic, and control operations and influence how the CPU executes subsequent instructions.

* * *

**1. Structure of EFLAGS (Bit Layout)**
---------------------------------------

| Bit | Name | Description |
| --- | --- | --- |
| 0 | **CF** (Carry Flag) | Set if an operation generates a carry/borrow. |
| 1 | **(Reserved)** | Always 1. |
| 2 | **PF** (Parity Flag) | Set if the least significant byte has an even number of 1s. |
| 3 | **(Reserved)** | Always 0. |
| 4 | **AF** (Auxiliary Carry Flag) | Set if there is a carry from the low nibble (used in BCD arithmetic). |
| 5 | **(Reserved)** | Always 0. |
| 6 | **ZF** (Zero Flag) | Set if the result of an operation is zero. |
| 7 | **SF** (Sign Flag) | Set if the result is negative (MSB is 1). |
| 8 | **TF** (Trap Flag) | Enables single-step debugging mode. |
| 9 | **IF** (Interrupt Flag) | Enables or disables hardware interrupts. |
| 10 | **DF** (Direction Flag) | Controls string operations (increment or decrement). |
| 11 | **OF** (Overflow Flag) | Set if an operation produces a signed overflow. |
| 12-13 | **IOPL** (I/O Privilege Level) | Defines privilege level for I/O operations (Ring 0-3). |
| 14 | **NT** (Nested Task) | Used in task switching. |
| 15 | **(Reserved)** | Always 0. |
| 16 | **RF** (Resume Flag) | Used to continue execution after a debug exception. |
| 17 | **VM** (Virtual 8086 Mode) | Enables execution in Virtual 8086 mode. |
| 18 | **AC** (Alignment Check) | Detects unaligned memory accesses. |
| 19 | **VIF** (Virtual Interrupt Flag) | Virtualized `IF` for VMs. |
| 20 | **VIP** (Virtual Interrupt Pending) | Tracks pending virtual interrupts. |
| 21 | **ID** (ID Flag) | Can be used to detect CPU capabilities. |
| 22-31 | **(Reserved)** | Always 0. |

* * *

**2. Condition Flags (Status Flags)**
-------------------------------------

These **flags change based on arithmetic/logical operations**:

| Flag | Set When |
| --- | --- |
| **CF** (Carry Flag) | Addition results in a carry or subtraction borrows. |
| **PF** (Parity Flag) | Result's LSB byte has an even number of 1s. |
| **AF** (Auxiliary Carry) | Carry occurs between **bit 3 and 4** (used in BCD). |
| **ZF** (Zero Flag) | Result of an operation is **zero**. |
| **SF** (Sign Flag) | Result is **negative** (MSB = 1). |
| **OF** (Overflow Flag) | Signed overflow occurs (e.g., adding two positives results in negative). |

* * *

**3. Control and System Flags**
-------------------------------

These **affect CPU execution**:

| Flag | Description |
| --- | --- |
| **TF** (Trap Flag) | Enables **single-step debugging** (triggers exception after each instruction). |
| **IF** (Interrupt Flag) | Enables/disables **hardware interrupts** (`CLI` clears, `STI` sets). |
| **DF** (Direction Flag) | Controls **string instructions** (`STD` sets to decrement, `CLD` clears to increment). |
| **IOPL** (I/O Privilege Level) | Controls access to **I/O ports** (Ring 0 = full access). |
| **NT** (Nested Task) | Used in **task switching**. |
| **VM** (Virtual 8086 Mode) | Enables **real-mode execution inside protected mode**. |
| **AC** (Alignment Check) | Detects **misaligned memory accesses**. |
| **VIF** (Virtual Interrupt Flag) | Used in **virtualized environments**. |
| **VIP** (Virtual Interrupt Pending) | Tracks **pending interrupts**. |
| **ID** (ID Flag) | Allows **CPU feature detection** (`CPUID` instruction). |

* * *

**4. Example: How EFLAGS Are Affected by Instructions**
-------------------------------------------------------

### **Example 1: Arithmetic Operations**

```assembly
mov eax, 0xFFFFFFFF
add eax, 1      ; EAX = 0x00000000
```

* **ZF = 1** (Result is zero)
* **CF = 1** (Carry occurred)
* **OF = 0** (No signed overflow)

### **Example 2: Subtraction**

```assembly
mov eax, 5
sub eax, 7      ; EAX = -2
```

* **ZF = 0** (Result is not zero)
* **SF = 1** (Negative result)
* **CF = 1** (Borrow occurred)
* **OF = 0** (No signed overflow)

### **Example 3: Logical AND**

```assembly
mov eax, 0xFFFFFFFF
and eax, 0      ; EAX = 0x00000000
```

* **ZF = 1** (Result is zero)
* **SF = 0** (Result is non-negative)
* **OF and CF = 0** (Logical operations don't affect these)

* * *

**5. How to Read EFLAGS in Assembly**
-------------------------------------

You can access the **EFLAGS register** using:

```assembly
pushfd       ; Push EFLAGS onto stack
pop eax      ; Store into EAX for inspection
```

To modify specific flags, use:

```assembly
clc          ; Clear Carry Flag (CF = 0)
stc          ; Set Carry Flag (CF = 1)
cli          ; Disable Interrupts (IF = 0)
sti          ; Enable Interrupts (IF = 1)
```

* * *

**6. Summary**
--------------

* **EFLAGS is a 32-bit register storing status and control flags.**
* **Condition flags** (`CF`, `ZF`, `SF`, `OF`, `PF`, `AF`) reflect results of arithmetic operations.
* **Control flags** (`TF`, `IF`, `DF`) affect **execution behavior**.
* **System flags** (`IOPL`, `VM`, `ID`) enable **privileged features**.


## How Angr get EFLAGS

In Angr, the `EFLAGS` register is modeled as a bitvector inside the symbolic state. We can access, modify, and analyze individual flags using Claripy expressions.


In x86 symbolic execution, EFLAGS is stored as a 32-bit symbolic bitvector.
Angr does not track flags explicitly but computes them dynamically using `cc_op`, `cc_dep1`, and `cc_dep2`.
You can extract specific flags using bitwise operations (`claripy.Extract`).


### How Angr Extract arm and x86 Flags Differently?

#### arm
- arm does not have dedicated flag register
- Flags are comoputed dynamially based on previous operatins.
- The `armg_calculate_flag_*` functions recompute `NF, ZF, CF, VF` dynamically based on previous computations.
- The method `extract_condition_flag` derives flag values using `cc_op`, `cc_dep1`, `cc_dep2`, and `cc_ndep`.
- Reason: ARM’s condition flags aren’t stored persistently in registers; they depend on the last instruction that updated them.
```py
  def extract_condition_flag(self, state: angr.SimState, flag):
      cc_op = state.regs.cc_op
      cc_dep1 = state.regs.cc_dep1
      cc_dep2 = state.regs.cc_dep2
      cc_ndep = state.regs.cc_ndep
      if cc_op.op == "If":
          ValueError("Unsupport ...")
      if flag == "nf":
          f = s_ccall.armg_calculate_flag_n(state, cc_op, cc_dep1, cc_dep2, cc_ndep)
      elif flag == "zf":
          f = s_ccall.armg_calculate_flag_z(state, cc_op, cc_dep1, cc_dep2, cc_ndep)
      elif flag == "cf":
          f = s_ccall.armg_calculate_flag_c(state, cc_op, cc_dep1, cc_dep2, cc_ndep)
      elif flag == "vf":
          f = s_ccall.armg_calculate_flag_v(state, cc_op, cc_dep1, cc_dep2, cc_ndep)
      return f
```


### x86 data in Angr

```py

data["X86"]["CondTypes"]["CondO"] = 0
data["X86"]["CondTypes"]["CondNO"] = 1
data["X86"]["CondTypes"]["CondB"] = 2
data["X86"]["CondTypes"]["CondNB"] = 3
data["X86"]["CondTypes"]["CondZ"] = 4
data["X86"]["CondTypes"]["CondNZ"] = 5
data["X86"]["CondTypes"]["CondBE"] = 6
data["X86"]["CondTypes"]["CondNBE"] = 7
data["X86"]["CondTypes"]["CondS"] = 8
data["X86"]["CondTypes"]["CondNS"] = 9
data["X86"]["CondTypes"]["CondP"] = 10
data["X86"]["CondTypes"]["CondNP"] = 11
data["X86"]["CondTypes"]["CondL"] = 12
data["X86"]["CondTypes"]["CondNL"] = 13
data["X86"]["CondTypes"]["CondLE"] = 14
data["X86"]["CondTypes"]["CondNLE"] = 15
data["X86"]["CondTypes"]["CondAlways"] = 16

data["X86"]["CondBitOffsets"]["G_CC_SHIFT_O"] = 11 # OF
data["X86"]["CondBitOffsets"]["G_CC_SHIFT_S"] = 7  # SF
data["X86"]["CondBitOffsets"]["G_CC_SHIFT_Z"] = 6  # ZF
data["X86"]["CondBitOffsets"]["G_CC_SHIFT_A"] = 4  # AF
data["X86"]["CondBitOffsets"]["G_CC_SHIFT_C"] = 0  # CF
data["X86"]["CondBitOffsets"]["G_CC_SHIFT_P"] = 2  # PF

# masks
data["X86"]["CondBitMasks"]["G_CC_MASK_O"] = 1 << data["X86"]["CondBitOffsets"]["G_CC_SHIFT_O"]
data["X86"]["CondBitMasks"]["G_CC_MASK_S"] = 1 << data["X86"]["CondBitOffsets"]["G_CC_SHIFT_S"]
data["X86"]["CondBitMasks"]["G_CC_MASK_Z"] = 1 << data["X86"]["CondBitOffsets"]["G_CC_SHIFT_Z"]
data["X86"]["CondBitMasks"]["G_CC_MASK_A"] = 1 << data["X86"]["CondBitOffsets"]["G_CC_SHIFT_A"]
data["X86"]["CondBitMasks"]["G_CC_MASK_C"] = 1 << data["X86"]["CondBitOffsets"]["G_CC_SHIFT_C"]
data["X86"]["CondBitMasks"]["G_CC_MASK_P"] = 1 << data["X86"]["CondBitOffsets"]["G_CC_SHIFT_P"]
```
- In angr, there are multiple operation types and condtion types perpared for condtion validation.
- For example, when computing the condition code and validating it, we can use functions like `x86g_calculate_eflags_all` or `pc_calculate_rdata_all_WRK`
```python
# Used for x86, compute all flags based on cc_op, dep1, dep2
def x86g_calculate_eflags_all(state, cc_op, cc_dep1, cc_dep2, cc_ndep):
    return pc_calculate_rdata_all(state, cc_op, cc_dep1, cc_dep2, cc_ndep, platform="X86")

# This function returns all the data
def pc_calculate_rdata_all(state, cc_op, cc_dep1, cc_dep2, cc_ndep, platform=None):
    rdata_all = pc_calculate_rdata_all_WRK(state, cc_op, cc_dep1, cc_dep2, cc_ndep, platform=platform)
    if isinstance(rdata_all, tuple):
        return pc_make_rdata_if_necessary(data[platform]["size"], *rdata_all, platform=platform)
    else:
        return rdata_all

def pc_calculate_rdata_all_WRK(state, cc_op, cc_dep1_formal, cc_dep2_formal, cc_ndep_formal, platform=None):
    # sanity check
    cc_op = op_concretize(cc_op)

    if cc_op == data[platform]["OpTypes"]["G_CC_OP_COPY"]:
        l.debug("cc_op == data[platform]['OpTypes']['G_CC_OP_COPY']")
        return cc_dep1_formal & (
            data[platform]["CondBitMasks"]["G_CC_MASK_O"]
            | data[platform]["CondBitMasks"]["G_CC_MASK_S"]
            | data[platform]["CondBitMasks"]["G_CC_MASK_Z"]
            | data[platform]["CondBitMasks"]["G_CC_MASK_A"]
            | data[platform]["CondBitMasks"]["G_CC_MASK_C"]
            | data[platform]["CondBitMasks"]["G_CC_MASK_P"]
        )

    cc_str = data_inverted[platform]["OpTypes"][cc_op]

    nbits = _get_nbits(cc_str)
    l.debug("nbits == %d", nbits)

    cc_dep1_formal = cc_dep1_formal[nbits - 1 : 0]
    cc_dep2_formal = cc_dep2_formal[nbits - 1 : 0]
    cc_ndep_formal = cc_ndep_formal[nbits - 1 : 0]

    if cc_str in ["G_CC_OP_ADDB", "G_CC_OP_ADDW", "G_CC_OP_ADDL", "G_CC_OP_ADDQ"]:
        l.debug("cc_str: ADD")
        return pc_actions_ADD(state, nbits, cc_dep1_formal, cc_dep2_formal, cc_ndep_formal, platform=platform)

    if cc_str in ["G_CC_OP_ADCB", "G_CC_OP_ADCW", "G_CC_OP_ADCL", "G_CC_OP_ADCQ"]:
        l.debug("cc_str: ADC")
        return pc_actions_ADC(state, nbits, cc_dep1_formal, cc_dep2_formal, cc_ndep_formal, platform=platform)

    if cc_str in ["G_CC_OP_SUBB", "G_CC_OP_SUBW", "G_CC_OP_SUBL", "G_CC_OP_SUBQ"]:
        l.debug("cc_str: SUB")
        return pc_actions_SUB(state, nbits, cc_dep1_formal, cc_dep2_formal, cc_ndep_formal, platform=platform)

    if cc_str in ["G_CC_OP_SBBB", "G_CC_OP_SBBW", "G_CC_OP_SBBL", "G_CC_OP_SBBQ"]:
        l.debug("cc_str: SBB")
        return pc_actions_SBB(state, nbits, cc_dep1_formal, cc_dep2_formal, cc_ndep_formal, platform=platform)

    if cc_str in ["G_CC_OP_LOGICB", "G_CC_OP_LOGICW", "G_CC_OP_LOGICL", "G_CC_OP_LOGICQ"]:
        l.debug("cc_str: LOGIC")
        return pc_actions_LOGIC(state, nbits, cc_dep1_formal, cc_dep2_formal, cc_ndep_formal, platform=platform)

    if cc_str in ["G_CC_OP_INCB", "G_CC_OP_INCW", "G_CC_OP_INCL", "G_CC_OP_INCQ"]:
        l.debug("cc_str: INC")
        return pc_actions_INC(state, nbits, cc_dep1_formal, cc_dep2_formal, cc_ndep_formal, platform=platform)

    if cc_str in ["G_CC_OP_DECB", "G_CC_OP_DECW", "G_CC_OP_DECL", "G_CC_OP_DECQ"]:
        l.debug("cc_str: DEC")
        return pc_actions_DEC(state, nbits, cc_dep1_formal, cc_dep2_formal, cc_ndep_formal, platform=platform)

    if cc_str in ["G_CC_OP_SHLB", "G_CC_OP_SHLW", "G_CC_OP_SHLL", "G_CC_OP_SHLQ"]:
        l.debug("cc_str: SHL")
        return pc_actions_SHL(state, nbits, cc_dep1_formal, cc_dep2_formal, cc_ndep_formal, platform=platform)

    if cc_str in ["G_CC_OP_SHRB", "G_CC_OP_SHRW", "G_CC_OP_SHRL", "G_CC_OP_SHRQ"]:
        l.debug("cc_str: SHR")
        return pc_actions_SHR(state, nbits, cc_dep1_formal, cc_dep2_formal, cc_ndep_formal, platform=platform)

    if cc_str in ["G_CC_OP_ROLB", "G_CC_OP_ROLW", "G_CC_OP_ROLL", "G_CC_OP_ROLQ"]:
        l.debug("cc_str: ROL")
        return pc_actions_ROL(state, nbits, cc_dep1_formal, cc_dep2_formal, cc_ndep_formal, platform=platform)

    if cc_str in ["G_CC_OP_RORB", "G_CC_OP_RORW", "G_CC_OP_RORL", "G_CC_OP_RORQ"]:
        l.debug("cc_str: ROR")
        return pc_actions_ROR(state, nbits, cc_dep1_formal, cc_dep2_formal, cc_ndep_formal, platform=platform)

    if cc_str in ["G_CC_OP_UMULB", "G_CC_OP_UMULW", "G_CC_OP_UMULL", "G_CC_OP_UMULQ"]:
        l.debug("cc_str: UMUL")
        return pc_actions_UMUL(state, nbits, cc_dep1_formal, cc_dep2_formal, cc_ndep_formal, platform=platform)
    if cc_str == "G_CC_OP_UMULQ":
        l.debug("cc_str: UMULQ")
        return pc_actions_UMULQ(state, nbits, cc_dep1_formal, cc_dep2_formal, cc_ndep_formal, platform=platform)
    if cc_str in ["G_CC_OP_SMULB", "G_CC_OP_SMULW", "G_CC_OP_SMULL", "G_CC_OP_SMULQ"]:
        l.debug("cc_str: SMUL")
        return pc_actions_SMUL(state, nbits, cc_dep1_formal, cc_dep2_formal, cc_ndep_formal, platform=platform)
    if cc_str == "G_CC_OP_SMULQ":
        l.debug("cc_str: SMULQ")
        return pc_actions_SMULQ(state, nbits, cc_dep1_formal, cc_dep2_formal, cc_ndep_formal, platform=platform)
    if cc_str in ["G_CC_OP_ANDN32", "G_CC_OP_ANDN64"]:
        l.debug("cc_str: ANDN")
        return pc_actions_ANDN(state, nbits, cc_dep1_formal, cc_dep2_formal, cc_ndep_formal, platform=platform)
    if cc_str in ["G_CC_OP_BLSI32", "G_CC_OP_BLSI64"]:
        l.debug("cc_str: BLSI")
        return pc_actions_BLSI(state, nbits, cc_dep1_formal, cc_dep2_formal, cc_ndep_formal, platform=platform)
    if cc_str in ["G_CC_OP_BLSMSK32", "G_CC_OP_BLSMSK64"]:
        l.debug("cc_str: BLSMSK")
        return pc_actions_BLSMSK(state, nbits, cc_dep1_formal, cc_dep2_formal, cc_ndep_formal, platform=platform)
    if cc_str in ["G_CC_OP_BLSR32", "G_CC_OP_BLSR64"]:
        l.debug("cc_str: BLSR")
        return pc_actions_BLSR(state, nbits, cc_dep1_formal, cc_dep2_formal, cc_ndep_formal, platform=platform)
    if cc_str in ["G_CC_OP_ADOXL", "G_CC_OP_ADOXQ"]:
        l.debug("cc_str: ADOX")
        return pc_actions_ADCX(state, nbits, cc_dep1_formal, cc_dep2_formal, cc_ndep_formal, False, platform=platform)
    if cc_str in ["G_CC_OP_ADCXL", "G_CC_OP_ADCXQ"]:
        l.debug("cc_str: ADCX")
        return pc_actions_ADCX(state, nbits, cc_dep1_formal, cc_dep2_formal, cc_ndep_formal, True, platform=platform)

    l.error("Unsupported cc_op %d in in pc_calculate_rdata_all_WRK", cc_op)
    raise SimCCallError("Unsupported cc_op in pc_calculate_rdata_all_WRK")
```
- Sometimes we want to check whether a conditional branch will be taken
```python
def x86g_calculate_condition(state, cond, cc_op, cc_dep1, cc_dep2, cc_ndep):
    if USE_SIMPLIFIED_CCALLS in state.options:
        return pc_calculate_condition_simple(state, cond, cc_op, cc_dep1, cc_dep2, cc_ndep, platform="X86")
    else:
        return pc_calculate_condition(state, cond, cc_op, cc_dep1, cc_dep2, cc_ndep, platform="X86")
# in arm, the computation is like:

def armg_calculate_condition(state, cond_n_op, cc_dep1, cc_dep2, cc_dep3):
    concrete_cond_n_op = op_concretize(cond_n_op)

    cond = concrete_cond_n_op >> 4
    cc_op = concrete_cond_n_op & 0xF
    inv = cond & 1

    concrete_cond = op_concretize(cond)
    flag = None

    # NOTE: adding constraints afterwards works here *only* because the constraints are actually useless, because we
    # require cc_op to be unique. If we didn't, we'd need to pass the constraints into any functions called after the
    # constraints were created.

    if concrete_cond == ARMCondAL:
        flag = claripy.BVV(1, 32)
    elif concrete_cond in [ARMCondEQ, ARMCondNE]:
        zf = armg_calculate_flag_z(state, cc_op, cc_dep1, cc_dep2, cc_dep3)
        flag = inv ^ zf
    elif concrete_cond in [ARMCondHS, ARMCondLO]:
        cf = armg_calculate_flag_c(state, cc_op, cc_dep1, cc_dep2, cc_dep3)
        flag = inv ^ cf
    elif concrete_cond in [ARMCondMI, ARMCondPL]:
        nf = armg_calculate_flag_n(state, cc_op, cc_dep1, cc_dep2, cc_dep3)
        flag = inv ^ nf
    elif concrete_cond in [ARMCondVS, ARMCondVC]:
        vf = armg_calculate_flag_v(state, cc_op, cc_dep1, cc_dep2, cc_dep3)
        flag = inv ^ vf
    elif concrete_cond in [ARMCondHI, ARMCondLS]:
        cf = armg_calculate_flag_c(state, cc_op, cc_dep1, cc_dep2, cc_dep3)
        zf = armg_calculate_flag_z(state, cc_op, cc_dep1, cc_dep2, cc_dep3)
        flag = inv ^ (cf & ~zf)
    elif concrete_cond in [ARMCondGE, ARMCondLT]:
        nf = armg_calculate_flag_n(state, cc_op, cc_dep1, cc_dep2, cc_dep3)
        vf = armg_calculate_flag_v(state, cc_op, cc_dep1, cc_dep2, cc_dep3)
        flag = inv ^ (1 & ~(nf ^ vf))
    elif concrete_cond in [ARMCondGT, ARMCondLE]:
        nf = armg_calculate_flag_n(state, cc_op, cc_dep1, cc_dep2, cc_dep3)
        vf = armg_calculate_flag_v(state, cc_op, cc_dep1, cc_dep2, cc_dep3)
        zf = armg_calculate_flag_z(state, cc_op, cc_dep1, cc_dep2, cc_dep3)
        flag = inv ^ (1 & ~(zf | (nf ^ vf)))

    if flag is not None:
        return flag

    l.error("Unrecognized condition %d in armg_calculate_condition", concrete_cond)
    raise SimCCallError("Unrecognized condition %d in armg_calculate_condition" % concrete_cond)

```
The code above is to **check whether the x86 Carry Flag (CF) would cause a conditional branch** to be taken — i.e., _“Would a `jb` (jump if below) or `jc` (jump if carry) actually occur?”_

So instead of **directly extracting the CF**, it evaluates whether the **CondB condition is satisfied**, which is:

> `CondB ≡ (CF == 1)`

This makes sense in context when:

* You're verifying behavior from the perspective of conditional control flow (e.g. jump instructions).
    
* You're not just reading CF, but asking: **“Would CF trigger a branch here?”**

So the code here is to check whether the cf will trigger a branch here, not to extract the cflags.

```python
cf_normal = (claripy.LShR(eflags, s_ccall.data["X86"]["CondBitOffsets"]["G_CC_SHIFT_C"]) & 1 ) # normal cf
cf_shl = claripy.Extract(0, 0, 
            s_ccall.x86g_calculate_condition(
            state,
            s_ccall.data["X86"]["CondTypes"]["CondB"],
            regs.cc_op,
            regs.cc_dep2,
            regs.cc_dep2,
            regs.cc_ndep,
        )
    )
is_shl = claripy.Or(
    regs.cc_op == s_ccall.data["X86"]["OpTypes"]["G_CC_OP_SHLB"],
    claripy.Or(
        regs.cc_op == s_ccall.data["X86"]["OpTypes"]["G_CC_OP_SHLW"],
        claripy.Or(
            regs.cc_op == s_ccall.data["X86"]["OpTypes"]["G_CC_OP_SHLL"],
            regs.cc_op == s_ccall.data["X86"]["OpTypes"]["G_CC_OP_SHLQ"],
        ),
    ),
)
reg_state["cf"] = claripy.If(is_shl, cf_shl, cf_normal)
```

### tips when preparing for combination: Getting from dbt-learning
- Context: registers in host is not used 
  - However, it only saves minimum of context if multiple exist
  - 

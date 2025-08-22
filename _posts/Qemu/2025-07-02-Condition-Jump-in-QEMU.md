---
title: Conditional Jump in QEMU and Label Usage
date: 2025-06-20
categories: [Qemu]
tags: [qemu]     # TAG names should always be lowercase
published: false
---

Problem:
> 1. How to set some jump conditions at any specific instructions and skip it.
> 2. How it is managed in original QEMU?
> 3. Internal mechanism: IRs emission and Instruction generation.

# Original conditional jump
`condjmp` is set when there is conditional instr:

```c
// disas_arm_insn
    if (cond != 0xe) {
        /* if not always execute, we generate a conditional jump to
           next instruction */
        arm_skip_unless(s, cond);
    }

/* Generate a label used for skipping this instruction */
void arm_gen_condlabel(DisasContext *s)
{
    if (!s->condjmp) {
        s->condlabel = gen_disas_label(s);
        s->condjmp = 1;
    }
    else{
      // seems never reach
        fprintf(stderr, "[DEBUG INFO] condlabel already exists\n");
    }
}
// only for instrs that have no control flow change is_jmp == DISAS_NEXT
static void arm_post_translate_insn(DisasContext *dc)
{
    if (dc->condjmp && dc->base.is_jmp == DISAS_NEXT) {
        if (dc->pc_save != dc->condlabel.pc_save) {
            gen_update_pc(dc, dc->condlabel.pc_save - dc->pc_save);
        }
        gen_set_label(dc->condlabel.label);
        dc->condjmp = 0;
    }
}

// when the tb is end
static void arm_tr_tb_stop(DisasContextBase *dcbase, CPUState *cpu)
{
   // skipped ..... 
    /* At this stage dc->condjmp will only be set when the skipped
       instruction was a conditional branch or trap, and the PC has
       already been written.  */
    if (dc->condjmp) {
        /* "Condition failed" instruction codepath for the branch/trap insn */
        set_disas_label(dc, dc->condlabel);
        gen_set_condexec(dc);
        if (unlikely(dc->ss_active)) {
            gen_update_pc(dc, curr_insn_len(dc));
            gen_singlestep_exception(dc);
        } else {
            gen_goto_tb(dc, 1, curr_insn_len(dc));
        }
    }
}

```

Key steps in a Translation Block cycle:
- 1. Dissemble context initialization in `gen_intermediate_code/init_disas_context`: `condjmp = 0`, `pc_start/pc_first = tb->pc`, `is_jmp = DISAS_NEXT (0)`, `condlable (implicitly as NULL)`, `pc_save = tb->pc` (related to condlabel)
- 2. iterating the instruction one by one, then emit the corresponding IRs for each instr.
  - a. if the instruction is conditionally executed (`cond` is not 0xe)
    - i. the first instr in a tb of which cond is not AL, set `condjmp=1`, then generate a jump label through `arm_skip_unless`: means if the condition is not met, then jump to the label (skip the instr) --> then set the condjmp as 0 if `dc->condjmp && dc->base.is_jmp == DISAS_NEXT`
    - ii. `arm_post_translate_insn` will set the jump label after this instr, if condition is not met.
- 3. `am_tr_tb_stop` will process condition failed instruction code path.

If the conditional instr has `is_jmp != DISAS_NEXT` (`jne`), then `arm_post_translate_insn` will not set the jump label until end of the tb, `am_tr_tb_stop` will process this condition failed instruction code path.

## `DisasContext` and `TCGLabel`

```c
typedef struct DisasContext {
    target_ulong pc;
    uint32_t insn;
    int is_jmp;  
    /* Nonzero if this instruction has been conditionally skipped.  */
    int condjmp;
    /* The label that will be jumped to when the instruction is skipped.  */
    TCGLabel *condlabel;
}
typedef struct TCGLabel {
    unsigned has_value : 1;
    unsigned id : 31;
    union {
        uintptr_t value;
        tcg_insn_unit *value_ptr;
        TCGRelocation *first_reloc;
    } u;
} TCGLabel;
```

## `is_jmp` categories: DisasJumpType

`DISAS_*` codes are internal flags used during translation of a Translation Block (TB) in QEMU. They indicate what kind of control flow behavior should happen at the end of the TB.

```c
/**
 * DisasJumpType:
 * @DISAS_NEXT: Next instruction in program order.
 * @DISAS_TOO_MANY: Too many instructions translated.
 * @DISAS_NORETURN: Following code is dead.
 * @DISAS_TARGET_*: Start of target-specific conditions.
 *
 * What instruction to disassemble next.
 */
typedef enum DisasJumpType {
    DISAS_NEXT,
    DISAS_TOO_MANY,
    DISAS_NORETURN,
    DISAS_TARGET_0,
    DISAS_TARGET_1,
    DISAS_TARGET_2,
    DISAS_TARGET_3,
    DISAS_TARGET_4,
    DISAS_TARGET_5,
    DISAS_TARGET_6,
    DISAS_TARGET_7,
    DISAS_TARGET_8,
    DISAS_TARGET_9,
    DISAS_TARGET_10,
    DISAS_TARGET_11,
} DisasJumpType;

/* is_jmp field values */
#define DISAS_JUMP      DISAS_TARGET_0 /* only pc was modified dynamically */
/* CPU state was modified dynamically; exit to main loop for interrupts. */
#define DISAS_UPDATE_EXIT  DISAS_TARGET_1
/* These instructions trap after executing, so the A32/T32 decoder must
 * defer them until after the conditional execution state has been updated.
 * WFI also needs special handling when single-stepping.
 */
#define DISAS_WFI       DISAS_TARGET_2
#define DISAS_SWI       DISAS_TARGET_3
/* WFE */
#define DISAS_WFE       DISAS_TARGET_4
#define DISAS_HVC       DISAS_TARGET_5
#define DISAS_SMC       DISAS_TARGET_6
#define DISAS_YIELD     DISAS_TARGET_7
/* M profile branch which might be an exception return (and so needs
 * custom end-of-TB code)
 */
#define DISAS_BX_EXCRET DISAS_TARGET_8
/*
 * For instructions which want an immediate exit to the main loop, as opposed
 * to attempting to use lookup_and_goto_ptr.  Unlike DISAS_UPDATE_EXIT, this
 * doesn't write the PC on exiting the translation loop so you need to ensure
 * something (gen_a64_update_pc or runtime helper) has done so before we reach
 * return from cpu_tb_exec.
 */
#define DISAS_EXIT      DISAS_TARGET_9
/* CPU state was modified dynamically; no need to exit, but do not chain. */
#define DISAS_UPDATE_NOCHAIN  DISAS_TARGET_10
```

### Normal Case
```c
/* Jump, specifying which TB number to use if we gen_goto_tb() */
static void gen_jmp_tb(DisasContext *s, target_long diff, int tbno)
{
    if (unlikely(s->ss_active)) {
        /* An indirect jump so that we still trigger the debug exception.  */
        gen_update_pc(s, diff);
        s->base.is_jmp = DISAS_JUMP;
        return;
    }
    switch (s->base.is_jmp) {
    case DISAS_NEXT:
    case DISAS_TOO_MANY:
    case DISAS_NORETURN:
        /*
         * The normal case: just go to the destination TB.
         * NB: NORETURN happens if we generate code like
         *    gen_brcondi(l);
         *    gen_jmp();
         *    gen_set_label(l);
         *    gen_jmp();
         * on the second call to gen_jmp().
         */
        gen_goto_tb(s, tbno, diff);
        break;
    case DISAS_UPDATE_NOCHAIN:
    case DISAS_UPDATE_EXIT:
        /*
         * We already decided we're leaving the TB for some other reason. (e.g., page boundary, memory trap, signal, syscall)
         * Avoid using goto_tb so we really do exit back to the main loop
         * and don't chain to another TB.
         */
        gen_update_pc(s, diff);
        gen_goto_ptr();
        s->base.is_jmp = DISAS_NORETURN;
        break;
    default:
        /*
         * We shouldn't be emitting code for a jump and also have
         * is_jmp set to one of the special cases like DISAS_SWI.
         */
        g_assert_not_reached();
    }
}
```
| Code | Meaning | Typical Use | TB Chaining? | TB Ends? |
| --- | --- | --- | --- | --- |
| `DISAS_NEXT` | Continue to next instruction (fall-through) | Normal instruction sequence | ✅ Yes | ❌ No |
| `DISAS_TOO_MANY` | Too many instructions in this TB | TB limit hit, must end now | ✅ Yes | ✅ Yes |
| `DISAS_NORETURN` | No more control flow possible (e.g. `bx lr`) | Ends in a return, exception, or jump | ❌ No | ✅ Yes |

* * *
| When used | `DISAS_NEXT` | `DISAS_TOO_MANY` | `DISAS_NORETURN` |
| --- | --- | --- | --- |
| Fall-through? | ✅ Yes | ✅ Yes | ❌ No |
| Ends TB? | ❌ Not by itself | ✅ Yes (limit) | ✅ Yes (by nature) |
| Used with `gen_goto_tb()` | ✅ | ✅ | ❌ (use `exit_tb()` instead) |

* * *

- `DISAS_NEXT` is set as initial value:
  - Normal case, move on to the next instruction.
  - If at the end of the TB → use `gen_goto_tb()` to chain to next TB.
  - Doesn't terminate the TB unless translation naturally ends.
  - Special case: Transfers data from a CPU register to a coprocessor register, and there are no traps.
- `DISAS_TOO_MANY` is set when translation has to stop after too many instructions:
  - Still allows `gen_goto_tb()` (i.e., the TB ends with a direct jump). But forces the end of TB generation.
- `DISAS_NORETURN` means this TB will never return.
  - bx lr into non-emulated code
  - Exception, syscall ...
  - Indicates no need to emit `goto_tb()` (special case in `gen_jmp_tb()`). Just `exit_tb()` or return.
  - Some priority reasons need to abort and process (pc alignment)
- `DISAS_JUMP` is set when:
  - single-step state (most likely a GDB-support)
  - a bx indirect jump: 
    - `store_reg_from_load`: LDR/LDM/POP into r15
  - exception is generated by I/O operations, MRS...


### Main `translator_loop` in `translator.c`
If the `is_jmp` is not `DISAS_NEXT`, then leave the translation process.
```c
// ...
/* Stop translation if translate_insn so indicated.  */
if (db->is_jmp != DISAS_NEXT) {
    break;
}

/* Stop translation if the output buffer is full,
   or we have executed all of the allowed instructions.  */
if (tcg_op_buf_full() || db->num_insns >= db->max_insns) {
    db->is_jmp = DISAS_TOO_MANY;
    break;
}
```


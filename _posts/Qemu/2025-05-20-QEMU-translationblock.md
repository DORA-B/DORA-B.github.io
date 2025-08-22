---
title: QEMU Translation block
date: 2025-05-20
categories: [Qemu]
tags: [qemu]     # TAG names should always be lowercase
published: false
---


> Problem: I need to know how blocks are chained during tcg_out backend process.

# Translation Block structure

```c
struct TranslationBlock {
    /*
     * Guest PC corresponding to this block.  This must be the true
     * virtual address.  Therefore e.g. x86 stores EIP + CS_BASE, and
     * targets like Arm, MIPS, HP-PA, which reuse low bits for ISA or
     * privilege, must store those bits elsewhere.
     *
     * If CF_PCREL, the opcodes for the TranslationBlock are written
     * such that the TB is associated only with the physical page and
     * may be run in any virtual address context.  In this case, PC
     * must always be taken from ENV in a target-specific manner.
     * Unwind information is taken as offsets from the page, to be
     * deposited into the "current" PC.
     */
    vaddr pc;

    /*
     * Target-specific data associated with the TranslationBlock, e.g.:
     * x86: the original user, the Code Segment virtual base,
     * arm: an extension of tb->flags,
     * s390x: instruction data for EXECUTE,
     * sparc: the next pc of the instruction queue (for delay slots).
     */
    uint64_t cs_base;

    uint32_t flags; /* flags defining in which context the code was generated */
    uint32_t cflags;    /* compile flags */

/* Note that TCG_MAX_INSNS is 512; we validate this match elsewhere. */
#define CF_COUNT_MASK    0x000001ff
#define CF_NO_GOTO_TB    0x00000200 /* Do not chain with goto_tb */
#define CF_NO_GOTO_PTR   0x00000400 /* Do not chain with goto_ptr */
#define CF_SINGLE_STEP   0x00000800 /* gdbstub single-step in effect */
#define CF_MEMI_ONLY     0x00001000 /* Only instrument memory ops */
#define CF_USE_ICOUNT    0x00002000
#define CF_INVALID       0x00004000 /* TB is stale. Set with @jmp_lock held */
#define CF_PARALLEL      0x00008000 /* Generate code for a parallel context */
#define CF_NOIRQ         0x00010000 /* Generate an uninterruptible TB */
#define CF_PCREL         0x00020000 /* Opcodes in TB are PC-relative */
#define CF_CLUSTER_MASK  0xff000000 /* Top 8 bits are cluster ID */
#define CF_CLUSTER_SHIFT 24

    /*
     * Above fields used for comparing
     */

    /* size of target code for this block (1 <= size <= TARGET_PAGE_SIZE) */
    uint16_t size;
    uint16_t icount;

    struct tb_tc tc;

    /*
     * Track tb_page_addr_t intervals that intersect this TB.
     * For user-only, the virtual addresses are always contiguous,
     * and we use a unified interval tree.  For system, we use a
     * linked list headed in each PageDesc.  Within the list, the lsb
     * of the previous pointer tells the index of page_next[], and the
     * list is protected by the PageDesc lock(s).
     */
#ifdef CONFIG_USER_ONLY
    IntervalTreeNode itree;
#else
    uintptr_t page_next[2];
    tb_page_addr_t page_addr[2];
#endif

    /* jmp_lock placed here to fill a 4-byte hole. Its documentation is below */
    QemuSpin jmp_lock;

    /* The following data are used to directly call another TB from
     * the code of this one. This can be done either by emitting direct or
     * indirect native jump instructions. These jumps are reset so that the TB
     * just continues its execution. The TB can be linked to another one by
     * setting one of the jump targets (or patching the jump instruction). Only
     * two of such jumps are supported.
     */
#define TB_JMP_OFFSET_INVALID 0xffff /* indicates no jump generated */
    uint16_t jmp_reset_offset[2]; /* offset of original jump target */
    uint16_t jmp_insn_offset[2];  /* offset of direct jump insn */
    uintptr_t jmp_target_addr[2]; /* target address */

    /*
     * Each TB has a NULL-terminated list (jmp_list_head) of incoming jumps.
     * Each TB can have two outgoing jumps, and therefore can participate
     * in two lists. The list entries are kept in jmp_list_next[2]. The least
     * significant bit (LSB) of the pointers in these lists is used to encode
     * which of the two list entries is to be used in the pointed TB.
     *
     * List traversals are protected by jmp_lock. The destination TB of each
     * outgoing jump is kept in jmp_dest[] so that the appropriate jmp_lock
     * can be acquired from any origin TB.
     *
     * jmp_dest[] are tagged pointers as well. The LSB is set when the TB is
     * being invalidated, so that no further outgoing jumps from it can be set.
     *
     * jmp_lock also protects the CF_INVALID cflag; a jump must not be chained
     * to a destination TB that has CF_INVALID set.
     */
    uintptr_t jmp_list_head;
    uintptr_t jmp_list_next[2];
    uintptr_t jmp_dest[2];
}
```

## Operations related to translation block

- `tcg_gen_exit_tb`: where `exit_tb` is generated.
```c
/**
 * tcg_gen_exit_tb() - output exit_tb TCG operation
 * @tb: The TranslationBlock from which we are exiting
 * @idx: Direct jump slot index, or exit request
 *
 * See tcg/README for more info about this TCG operation.
 * See also tcg.h and the block comment above TB_EXIT_MASK.
 *
 * For a normal exit from the TB, back to the main loop, @tb should
 * be NULL and @idx should be 0.  Otherwise, @tb should be valid and
 * @idx should be one of the TB_EXIT_ values.
 */
```
- tcg_gen_goto_tb
```c
/**
 * tcg_gen_goto_tb() - output goto_tb TCG operation
 * @idx: Direct jump slot index (0 or 1)
 *
 * See tcg/README for more info about this TCG operation.
 *
 * NOTE: In system emulation, direct jumps with goto_tb are only safe within
 * the pages this TB resides in because we don't take care of direct jumps when
 * address mapping changes, e.g. in tlb_flush(). In user mode, there's only a
 * static address translation, so the destination address is always valid, TBs
 * are always invalidated properly, and direct jumps are reset when mapping
 * changes.
 */
```
- tcg_gen_lookup_and_goto_ptr
```c
/**
 * tcg_gen_lookup_and_goto_ptr() - look up the current TB, jump to it if valid
 * @addr: Guest address of the target TB
 *
 * If the TB is not valid, jump to the epilogue.
 *
 * This operation is optional. If the TCG backend does not implement goto_ptr,
 * this op is equivalent to calling tcg_gen_exit_tb() with 0 as the argument.
 */
```
```c
/* QEMU specific operations.  */

void tcg_gen_exit_tb(const TranslationBlock *tb, unsigned idx)
{
    /*
     * Let the jit code return the read-only version of the
     * TranslationBlock, so that we minimize the pc-relative
     * distance of the address of the exit_tb code to TB.
     * This will improve utilization of pc-relative address loads.
     *
     * TODO: Move this to translator_loop, so that all const
     * TranslationBlock pointers refer to read-only memory.
     * This requires coordination with targets that do not use
     * the translator_loop.
     */
    uintptr_t val = (uintptr_t)tcg_splitwx_to_rx((void *)tb) + idx;

    if (tb == NULL) {
        tcg_debug_assert(idx == 0);
    } else if (idx <= TB_EXIT_IDXMAX) {
#ifdef CONFIG_DEBUG_TCG
        /* This is an exit following a goto_tb.  Verify that we have
           seen this numbered exit before, via tcg_gen_goto_tb.  */
        tcg_debug_assert(tcg_ctx->goto_tb_issue_mask & (1 << idx));
#endif
    } else {
        /* This is an exit via the exitreq label.  */
        tcg_debug_assert(idx == TB_EXIT_REQUESTED);
    }

    tcg_gen_op1i(INDEX_op_exit_tb, val);
}

void tcg_gen_goto_tb(unsigned idx)
{
    /* We tested CF_NO_GOTO_TB in translator_use_goto_tb. */
    tcg_debug_assert(!(tcg_ctx->gen_tb->cflags & CF_NO_GOTO_TB));
    /* We only support two chained exits.  */
    tcg_debug_assert(idx <= TB_EXIT_IDXMAX);
#ifdef CONFIG_DEBUG_TCG
    /* Verify that we haven't seen this numbered exit before.  */
    tcg_debug_assert((tcg_ctx->goto_tb_issue_mask & (1 << idx)) == 0);
    tcg_ctx->goto_tb_issue_mask |= 1 << idx;
#endif
    plugin_gen_disable_mem_helpers();
    tcg_gen_op1i(INDEX_op_goto_tb, idx);
}

void tcg_gen_lookup_and_goto_ptr(void)
{
    TCGv_ptr ptr;

    if (tcg_ctx->gen_tb->cflags & CF_NO_GOTO_PTR) {
        tcg_gen_exit_tb(NULL, 0);
        return;
    }

    plugin_gen_disable_mem_helpers();
    ptr = tcg_temp_ebb_new_ptr();
    gen_helper_lookup_tb_ptr(ptr, tcg_env);
    tcg_gen_op1i(INDEX_op_goto_ptr, tcgv_ptr_arg(ptr));
    tcg_temp_free_ptr(ptr);
}
```
## Order
- 0. `tcg_gen_exit_tb` generates `INDEX_op_exit_tb`.
- 1. `tcg_gen_exit_tb` called by 
    - `gen_tb_end`: `accel/tcg/translator.c`
    - `gen_goto_tb`: `target/arm/tcg/translate.c`
    - (`gen_bx_excret_final_code`, `arm_tr_tb_stop` --> For debug exception)
### Call `tcg_gen_exit_tb` in `gen_tb_end`
```c
/**
 * tcg_qemu_tb_exec:
 * @env: pointer to CPUArchState for the CPU
 * @tb_ptr: address of generated code for the TB to execute
 *
 * Start executing code from a given translation block.
 * Where translation blocks have been linked, execution
 * may proceed from the given TB into successive ones.
 * Control eventually returns only when some action is needed
 * from the top-level loop: either control must pass to a TB
 * which has not yet been directly linked, or an asynchronous
 * event such as an interrupt needs handling.
 *
 * Return: The return value is the value passed to the corresponding
 * tcg_gen_exit_tb() at translation time of the last TB attempted to execute.
 * The value is either zero or a 4-byte aligned pointer to that TB combined
 * with additional information in its two least significant bits. The
 * additional information is encoded as follows:
 *  0, 1: the link between this TB and the next is via the specified
 *        TB index (0 or 1). That is, we left the TB via (the equivalent
 *        of) "goto_tb <index>". The main loop uses this to determine
 *        how to link the TB just executed to the next.
 *  2:    we are using instruction counting code generation, and we
 *        did not start executing this TB because the instruction counter
 *        would hit zero midway through it. In this case the pointer
 *        returned is the TB we were about to execute, and the caller must
 *        arrange to execute the remaining count of instructions.
 *  3:    we stopped because the CPU's exit_request flag was set
 *        (usually meaning that there is an interrupt that needs to be
 *        handled). The pointer returned is the TB we were about to execute
 *        when we noticed the pending exit request.
 *
 * If the bottom two bits indicate an exit-via-index then the CPU
 * state is correctly synchronised and ready for execution of the next
 * TB (and in particular the guest PC is the address to execute next).
 * Otherwise, we gave up on execution of this TB before it started, and
 * the caller must fix up the CPU state by calling the CPU's
 * synchronize_from_tb() method with the TB pointer we return (falling
 * back to calling the CPU's set_pc method with tb->pb if no
 * synchronize_from_tb() method exists).
 *
 * Note that TCG targets may use a different definition of tcg_qemu_tb_exec
 * to this default (which just calls the prologue.code emitted by
 * tcg_target_qemu_prologue()).
 */
#define TB_EXIT_MASK      3
#define TB_EXIT_IDX0      0
#define TB_EXIT_IDX1      1
#define TB_EXIT_IDXMAX    1
#define TB_EXIT_REQUESTED 3

static void gen_tb_end(const TranslationBlock *tb, uint32_t cflags,
                       TCGOp *icount_start_insn, int num_insns)
{
    if (cflags & CF_USE_ICOUNT) {
        /*
         * Update the num_insn immediate parameter now that we know
         * the actual insn count.
         */
        tcg_set_insn_param(icount_start_insn, 2,
                           tcgv_i32_arg(tcg_constant_i32(num_insns)));
    }

    if (tcg_ctx->exitreq_label) {
        gen_set_label(tcg_ctx->exitreq_label);
        tcg_gen_exit_tb(tb, TB_EXIT_REQUESTED);
    }
}
```
`gen_tb_end` --> `translator_loop` --> `gen_intermediate_code` --> `setjmp_gen_code` --> `tb_gen_code` --> `cpu_exec_loop`

#### How to set/use the execution pc during execution 
- `cpu_get_tb_cpu_state` set the pc from env to execute.
- `tb_gen_code` execute the tb from pc
  - Init DisasmbleContext
    - set `pc_first`, `pc_curr` --> start addr of the tb
    - set `pc_next` --> pc + len(instr)
    - when there is a control flow change --> then change the `pc_save`
- `cpu_loop_exec_tb` will be executed after `tb_gen_code`
- `cpu_tb_exec` --> set pc back in env

### Call `tcg_gen_exit_tb` in `gen_goto_tb`

```c

/* This will end the TB but doesn't guarantee we'll return to
 * cpu_loop_exec. Any live exit_requests will be processed as we
 * enter the next TB.
 */
static void gen_goto_tb(DisasContext *s, int n, target_long diff)
{
    if (translator_use_goto_tb(&s->base, s->pc_curr + diff)) {
        /*
         * For pcrel, the pc must always be up-to-date on entry to
         * the linked TB, so that it can use simple additions for all
         * further adjustments.  For !pcrel, the linked TB is compiled
         * to know its full virtual address, so we can delay the
         * update to pc to the unlinked path.  A long chain of links
         * can thus avoid many updates to the PC.
         */
        if (tb_cflags(s->base.tb) & CF_PCREL) {
            gen_update_pc(s, diff);
            tcg_gen_goto_tb(n);
        } else {
            tcg_gen_goto_tb(n);
            gen_update_pc(s, diff);
        }
        tcg_gen_exit_tb(s->base.tb, n);
    } else {
        gen_update_pc(s, diff);
        gen_goto_ptr();
    }
    s->base.is_jmp = DISAS_NORETURN;
}
```
- `gen_goto_tb` --> `arm_tr_tb_stop`  --> `arm_translator_ops.tb_stop` --> `ops->tb_stop(db, cpu);` --> `translator_loop`

- `gen_goto_tb` --> `gen_jmp_tb` --> `gen_jmp`
```c
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
```

### `gen_jmp`
```c
/*
 * Branch, branch with link
 */

static bool trans_B(DisasContext *s, arg_i *a)
{
    gen_jmp(s, jmp_diff(s, a->imm));
    return true;
}
/* The pc_curr difference for an architectural jump. */
static target_long jmp_diff(DisasContext *s, target_long diff)
{
    return diff + (s->thumb ? 4 : 8);
}
static inline void gen_jmp(DisasContext *s, target_long diff)
{
    gen_jmp_tb(s, diff, 0);
}
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
         * We already decided we're leaving the TB for some other reason.
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

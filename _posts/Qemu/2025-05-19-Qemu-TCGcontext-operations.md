---
title: TCGContext operations
date: 2025-05-18
categories: [Qemu]
tags: [qemu]     # TAG names should always be lowercase
published: false
---

> Problem1: I need to find out how `TCGContext` is maintained.


# Functions in Liveness Analysis
```c
/* liveness analysis: end of function: all temps are dead, and globals
   should be in memory. */
static void la_func_end(TCGContext *s, int ng, int nt)
{
    int i;

    for (i = 0; i < ng; ++i) {
        s->temps[i].state = TS_DEAD | TS_MEM;
        la_reset_pref(&s->temps[i]);
    }
    for (i = ng; i < nt; ++i) {
        s->temps[i].state = TS_DEAD;
        la_reset_pref(&s->temps[i]);
    }
}
/* liveness analysis: end of basic block: all temps are dead, globals
   and local temps should be in memory. */
static void la_bb_end(TCGContext *s, int ng, int nt);
/* liveness analysis: sync globals back to memory.  */
static void la_global_sync(TCGContext *s, int ng)
{
    int i;

    for (i = 0; i < ng; ++i) {
        int state = s->temps[i].state;
        s->temps[i].state = state | TS_MEM;
        if (state == TS_DEAD) {
            /* If the global was previously dead, reset prefs.  */
            la_reset_pref(&s->temps[i]);
        }
    }
}
/*
 * liveness analysis: conditional branch: all temps are dead unless
 * explicitly live-across-conditional-branch, globals and local temps
 * should be synced.
 */
static void la_bb_sync(TCGContext *s, int ng, int nt);
/* liveness analysis: sync globals back to memory and kill.  */
static void la_global_kill(TCGContext *s, int ng);
/* liveness analysis: note live globals crossing calls.  */
static void la_cross_call(TCGContext *s, int nt);

```

# Functions in tcg_out

```c
/* Assign @reg to @ts, and update reg_to_temp[]. */
static void set_temp_val_reg(TCGContext *s, TCGTemp *ts, TCGReg reg)
{
    if (ts->val_type == TEMP_VAL_REG) {
        TCGReg old = ts->reg;
        tcg_debug_assert(s->reg_to_temp[old] == ts);
        if (old == reg) {
            return;
        }
        s->reg_to_temp[old] = NULL;
    }
    tcg_debug_assert(s->reg_to_temp[reg] == NULL);
    s->reg_to_temp[reg] = ts;
    ts->val_type = TEMP_VAL_REG;
    ts->reg = reg;
}
/* Assign a non-register value type to @ts, and update reg_to_temp[]. */
static void set_temp_val_nonreg(TCGContext *s, TCGTemp *ts, TCGTempVal type)
{
    tcg_debug_assert(type != TEMP_VAL_REG);
    if (ts->val_type == TEMP_VAL_REG) {
        TCGReg reg = ts->reg;
        tcg_debug_assert(s->reg_to_temp[reg] == ts);
        s->reg_to_temp[reg] = NULL;
    }
    ts->val_type = type;
}
/* Make sure the temporary is in a register.  If needed, allocate the register
   from DESIRED while avoiding ALLOCATED.  */
static void temp_load(TCGContext *s, TCGTemp *ts, TCGRegSet desired_regs,
                      TCGRegSet allocated_regs, TCGRegSet preferred_regs);
/* Mark a temporary as free or dead.  If 'free_or_dead' is negative,
   mark it free; otherwise mark it dead.  */
static void temp_free_or_dead(TCGContext *s, TCGTemp *ts, int free_or_dead);
/* Mark a temporary as dead.  */
static inline void temp_dead(TCGContext *s, TCGTemp *ts)
{
    temp_free_or_dead(s, ts, 1);
}
/* Sync a temporary to memory. 'allocated_regs' is used in case a temporary
   registers needs to be allocated to store a constant.  If 'free_or_dead'
   is non-zero, subsequently release the temporary; if it is positive, the
   temp is dead; if it is negative, the temp is free.  */
static void temp_sync(TCGContext *s, TCGTemp *ts, TCGRegSet allocated_regs,
                      TCGRegSet preferred_regs, int free_or_dead)
{
    if (!temp_readonly(ts) && !ts->mem_coherent) {
        if (!ts->mem_allocated) {
            temp_allocate_frame(s, ts);
        }
        switch (ts->val_type) {
        case TEMP_VAL_CONST:
            /* If we're going to free the temp immediately, then we won't
               require it later in a register, so attempt to store the
               constant to memory directly.  */
            if (free_or_dead
                && tcg_out_sti(s, ts->type, ts->val,
                               ts->mem_base->reg, ts->mem_offset)) {
                break;
            }
            temp_load(s, ts, tcg_target_available_regs[ts->type],
                      allocated_regs, preferred_regs);
            /* fallthrough */

        case TEMP_VAL_REG:
            tcg_out_st(s, ts->type, ts->reg,
                       ts->mem_base->reg, ts->mem_offset);
            break;

        case TEMP_VAL_MEM:
            break;

        case TEMP_VAL_DEAD:
        default:
            g_assert_not_reached();
        }
        ts->mem_coherent = 1;
    }
    if (free_or_dead) {
        temp_free_or_dead(s, ts, free_or_dead);
    }
}
/* free register 'reg' by spilling the corresponding temporary if necessary */
static void tcg_reg_free(TCGContext *s, TCGReg reg, TCGRegSet allocated_regs)
{
    TCGTemp *ts = s->reg_to_temp[reg];
    if (ts != NULL) {
        temp_sync(s, ts, allocated_regs, 0, -1);
    }
}
/* Make sure the temporary is in a register.  If needed, allocate the register
   from DESIRED while avoiding ALLOCATED.  */
static void temp_load(TCGContext *s, TCGTemp *ts, TCGRegSet desired_regs,
                      TCGRegSet allocated_regs, TCGRegSet preferred_regs);
/* Save a temporary to memory. 'allocated_regs' is used in case a
   temporary registers needs to be allocated to store a constant.  */
static void temp_save(TCGContext *s, TCGTemp *ts, TCGRegSet allocated_regs);
/* save globals to their canonical location and assume they can be
   modified be the following code. 'allocated_regs' is used in case a
   temporary registers needs to be allocated to store a constant. */
static void save_globals(TCGContext *s, TCGRegSet allocated_regs);

/* sync globals to their canonical location and assume they can be
   read by the following code. 'allocated_regs' is used in case a
   temporary registers needs to be allocated to store a constant. */
static void sync_globals(TCGContext *s, TCGRegSet allocated_regs);
/* at the end of a basic block, we assume all temporaries are dead and
   all globals are stored at their canonical location. */
static void tcg_reg_alloc_bb_end(TCGContext *s, TCGRegSet allocated_regs);

/*
 * At a conditional branch, we assume all temporaries are dead unless
 * explicitly live-across-conditional-branch; all globals and local
 * temps are synced to their location.
 */
static void tcg_reg_alloc_cbranch(TCGContext *s, TCGRegSet allocated_regs) 

```
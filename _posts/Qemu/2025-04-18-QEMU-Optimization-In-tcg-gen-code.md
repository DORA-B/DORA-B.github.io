---
title: Optimization in tcg_gen_code
date: 2025-04-05
categories: [Qemu]
tags: [qemu_opt]     # TAG names should always be lowercase
published: false
---

> When look into the code of tcg_gen_code, there are multiple steps of optimization

```c
int tcg_gen_code(TCGContext *s, TranslationBlock *tb, uint64_t pc_start)
{
    int i, start_words, num_insns;
    TCGOp *op;

    // log and debug tcg enabled
    // ...

    tcg_optimize(s);
    reachable_code_pass(s);
    liveness_pass_0(s);
    liveness_pass_1(s);
    // 
}
```

> I need to know the logic of `tcg_optmize()` before I move on.

# Whole process

## 1. Initialization Process
### Part A: Context Setup:
- Creates an optimization context structure
```c
OptContext ctx = { .tcg = s };
QSIMPLEQ_INIT(&ctx.mem_free);
```  
- `OptContext` Structure.
```c
typedef struct OptContext {
    TCGContext *tcg;
    TCGOp *prev_mb;
    TCGTempSet temps_used;

    IntervalTreeRoot mem_copy;
    QSIMPLEQ_HEAD(, MemCopyInfo) mem_free;

    /* In flight values from optimization. */
    uint64_t a_mask;  /* mask bit is 0 iff value identical to first input */
    uint64_t z_mask;  /* mask bit is 0 iff value bit is 0 */
    uint64_t s_mask;  /* mask of clrsb(value) bits */
    TCGType type;
} OptContext;
```
- the optimization state stored per temp, specifically via the TempOptInfo struct, which is attached to each TCGTemp.
```c
typedef struct MemCopyInfo {
    IntervalTreeNode itree;
    QSIMPLEQ_ENTRY (MemCopyInfo) next;
    TCGTemp *ts;
    TCGType type;
} MemCopyInfo;

typedef struct TempOptInfo {
    bool is_const;       // a 
    TCGTemp *prev_copy;
    TCGTemp *next_copy; // circular linked list
    QSIMPLEQ_HEAD(, MemCopyInfo) mem_copy;
    uint64_t val;
    uint64_t z_mask;  /* mask bit is 0 if and only if value bit is 0 */
    uint64_t s_mask;  /* a left-aligned mask of clrsb(value) bits. */
} TempOptInfo;

typedef struct TCGTemp {
    TCGReg reg:8;
    TCGTempVal val_type:8;
    TCGType base_type:8;
    TCGType type:8; // e.g. readonly: TEMP_FIXED
    TCGTempKind kind:3;
    unsigned int indirect_reg:1;
    unsigned int indirect_base:1;
    unsigned int mem_coherent:1;
    unsigned int mem_allocated:1;
    unsigned int temp_allocated:1;
    unsigned int temp_subindex:2;

    int64_t val;
    struct TCGTemp *mem_base;
    intptr_t mem_offset;
    const char *name;

    /* Pass-specific information that can be stored for a temporary.
       One word worth of integer data, and one pointer to data
       allocated separately.  */
    uintptr_t state;
    void *state_ptr; //
} TCGTemp;
```

### Part B: Temporary Register State Initialization
- Prepares temp regs for tracking constants and copies during optimization
```c
/* Array VALS has an element for each temp.
 If this temp holds a constant then its value is kept in VALS' element.
 If this temp is a copy of other ones then the other copies are
 available through the doubly linked circular list. */
nb_temps = s->nb_temps;
for (i = 0; i < nb_temps; ++i) {
    s->temps[i].state_ptr = NULL;
}
```
- and later in `init_ts_info`
```c
/* Initialize and activate a temporary.  */
static void init_ts_info(OptContext *ctx, TCGTemp *ts)
{
    size_t idx = temp_idx(ts);
    TempOptInfo *ti;
    // avoid redundant initialization
    if (test_bit(idx, ctx->temps_used.l)) {
        return;
    }
    set_bit(idx, ctx->temps_used.l);

    ti = ts->state_ptr;
    if (ti == NULL) {
        ti = tcg_malloc(sizeof(TempOptInfo));
        ts->state_ptr = ti;
    }
    // Allocate TempOptInfo and attach it
    ti->next_copy = ts;
    ti->prev_copy = ts;
    QSIMPLEQ_INIT(&ti->mem_copy);
    if (ts->kind == TEMP_CONST) {
        ti->is_const = true;
        ti->val = ts->val;
        ti->z_mask = ts->val;
        ti->s_mask = smask_from_value(ts->val);
    } else {
        ti->is_const = false;
        ti->z_mask = -1;
        ti->s_mask = 0;
    }
}
```
## 2. Propagation Process

For each operation in the TCG operations list, the following steps occur:
- Specially handle call operation. 
- Gets the operation definition
- Initializes arguments
- Performs copy propagation
- Sets up operation type (32-bit, 64-bit, or vector)


### Part A: Specially handle function call operation.
- In `tcg_optimize()` process, why calls are special?
```c
/* Calls are special. */
if (opc == INDEX_op_call) {
    fold_call(&ctx, op);
    continue;
}
```
- might modify global memory
- might modify global variables
- might have side effects (e.g., I/O, syscalls)
- don't know what the callee function will touch (unless flags tell you)

- fold_call: specially handles calls
```c
static bool fold_call(OptContext *ctx, TCGOp *op)
{
    TCGContext *s = ctx->tcg;
    int nb_oargs = TCGOP_CALLO(op);
    int nb_iargs = TCGOP_CALLI(op);
    int flags, i;
    // This initializes the optimization context’s tracking info (temps_used, temps_read, temps_written) for the arguments used in this call
    init_arguments(ctx, op, nb_oargs + nb_iargs); //  [0, ... nb_oargs -1 , nb_oargs, ... nb_oargs + nb_iargs - 1]
    //op->args[nb_oargs .. nb_oargs+nb_iargs-1] → input arguments (read by call)
    // copy propagation is only performed for the input arguments.
    copy_propagate(ctx, op, nb_oargs, nb_iargs);

    /* If the function reads or writes globals, reset temp data. */
    flags = tcg_call_flags(op);
    if (!(flags & (TCG_CALL_NO_READ_GLOBALS | TCG_CALL_NO_WRITE_GLOBALS))) {
        int nb_globals = s->nb_globals;

        for (i = 0; i < nb_globals; i++) {
            if (test_bit(i, ctx->temps_used.l)) {
                reset_ts(ctx, &ctx->tcg->temps[i]);
            }
        }
    }

    /* If the function has side effects, reset mem data. */
    if (!(flags & TCG_CALL_NO_SIDE_EFFECTS)) {
        remove_mem_copy_all(ctx);
    }

    /* Reset temp data for outputs. */
    for (i = 0; i < nb_oargs; i++) {
        reset_temp(ctx, op->args[i]);
    }

    /* Stop optimizing MB across calls. */
    ctx->prev_mb = NULL;
    return true;
}
```
- TCGOp structure
```c
struct TCGOp {
    TCGOpcode opc   : 8;
    unsigned nargs  : 8;

    /* Parameters for this opcode.  See below.  */
    unsigned param1 : 8;
    unsigned param2 : 8;

    /* Lifetime data of the operands.  */
    TCGLifeData life;

    /* Next and previous opcodes.  */
    QTAILQ_ENTRY(TCGOp) link;

    /* Register preferences for the output(s).  */
    TCGRegSet output_pref[2];

    /* Arguments for the opcode.  */
    TCGArg args[];
};
```

#### In `init_arguments`
- calls `init_ts_info`, init every arguments in the TCGop
```c
static void init_arguments(OptContext *ctx, TCGOp *op, int nb_args)
{
    for (int i = 0; i < nb_args; i++) {
        TCGTemp *ts = arg_temp(op->args[i]);
        init_ts_info(ctx, ts);
    }
}
```
#### In `copy_propagate`
- The propagation is only performed for the input arguments. It will find better copy to replace the current one, 
```c
static void copy_propagate(OptContext *ctx, TCGOp *op,
                           int nb_oargs, int nb_iargs)
{
    for (int i = nb_oargs; i < nb_oargs + nb_iargs; i++) {
        TCGTemp *ts = arg_temp(op->args[i]);
        if (ts_is_copy(ts)) {
            op->args[i] = temp_arg(find_better_copy(ts));
        }
    }
}
static TCGTemp *find_better_copy(TCGTemp *ts)
{
    TCGTemp *i, *ret;

    /* If this is already readonly, we can't do better. */
    // If the temporary is readonly (i.e., it refers to something like a global, constant, or memory input), 
    // then it’s already as "canonical" as it gets. So, return it immediately.
    if (temp_readonly(ts)) {
        return ts;
    }
    ret = ts;
    // This iterates over the circular linked list of copies.
    for (i = ts_info(ts)->next_copy; i != ts; i = ts_info(i)->next_copy) {
        ret = cmp_better_copy(ret, i);
    }
    return ret;
}
```
  - It improves the quality of generated code by collapsing multiple temps into one.
  - This can reduce register pressure and increase chances of further optimization.
  - Also helps avoid unnecessary moves like `mov r1, r2`; `mov r2, r3` chains.
#### Memory values handler: remove_mem_copy_all
```c
/* If the function has side effects, reset mem data. */
// then Discard all assumptions about memory contents — e.g., copied memory values
// This prevents incorrect propagation across potentially mutating operations.
if (!(flags & TCG_CALL_NO_SIDE_EFFECTS)) {
    remove_mem_copy_all(ctx);
}
static void remove_mem_copy_all(OptContext *ctx)
{
    remove_mem_copy_in(ctx, 0, -1);
    tcg_debug_assert(interval_tree_is_empty(&ctx->mem_copy));
}
static void remove_mem_copy_in(OptContext *ctx, intptr_t s, intptr_t l)
{
    while (true) {
        MemCopyInfo *mc = mem_copy_first(ctx, s, l);
        if (!mc) {
            break;
        }
        remove_mem_copy(ctx, mc);
    }
}
static MemCopyInfo *mem_copy_first(OptContext *ctx, intptr_t s, intptr_t l)
{
    IntervalTreeNode *r = interval_tree_iter_first(&ctx->mem_copy, s, l);
    return r ? container_of(r, MemCopyInfo, itree) : NULL;
}
IntervalTreeNode *interval_tree_iter_first(IntervalTreeRoot *root,
                                           uint64_t start, uint64_t last)
{
    IntervalTreeNode *node, *leftmost;

    if (!root || !root->rb_root.rb_node) {
        return NULL;
    }

    /*
     * Fastpath range intersection/overlap between A: [a0, a1] and
     * B: [b0, b1] is given by:
     *
     *         a0 <= b1 && b0 <= a1
     *
     *  ... where A holds the lock range and B holds the smallest
     * 'start' and largest 'last' in the tree. For the later, we
     * rely on the root node, which by augmented interval tree
     * property, holds the largest value in its last-in-subtree.
     * This allows mitigating some of the tree walk overhead for
     * for non-intersecting ranges, maintained and consulted in O(1).
     */
    node = rb_to_itree(root->rb_root.rb_node);
    if (node->subtree_last < start) {
        return NULL;
    }

    leftmost = rb_to_itree(root->rb_leftmost);
    if (leftmost->start > last) {
        return NULL;
    }

    return interval_tree_subtree_search(node, start, last);
}
```
- Original mechanism for tracks actual memory contents (aliases): load-store forwarding
```c
st_i32 r1, r2     // store r2 into MEM[r1]
ld_i32 r3, r1     // load from MEM[r1]
// after using load-store forwarding
mov_i32 r3, r2
```
it’s only safe if there is no function call in between that might change memory.
it is crucial for preserving correctness across function calls that may mutate memory.
- If the function call has side effects (i.e., it might write to memory), we must discard all previously known memory equivalences or copies.
```c
st_i32 r1, r2     // MEM[r1] = r2
// call f() BUT if there's a function call in between: f() might write to MEM[r1], we must invalidate the memory copy info.
ld_i32 r3, r1     // r3 = MEM[r1]
```
- in theory it could just clear affected memory regions, but in practice, QEMU uses a simpler conservative approach:
- Invalidate all memory state if the function call might touch memory.
### Part B: Init arguments and do copy propagation

#### In init_argument
- TCGOpDef
- where each entry describes:
- How many output arguments the opcode has (nb_oargs)
- How many input arguments it has (nb_iargs)
- What flags the opcode has (flags), such as `TCG_OPF_64BIT`, `TCG_OPF_VECTOR`, etc.
```c
typedef struct TCGOpDef {
    const char *name;
    uint8_t nb_oargs, nb_iargs, nb_cargs, nb_args;
    uint8_t flags;
    TCGArgConstraint *args_ct;
} TCGOpDef;
```
#### In copy_propagate, it checks 
- Is this temp just a copy of another one?
- If yes → replace it with the best available copy using find_better_copy()


Together, this prepares each instruction in the TCG IR for further optimization by:
- Understanding which temps are involved
- Propagating known copies
- Enabling later stages (like constant folding, dead code removal)

The propagation maintains a doubly-linked circular list of copies, allowing the optimizer to:
- Track which temporaries are copies of each other
- Propagate constants through copy chains
- Optimize away redundant moves
- Track memory operations, but only for register (temp) arguments in memory ops.
  - does not eliminate loads/store directly (alias analysis, load-store forwarding, memory SSA)
```c
mov_i32 r1, r0          // r1 = r0
ld_i32 r2, r1           // r2 = MEM[r1]
/*******
after copy propagation
********/
ld_i32 r2, r0           // r2 = MEM[r0]
```

## 3. Process each opcode
The core of the constant folding / strength reduction pass in QEMU’s TCG optimizer.


```c
/*
 * Process each opcode.
 * Sorted alphabetically by opcode as much as possible.
 */
switch (opc) {
CASE_OP_32_64(add):
    // add_i32 r0, r1, 0 --> mov_i32 r0, r1
    done = fold_add(&ctx, op);
    break;
case INDEX_op_add_vec:
    done = fold_add_vec(&ctx, op);
    break;
CASE_OP_32_64(add2):
    done = fold_add2(&ctx, op);
    break;
CASE_OP_32_64_VEC(and):
    // and_i32 r0, r1, -1  // bitwise AND with all 1s --> mov_i32 r0, r1
    done = fold_and(&ctx, op);
    break;
CASE_OP_32_64_VEC(andc):
    done = fold_andc(&ctx, op);
    break;
CASE_OP_32_64(brcond):
    // if (r1 == 0) goto L1
    // mov_i32 r1, 0
    // brcond_i32 r1, 0, TCG_COND_EQ, label1
      // goto label1
      // Condition is false — branch removed
    done = fold_brcond(&ctx, op);
    break;
```

### Specailly For ldr/str
```c
static void record_mem_copy(OptContext *ctx, TCGType type,
                            TCGTemp *ts, intptr_t start, intptr_t last)
{
    MemCopyInfo *mc;
    TempOptInfo *ti;

    mc = QSIMPLEQ_FIRST(&ctx->mem_free);
    if (mc) {
        QSIMPLEQ_REMOVE_HEAD(&ctx->mem_free, next);
    } else {
        mc = tcg_malloc(sizeof(*mc));
    }

    memset(mc, 0, sizeof(*mc));
    mc->itree.start = start;
    mc->itree.last = last;
    mc->type = type;
    interval_tree_insert(&mc->itree, &ctx->mem_copy);

    ts = find_better_copy(ts);
    ti = ts_info(ts);
    mc->ts = ts;
    QSIMPLEQ_INSERT_TAIL(&ti->mem_copy, mc, next);
}

static bool fold_tcg_ld_memcopy(OptContext *ctx, TCGOp *op)
{
    TCGTemp *dst, *src;
    intptr_t ofs;
    TCGType type;

    if (op->args[1] != tcgv_ptr_arg(tcg_env)) {
        return false;
    }

    type = ctx->type;
    ofs = op->args[2];
    dst = arg_temp(op->args[0]);
    src = find_mem_copy_for(ctx, type, ofs);
    if (src && src->base_type == type) {
        return tcg_opt_gen_mov(ctx, op, temp_arg(dst), temp_arg(src));
    }

    reset_ts(ctx, dst);
    record_mem_copy(ctx, type, dst, ofs, ofs + tcg_type_size(type) - 1);
    return true;
}

static bool fold_tcg_st(OptContext *ctx, TCGOp *op)
{
    intptr_t ofs = op->args[2];
    intptr_t lm1;

    if (op->args[1] != tcgv_ptr_arg(tcg_env)) {
        remove_mem_copy_all(ctx);
        return false;
    }

    switch (op->opc) {
    CASE_OP_32_64(st8):
        lm1 = 0;
        break;
    CASE_OP_32_64(st16):
        lm1 = 1;
        break;
    case INDEX_op_st32_i64:
    case INDEX_op_st_i32:
        lm1 = 3;
        break;
    case INDEX_op_st_i64:
        lm1 = 7;
        break;
    case INDEX_op_st_vec:
        lm1 = tcg_type_size(ctx->type) - 1;
        break;
    default:
        g_assert_not_reached();
    }
    remove_mem_copy_in(ctx, ofs, ofs + lm1);
    return false;
}

static bool fold_tcg_st_memcopy(OptContext *ctx, TCGOp *op)
{
    TCGTemp *src;
    intptr_t ofs, last;
    TCGType type;

    if (op->args[1] != tcgv_ptr_arg(tcg_env)) {
        fold_tcg_st(ctx, op);
        return false;
    }

    src = arg_temp(op->args[0]);
    ofs = op->args[2];
    type = ctx->type;

    /*
     * Eliminate duplicate stores of a constant.
     * This happens frequently when the target ISA zero-extends.
     */
    if (ts_is_const(src)) {
        TCGTemp *prev = find_mem_copy_for(ctx, type, ofs);
        if (src == prev) {
            tcg_op_remove(ctx->tcg, op);
            return true;
        }
    }

    last = ofs + tcg_type_size(type) - 1;
    remove_mem_copy_in(ctx, ofs, last);
    record_mem_copy(ctx, type, src, ofs, last);
    return false;
}
```
QEMU keeps track of known memory contents using a structure called MemCopyInfo.
This allows it to:

- Eliminate redundant loads: if we just stored a value to an address, and load it again, we can skip the memory access.
- Eliminate redundant stores: if the same constant is being stored again to the same address, skip the store.
- record_mem_copy()
  - Records that memory at [start, last] is now equal to the value of a particular temp `ts`
  - Inserts that info into an interval tree `ctx->mem_copy`
  - Links this `MemCopyInfo` into the temp’s `mem_copy` list
- fold_tcg_ld_memcopy()
  - remember that we just loaded a value from memory
- fold_tcg_st_memcopy()
  - Case 1: Store of the same constant again
    - Second store is redundant, so it's removed.
  - Case 2: General case
    - Remove prior memory copy info in that range
    - to remember that we just stored a value to memory
- fold_tcg_st()
  - If the store base is not env (i.e., unknown aliasing), invalidate all memory copy info.
  - Otherwise, for safe known memory (e.g., env)
    - Determine how many bytes are being affected by this st
    - then removes any stale copy info from overlapping memory ranges.

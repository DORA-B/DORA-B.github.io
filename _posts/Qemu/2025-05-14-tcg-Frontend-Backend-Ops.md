---
title: TCG operations
date: 2025-05-14
categories: [Qemu]
tags: [qemu]     # TAG names should always be lowercase
published: false
---


> Problem: I am looking into the tcg translation process, There are lots of tcg operations.

> `tcg_out_*` means “write something (an instruction or part of it) out to the code buffer”.

| Layer               | Purpose                                                                                                                                                                 | Typical prefix                                                                 |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| **Encoder helpers** | Emit *bytes* of host machine-code. They understand the instruction format (OP-code, ModR/M, displacement, …) but **not** the TCG IR.                                    | `tcg_out8`, `tcg_out16`, `tcg_out32`,<br>`tcg_out_modrm()`, `tcg_out_ldst()` … |
| **Opcode helpers**  | Generate one concrete TCG opcode for this host, using the encoder helpers above. 1-to-1 with the entries in the big switch-statement that converts TCG IR to host code. | `tcg_out_addi()`, `tcg_out_mov()`,<br>`tcg_out_qemu_ld()`, `tcg_out_jxx()` …   |

> The naming convention is mostly
```txt
tcg_out_<instr mnemonic>[extra info]_[r/m/i]
                ↑                     ↑
          target ISA op        operand flavour
```

# Phase in Translation Process

- In translation process, TCG operations emission happens in `tcg_gen_code`, after optimization, and liveness analysis, dead register removal. There is `tcg_out_tb_start` means starting of the `tcg_out_*`.
- For each op, emit the corresponding binary code.
```c
case INDEX_op_insn_start:
    if (num_insns >= 0) {
        size_t off = tcg_current_code_size(s);
        s->gen_insn_end_off[num_insns] = off;
        /* Assert that we do not overflow our stored offset.  */
        assert(s->gen_insn_end_off[num_insns] == off);
    }
    num_insns++;
    for (i = 0; i < start_words; ++i) {
        s->gen_insn_data[num_insns * start_words + i] =
            tcg_get_insn_start_param(op, i);
    }
    break;
```
- Ops can be target-independent (Ops definded in `tcg.c`) or target-dependent (defined in tcg/[arch]/tcg-target.c.inc).
- For each op in TCGContext, alloc and emit different ops.
```c
// if there is a start of op
    case INDEX_op_insn_start:
        if (num_insns >= 0) {
            size_t off = tcg_current_code_size(s);
            s->gen_insn_end_off[num_insns] = off;
            /* Assert that we do not overflow our stored offset.  */
            assert(s->gen_insn_end_off[num_insns] == off);
        }
        num_insns++;
        for (i = 0; i < start_words; ++i) {
            s->gen_insn_data[num_insns * start_words + i] =
                tcg_get_insn_start_param(op, i);
        }
        break;
// ... other ops
      /* Test for (pending) buffer overflow.  The assumption is that any
     one operation beginning below the high water mark cannot overrun
     the buffer completely.  Thus we can test for overflow after
     generating code without having to check during generation.  */
    if (unlikely((void *)s->code_ptr > s->code_gen_highwater)) {
        return -1;
    }
    /* Test for TB overflow, as seen by gen_insn_end_off.  */
    if (unlikely(tcg_current_code_size(s) > UINT16_MAX)) {
        return -2;
    }
```
- Then finalize the process, return the currrent code size.
```c
    tcg_debug_assert(num_insns + 1 == s->gen_tb->icount);
    s->gen_insn_end_off[num_insns] = tcg_current_code_size(s);

    /* Generate TB finalization at the end of block */
#ifdef TCG_TARGET_NEED_LDST_LABELS
    i = tcg_out_ldst_finalize(s);
    if (i < 0) {
        return i;
    }
#endif
#ifdef TCG_TARGET_NEED_POOL_LABELS
    i = tcg_out_pool_finalize(s);
    if (i < 0) {
        return i;
    }
#endif
    if (!tcg_resolve_relocs(s)) {
        return -2;
    }
```

# When to emit ops
## TCGContext
`TCGContext` has ops, which is a list of translation ops that can be translated in to code. 
```c
struct TCGContext {
    uint8_t *pool_cur, *pool_end;
    TCGPool *pool_first, *pool_current, *pool_first_large;
    int nb_labels;
    int nb_globals;  // tcg_global_alloc, tcg_context_init, TCG_MAX_TEMPS maximum
    int nb_temps;    // tcg_temp_alloc, tcg_func_start (equal to nb_globals), TCG_MAX_TEMPS maximum
    int nb_indirects;
    int nb_ops;
    TCGType addr_type;            /* TCG_TYPE_I32 or TCG_TYPE_I64 */
    GHashTable *const_table[TCG_TYPE_COUNT];
    TCGTempSet free_temps[TCG_TYPE_COUNT];
    TCGTemp temps[TCG_MAX_TEMPS]; /* globals first, temps after */
    // .....
    QTAILQ_HEAD(, TCGOp) ops, free_ops;
    QSIMPLEQ_HEAD(, TCGLabel) labels;
    // .....
```
## Functions related to update the TCG ops
These functions are called during `gen_intermediate_code`, in `accel/tcg/trasnslator_loop`, after emit the start of a instruction by using:
- 1. ops->init_disas_context(db, cpu);
- 2. icount_start_insn = gen_tb_start(db, cflags);
- 3. ops->tb_start(db, cpu);
- 4. loop for every instruction: 
  - a. ops->insn_start(db, cpu);
  - b. ops->translate_insn(db, cpu); (`arm_tr_translate_insn`) (emit all ops and insert into ops.)
    - (1)  e.g. `disas_arm_insn` --> `disas_a32`: disamble a 32-bit section to arm instr, fill different fields in a instr structure.
    - (2)  e.g. `trans_MOV_rxi`: generate the wanted front-end ops of one instr.
- 5. ops->tb_stop(db, cpu);
- 6. gen_tb_end(tb, cflags, icount_start_insn, db->num_insns);

## Description of the macro of one generation of op
- `trans_MOV_rxi` in `disas_a32`
```c
        case 0xd:
            /* ....0011 101..... ........ ........ */
            disas_a32_extract_s_rxi_rot(ctx, &u.f_s_rri_rot, insn);
            switch ((insn >> 16) & 0xf) {
            case 0x0:
                /* ....0011 101.0000 ........ ........ */
                /* ../target/arm/tcg/a32.decode:135 */
                if (trans_MOV_rxi(ctx, &u.f_s_rri_rot)) return true;
                break;
            }
            break;
```
- macro in `target/arm/tcg/translate.c` --> `trans_MOV_rxi`
```c
#define DO_ANY2(NAME, OP, L, K)                                         \
    static bool trans_##NAME##_rxri(DisasContext *s, arg_s_rrr_shi *a)  \
    { StoreRegKind k = (K); return op_s_rxr_shi(s, a, OP, L, k); }      \
    static bool trans_##NAME##_rxrr(DisasContext *s, arg_s_rrr_shr *a)  \
    { StoreRegKind k = (K); return op_s_rxr_shr(s, a, OP, L, k); }      \
    static bool trans_##NAME##_rxi(DisasContext *s, arg_s_rri_rot *a)   \
    { StoreRegKind k = (K); return op_s_rxi_rot(s, a, OP, L, k); } // --> can the last one `op_s_rxi_rot`
// here arg_s_rri_rot *a = &u.f_s_rri_rot;
DO_ANY2(MOV, tcg_gen_mov_i32, a->s,
        ({
            StoreRegKind ret = STREG_NORMAL;
            if (a->rd == 15 && a->s) {
                /*
                 * See ALUExceptionReturn:
                 * In User mode, UNPREDICTABLE; we choose UNDEF.
                 * In Hyp mode, UNDEFINED.
                 */
                if (IS_USER(s) || s->current_el == 2) {
                    unallocated_encoding(s);
                    return true;
                }
                /* There is no writeback of nzcv to PSTATE.  */
                a->s = 0;
                ret = STREG_EXC_RET;
            } else if (a->rd == 13) {
                ret = STREG_SP_CHECK;
            }
            ret;
        }))
```
- finally gen op
```c
static bool op_s_rxi_rot(DisasContext *s, arg_s_rri_rot *a,
                         void (*gen)(TCGv_i32, TCGv_i32),
                         int logic_cc, StoreRegKind kind)
{
    TCGv_i32 tmp;
    uint32_t imm;

    imm = ror32(a->imm, a->rot);
    if (logic_cc && a->rot) {
        tcg_gen_movi_i32(cpu_CF, imm >> 31);
    }

    tmp = tcg_temp_new_i32();
    gen(tmp, tcg_constant_i32(imm)); // call tcg_gen_mov_i32

    if (logic_cc) {
        gen_logic_CC(tmp);
    }
    return store_reg_kind(s, a->rd, tmp, kind);
}
```
- in `tcg_gen_mov_i32`: allocate op and add the args into it
```c
void tcg_gen_mov_i32(TCGv_i32 ret, TCGv_i32 arg)
{
    if (ret != arg) {
        tcg_gen_op2_i32(INDEX_op_mov_i32, ret, arg); 
    }
}
// tcg-op.c
void NI tcg_gen_op2(TCGOpcode opc, TCGArg a1, TCGArg a2)
{
    TCGOp *op = tcg_emit_op(opc, 2);
    op->args[0] = a1;
    op->args[1] = a2;
}
// arg_s_rri_rot --> decond-a32.c.inc
typedef struct {
    int s;
    int rn;
    int rd;
    int imm;
    int rot;
} arg_s_rri_rot;
// arg_s_rri_rot a = {
//     .s = 1,         // 'S' suffix is present → updates condition flags
//     .rn = 1,        // source register is r1
//     .rd = 0,        // destination register is r0
//     .imm = 0x80,    // immediate value
//     .rot = 8        // rotate right by 8 bits
// };
```
- store the reg
```c
/* Variant of store_reg which uses branch&exchange logic when storing
   to r15 in ARM architecture v7 and above. The source must be a temporary
   and will be marked as dead. */
static inline void store_reg_bx(DisasContext *s, int reg, TCGv_i32 var)
{
    if (reg == 15 && ENABLE_ARCH_7) {
        gen_bx(s, var);
    } else {
        store_reg(s, reg, var);
    }
}
/* Set a CPU register.  The source must be a temporary and will be
   marked as dead.  */
void store_reg(DisasContext *s, int reg, TCGv_i32 var)
{
    if (reg == 15) {
        /* In Thumb mode, we must ignore bit 0.
         * In ARM mode, for ARMv4 and ARMv5, it is UNPREDICTABLE if bits [1:0]
         * are not 0b00, but for ARMv6 and above, we must ignore bits [1:0].
         * We choose to ignore [1:0] in ARM mode for all architecture versions.
         */
        tcg_gen_andi_i32(var, var, s->thumb ? ~1 : ~3);
        s->base.is_jmp = DISAS_JUMP;
        s->pc_save = -1;
    } else if (reg == 13 && arm_dc_feature(s, ARM_FEATURE_M)) {
        /* For M-profile SP bits [1:0] are always zero */
        tcg_gen_andi_i32(var, var, ~3);
    }
    tcg_gen_mov_i32(cpu_R[reg], var);
}
```




### Trace Ops emission process.
```c
// accel/tcg/plugin-gen.c --> called by rm_ops --> sub areas
static TCGOp *rm_ops_range(TCGOp *begin, TCGOp *end)
{
    TCGOp *ret = QTAILQ_NEXT(end, link);

    QTAILQ_REMOVE_SEVERAL(&tcg_ctx->ops, begin, end, link);
    return ret;
}
// tcg.c --> tcg_func_start, init everything for tcx --> called by setjmp_gen_code, in main translation process, before gen_intermediate_code.
void tcg_func_start(TCGContext *s){
    tcg_pool_reset(s);
    s->nb_temps = s->nb_globals;

    /* No temps have been previously allocated for size or locality.  */
    memset(s->free_temps, 0, sizeof(s->free_temps));
    // ...

    QTAILQ_INIT(&s->ops);
    QTAILQ_INIT(&s->free_ops);
}

// tcg.c --> tcg_op_remove --> used in optimization process/livenesss pass process
void tcg_op_remove(TCGContext *s, TCGOp *op)
{
    switch (op->opc) {
    case INDEX_op_br:
        remove_label_use(op, 0);
        break;
    case INDEX_op_brcond_i32:
    case INDEX_op_brcond_i64:
        remove_label_use(op, 3);
        break;
    case INDEX_op_brcond2_i32:
        remove_label_use(op, 5);
        break;
    default:
        break;
    }

    QTAILQ_REMOVE(&s->ops, op, link);
    QTAILQ_INSERT_TAIL(&s->free_ops, op, link);
    s->nb_ops--;
}
// tcg.c --> tcg_op_insert_after --> used in optimization/liveness pass process
TCGOp *tcg_op_insert_after(TCGContext *s, TCGOp *old_op,
                           TCGOpcode opc, unsigned nargs)
{
    TCGOp *new_op = tcg_op_alloc(opc, nargs);
    QTAILQ_INSERT_AFTER(&s->ops, old_op, new_op, link);
    return new_op;
}
// tcg.c --> tcg_emit_op
TCGOp *tcg_emit_op(TCGOpcode opc, unsigned nargs)
{
    TCGOp *op = tcg_op_alloc(opc, nargs);

    if (tcg_ctx->emit_before_op) {
        QTAILQ_INSERT_BEFORE(tcg_ctx->emit_before_op, op, link);
    } else {
        QTAILQ_INSERT_TAIL(&tcg_ctx->ops, op, link);
    }
    return op;
}
```
### Different functions using `tcg_emit_op` 
- `tcg_gen_insn_start` in `tcg-op.h`
```c
#ifndef TARGET_INSN_START_EXTRA_WORDS
...
#elif TARGET_INSN_START_EXTRA_WORDS == 2
static inline void tcg_gen_insn_start(target_ulong pc, target_ulong a1,
                                      target_ulong a2)
{
    TCGOp *op = tcg_emit_op(INDEX_op_insn_start, 3 * 64 / TCG_TARGET_REG_BITS);
    tcg_set_insn_start_param(op, 0, pc);
    tcg_set_insn_start_param(op, 1, a1);
    tcg_set_insn_start_param(op, 2, a2);
}
```
- `tcg_gen_op[k]` k from 1 to 6
```c
// tcg/tcg-internal.h
void tcg_gen_op1(TCGOpcode, TCGArg);
void tcg_gen_op2(TCGOpcode, TCGArg, TCGArg);
void tcg_gen_op3(TCGOpcode, TCGArg, TCGArg, TCGArg);
void tcg_gen_op4(TCGOpcode, TCGArg, TCGArg, TCGArg, TCGArg);
void tcg_gen_op5(TCGOpcode, TCGArg, TCGArg, TCGArg, TCGArg, TCGArg);
void tcg_gen_op6(TCGOpcode, TCGArg, TCGArg, TCGArg, TCGArg, TCGArg, TCGArg);

void vec_gen_2(TCGOpcode, TCGType, unsigned, TCGArg, TCGArg);
void vec_gen_3(TCGOpcode, TCGType, unsigned, TCGArg, TCGArg, TCGArg);
void vec_gen_4(TCGOpcode, TCGType, unsigned, TCGArg, TCGArg, TCGArg, TCGArg);

// tcg/tc-op.c
void NI tcg_gen_op5(TCGOpcode opc, TCGArg a1, TCGArg a2, TCGArg a3,
                     TCGArg a4, TCGArg a5)
{
    TCGOp *op = tcg_emit_op(opc, 5);
    op->args[0] = a1;
    op->args[1] = a2;
    op->args[2] = a3;
    op->args[3] = a4;
    op->args[4] = a5;
}

void NI tcg_gen_op6(TCGOpcode opc, TCGArg a1, TCGArg a2, TCGArg a3,
                     TCGArg a4, TCGArg a5, TCGArg a6)
{
    TCGOp *op = tcg_emit_op(opc, 6);
    op->args[0] = a1;
    op->args[1] = a2;
    op->args[2] = a3;
    op->args[3] = a4;
    op->args[4] = a5;
    op->args[5] = a6;
}
```
- Generatic Ops: Other ops that are derived from `tcg_gen_op[k]` --> `tcg/tcg-op.c`
```c
tcg_gen_op[k]_i[32/64]
tcg_gen_ldst_op_i64
tcg_gen_op5ii_i32
/* Generic ops.  */

void gen_set_label(TCGLabel *l)
{
    l->present = 1;
    tcg_gen_op1(INDEX_op_set_label, label_arg(l));
}
// ...
void tcg_gen_mov_i32(TCGv_i32 ret, TCGv_i32 arg)
{
    if (ret != arg) {
        tcg_gen_op2_i32(INDEX_op_mov_i32, ret, arg);
    }
}

void tcg_gen_movi_i32(TCGv_i32 ret, int32_t arg)
{
    tcg_gen_mov_i32(ret, tcg_constant_i32(arg));
}

```
- translation for general ops `target/arm/tcg/translate.c`
```c

#define DO_ANY3(NAME, OP, L, K)                                         \
    static bool trans_##NAME##_rrri(DisasContext *s, arg_s_rrr_shi *a)  \
    { StoreRegKind k = (K); return op_s_rrr_shi(s, a, OP, L, k); }      \
    static bool trans_##NAME##_rrrr(DisasContext *s, arg_s_rrr_shr *a)  \
    { StoreRegKind k = (K); return op_s_rrr_shr(s, a, OP, L, k); }      \
    static bool trans_##NAME##_rri(DisasContext *s, arg_s_rri_rot *a)   \
    { StoreRegKind k = (K); return op_s_rri_rot(s, a, OP, L, k); }

#define DO_ANY2(NAME, OP, L, K)                                         \
    static bool trans_##NAME##_rxri(DisasContext *s, arg_s_rrr_shi *a)  \
    { StoreRegKind k = (K); return op_s_rxr_shi(s, a, OP, L, k); }      \
    static bool trans_##NAME##_rxrr(DisasContext *s, arg_s_rrr_shr *a)  \
    { StoreRegKind k = (K); return op_s_rxr_shr(s, a, OP, L, k); }      \
    static bool trans_##NAME##_rxi(DisasContext *s, arg_s_rri_rot *a)   \
    { StoreRegKind k = (K); return op_s_rxi_rot(s, a, OP, L, k); }

#define DO_CMP2(NAME, OP, L)                                            \
    static bool trans_##NAME##_xrri(DisasContext *s, arg_s_rrr_shi *a)  \
    { return op_s_rrr_shi(s, a, OP, L, STREG_NONE); }                   \
    static bool trans_##NAME##_xrrr(DisasContext *s, arg_s_rrr_shr *a)  \
    { return op_s_rrr_shr(s, a, OP, L, STREG_NONE); }                   \
    static bool trans_##NAME##_xri(DisasContext *s, arg_s_rri_rot *a)   \
    { return op_s_rri_rot(s, a, OP, L, STREG_NONE); }


DO_ANY3(AND, tcg_gen_and_i32, a->s, STREG_NORMAL)
DO_ANY3(EOR, tcg_gen_xor_i32, a->s, STREG_NORMAL)
DO_ANY3(ORR, tcg_gen_or_i32, a->s, STREG_NORMAL)
```

# `tcg_out_op` in `tcg_gen_code`
---
Ok, the previous part talked about when and how the ops are inserted or removed. the `tcg_out_...` process happens after `gen_intermediate_code`, in `tcg_gen_code`, calledy by `tb_gen_code`.
Generated codes should in the same arch with host machine --> tcg-target.c.inc
```c
/*
 * Isolate the portion of code gen which can setjmp/longjmp.
 * Return the size of the generated code, or negative on error.
 */
static int setjmp_gen_code(CPUArchState *env, TranslationBlock *tb,
                           vaddr pc, void *host_pc,
                           int *max_insns, int64_t *ti)
{
    int ret = sigsetjmp(tcg_ctx->jmp_trans, 0);
    if (unlikely(ret != 0)) {
        return ret;
    }
    // RULE NOTE: init the tcg context before disambling/storing arm instruction 
    tcg_func_start(tcg_ctx); // s->nb_temps = s->nb_globals;

    tcg_ctx->cpu = env_cpu(env);
    gen_intermediate_code(env_cpu(env), tb, max_insns, pc, host_pc);
    assert(tb->size != 0);
    tcg_ctx->cpu = NULL;
    *max_insns = tb->icount;

    return tcg_gen_code(tcg_ctx, tb, pc);
}
```

## `TCG_Out` whole process

```c
[guest instruction: ADD r0, r1, #5]
            ↓
[TCG IR emitted via tcg_gen_*]
            ↓
[s->ops list contains movi_i32, add_i32, mov_i32]
            ↓
[tcg_gen_code()]
    → allocate registers
    → emit host instructions
    → record instruction offsets
            ↓
[host binary code buffer complete]
```

### Overall Purpose

This function walks through the list of TCG operations (`TCGOp`s) stored in `s->ops` and, for each:

* **Performs register allocation**
    
* **Emits machine code**
    
* Tracks instruction metadata (such as code offsets)
    
* Performs overflow checks
    
It is the **core of instruction lowering and code emission** in the QEMU JIT pipeline.


## TCG Initialization the whole tcg context
- 1. In `linux-user/main.c`
```c
/* init tcg before creating CPUs */
{
    AccelState *accel = current_accel();
    AccelClass *ac = ACCEL_GET_CLASS(accel);

    accel_init_interfaces(ac);
    object_property_set_bool(OBJECT(accel), "one-insn-per-tb",
                             opt_one_insn_per_tb, &error_abort);
    ac->init_machine(NULL);
}
```
- 2. in `accel/tcg/tcg-all.c` defines the init function: `ac->init_machine = tcg_init_machine`
- 3. `tcg_init`
```c
void tcg_init(size_t tb_size, int splitwx, unsigned max_cpus)
{
    tcg_context_init(max_cpus); // all args, nb_globals, and spaces, tcg context, tcg_target_init
    tcg_region_init(tb_size, splitwx, max_cpus); // Region partitioning works by splitting code_gen_buffer into separate regions, and then assigning regions to TCG threads so that the threads can translate code in parallel without synchronization.
}
``` 

## How `tcg_out_[op]` works
- `tcg_reg_alloc_op` is used in `tcg_gen_code`.
- `tcg_out_op` in `tcg-target.c.inc`, called by `tcg_reg_alloc_op` (in `tcg.c`)
```c
static inline void tcg_out_op(TCGContext *s, TCGOpcode opc,
                              const TCGArg args[TCG_MAX_OP_ARGS],
                              const int const_args[TCG_MAX_OP_ARGS])
{
    TCGArg a0, a1, a2;
    int c, const_a2, vexop, rexw = 0;

#if TCG_TARGET_REG_BITS == 64
# define OP_32_64(x) \
        case glue(glue(INDEX_op_, x), _i64): \
            rexw = P_REXW; /* FALLTHRU */    \
        case glue(glue(INDEX_op_, x), _i32)
#else
# define OP_32_64(x) \
        case glue(glue(INDEX_op_, x), _i32)
#endif

    /* Hoist the loads of the most common arguments.  */
    a0 = args[0];
    a1 = args[1];
    a2 = args[2];
    const_a2 = const_args[2];

    switch (opc) {
    case INDEX_op_goto_ptr:
        /* jmp to the given host address (could be epilogue) */
        tcg_out_modrm(s, OPC_GRP5, EXT5_JMPN_Ev, a0);
        break;
    case INDEX_op_br:
        tcg_out_jxx(s, JCC_JMP, arg_label(a0), 0);
        break;
    OP_32_64(ld8u):
        /* Note that we can ignore REXW for the zero-extend to 64-bit.  */
        tcg_out_modrm_offset(s, OPC_MOVZBL, a0, a1, a2);
        break;
    OP_32_64(ld8s):
        tcg_out_modrm_offset(s, OPC_MOVSBL + rexw, a0, a1, a2);
        break;
    OP_32_64(ld16u):
        /* Note that we can ignore REXW for the zero-extend to 64-bit.  */
        tcg_out_modrm_offset(s, OPC_MOVZWL, a0, a1, a2);
        break;
    OP_32_64(ld16s):
        tcg_out_modrm_offset(s, OPC_MOVSWL + rexw, a0, a1, a2);
        break;
#if TCG_TARGET_REG_BITS == 64
    case INDEX_op_ld32u_i64:
#endif
    case INDEX_op_ld_i32:
        tcg_out_ld(s, TCG_TYPE_I32, a0, a1, a2);
        break;

    OP_32_64(st8):
        if (const_args[0]) {
            tcg_out_modrm_offset(s, OPC_MOVB_EvIz, 0, a1, a2);
            tcg_out8(s, a0);
        } else {
            tcg_out_modrm_offset(s, OPC_MOVB_EvGv | P_REXB_R, a0, a1, a2);
        }
        break;
    OP_32_64(st16):
        if (const_args[0]) {
            tcg_out_modrm_offset(s, OPC_MOVL_EvIz | P_DATA16, 0, a1, a2);
            tcg_out16(s, a0);
        } else {
            tcg_out_modrm_offset(s, OPC_MOVL_EvGv | P_DATA16, a0, a1, a2);
        }
        break;
#if TCG_TARGET_REG_BITS == 64
    case INDEX_op_st32_i64:
#endif
    case INDEX_op_st_i32:
        if (const_args[0]) {
            tcg_out_modrm_offset(s, OPC_MOVL_EvIz, 0, a1, a2);
            tcg_out32(s, a0);
        } else {
            tcg_out_st(s, TCG_TYPE_I32, a0, a1, a2);
        }
        break;

    OP_32_64(add):
        /* For 3-operand addition, use LEA.  */
        if (a0 != a1) {
            TCGArg c3 = 0;
            if (const_a2) {
                c3 = a2, a2 = -1;
            } else if (a0 == a2) {
                /* Watch out for dest = src + dest, since we've removed
                   the matching constraint on the add.  */
                tgen_arithr(s, ARITH_ADD + rexw, a0, a1);
                break;
            }

            tcg_out_modrm_sib_offset(s, OPC_LEA + rexw, a0, a1, a2, 0, c3);
            break;
        }
        c = ARITH_ADD;
        goto gen_arith;
    OP_32_64(sub):
        c = ARITH_SUB;
        goto gen_arith;
    OP_32_64(and):
        c = ARITH_AND;
        goto gen_arith;
    OP_32_64(or):
        c = ARITH_OR;
        goto gen_arith;
    OP_32_64(xor):
        c = ARITH_XOR;
        goto gen_arith;
    gen_arith:
        if (const_a2) {
            tgen_arithi(s, c + rexw, a0, a2, 0);
        } else {
            tgen_arithr(s, c + rexw, a0, a2);
        }
        break;

    OP_32_64(andc):
        if (const_a2) {
            tcg_out_mov(s, rexw ? TCG_TYPE_I64 : TCG_TYPE_I32, a0, a1);
            tgen_arithi(s, ARITH_AND + rexw, a0, ~a2, 0);
        } else {
            tcg_out_vex_modrm(s, OPC_ANDN + rexw, a0, a2, a1);
        }
        break;

    OP_32_64(mul):
        if (const_a2) {
            int32_t val;
            val = a2;
            if (val == (int8_t)val) {
                tcg_out_modrm(s, OPC_IMUL_GvEvIb + rexw, a0, a0);
                tcg_out8(s, val);
            } else {
                tcg_out_modrm(s, OPC_IMUL_GvEvIz + rexw, a0, a0);
                tcg_out32(s, val);
            }
        } else {
            tcg_out_modrm(s, OPC_IMUL_GvEv + rexw, a0, a2);
        }
        break;

    OP_32_64(div2):
        tcg_out_modrm(s, OPC_GRP3_Ev + rexw, EXT3_IDIV, args[4]);
        break;
    OP_32_64(divu2):
        tcg_out_modrm(s, OPC_GRP3_Ev + rexw, EXT3_DIV, args[4]);
        break;

    OP_32_64(shl):
        /* For small constant 3-operand shift, use LEA.  */
        if (const_a2 && a0 != a1 && (a2 - 1) < 3) {
            if (a2 - 1 == 0) {
                /* shl $1,a1,a0 -> lea (a1,a1),a0 */
                tcg_out_modrm_sib_offset(s, OPC_LEA + rexw, a0, a1, a1, 0, 0);
            } else {
                /* shl $n,a1,a0 -> lea 0(,a1,n),a0 */
                tcg_out_modrm_sib_offset(s, OPC_LEA + rexw, a0, -1, a1, a2, 0);
            }
            break;
        }
        c = SHIFT_SHL;
        vexop = OPC_SHLX;
        goto gen_shift_maybe_vex;
    OP_32_64(shr):
        c = SHIFT_SHR;
        vexop = OPC_SHRX;
        goto gen_shift_maybe_vex;
    OP_32_64(sar):
        c = SHIFT_SAR;
        vexop = OPC_SARX;
        goto gen_shift_maybe_vex;
    OP_32_64(rotl):
        c = SHIFT_ROL;
        goto gen_shift;
    OP_32_64(rotr):
        c = SHIFT_ROR;
        goto gen_shift;
    gen_shift_maybe_vex:
        if (have_bmi2) {
            if (!const_a2) {
                tcg_out_vex_modrm(s, vexop + rexw, a0, a2, a1);
                break;
            }
            tcg_out_mov(s, rexw ? TCG_TYPE_I64 : TCG_TYPE_I32, a0, a1);
        }
        /* FALLTHRU */
    gen_shift:
        if (const_a2) {
            tcg_out_shifti(s, c + rexw, a0, a2);
        } else {
            tcg_out_modrm(s, OPC_SHIFT_cl + rexw, c, a0);
        }
        break;

    OP_32_64(ctz):
        tcg_out_ctz(s, rexw, args[0], args[1], args[2], const_args[2]);
        break;
    OP_32_64(clz):
        tcg_out_clz(s, rexw, args[0], args[1], args[2], const_args[2]);
        break;
    OP_32_64(ctpop):
        tcg_out_modrm(s, OPC_POPCNT + rexw, a0, a1);
        break;

    OP_32_64(brcond):
        tcg_out_brcond(s, rexw, a2, a0, a1, const_args[1],
                       arg_label(args[3]), 0);
        break;
    // ...
}
```
- `tcg_out_brcond`
```c
static void tcg_out_brcond(TCGContext *s, int rexw, TCGCond cond,
                           TCGArg arg1, TCGArg arg2, int const_arg2,
                           TCGLabel *label, bool small)
{
    int jcc = tcg_out_cmp(s, cond, arg1, arg2, const_arg2, rexw);
    tcg_out_jxx(s, jcc, label, small);
}
```
- `tcg_out_jxx`
```c
/* Set SMALL to force a short forward branch.  */
static void tcg_out_jxx(TCGContext *s, int opc, TCGLabel *l, bool small)
{
    int32_t val, val1;

    if (l->has_value) {
        val = tcg_pcrel_diff(s, l->u.value_ptr);
        val1 = val - 2;
        if ((int8_t)val1 == val1) {
            if (opc == -1) {
                tcg_out8(s, OPC_JMP_short);
            } else {
                tcg_out8(s, OPC_JCC_short + opc);
            }
            tcg_out8(s, val1);
        } else {
            tcg_debug_assert(!small);
            if (opc == -1) {
                tcg_out8(s, OPC_JMP_long);
                tcg_out32(s, val - 5);
            } else {
                tcg_out_opc(s, OPC_JCC_long + opc, 0, 0, 0);
                tcg_out32(s, val - 6);
            }
        }
    } else if (small) {
        if (opc == -1) {
            tcg_out8(s, OPC_JMP_short);
        } else {
            tcg_out8(s, OPC_JCC_short + opc);
        }
        tcg_out_reloc(s, s->code_ptr, R_386_PC8, l, -1);
        s->code_ptr += 1;
    } else {
        if (opc == -1) {
            tcg_out8(s, OPC_JMP_long);
        } else {
            tcg_out_opc(s, OPC_JCC_long + opc, 0, 0, 0);
        }
        tcg_out_reloc(s, s->code_ptr, R_386_PC32, l, -4);
        s->code_ptr += 4;
    }
}
```
- `tcg_out32`: put the generated code into the code buffer and move the code_ptr in tcgcontext
```c
static __attribute__((unused)) inline void tcg_out32(TCGContext *s, uint32_t v)
{
    if (TCG_TARGET_INSN_UNIT_SIZE == 4) {
        *s->code_ptr++ = v;
    } else {
        tcg_insn_unit *p = s->code_ptr;
        memcpy(p, &v, sizeof(v));
        s->code_ptr = p + (4 / TCG_TARGET_INSN_UNIT_SIZE);
    }
}
```

- `tcg_gen_code` is the entry point of the entire backend translation, responsible for the synchronization logic between registers and memory areas, and calls related functions according to different instruction types to translate the intermediate code into host CPU instructions. By default, the function `tcg_gen_code` will be called `tcg_reg_alloc_op`, which will generate synchronization logic described by host CPU instructions and store it in TB, and finally call the function provided by developers of different architectures `tcg_out_op` to complete the translation of specific instructions. For addthe instruction, the function will eventually be called `tcg_out32()`, which is responsible for writing a 32-bit unsigned integer `v` to `s->code_ptr` the memory location corresponding to the pointer and updating the value of the pointer according to the instruction unit size of the target platform. [Resources](https://blog.jjl9807.com/archives/6/)

![`tcg_gen_code` process](/commons/images/qemu/tcg_backend_op.png)



==[Content Link for TCG Frontend-Ops](https://wiki.qemu.org/Documentation/TCG/frontend-ops)== 

# Frontend Ops

These are the supported operations as implemented by the TCG frontend for the target cpu (what QEMU executes; not where QEMU executes). This information is useful for people who want to port QEMU to emulate a new processor.

All of the frontend helpers in `tcg/tcg-op.h`. This page covers all them; it does not cover the mechanism for defining port-specific helpers.

# Conventions
+ When the term register is used unqualified, it most likely is referring to a TCG register.
+ The frontend helpers for generating TCG opcodes typically take the form: `tcg_gen_<op>[i]_<reg_size>`.
  + The `<op>` is the TCG operation that will be generated for its arguments.
  + The `[i]` suffix is used to indicate the TCG operation takes an immediate rather than a normal register.
  + The `<reg_size>` refers to the size of the TCG registers in use. The vast majority of the time, this will match the native size of the emulated target, so rather than force people to type i32 or i64 all the time, the shorthand tl is made available for all helpers. e.g. to perform a 32bit register move for a 32bit target, simply use tcg_gen_mov_tl rather than tcg_gen_mov_i32.
+ We won't cover the immediate variants of functions as their usage should be fairly obvious once you grasp the register version.
+ To keep things simple, we'll cover the tl variants here. If you need to delve into explicit types, then you should probably know what you're doing anyways, so you're beyond the current scope of this wiki page.
+ Similarly, rather than using `TCGv_i32` and `TCGv_i64` everywhere, people should stick to `TCGv` and `TCGv_ptr`.
+ The `tcg_gen_xxx` arguments typically place the return value first followed by the operands. So to add two registers (a = b + c), the code would be `tcg_gen_xxx(a, b, c);`.


# Non-Ops
Some non-ops helpers are also provided.

## Registers
Most of the core registers of the emulated processor should have a TCG register equivalent. For processor status flags (or similar oddities), ports will employ tricks to speed things up. This is obviously left up to the discretion of the processor port maintainer.

```c
TCGv reg = tcg_global_mem_new(TCG_AREG0, offsetof(CPUState, reg), "reg");	Declare a named TCG register
```
## Temporaries
Often times, target insns cannot be broken down into one or two simple RISC insns which means temporary registers might be necessary to store intermediate results. Rather than each frontend maintaining its own static set of temporary scratch registers, helpers are provided to manage these on the fly.

```c
TCGv tmp = tcg_temp_new();	Create a new temporary register
TCGv tmpl = tcg_temp_local_new();	Create a local temporary register. Simple temporary register cannot carry its value across jump/brcond, only local temporary can.
tcg_temp_free(tmp);	Free a temporary register
```
You should not try to "hold on" to temporary registers beyond the target insn you're currently generating. If you need long lived registers, consider allocating a proper one.

## Labels
Labels are used to generate conditional code. Here we'll just cover their management; later we'll get into using them in conjunction with opcodes.

```c
int l = gen_new_label();	Create a new label.
gen_set_label(l);	Label the current location.
```
# Ops
These frontend ops expand into `backend` ops. Consulting that page might be useful for deeper insight into the front/back end relationship.

## Math
Math operation on a single register:

```c
tcg_gen_mov_tl(ret, arg1);	Assign one register to another	ret = arg1
tcg_gen_neg_tl(ret, arg1);	Negate the sign of a register	ret = - arg1
```

Math operations on two registers:

```c
tcg_gen_sub_tl(ret, arg1, arg2);	Subtract two registers	ret = arg1 - arg2
tcg_gen_mul_tl(ret, arg1, arg2);	Multiply two signed registers and return the result	ret = arg1 * arg2
tcg_gen_mulu_tl(ret, arg1, arg2);	Multiply two unsigned registers and return the result	ret = arg1 * arg2
```

## Others 
- Bit Operations
  - Logic
  - Shift
  - Rotation
- Byte
- Load/Store: 
  - For moving data between registers and arbitrary host memory. Typically used for funky CPU state that is not represented by dedicated registers already and thus infrequently used. 
  ```c
  tcg_gen_ld64_tl(reg, cpu_env, offsetof(CPUState, reg));	Load a 64bit quantity from host memory
  tcg_gen_ld_tl(reg, cpu_env, offsetof(CPUState, reg));	Alias to target native sized load
  tcg_gen_st32_tl(reg, cpu_env, offsetof(CPUState, reg));	Store a 32bit quantity to host memory
  tcg_gen_st_tl(reg, cpu_env, offsetof(CPUState, reg));	Alias to target native sized store
  ```
  - for moving data between registers and arbitrary target memory. The address to load/store via is always the second argument while the first argument is always the value to be loaded/stored.
- Code Flow
```c
tcg_gen_brcond_tl(TCG_COND_XXX, arg1, arg2, label);	Test two operands and conditionally branch to a label	if (arg1 <condition> arg2) goto label
tcg_gen_goto_tb(num);	Goto translation block (TB chaining)	//Every TB can goto_tb to max two other different destinations. There are two jump slots. tcg_gen_goto_tb takes a jump slot index as an arg, 0 or 1. These jumps will only take place if the TB's get chained, you need to tcg_gen_exit_tb with (tb | index) for that to ever happen. tcg_gen_goto_tb may be issued at most once with each slot index per TB.

tcg_gen_exit_tb(num);	Exit translation block. //Num may be 0 or TB address ORed with the index of the taken jump slot. If you tcg_gen_exit_tb(0), chaining will not happen and a new TB will be looked up based on the CPU state.

tcg_gen_setcond_tl(TCG_COND_XXX, ret, arg1, arg2);	Compare two operands	ret = arg1 <condition> arg2
```


# TCG Backend-ops

TCG backend ops are ops used in `tcg_out_op`.

These are the supported operations as implemented by the TCG backend for the host cpu (where QEMU executes; not what QEMU executes). This information is useful for people who want to port QEMU run on a new processor.

## Math
```
ADD	Add two operands	ret = arg1 + arg2
DIV	Divide two signed operands and return the quotient	ret = arg1 / arg2
DIVU	Divide two unsigned operands and return the quotient	ret = arg1 / arg2
MOV	Assign one operand to another	ret = arg1
MUL	Multiply two signed operands	ret = arg1 * arg2
MULU	Multiply two unsigned operands	ret = arg1 * arg2
NEG	Negate the sign of an operand	ret = -arg1
REM	Divide two signed operands and return the remainder	ret = arg1 % arg2
REMU	Divide two unsigned operands and return the remainder	ret = arg1 % arg2
SUB	Subtract two operands	ret = arg1 - arg2

```
## Bit
```
AND	Logical AND two operands	ret = arg1 & arg2
ANDC	Logical AND one operand with the complement of another	ret = arg1 & ~arg2
EQV	Compute logical equivalent of two operands	ret = !(arg1 ^ arg2)
NAND	Logical NAND two operands	ret = arg1 ↑ arg2
NOR	Logical NOR two operands	ret = arg1 ↓ arg2
NOT	Logical NOT an operand	ret = !arg1
OR	Logical OR two operands	ret = arg1 | arg2
ORC	Logical OR one operand with the complement of another	ret = arg1 | ~arg2
ROTL	Rotate left one operand by magnitude of another	ret = arg1 rotl arg2
ROTR	Rotate right one operand by magnitude of another	ret = arg1 rotr arg2
SAR	Arithmetic shift right one operand by magnitude of another	ret = arg1 >> arg2 /* Sign bit used to fill vacant bits */
SHL	Logical shift left one operand by magnitude of another	ret = arg1 << arg2
SHR	Logical shift right one operand by magnitude of another	ret = arg1 >> arg2
XOR	Logical XOR two operands	ret = arg1 ^ arg2
```
## Byte
```
BSWAP16	Byte swap a 16bit quantity	ret = ((arg1 & 0xff00) >> 8) | ((arg1 & 0xff) << 8)
BSWAP32	Byte swap a 32bit quantity	ret = ...see bswap16 and extend to 32bits...
BSWAP64	Byte swap a 64bit quantity	ret = ...see bswap32 and extend to 64bits...
EXT8S	Sign extend an 8bit operand	ret = (int8_t)arg1
EXT8U	Zero extend an 8bit operand	ret = (uint8_t)arg1
EXT16S	Sign extend an 16bit operand	ret = (int16_t)arg1
EXT16U	Zero extend an 16bit operand	ret = (uint16_t)arg1
EXT32S	Sign extend an 32bit operand	ret = (int32_t)arg1
EXT32U	Zero extend an 32bit operand	ret = (uint32_t)arg1
```
## Load/Store
These are for moving data between registers and arbitrary host memory. Typically used for funky CPU state that is not represented by dedicated registers already and thus infrequently used. These are not for accessing the target's memory space; see the QEMU_XX helpers below for that.
```
LD8S	Load an 8bit quantity from host memory and sign extend
LD8U	Load an 8bit quantity from host memory and zero extend
LD16S	Load a 16bit quantity from host memory and sign extend
LD16U	Load a 16bit quantity from host memory and zero extend
LD32S	Load a 32bit quantity from host memory and sign extend
LD32U	Load a 32bit quantity from host memory and zero extend
LD64	Load a 64bit quantity from host memory
LD	Alias to target native sized load
ST8	Store a 8bit quantity to host memory
ST16	Store a 16bit quantity to host memory
ST32	Store a 32bit quantity to host memory
ST	Alias to target native sized store
```
These are for moving data between registers and arbitrary target memory. The address to load/store via is always the second argument while the first argument is always the value to be loaded/stored. The third argument (memory index) only makes sense for system targets; user targets will simply specify 0 all the time.
```
QEMU_LD8S	Load an 8bit quantity from target memory and sign extend	ret = *(int8_t *)arg1
QEMU_LD8U	Load an 8bit quantity from target memory and zero extend	ret = *(uint8_t *)arg1
QEMU_LD16S	Load a 16bit quantity from target memory and sign extend	ret = *(int8_t *)arg1
QEMU_LD16U	Load a 16bit quantity from target memory and zero extend	ret = *(uint8_t *)arg1
QEMU_LD32S	Load a 32bit quantity from target memory and sign extend	ret = *(int8_t *)arg1
QEMU_LD32U	Load a 32bit quantity from target memory and zero extend	ret = *(uint8_t *)arg1
QEMU_LD64	Load a 64bit quantity from target memory	ret = *(uint64_t *)arg1
QEMU_ST8	Store an 8bit quantity to target memory	*(uint8_t *)addr = arg
QEMU_ST16	Store a 16bit quantity to target memory	*(uint16_t *)addr = arg
QEMU_ST32	Store a 32bit quantity to target memory	*(uint32_t *)addr = arg
QEMU_ST64	Store a 64bit quantity to target memory	*(uint64_t *)addr = arg
```
## Code Flow
```
BR	Branch somewhere?	
BRCOND	Test two operands and conditionally branch to a label	if (arg1 <condition> arg2) goto label
CALL	Call a helper function	
GOTO_TB	Goto translation block	
EXIT_TB	Exit translation block	
SETCOND	Compare two operands	ret = arg1 <condition> arg2
SET_LABEL	Mark the current location with a label	label:
```

## Misc
```
DISCARD	Discard a register
NOP	Generate a No Operation insn
```
---
title: Target-dependent Data Usage
date: 2025-05-07
categories: [Qemu]
tags: [qemu]     # TAG names should always be lowercase
published: false
---

> Problem: I want to use target-related structures in the tcg process.

- What is 'target'?? [QEMU document](https://www.qemu.org/docs/master/glossary.html)
The term “target” can be ambiguous. In most places in QEMU it is used as a synonym for Guest. For example the code for emulating Arm CPUs is in target/arm/ . However in the TCG subsystem “target” refers to the architecture which QEMU is running on, i.e. the Host.
- So, Here, the target-related in the tcg translation process, it should be host "x86"

# Structure of inclusion

1. `target/arm/translate.c` --> `target_ulong` --> defined by `TARGET_LONG_BITS`.
2. `target/arm/cpu-params.h` defines `TARGET_LONG_BITS`.
3. `target_ulong` --> defined in `target_long.h`.
4. `target_long.h` is needed in `include/exec/cpu-defs.h` --> included in `cpu.h` --> `target/arm/translate.h` by `target/arm/translate.c`.

# Related structure

## pc --> vaddr
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

    tcg_func_start(tcg_ctx); // s->nb_temps = s->nb_globals;

    tcg_ctx->cpu = env_cpu(env);
    gen_intermediate_code(env_cpu(env), tb, max_insns, pc, host_pc);
    assert(tb->size != 0);
    tcg_ctx->cpu = NULL;
    *max_insns = tb->icount;

    return tcg_gen_code(tcg_ctx, tb, pc);
}
```
defined in `include/exec/vaddr.h`
```c
/**
 * vaddr:
 * Type wide enough to contain any #target_ulong virtual address.
 */
typedef uint64_t vaddr;
```
in `target/arm/cpu.h` when use `tb->pc`
```c
#ifdef CONFIG_TCG
void arm_cpu_synchronize_from_tb(CPUState *cs,
                                 const TranslationBlock *tb)
{
    /* The program counter is always up to date with CF_PCREL. */
    if (!(tb_cflags(tb) & CF_PCREL)) {
        CPUARMState *env = cpu_env(cs);
        /*
         * It's OK to look at env for the current mode here, because it's
         * never possible for an AArch64 TB to chain to an AArch32 TB.
         */
        if (is_a64(env)) {
            env->pc = tb->pc;
        } else {
            env->regs[15] = tb->pc;
        }
    }
}
void arm_restore_state_to_opc(CPUState *cs,
                              const TranslationBlock *tb,
                              const uint64_t *data)
{
    CPUARMState *env = cpu_env(cs);

    if (is_a64(env)) {
        if (tb_cflags(tb) & CF_PCREL) {
            env->pc = (env->pc & TARGET_PAGE_MASK) | data[0];
        } else {
            env->pc = data[0];
        }
        env->condexec_bits = 0;
        env->exception.syndrome = data[2] << ARM_INSN_START_WORD2_SHIFT;
    } else {
        if (tb_cflags(tb) & CF_PCREL) {
            env->regs[15] = (env->regs[15] & TARGET_PAGE_MASK) | data[0];
        } else {
            env->regs[15] = data[0];
        }
        env->condexec_bits = data[1];
        env->exception.syndrome = data[2] << ARM_INSN_START_WORD2_SHIFT;
    }
}
#endif /* CONFIG_TCG */
```
in `target/arm/cpu.c`
```c
#ifdef CONFIG_TCG
static const TCGCPUOps arm_tcg_ops = {
    .initialize = arm_translate_init,
    .synchronize_from_tb = arm_cpu_synchronize_from_tb,
    .debug_excp_handler = arm_debug_excp_handler,
    .restore_state_to_opc = arm_restore_state_to_opc,

#ifdef CONFIG_USER_ONLY
    .record_sigsegv = arm_cpu_record_sigsegv,
    .record_sigbus = arm_cpu_record_sigbus,
#else
    .tlb_fill = arm_cpu_tlb_fill,
    .cpu_exec_interrupt = arm_cpu_exec_interrupt,
    .do_interrupt = arm_cpu_do_interrupt,
    .do_transaction_failed = arm_cpu_do_transaction_failed,
    .do_unaligned_access = arm_cpu_do_unaligned_access,
    .adjust_watchpoint_address = arm_adjust_watchpoint_address,
    .debug_check_watchpoint = arm_debug_check_watchpoint,
    .debug_check_breakpoint = arm_debug_check_breakpoint,
#endif /* !CONFIG_USER_ONLY */
};
#endif /* CONFIG_TCG */
```

## IN CPUArchState --> CPUARMState
```c
typedef struct CPUArchState {
    /* Regs for current mode.  */
    uint32_t regs[16];

    /* 32/64 switch only happens when taking and returning from
     * exceptions so the overlap semantics are taken care of then
     * instead of having a complicated union.
     */
    /* Regs for A64 mode.  */
    uint64_t xregs[32];
    uint64_t pc;
    ...
}
static void arm_cpu_set_pc(CPUState *cs, vaddr value)
{
    ARMCPU *cpu = ARM_CPU(cs);
    CPUARMState *env = &cpu->env;

    if (is_a64(env)) {
        env->pc = value;
        env->thumb = false;
    } else {
        env->regs[15] = value & ~1;
        env->thumb = value & 1;
    }
}

static vaddr arm_cpu_get_pc(CPUState *cs)
{
    ARMCPU *cpu = ARM_CPU(cs);
    CPUARMState *env = &cpu->env;

    if (is_a64(env)) {
        return env->pc;
    } else {
        return env->regs[15];
    }
}
```

so there is a truncation in uint32_t env->regs[15] = tb->pc, and unsigned extend the content of pc


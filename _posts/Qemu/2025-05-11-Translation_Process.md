---
title: Translation Process (in tb_gen_code)
date: 2025-05-11
categories: [Qemu]
tags: [qemu]     # TAG names should always be lowercase
published: false
---

> - Problem1: During translation, I need to know more about how target-specific translation processs interacts with the target-independent translation pipeline
> - Problem2: Modularized code maintaince

## The common translators
- `include/exec/translator.h`: Used in general translation framework, intergrated with target-specific DsiasContext.
```c

/**
 * TranslatorOps:
 * @init_disas_context:
 *      Initialize the target-specific portions of DisasContext struct.
 *      The generic DisasContextBase has already been initialized.
 *
 * @tb_start:
 *      Emit any code required before the start of the main loop,
 *      after the generic gen_tb_start().
 *
 * @insn_start:
 *      Emit the tcg_gen_insn_start opcode.
 *
 * @translate_insn:
 *      Disassemble one instruction and set db->pc_next for the start
 *      of the following instruction.  Set db->is_jmp as necessary to
 *      terminate the main loop.
 *
 * @tb_stop:
 *      Emit any opcodes required to exit the TB, based on db->is_jmp.
 *
 * @disas_log:
 *      Print instruction disassembly to log.
 */
typedef struct TranslatorOps {
    void (*init_disas_context)(DisasContextBase *db, CPUState *cpu);
    void (*tb_start)(DisasContextBase *db, CPUState *cpu);
    void (*insn_start)(DisasContextBase *db, CPUState *cpu);
    void (*translate_insn)(DisasContextBase *db, CPUState *cpu);
    void (*tb_stop)(DisasContextBase *db, CPUState *cpu);
    void (*disas_log)(const DisasContextBase *db, CPUState *cpu, FILE *f);
} TranslatorOps;

```
- `gen_intermediate_code`, used in `translate-all.c` provided by the target, which should create the target-specific `DisasContext`, and then invoke translator_loop.
```c
/**
 * gen_intermediate_code
 * @cpu: cpu context
 * @tb: translation block
 * @max_insns: max number of instructions to translate
 * @pc: guest virtual program counter address
 * @host_pc: host physical program counter address
 *
 * This function must be provided by the target, which should create
 * the target-specific DisasContext, and then invoke translator_loop.
 */
void gen_intermediate_code(CPUState *cpu, TranslationBlock *tb, int *max_insns,
                           vaddr pc, void *host_pc);
```

- DisasContextBase
```c
/**
 * DisasContextBase:
 * @tb: Translation block for this disassembly.
 * @pc_first: Address of first guest instruction in this TB.
 * @pc_next: Address of next guest instruction in this TB (current during
 *           disassembly).
 * @is_jmp: What instruction to disassemble next.
 * @num_insns: Number of translated instructions (including current).
 * @max_insns: Maximum number of instructions to be translated in this TB.
 * @singlestep_enabled: "Hardware" single stepping enabled.
 * @saved_can_do_io: Known value of cpu->neg.can_do_io, or -1 for unknown.
 * @plugin_enabled: TCG plugin enabled in this TB.
 * @insn_start: The last op emitted by the insn_start hook,
 *              which is expected to be INDEX_op_insn_start.
 *
 * Architecture-agnostic disassembly context.
 */
typedef struct DisasContextBase {
    TranslationBlock *tb;
    vaddr pc_first;
    vaddr pc_next;
    DisasJumpType is_jmp;
    int num_insns;
    int max_insns;
    bool singlestep_enabled;
    bool plugin_enabled;
    struct TCGOp *insn_start;
    void *host_addr[2];
} DisasContextBase;
```


- `DisasContext` defined in each target's `translate.h`
```c
typedef struct DisasContext {
    DisasContextBase base;
    const ARMISARegisters *isar;

    /* The address of the current instruction being translated. */
    target_ulong pc_curr;
    /*
     * For CF_PCREL, the full value of cpu_pc is not known
     * (although the page offset is known).  For convenience, the
     * translation loop uses the full virtual address that triggered
     * the translation, from base.pc_start through pc_curr.
     * For efficiency, we do not update cpu_pc for every instruction.
     * Instead, pc_save has the value of pc_curr at the time of the
     * last update to cpu_pc, which allows us to compute the addend
     * needed to bring cpu_pc current: pc_curr - pc_save.
     * If cpu_pc now contains the destination of an indirect branch,
     * pc_save contains -1 to indicate that relative updates are no
     * longer possible.
     */
    target_ulong pc_save;
    target_ulong page_start;
    uint32_t insn;
    ...
} DisasContext;
```




## Target-specific Translator
- `target/arm/tcg/translate.c`
```c
static void arm_tr_disas_log(const DisasContextBase *dcbase,
                             CPUState *cpu, FILE *logfile)
{
    DisasContext *dc = container_of(dcbase, DisasContext, base);

    fprintf(logfile, "IN: %s\n", lookup_symbol(dc->base.pc_first));
    target_disas(logfile, cpu, dc->base.pc_first, dc->base.tb->size);
}

static const TranslatorOps arm_translator_ops = {
    .init_disas_context = arm_tr_init_disas_context,
    .tb_start           = arm_tr_tb_start,
    .insn_start         = arm_tr_insn_start,
    .translate_insn     = arm_tr_translate_insn,
    .tb_stop            = arm_tr_tb_stop,
    .disas_log          = arm_tr_disas_log,
};
```

- Example `translate_insn` in arm ISA
```c
static void arm_tr_translate_insn(DisasContextBase *dcbase, CPUState *cpu)
{
    DisasContext *dc = container_of(dcbase, DisasContext, base);
    CPUARMState *env = cpu_env(cpu);
    uint32_t pc = dc->base.pc_next;
    unsigned int insn;

    /* Singlestep exceptions have the highest priority. */
    if (arm_check_ss_active(dc)) {
        dc->base.pc_next = pc + 4;
        return;
    }

    if (pc & 3) {
        /*
         * PC alignment fault.  This has priority over the instruction abort
         * that we would receive from a translation fault via arm_ldl_code
         * (or the execution of the kernelpage entrypoint). This should only
         * be possible after an indirect branch, at the start of the TB.
         */
        assert(dc->base.num_insns == 1);
        gen_helper_exception_pc_alignment(tcg_env, tcg_constant_tl(pc));
        dc->base.is_jmp = DISAS_NORETURN;
        dc->base.pc_next = QEMU_ALIGN_UP(pc, 4);
        return;
    }

    if (arm_check_kernelpage(dc)) {
        dc->base.pc_next = pc + 4;
        return;
    }

    dc->pc_curr = pc;
    insn = arm_ldl_code(env, &dc->base, pc, dc->sctlr_b);
    dc->insn = insn;
    dc->base.pc_next = pc + 4;

    disas_arm_insn(dc, insn);

    arm_post_translate_insn(dc);

    /* ARM is a fixed-length ISA.  We performed the cross-page check
       in init_disas_context by adjusting max_insns.  */
}
```
- In `disas_arm_insn`, execution paths varies according to condition execution. Calling meson.build to decode the instruction through templates. `target/arm/tcg/meson.build`
```shell
gen_a32 = [
  decodetree.process('neon-shared.decode', extra_args: '--decode=disas_neon_shared'),
  decodetree.process('neon-dp.decode', extra_args: '--decode=disas_neon_dp'),
  decodetree.process('neon-ls.decode', extra_args: '--decode=disas_neon_ls'),
  decodetree.process('vfp.decode', extra_args: '--decode=disas_vfp'),
  decodetree.process('vfp-uncond.decode', extra_args: '--decode=disas_vfp_uncond'),
  decodetree.process('m-nocp.decode', extra_args: '--decode=disas_m_nocp'),
  decodetree.process('mve.decode', extra_args: '--decode=disas_mve'),
  decodetree.process('a32.decode', extra_args: '--static-decode=disas_a32'),
  decodetree.process('a32-uncond.decode', extra_args: '--static-decode=disas_a32_uncond'),
  decodetree.process('t32.decode', extra_args: '--static-decode=disas_t32'),
  decodetree.process('t16.decode', extra_args: ['-w', '16', '--static-decode=disas_t16']),
]

arm_ss.add(gen_a32)
```
- `t32.decode` -> more modularized 
```txt
# Data processing (plain binary immediate)

%imm12_26_12_0   26:1 12:3 0:8
%neg12_26_12_0   26:1 12:3 0:8 !function=negate
@s0_rri_12       .... ... .... . rn:4 . ... rd:4 ........ \
                 &s_rri_rot imm=%imm12_26_12_0 rot=0 s=0
{
  ADR            1111 0.1 0000 0 1111 0 ... rd:4 ........ \
                 &ri imm=%imm12_26_12_0
  ADD_rri        1111 0.1 0000 0 .... 0 ... .... ........     @s0_rri_12
}
{
  ADR            1111 0.1 0101 0 1111 0 ... rd:4 ........ \
                 &ri imm=%neg12_26_12_0
  SUB_rri        1111 0.1 0101 0 .... 0 ... .... ........     @s0_rri_12
}
```


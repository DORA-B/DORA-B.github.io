---
title: Trace the Binary Label in qemu-6.2.0 and the Translation steps
date: 2024-02-25
categories: [Compiler]
tags: [compiler, angr]     # TAG names should always be lowercase
published: false
---
## How to Find `Binary:`
+ Find `void mips_tr_disas_log` in `target/mips/tcg/translate.c`
```c++
static void mips_tr_disas_log(const DisasContextBase *dcbase, CPUState *cs)
{
    qemu_log("IN: %s\n", lookup_symbol(dcbase->pc_first));
    log_target_disas(cs, dcbase->pc_first, dcbase->tb->size);
    fprintf(stderr, "Binary:\n", dcbase->tb->size);
    for(int i = 0; i < dcbase->tb->size; i+=4) {
        for (int j = 0; j < 4; j++) {
            uint8_t *p = (uint8_t *)dcbase->pc_first + i + j;
            uint8_t *hp = (uint8_t *)g2h(env_cpu(cpu_env), p);
            fprintf(stderr, "%02x", *hp);
        }
        fprintf(stderr, "\n");
    }
}
```
+ Find mips_tr_disas_log:
```shell
static const TranslatorOps mips_tr_ops = {
    .init_disas_context = mips_tr_init_disas_context,
    .tb_start           = mips_tr_tb_start,
    .insn_start         = mips_tr_insn_start,
    .translate_insn     = mips_tr_translate_insn,
    .tb_stop            = mips_tr_tb_stop,
    .disas_log          = mips_tr_disas_log,
};
```

+ And finally it is called here:
```c++
void gen_intermediate_code(CPUState *cs, TranslationBlock *tb, int max_insns)
{
    DisasContext ctx;

    translator_loop(&mips_tr_ops, &ctx.base, cs, tb, max_insns);
}
```

So, `translator_loop` is the function that `qemu-bin -d in_asm,out_asm` uses, and different arches will have different `gen_intermediate_code()` function

## different arches use different strategies to generate qemu log
+ arm
```c++
/* generate intermediate code for basic block 'tb'.  */
void gen_intermediate_code(CPUState *cpu, TranslationBlock *tb, int max_insns)
{
    DisasContext dc = { };
    const TranslatorOps *ops = &arm_translator_ops;
    CPUARMTBFlags tb_flags = arm_tbflags_from_tb(tb);

    if (EX_TBFLAG_AM32(tb_flags, THUMB)) {
        ops = &thumb_translator_ops;
    }
#ifdef TARGET_AARCH64
    if (EX_TBFLAG_ANY(tb_flags, AARCH64_STATE)) {
        ops = &aarch64_translator_ops;
    }
#endif

    translator_loop(ops, &dc.base, cpu, tb, max_insns);
}
static const TranslatorOps arm_translator_ops = {
    .init_disas_context = arm_tr_init_disas_context,
    .tb_start           = arm_tr_tb_start,
    .insn_start         = arm_tr_insn_start,
    .translate_insn     = arm_tr_translate_insn,
    .tb_stop            = arm_tr_tb_stop,
    .disas_log          = arm_tr_disas_log,
};

static const TranslatorOps thumb_translator_ops = {
    .init_disas_context = arm_tr_init_disas_context,
    .tb_start           = arm_tr_tb_start,
    .insn_start         = arm_tr_insn_start,
    .translate_insn     = thumb_tr_translate_insn,
    .tb_stop            = arm_tr_tb_stop,
    .disas_log          = arm_tr_disas_log,
};
const TranslatorOps aarch64_translator_ops = {
    .init_disas_context = aarch64_tr_init_disas_context,
    .tb_start           = aarch64_tr_tb_start,
    .insn_start         = aarch64_tr_insn_start,
    .translate_insn     = aarch64_tr_translate_insn,
    .tb_stop            = aarch64_tr_tb_stop,
    .disas_log          = aarch64_tr_disas_log,
};
```

## Tiny Code Generator (TCG)
The Tiny Code Generator (TCG) exists to transform target insns (the processor being emulated) via the TCG frontend to TCG ops which are then transformed into host insns (the processor executing QEMU itself) via the TCG backend.

People who wish to port QEMU to run on a new processor need to be concerned with the backend. There also exists the TCI (TCG Interpreter) effort which provides a backend agnostic interpreter for TCGops.

People who wish to port QEMU to emulate a new processor need to be concerned with the front-end.

## Project validation between guest and host

```py
def do_validation(self):
	print('## Create guest ' + guest.isa + ' symbol machine ...')
	print('## Create host ' + host.isa + ' symbol machine ...')

	print('## Init guest ' + guest.isa + ' machine state ...')

	print('## Run guest ' + guest.isa + ' machine ...')

	print('## Init host ' + host.isa + ' machine state ...')

	print('## Run host ' + host.isa + ' machine ...')

	print('## Validate the states ...')

	if len(gexcep_states) != len(hexcep_states):
		print('## Found different numbers of exception states: guest (', \
			  len(gexcep_states), ') vs host (', len(hexcep_states), ')')
	if len(gunsat_states) != len(hunsat_states):
		if len(gfini_states) != 1 or len(hfini_states) != 2:
			print('## Found different numbers of unsatisfiable states: guest (', \
				  len(gunsat_states), ') vs host (', len(hunsat_states), ')')
			print('## One guest state and two host states. ', end='')
			print('Merge host states for validation ...')
			gfstate = gfini_states[0]
			hfstate, _, _ = hfini_states[0].merge(hfini_states[1])
			if self.do_state_validation(gfstate, hfstate):
				print('## Pass translation validation')
			else:
				print('## Fail translation validation')
		else:
			print('## Found different numbers of fini states: guest (', \
				  len(gfini_states), ') vs host (', len(hfini_states), ')')
	else:
		if len(gfini_states) == 0:
			print('## Found zero fini state for validation')
		else:
			print('## Number of fini states to validate:', len(gfini_states))
			sidx = 0
			for gfstate in gfini_states:
				hfstate = self.find_state_with_equivalent_constraints(gfstate, hfini_states)
				if hfstate == None:
					print('## Cannot find equivalent host state')
					continue
				if self.do_state_validation(gfstate, hfstate):
					print('## Pass translation validation for state', sidx)
				else:
					print('## Fail translation validation for state', sidx)
```

+ First, remove the first three and last two host instructions in qemu, which are used to exist from the code cache
+ Create guest Guest.symbolicExecutionMachine to do some initialization and extract the state of the symbolic execution.
	+ initialize registers and condition code
		+  (sp, lr, pc)
		+  condition flag (nf, zf, cf, vf)
		+  (cc_dep1_, cc_dep2_, cc_ndep_)
		+  (qflag32, geflag0, geflag1, geflag2, geflag3)
	+ extract machine state
		+ (sp, lr, pc)
		+ extract condition flag using angr (nf, zf, cf, vf)
		+ (qflag32, geflag0, geflag1, geflag2, geflag3)
+ Guest SymbolMachine Execution to get the state
	+ Init the machine state:
		+ Init registers --> for the subclass that have arch-specific implementation
		+ Init Memory --> **`Class Memory`**
	+ Run Guest Machine
		+ Use init machine state to set up breakpoints to capture memory accesses
		+ Step forward
		+ Clear up error states
		+ Drop potential dead states
+ Create host Host.symbolicExecutionMachine to do some initialization and extract state of the symbolic execution.
	+ Using insn_list and data_list to initialize the Symbolic Machine state
+ Host SymbolMachine Execution to get state
	+ get guest isa, memory_state_end,  and guest initstate (init_reg_state, init_mem_state), use them to init the host machine
	+ Run Host machine
+ Extract the final states of two machines and compare
	```py
	def find_state_with_equivalent_constraints(self, state, state_list):
        for s in state_list:
            cons = ()
            for c in s.solver.constraints:
                cons += (c,)
            if state.solver.satisfiable(extra_constraints=cons):
                return s
        return None
	```

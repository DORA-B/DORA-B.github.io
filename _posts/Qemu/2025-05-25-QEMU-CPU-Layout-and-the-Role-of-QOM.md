---
title: QEMU CPU Layout and the Role of QOM
date: 2025-05-25
categories: [Qemu]
tags: [qemu]     # TAG names should always be lowercase
published: false
---

[TOC]


> Problem1: There is a initialization of translation in `tb_gen_code`, before `tcg_gen_code`, and `offsetof` actually gives the offset of one register, represented as a section of memory to store the value of a specific register.
> Problem2: how TCGContext is set and used? there is a CPUState * cpu in it, how it is used?

```c
/* initialize TCG globals.  */
void arm_translate_init(void)
{
    int i;

    for (i = 0; i < 16; i++) {
        cpu_R[i] = tcg_global_mem_new_i32(tcg_env,
                                          offsetof(CPUARMState, regs[i]),
                                          regnames[i]);
    }
    cpu_CF = tcg_global_mem_new_i32(tcg_env, offsetof(CPUARMState, CF), "CF");
    cpu_NF = tcg_global_mem_new_i32(tcg_env, offsetof(CPUARMState, NF), "NF");
    cpu_VF = tcg_global_mem_new_i32(tcg_env, offsetof(CPUARMState, VF), "VF");
    cpu_ZF = tcg_global_mem_new_i32(tcg_env, offsetof(CPUARMState, ZF), "ZF");

    // add for rule-translation
    cpu_CC_Source = tcg_global_mem_new_i32(tcg_env, offsetof(CPUARMState, cc_source), "CC_SOURCE");

    cpu_exclusive_addr = tcg_global_mem_new_i64(tcg_env,
        offsetof(CPUARMState, exclusive_addr), "exclusive_addr");
    cpu_exclusive_val = tcg_global_mem_new_i64(tcg_env,
        offsetof(CPUARMState, exclusive_val), "exclusive_val");

    a64_translate_init();
}
```

# Key structures

QEMU models CPUs using multiple layers:

```c
// include/qemu/typedefs.h --> include/hw/core/cpu.h  
/**
 * CPUState:
 * @cpu_index: CPU index (informative).
 * @cluster_index: Identifies which cluster this CPU is in.
 *   For boards which don't define clusters or for "loose" CPUs not assigned
 *   to a cluster this will be UNASSIGNED_CLUSTER_INDEX; otherwise it will
 *   be the same as the cluster-id property of the CPU object's TYPE_CPU_CLUSTER
 *   QOM parent.
 *   Under TCG this value is propagated to @tcg_cflags.
 *   See TranslationBlock::TCG CF_CLUSTER_MASK.
 * @tcg_cflags: Pre-computed cflags for this cpu.
 * @nr_cores: Number of cores within this CPU package.
 * @nr_threads: Number of threads within this CPU core.
 * @running: #true if CPU is currently running (lockless).
 * @has_waiter: #true if a CPU is currently waiting for the cpu_exec_end;
 * valid under cpu_list_lock.
 * @created: Indicates whether the CPU thread has been successfully created.
 * @interrupt_request: Indicates a pending interrupt request.
 * @halted: Nonzero if the CPU is in suspended state.
 * @stop: Indicates a pending stop request.
 * @stopped: Indicates the CPU has been artificially stopped.
 * @unplug: Indicates a pending CPU unplug request.
 * @crash_occurred: Indicates the OS reported a crash (panic) for this CPU
 * @singlestep_enabled: Flags for single-stepping.
 * @icount_extra: Instructions until next timer event.
 * @neg.can_do_io: True if memory-mapped IO is allowed.
 * @cpu_ases: Pointer to array of CPUAddressSpaces (which define the
 *            AddressSpaces this CPU has)
 * @num_ases: number of CPUAddressSpaces in @cpu_ases
 * @as: Pointer to the first AddressSpace, for the convenience of targets which
 *      only have a single AddressSpace
 * @gdb_regs: Additional GDB registers.
 * @gdb_num_regs: Number of total registers accessible to GDB.
 * @gdb_num_g_regs: Number of registers in GDB 'g' packets.
 * @node: QTAILQ of CPUs sharing TB cache.
 * @opaque: User data.
 * @mem_io_pc: Host Program Counter at which the memory was accessed.
 * @accel: Pointer to accelerator specific state.
 * @kvm_fd: vCPU file descriptor for KVM.
 * @work_mutex: Lock to prevent multiple access to @work_list.
 * @work_list: List of pending asynchronous work.
 * @plugin_mem_cbs: active plugin memory callbacks
 * @plugin_state: per-CPU plugin state
 * @ignore_memory_transaction_failures: Cached copy of the MachineState
 *    flag of the same name: allows the board to suppress calling of the
 *    CPU do_transaction_failed hook function.
 * @kvm_dirty_gfns: Points to the KVM dirty ring for this CPU when KVM dirty
 *    ring is enabled.
 * @kvm_fetch_index: Keeps the index that we last fetched from the per-vCPU
 *    dirty ring structure.
 *
 * State of one CPU core or thread.
 *
 * Align, in order to match possible alignment required by CPUArchState,
 * and eliminate a hole between CPUState and CPUArchState within ArchCPU.
 */
struct CPUState {
    /*< private >*/
    DeviceState parent_obj;
    /* cache to avoid expensive CPU_GET_CLASS */
    CPUClass *cc;
    /*< public >*/

    int nr_cores;
    int nr_threads;

    struct QemuThread *thread;
    //...
};


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
    /* PSTATE isn't an architectural register for ARMv8. However, it is
     * convenient for us to assemble the underlying state into a 32 bit format
     * identical to the architectural format used for the SPSR. (This is also
     * what the Linux kernel's 'pstate' field in signal handlers and KVM's
     * 'pstate' register are.) Of the PSTATE bits:
     *  NZCV are kept in the split out env->CF/VF/NF/ZF, (which have the same
     *    semantics as for AArch32, as described in the comments on each field)
     *  nRW (also known as M[4]) is kept, inverted, in env->aarch64
     *  DAIF (exception masks) are kept in env->daif
     *  BTYPE is kept in env->btype
     *  SM and ZA are kept in env->svcr
     *  all other bits are stored in their correct places in env->pstate
     */
    uint32_t pstate;
    bool aarch64; /* True if CPU is in aarch64 state; inverse of PSTATE.nRW */
    bool thumb;   /* True if CPU is in thumb mode; cpsr[5] */

    /* Cached TBFLAGS state.  See below for which bits are included.  */
    CPUARMTBFlags hflags;
    // ...
} CPUARMState;


/**
 * ARMCPU:
 * @env: #CPUARMState
 *
 * An ARM CPU core.
 */
struct ArchCPU {
    CPUState parent_obj;
    CPUARMState env;
    // Additional ARM-specific fields
};
```

## Memory Layout Comparison
---------------------------

### System Mode (full QOM object model)

```c
struct ARMCPU {
    CPUState parent_obj;
    CPUARMState env;
    ...
};
```

**Layout:**

```
[ ARMCPU ]
  ├── [ CPUState parent_obj ]
  └── [ CPUARMState env ]
```

* `ARMCPU` is a full QOM object: created with `object_new(TYPE_ARM_CPU)`
    
* Fully integrated into QEMU’s type system (QOM)
    
* Used in board-level and SoC-level emulation
    
* Has support for interrupts, devices, peripherals, MMUs
    

### User Mode (manual flat allocation)

```c
// allocation:
cpu = g_malloc0(sizeof(CPUState) + sizeof(CPUArchState));
```

**Layout:**

```
[ CPUState ]
[ CPUArchState ]
```

* No `ARMCPU` struct at all
    
* Not a QOM object
    
* No inheritance or device model
    
* Only what's needed for executing user-space ELF binaries (e.g., `qemu-arm ./a.out`)
    

## Code View: How Each Is Used
------------------------------

* **System Mode** = Full class with inheritance and method dispatch (`QOM`)
    
* **User Mode** = Manually laid-out struct with no inheritance

* **System Mode** needs peripheral support, QMP interaction, board integration
    
* **User Mode** needs minimal CPU state, speed, and simplicity
    
    * It avoids the overhead of QOM and devices
        
    * All it needs is `CPUState` + `CPUArchState`

### In System Mode:

```c
ARMCPU *cpu = ARM_CPU(object_new(TYPE_ARM_CPU));
CPUState *cs = CPU(cpu);
CPUARMState *env = &cpu->env;
```

### In User Mode:

```c
CPUState *cpu = g_malloc0(sizeof(CPUState) + sizeof(CPUArchState));
CPUArchState *env = cpu_env(cpu); // defined as (CPUArchState *)(cpu + 1)
```



# Visual Diagram
------------------------

```text
System Mode (ARMCPU object)
---------------------------
// In system-mode emulation (e.g. full machine), QEMU uses a full object:
// ARMCPU *cpu = ARM_CPU(object_new(TYPE_ARM_CPU));

[ ARMCPU = { CPUState | CPUARMState | ARM-specific fields } ]
         ↑             ↑
       (CPUState*)     &cpu->env

[ ARMCPU ]
  ├── [ CPUState parent_obj ]
  └── [ CPUARMState env ]

User Mode (flat allocation)
---------------------------
// In user-mode emulation, QEMU doesn't use full QOM inheritance. It simply does:
// cpu = g_malloc0(sizeof(CPUState) + sizeof(CPUArchState));

[ CPUState ][ CPUArchState ]
     ↑             ↑
    cpu        cpu_env(cpu)
```


## From cpu to env (User mode) `include/hw/core/cpu.h`
```c
// (CPUArchState *) (cpu + sizeof(CPUState)|sizeof(cpu))
static inline CPUArchState *cpu_env(CPUState *cpu)
{
    /* We validate that CPUArchState follows CPUState in cpu-all.h. */
    return (CPUArchState *)(cpu + 1);
}
```

## From env to cpu (User mode) `include/exec/cpu-common.h`


```c
/**
 * env_cpu(env)
 * @env: The architecture environment
 *
 * Return the CPUState associated with the environment.
 */
static inline CPUState *env_cpu(CPUArchState *env)
{
    return (void *)env - sizeof(CPUState);
}
/**
 * env_archcpu(env)
 * @env: The architecture environment
 *
 * Return the ArchCPU associated with the environment.
 */
static inline ArchCPU *env_archcpu(CPUArchState *env)
{
    return (void *)env - sizeof(CPUState);
}

```

# Execution Flow (User Mode)
-----------------------------

```c
cpu = cpu_create(cpu_type);       // allocates CPUState + CPUArchState
env = cpu_env(cpu);               // returns pointer to env part
cpu_reset(cpu);                   // initializes state
cpu_loop(env);                    // enters the main emulation loop
```

This all works thanks to predictable layout and helper macros.


# How TCGContext is initialized? and its inner pointer of CPUState

## TCGContexts
- 1. Global `TCGContext` and definition in `tcg.c`
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

    int page_mask;
    uint8_t page_bits;
    uint8_t tlb_dyn_max_bits;
    uint8_t insn_start_words; // /* ARM-specific extra insn start words: * 1: Conditional execution bits * 2: Partial exception syndrome for data aborts */
    TCGBar guest_mo;

    TCGRegSet reserved_regs;
    intptr_t current_frame_offset;
    intptr_t frame_start;
    intptr_t frame_end;
    TCGTemp *frame_temp;

    TranslationBlock *gen_tb;     /* tb for which code is being generated */
    tcg_insn_unit *code_buf;      /* pointer for start of tb */
    tcg_insn_unit *code_ptr;      /* pointer for running end of tb */

#ifdef CONFIG_DEBUG_TCG
    int goto_tb_issue_mask;
    const TCGOpcode *vecop_list;
#endif

    /* Code generation.  Note that we specifically do not use tcg_insn_unit
       here, because there's too much arithmetic throughout that relies
       on addition and subtraction working on bytes.  Rely on the GCC
       extension that allows arithmetic on void*.  */
    void *code_gen_buffer;
    size_t code_gen_buffer_size;
    void *code_gen_ptr;
    void *data_gen_ptr;

    /* Threshold to flush the translated code buffer.  */
    void *code_gen_highwater;

    /* Track which vCPU triggers events */
    CPUState *cpu;                      /* *_trans */
    /// ...
}
TCGContext tcg_init_ctx;
__thread TCGContext *tcg_ctx;
```

- 2. Initialize of `TCGContext`
```c
// linux-user/main.c
    /* init tcg before creating CPUs */
    // RULE NOTE: in init machine, there is tcg_init
    // which will call 
    // * tcg_context_init 
    // * tcg_region_init
    {
        AccelState *accel = current_accel();
        AccelClass *ac = ACCEL_GET_CLASS(accel);

        accel_init_interfaces(ac);
        object_property_set_bool(OBJECT(accel), "one-insn-per-tb",
                                 opt_one_insn_per_tb, &error_abort);
        ac->init_machine(NULL); // call `tcg_init`
    }
// tcg.c
void tcg_init(size_t tb_size, int splitwx, unsigned max_cpus)
{
    tcg_context_init(max_cpus);
    tcg_region_init(tb_size, splitwx, max_cpus);
}
```
- 3. init the registers memory layout in `env` before `cpu_loop` in `main.c`：  1. Copies general registers (R0–R15)； 2. Sets up process memory layout (stack, heap)
```c
void target_cpu_copy_regs(CPUArchState *env, struct target_pt_regs *regs)
{
    CPUState *cpu = env_cpu(env);
    TaskState *ts = get_task_state(cpu);
    struct image_info *info = ts->info;
    int i;

    cpsr_write(env, regs->uregs[16], CPSR_USER | CPSR_EXEC,
               CPSRWriteByInstr);
    for(i = 0; i < 16; i++) {
        env->regs[i] = regs->uregs[i];
    }
#if TARGET_BIG_ENDIAN
    /* Enable BE8.  */
    if (EF_ARM_EABI_VERSION(info->elf_flags) >= EF_ARM_EABI_VER4
        && (info->elf_flags & EF_ARM_BE8)) {
        env->uncached_cpsr |= CPSR_E;
        env->cp15.sctlr_el[1] |= SCTLR_E0E;
    } else {
        env->cp15.sctlr_el[1] |= SCTLR_B;
    }
    arm_rebuild_hflags(env);
#endif

    ts->stack_base = info->start_stack;
    ts->heap_base = info->brk;
    /* This will be filled in on the first SYS_HEAPINFO call.  */
    ts->heap_limit = 0;
}
```
- 4. init and reset `tcg_ctx->cpu` before and after `gen_intermediate_code`.
```c
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

# Situations when write to the `env` in `ArchCPUState`

- `translate-a32.h`：store and load according to the `offsetof`.
```c
#define store_cpu_field(val, name)                                      \
    ({                                                                  \
        QEMU_BUILD_BUG_ON(sizeof_field(CPUARMState, name) != 4          \
                          && sizeof_field(CPUARMState, name) != 1);     \
        store_cpu_offset(val, offsetof(CPUARMState, name),              \
                         sizeof_field(CPUARMState, name));              \
    })

#define store_cpu_field_constant(val, name) \
    store_cpu_field(tcg_constant_i32(val), name)

/*
 * Store var into env + offset to a member with size bytes.
 * Free var after use.
 */
void store_cpu_offset(TCGv_i32 var, int offset, int size)
{
    switch (size) {
    case 1:
        tcg_gen_st8_i32(var, tcg_env, offset);
        break;
    case 4:
        tcg_gen_st_i32(var, tcg_env, offset);
        break;
    default:
        g_assert_not_reached();
    }
}

/* Load from a 32-bit field to a TCGv_i32 */
#define load_cpu_field(name)                                            \
    ({                                                                  \
        QEMU_BUILD_BUG_ON(sizeof_field(CPUARMState, name) != 4);        \
        load_cpu_offset(offsetof(CPUARMState, name));                   \
    })

/* Load from the low half of a 64-bit field to a TCGv_i32 */
#define load_cpu_field_low32(name)                                      \
    ({                                                                  \
        QEMU_BUILD_BUG_ON(sizeof_field(CPUARMState, name) != 8);        \
        load_cpu_offset(offsetoflow32(CPUARMState, name));              \
    })

static inline TCGv_i32 load_cpu_offset(int offset)
{
    TCGv_i32 tmp = tcg_temp_new_i32();
    tcg_gen_ld_i32(tmp, tcg_env, offset);
    return tmp;
}
```

- How is `env` itself handled? QEMU ensures that `%rbp` points to the base of `env`, and offsets are calculated based on offsetof(env, regs[i]).

```c
static void tcg_context_init(unsigned max_cpus)
{
    TCGContext *s = &tcg_init_ctx;
    int op, total_args, n, i;
    TCGOpDef *def;
    TCGArgConstraint *args_ct;
    TCGTemp *ts;
    // ....
    tcg_debug_assert(!tcg_regset_test_reg(s->reserved_regs, TCG_AREG0));
    ts = tcg_global_reg_new_internal(s, TCG_TYPE_PTR, TCG_AREG0, "env");
    tcg_env = temp_tcgv_ptr(ts);
}
// in tcg/i386/tcg-taret.h
{
    // ...
    TCG_AREG0 = TCG_REG_EBP,
    TCG_REG_CALL_STACK = TCG_REG_ESP
} TCGReg;
```

- The mapping from `env->regs[i]` to `offset(%rbp)` is computed as:
```c
[base host register holding env] + offset(env->regs[i])
```


# How guest registers are set during front-end disambling process?


- origianl guest code
```c
----------------
IN: __tunables_init
0x0002c4e0:  e35b0000  cmp      fp, #0
0x0002c4e4:  0a000050  beq      #0x2c62c

OP:
 ld_i32 loc3,env,$0xfffffffffffffff0
 brcond_i32 loc3,$0x0,lt,$L0
 st8_i32 $0x0,env,$0xfffffffffffffff4

 ---- 000000000002c4e0 0000000000000000 0000000000000000
 mov_i32 loc5,r11
 sub_i32 NF,loc5,$0x0
 mov_i32 ZF,NF
 setcond_i32 CF,loc5,$0x0,geu
 xor_i32 VF,NF,loc5
 xor_i32 loc6,loc5,$0x0
 and_i32 VF,VF,loc6
 mov_i32 loc5,NF
 st8_i32 $0x1,env,$0xfffffffffffffff4

 ---- 000000000002c4e4 0000000000000000 0000000000000000
 brcond_i32 ZF,$0x0,ne,$L1
 goto_tb $0x0
 mov_i32 pc,$0x2c62c
 exit_tb $0x7fb894004280
 set_label $L1
 goto_tb $0x1
 mov_i32 pc,$0x2c4e8
 exit_tb $0x7fb894004281
 set_label $L0
 exit_tb $0x7fb894004283

OP after optimization and liveness analysis:
 ld_i32 tmp3,env,$0xfffffffffffffff0      pref=0xffff
 brcond_i32 tmp3,$0x0,lt,$L0              dead: 0
 st8_i32 $0x0,env,$0xfffffffffffffff4   

 ---- 000000000002c4e0 0000000000000000 0000000000000000
 mov_i32 NF,r11                           sync: 0  dead: 1  pref=0xffff
 mov_i32 ZF,NF                            sync: 0  dead: 1  pref=0xffff
 mov_i32 CF,$0x1                          sync: 0  dead: 0  pref=0xffff
 mov_i32 VF,$0x0                          sync: 0  dead: 0  pref=0xffff
 st8_i32 $0x1,env,$0xfffffffffffffff4     dead: 0 1

 ---- 000000000002c4e4 0000000000000000 0000000000000000
 brcond_i32 ZF,$0x0,ne,$L1                dead: 0 1
 goto_tb $0x0                           
 mov_i32 pc,$0x2c62c                      sync: 0  dead: 0 1  pref=0xffff
 exit_tb $0x7fb894004280                
 set_label $L1                          
 goto_tb $0x1                           
 mov_i32 pc,$0x2c4e8                      sync: 0  dead: 0 1  pref=0xffff
 exit_tb $0x7fb894004281                
 set_label $L0                          
 exit_tb $0x7fb894004283                

OUT: [size=123]
  -- guest addr 0x000000000002c4e0 + tb prologue
0x7fb894004340:  8b 5d f0                 movl     -0x10(%rbp), %ebx
0x7fb894004343:  85 db                    testl    %ebx, %ebx
0x7fb894004345:  0f 8c 64 00 00 00        jl       0x7fb8940043af
0x7fb89400434b:  c6 45 f4 00              movb     $0, -0xc(%rbp)
0x7fb89400434f:  8b 5d 2c                 movl     0x2c(%rbp), %ebx
0x7fb894004352:  89 9d 10 02 00 00        movl     %ebx, 0x210(%rbp)
0x7fb894004358:  89 9d 14 02 00 00        movl     %ebx, 0x214(%rbp)
0x7fb89400435e:  c7 85 08 02 00 00 01 00  movl     $1, 0x208(%rbp)
0x7fb894004366:  00 00
0x7fb894004368:  c7 85 0c 02 00 00 00 00  movl     $0, 0x20c(%rbp)
0x7fb894004370:  00 00
0x7fb894004372:  c6 45 f4 01              movb     $1, -0xc(%rbp)
  -- guest addr 0x000000000002c4e4
0x7fb894004376:  85 db                    testl    %ebx, %ebx
0x7fb894004378:  0f 85 19 00 00 00        jne      0x7fb894004397
0x7fb89400437e:  90                       nop      
0x7fb89400437f:  e9 00 00 00 00           jmp      0x7fb894004384
0x7fb894004384:  c7 45 3c 2c c6 02 00     movl     $0x2c62c, 0x3c(%rbp)
0x7fb89400438b:  48 8d 05 ee fe ff ff     leaq     -0x112(%rip), %rax
0x7fb894004392:  e9 81 bc ff ff           jmp      0x7fb894000018
0x7fb894004397:  e9 00 00 00 00           jmp      0x7fb89400439c
0x7fb89400439c:  c7 45 3c e8 c4 02 00     movl     $0x2c4e8, 0x3c(%rbp)
0x7fb8940043a3:  48 8d 05 d7 fe ff ff     leaq     -0x129(%rip), %rax
0x7fb8940043aa:  e9 69 bc ff ff           jmp      0x7fb894000018
0x7fb8940043af:  48 8d 05 cd fe ff ff     leaq     -0x133(%rip), %rax
0x7fb8940043b6:  e9 5d bc ff ff           jmp      0x7fb894000018

R00=407ffb7c R01=00000001 R02=00010000 R03=00000000
R04=00011da8 R05=00010158 R06=00000000 R07=00000000
R08=00000000 R09=00000000 R10=00087b7c R11=407ffb7c
R12=ffffffff R13=407ffa00 R14=00011734 R15=0002c4e0
PSR=60000010 -ZC- A usr32
```
In this code, need to translate `cmp` and `beq` to OPs and then host code.
- `tb_gen_code` process --> `generate_intermediate_code`, emit ops
```c
case 0x1:
    /* ....0001 0.01.... ........ ...0.... */
    disas_a32_extract_S_xrr_shi(ctx, &u.f_s_rrr_shi, insn);
    switch (insn & 0x0040f000) {
    case 0x00000000:
        /* ....0001 0001.... 0000.... ...0.... */
        /* ../target/arm/tcg/a32.decode:70 */
        if (trans_TST_xrri(ctx, &u.f_s_rrr_shi)) return true;
        break;
    case 0x00400000:
        /* ....0001 0101.... 0000.... ...0.... */
        /* ../target/arm/tcg/a32.decode:72 */
        if (trans_CMP_xrri(ctx, &u.f_s_rrr_shi)) return true;
        break;
    }
    break;
}
break;
```
- the structure related to `cmp` instruction
```c
typedef struct {
    int s;
    int rn;
    int rd;
    int imm;
    int rot;
} arg_s_rri_rot;
```
- tcg gen op through macro `#define DO_CMP2(NAME, OP, L)`
```c
DO_CMP2(CMP, gen_sub_CC, false) // --> give the CMP's front end op
/*
 * Data-processing (immediate)
 *
 * Operate, with set flags, one register source,
 * one rotated immediate, and a destination.
 *
 * Note that logic_cc && a->rot setting CF based on the msb of the
 * immediate is the reason why we must pass in the unrotated form
 * of the immediate.
 */
static bool op_s_rri_rot(DisasContext *s, arg_s_rri_rot *a,
                         void (*gen)(TCGv_i32, TCGv_i32, TCGv_i32),
                         int logic_cc, StoreRegKind kind)
{
    TCGv_i32 tmp1;
    uint32_t imm;

    imm = ror32(a->imm, a->rot); // ARM immediates in many instructions are encoded as a rotated 8-bit value.reconstructs the real 32-bit immediate by rotating a->imm right by a->rot.
    if (logic_cc && a->rot) {
        tcg_gen_movi_i32(cpu_CF, imm >> 31);
    }
    tmp1 = load_reg(s, a->rn); // new a tmp then load the reg to the tmp

    gen(tmp1, tmp1, tcg_constant_i32(imm)); // call front-end tcg op for cmp

    if (logic_cc) {
        gen_logic_CC(tmp1);
    }
    return store_reg_kind(s, a->rd, tmp1, kind);
}
```
- call front-end tcg op to generate the IR (without optimization) 
```c
/* dest = T0 - T1. Compute C, N, V and Z flags */
static void gen_sub_CC(TCGv_i32 dest, TCGv_i32 t0, TCGv_i32 t1)
{
    TCGv_i32 tmp;
    tcg_gen_sub_i32(cpu_NF, t0, t1);
    tcg_gen_mov_i32(cpu_ZF, cpu_NF);
    tcg_gen_setcond_i32(TCG_COND_GEU, cpu_CF, t0, t1);
    tcg_gen_xor_i32(cpu_VF, cpu_NF, t0);
    tmp = tcg_temp_new_i32();
    tcg_gen_xor_i32(tmp, t0, t1);
    tcg_gen_and_i32(cpu_VF, cpu_VF, tmp);
    tcg_gen_mov_i32(dest, cpu_NF);
}
```

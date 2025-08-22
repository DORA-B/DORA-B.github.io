---
title: QEMU TCG Optimization, Temp Value Arrangement
date: 2025-06-20
categories: [Qemu]
tags: [qemu_opt]     # TAG names should always be lowercase
published: false
---

> 1. How `TCGContext` is maintained in global/temp value level?
> 2. How data is allocated in TCG?

```c
struct TCGContext {
    uint8_t *pool_cur, *pool_end;
    TCGPool *pool_first, *pool_current, *pool_first_large;
    int nb_labels;
    int nb_globals;  // tcg_global_alloc, tcg_context_init, TCG_MAX_TEMPS maximum
    int nb_temps;    // tcg_temp_alloc, tcg_func_start (equal to nb_globals), TCG_MAX_TEMPS maximum
    int nb_indirects;
    int nb_ops;
    // ...

    GHashTable *const_table[TCG_TYPE_COUNT];
    TCGTempSet free_temps[TCG_TYPE_COUNT];
    TCGTemp temps[TCG_MAX_TEMPS]; // RULE NOTE: /* globals first, temps after */

    QTAILQ_HEAD(, TCGOp) ops, free_ops;
    QSIMPLEQ_HEAD(, TCGLabel) labels;
    // ...
}
```

# Allocation for global values
We had a discussion before about how the whole optimization process works in `tcg_gen_code`, `nb_ops` in `TCGContext` is maintained during `gen_intermediate_code`, while emitting ops.
Then, each op has different `args`, which are also allocated during emitting ops.
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
## Global Variables Allocation
1. global register allocation: `tcg_global_reg_new_internal`, the frame (call frame, initialized in prologue), global registers (env, _frame)
2. global memory allocation: `tcg_global_mem_new_internal`, arm registers (r0-r15) and condition code (cc).
3. for global allocation, increase the `nb_global`, and `nb_temps`
```c
/* tcg_global_mem_new_internal, tcg_global_reg_new_internal */
static TCGTemp *tcg_global_alloc(TCGContext *s)
{
    TCGTemp *ts;

    tcg_debug_assert(s->nb_globals == s->nb_temps); // RULE NOTE: Global allocations happen before temp allocation
    tcg_debug_assert(s->nb_globals < TCG_MAX_TEMPS);
    s->nb_globals++;
    ts = tcg_temp_alloc(s);
    ts->kind = TEMP_GLOBAL;

    return ts;
}
static TCGTemp *tcg_temp_alloc(TCGContext *s)
{
    int n = s->nb_temps++;

    if (n >= TCG_MAX_TEMPS) {
        tcg_raise_tb_overflow(s);
    }
    return memset(&s->temps[n], 0, sizeof(TCGTemp));
}
```
### Pre-settings with global values
- `nb_temps` is the number of temps allocated, initialized as the global temps number in `tcg_func_start`
```c
void tcg_func_start(TCGContext *s)
{
    tcg_pool_reset(s);
    s->nb_temps = s->nb_globals;

    /* No temps have been previously allocated for size or locality.  */
    memset(s->free_temps, 0, sizeof(s->free_temps));

    /* No constant temps have been previously allocated. */
    for (int i = 0; i < TCG_TYPE_COUNT; ++i) {
        if (s->const_table[i]) {
            g_hash_table_remove_all(s->const_table[i]);
        }
    }

    s->nb_ops = 0;
    s->nb_labels = 0;
    s->current_frame_offset = s->frame_start;
    //....
}
```
- Global temps are inited following the order:
  0: `tcg_init --> tcg_global_reg_new_internal(s, TCG_TYPE_PTR, TCG_AREG0, "env"); --> tcg_env`: `TCG_AREG0 --> TCG_REG_CALL_STACK`, `env`: TEMP_FIXED
  1: `arm_translate_init`: `r0 --> r15`, `cc`, `exclusive_addr`, `exclusive_val`: TEMP_GLOBAL
  2: `tcg_prologue_init --> tcg_set_frame`: `tcg_set_frame(s, TCG_REG_CALL_STACK, TCG_STATIC_CALL_ARGS_SIZE, CPU_TEMP_BUF_NLONGS * sizeof(long));`: TEMP_FIXED
- Backtrack of `arm_translate_init`
```shell
#0  arm_translate_init () at ../target/arm/tcg/translate.c:64
#1   tcg_exec_realizefn (cpu=0x55555595bf10, errp=0x7fffffffcd78) at ../accel/tcg/cpu-exec.c:1072
#2   accel_cpu_common_realize (cpu=0x55555595bf10, errp=0x7fffffffcd78) at ../accel/accel-target.c:135
#3   cpu_exec_realizefn (cpu=0x55555595bf10, errp=0x7fffffffcd78) at ../cpu-target.c:137
#4   arm_cpu_realizefn (dev=0x55555595bf10, errp=0x7fffffffcef0) at ../target/arm/cpu.c:1906
#5   device_set_realized (obj=0x55555595bf10, value=true, errp=0x7fffffffd128) at ../hw/core/qdev.c:510
#6   property_set_bool (obj=0x55555595bf10, v=0x55555595a310, name=0x555555842741 "realized", opaque=0x555555957b40, errp=0x7fffffffd128) at ../qom/object.c:2358
#7   object_property_set (obj=0x55555595bf10, name=0x555555842741 "realized", v=0x55555595a310, errp=0x7fffffffd128) at ../qom/object.c:1472
#8   object_property_set_qobject (obj=0x55555595bf10, name=0x555555842741 "realized", value=0x555555960750, errp=0x7fffffffd128) at ../qom/qom-qobject.c:28
#9   object_property_set_bool (obj=0x55555595bf10, name=0x555555842741 "realized", value=true, errp=0x7fffffffd128) at ../qom/object.c:1541
#10  qdev_realize (dev=0x55555595bf10, bus=0x0, errp=0x7fffffffd128) at ../hw/core/qdev.c:292
#11  cpu_create (typename=0x555555952d70 "max-arm-cpu") at ../hw/core/cpu-common.c:58
#12  main (argc=2, argv=0x7fffffffd8a8, envp=0x7fffffffd8c0) at ../linux-user/main.c:810
```

### Temp allocation happens after Global allocations 
- `tcg_global_alloc` has a checker whether `(tcg_debug_assert(s->nb_globals == s->nb_temps);)`.
- `tcg_temp_alloc` in `tcg_temp_new_internal` and `tcg_constant_internal` and `liveness_pass_2` (convert indirect temporaries to direct temps)
- temps/const allocation happens during gen_intermediate_code when there are ops that need new tmps/const.

- When there is a `tcg_func_start`, `nb_temps` is inited as `nb_globals`.

# Base Allocation: `g_malloc` --> `tcg_malloc_internal` --> `tcg_malloc`

- `tcg_malloc` is used during allocations: 
  - 1. allocation for instructions in a translation block, 
  - 2. allocation for tcg ops. (op's args)
  - 3. allocation for labels: `gen_new_label`
  - 4. Other spaces.
```c
/* user-mode: Called with mmap_lock held.  */
static inline void *tcg_malloc(int size)
{
    TCGContext *s = tcg_ctx;
    uint8_t *ptr, *ptr_end;

    /* ??? This is a weak placeholder for minimum malloc alignment.  */
    size = QEMU_ALIGN_UP(size, 8);

    ptr = s->pool_cur;
    ptr_end = ptr + size;
    if (unlikely(ptr_end > s->pool_end)) {
        return tcg_malloc_internal(tcg_ctx, size);
    } else {
        s->pool_cur = ptr_end;
        return ptr;
    }
}
```
- `tcg_malloc_internal`: pool based memory allocation
```c
/* pool based memory allocation */
void *tcg_malloc_internal(TCGContext *s, int size)
{
    TCGPool *p;
    int pool_size;

    if (size > TCG_POOL_CHUNK_SIZE) {
        /* big malloc: insert a new pool (XXX: could optimize) */
        p = g_malloc(sizeof(TCGPool) + size);
        p->size = size;
        p->next = s->pool_first_large;
        s->pool_first_large = p;
        return p->data;
    } else {
        p = s->pool_current;
        if (!p) {
            p = s->pool_first;
            if (!p)
                goto new_pool;
        } else {
            if (!p->next) {
            new_pool:
                pool_size = TCG_POOL_CHUNK_SIZE;
                p = g_malloc(sizeof(TCGPool) + pool_size);
                p->size = pool_size;
                p->next = NULL;
                if (s->pool_current) {
                    s->pool_current->next = p;
                } else {
                    s->pool_first = p;
                }
            } else {
                p = p->next;
            }
        }
    }
    s->pool_current = p;
    s->pool_cur = p->data + size;
    s->pool_end = p->data + p->size;
    return p->data;
}
```
# Other Structures in `TCGContext`

- `GHashTable *const_table[TCG_TYPE_COUNT];`
  - 1. Initialized in `tcg_func_start`
  - 2. Allocated in `tcg_constant_internal`: if exists, then look up the `const_table`, otherwise, gen a hash for the new const.
- `TCGTempSet free_temps[TCG_TYPE_COUNT];`
  - `tcg_temp_new_internal`: Clear the bit. There is a special case: when need to allocate a new one, the index is picked from the first bit that is free, and returned the one in `tcgcontext.temps` array.
  - `tcg_temp_free_internal`: Set bit if the temp is free, index in `temps` is mapped to index in `free_temps`
- `TCGTemp temps[TCG_MAX_TEMPS];` 
  - In this array, we can know the index between [`s->nb_globals`, `s->nb_temps`] should be type in EBB, TB, CONST.

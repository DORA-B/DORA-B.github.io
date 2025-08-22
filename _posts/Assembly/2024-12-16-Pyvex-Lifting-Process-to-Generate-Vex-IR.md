---
title: Pyvex library's lifting process to generate the Vex IR
date: 2024-12-16
categories: [Assembly]
tags: [assembly,python,angr]     # TAG names should always be lowercase
published: false
---

[TOC]

# Problem description

We all know that if perform this process, we can get different optimized levels of the the VEX IR that enables us to further analyze different ISAs.

```py
guestblock = project.factory.block(0, opt_level=1, cross_insn_opt=False)
irsb = guestblock.vex
print(f"{irsb}")
for stmt in irsb.statements:
    # capture and encoding the statements into the RDG
    if stmt.tag == "Ist_WrTmp":
        print(f"{stmt}")
```
But We need to know how this process is performed, and how the statements are generated, and the basic data structure of `stmt`.

# Basic `Block` class and the lifting process

```py
class Block(Serializable):
    """
    Represents a basic block in a binary or a program.
    """

    BLOCK_MAX_SIZE = 4096

    __slots__ = [
        "_project",
        "_bytes",
        "_vex",
        "thumb",
        "_disassembly",
        "_capstone",
        "addr",
        "size",
        "arch",
        "_instructions",
        "_instruction_addrs",
        "_opt_level",
        "_vex_nostmt",
        "_collect_data_refs",
        "_strict_block_end",
        "_cross_insn_opt",
        "_load_from_ro_regions",
        "_initial_regs",
    ]

    def __init__(
        self,
        addr,
        project=None,
        arch=None,
        size=None,
        max_size=None,
        byte_string=None,
        vex=None,
        thumb=False,
        backup_state=None,
        extra_stop_points=None,
        opt_level=None,
        num_inst=None,
        traceflags=0,
        strict_block_end=None,
        collect_data_refs=False,
        cross_insn_opt=True,
        load_from_ro_regions=False,
        initial_regs=None,
        skip_stmts=False,
    ):
        # set up arch
        if project is not None:
            self.arch = project.arch
        else:
            self.arch = arch

        if self.arch is None:
            raise ValueError('Either "project" or "arch" has to be specified.')

        if project is not None and backup_state is None and project.kb.patches.values():
            backup_state = project.kb.patches.patched_entry_state

        if isinstance(self.arch, ArchARM):
            if addr & 1 == 1:
                thumb = True
            elif thumb:
                addr |= 1
        else:
            thumb = False

        self._project = project
        self.thumb = thumb
        self.addr = addr
        self._opt_level = opt_level
        self._initial_regs: list[tuple[int, int, int]] | None = initial_regs if collect_data_refs else None

        if self._project is None and byte_string is None:
            raise ValueError('"byte_string" has to be specified if "project" is not provided.')

        if size is None:
            if byte_string is not None:
                size = len(byte_string)
            elif vex is not None:
                size = vex.size
            else:
                if self._initial_regs:
                    self.set_initial_regs()
                vex = self._vex_engine.lift_vex(   # NOTE: Enter in this lifting process
                    clemory=project.loader.memory,
                    state=backup_state,
                    insn_bytes=byte_string,
                    addr=addr,
                    size=max_size,
                    thumb=thumb,
                    extra_stop_points=extra_stop_points,
                    opt_level=opt_level,
                    num_inst=num_inst,
                    traceflags=traceflags,
                    strict_block_end=strict_block_end,
                    collect_data_refs=collect_data_refs,
                    load_from_ro_regions=load_from_ro_regions,
                    cross_insn_opt=cross_insn_opt,
                    skip_stmts=skip_stmts,
                )
                if self._initial_regs:
                    self.reset_initial_regs()
                size = vex.size

        if skip_stmts:
            self._vex = None
            self._vex_nostmt = vex
        else:
            self._vex = vex
            self._vex_nostmt = None
        self._disassembly = None
        self._capstone = None
        self.size = size
        self._collect_data_refs = collect_data_refs
        self._strict_block_end = strict_block_end
        self._cross_insn_opt = cross_insn_opt
        self._load_from_ro_regions = load_from_ro_regions

        self._instructions = num_inst
        self._instruction_addrs: list[int] = []

        if skip_stmts:
            self._parse_vex_info(self._vex_nostmt)
        else:
            self._parse_vex_info(self._vex)
```

## `VEXLifter` class --> `lift_vex` function

`/site-packages/angr/engines/vex/lifter.py`: `lift_vex` function is the main process of pre-setting and lifting.

```py
def lift_vex(
    self,
    addr=None,
    state=None,
    clemory=None,
    insn_bytes=None,
    offset=None,
    arch=None,
    size=None,
    num_inst=None,
    traceflags=0,
    thumb=False,
    extra_stop_points=None,
    opt_level=None,
    strict_block_end=None,
    skip_stmts=False,
    collect_data_refs=False,
    cross_insn_opt=None,
    load_from_ro_regions=False,
):
    """
    Lift an IRSB.

    There are many possible valid sets of parameters. You at the very least must pass some
    source of data, some source of an architecture, and some source of an address.

    Sources of data in order of priority: insn_bytes, clemory, state

    Sources of an address, in order of priority: addr, state

    Sources of an architecture, in order of priority: arch, clemory, state

    :param state:           A state to use as a data source.
    :param clemory:         A cle.memory.Clemory object to use as a data source.
    :param addr:            The address at which to start the block.
    :param thumb:           Whether the block should be lifted in ARM's THUMB mode.
    :param opt_level:       The VEX optimization level to use. The final IR optimization level is determined by
                            (ordered by priority):
                            - Argument opt_level
                            - opt_level is set to 1 if OPTIMIZE_IR exists in state options
                            - self._default_opt_level is 1
    :param insn_bytes:      A string of bytes to use as a data source.
    :param offset:          If using insn_bytes, the number of bytes in it to skip over.
    :param size:            The maximum size of the block, in bytes.
    :param num_inst:        The maximum number of instructions.
    :param traceflags:      traceflags to be passed to VEX. (default: 0)
    :param strict_block_end:   Whether to force blocks to end at all conditional branches (default: false)
    """
    # phase 0: sanity check ...
    # phase 1: parameter defaults ... for cross_insn_opt, skip_stmts, opt_level, ...
    # phase 2: thumb normalization
    # phase 3: check cache ....
    # call pyvex.lift
    try:
        for subphase in range(2):
            irsb = pyvex.lift(
                buff,
                addr + thumb,
                arch,
                max_bytes=size,
                max_inst=num_inst,
                bytes_offset=offset + thumb,
                traceflags=traceflags,
                opt_level=opt_level,
                strict_block_end=strict_block_end,
                skip_stmts=skip_stmts,
                collect_data_refs=collect_data_refs,
                load_from_ro_regions=load_from_ro_regions,
                cross_insn_opt=cross_insn_opt,
            )

            if subphase == 0 and irsb.statements is not None:
                # check for possible stop points
                stop_point = self._first_stoppoint(irsb, extra_stop_points)
                if stop_point is not None:
                    size = stop_point - addr
                    continue

            if use_cache:
                self._block_cache[cache_key] = irsb
            if state:
                state._inspect("vex_lift", BP_AFTER, vex_lift_addr=addr, vex_lift_size=size)
            return irsb    
```

## pyvex's lifting process

+ `/site-packages/pyvex/lifting/lift_function.py`

```py

def lift(
    data: LiftSource,
    addr,
    arch,
    max_bytes=None,
    max_inst=None,
    bytes_offset=0,
    opt_level=1,
    traceflags=0,
    strict_block_end=True,
    inner=False,
    skip_stmts=False,
    collect_data_refs=False,
    cross_insn_opt=True,
    load_from_ro_regions=False,
):
    """
    Recursively lifts blocks using the registered lifters and postprocessors. Tries each lifter in the order in
    which they are registered on the data to lift.

    If a lifter raises a LiftingException on the data, it is skipped.
    If it succeeds and returns a block with a jumpkind of Ijk_NoDecode, all of the lifters are tried on the rest
    of the data and if they work, their output is appended to the first block.

    :param arch:            The arch to lift the data as.
    :param addr:            The starting address of the block. Effects the IMarks.
    :param data:            The bytes to lift as either a python string of bytes or a cffi buffer object.
    :param max_bytes:       The maximum number of bytes to lift. If set to None, no byte limit is used.
    :param max_inst:        The maximum number of instructions to lift. If set to None, no instruction limit is used.
    :param bytes_offset:    The offset into `data` to start lifting at.
    :param opt_level:       The level of optimization to apply to the IR, -1 through 2. -1 is the strictest
                            unoptimized level, 0 is unoptimized but will perform some lookahead/lookbehind
                            optimizations, 1 performs constant propogation, and 2 performs loop unrolling,
                            which honestly doesn't make much sense in the context of pyvex. The default is 1.
    :param traceflags:      The libVEX traceflags, controlling VEX debug prints.

    .. note:: Explicitly specifying the number of instructions to lift (`max_inst`) may not always work
              exactly as expected. For example, on MIPS, it is meaningless to lift a branch or jump
              instruction without its delay slot. VEX attempts to Do The Right Thing by possibly decoding
              fewer instructions than requested. Specifically, this means that lifting a branch or jump
              on MIPS as a single instruction (`max_inst=1`) will result in an empty IRSB, and subsequent
              attempts to run this block will raise `SimIRSBError('Empty IRSB passed to SimIRSB.')`.

    .. note:: If no instruction and byte limit is used, pyvex will continue lifting the block until the block
              ends properly or until it runs out of data to lift.
    """

    # ignore the previous code
    # In order to attempt to preserve the property that
    # VEX lifts the same bytes to the same IR at all times when optimizations are disabled
    # we hack off all of VEX's non-IROpt optimizations when opt_level == -1.
    # This is intended to enable comparisons of the lifted IR between code that happens to be
    # found in different contexts.
    if opt_level < 0:
        allow_arch_optimizations = False
        opt_level = 0

    for lifter in lifters[arch.name]:
        try:
            u_data: LiftSource = data
            if lifter.REQUIRE_DATA_C:
                if c_data is None:
                    assert py_data is not None
                    if isinstance(py_data, (bytearray, memoryview)):
                        u_data = ffi.from_buffer(ffi.BVoidP, py_data)
                    else:
                        u_data = ffi.from_buffer(ffi.BVoidP, py_data + b"\0" * 8)
                    max_bytes = min(len(py_data), max_bytes) if max_bytes is not None else len(py_data)
                else:
                    u_data = c_data
                skip = 0
            elif lifter.REQUIRE_DATA_PY:
                if bytes_offset and arch.name.startswith("ARM") and (addr & 1) == 1:
                    skip = bytes_offset - 1
                else:
                    skip = bytes_offset
                if py_data is None:
                    assert c_data is not None
                    if max_bytes is None:
                        log.debug("Cannot create py_data from c_data when no max length is given")
                        continue
                    u_data = ffi.buffer(c_data + skip, max_bytes)[:]
                else:
                    if max_bytes is None:
                        u_data = py_data[skip:]
                    else:
                        u_data = py_data[skip : skip + max_bytes]
            else:
                raise RuntimeError(
                    "Incorrect lifter configuration. What type of data does %s expect?" % lifter.__class__
                )

            try:
                final_irsb = lifter(arch, addr).lift(
                    u_data,
                    bytes_offset - skip,
                    max_bytes,
                    max_inst,
                    opt_level,
                    traceflags,
                    allow_arch_optimizations,
                    strict_block_end,
                    skip_stmts,
                    collect_data_refs=collect_data_refs,
                    cross_insn_opt=cross_insn_opt,
                    load_from_ro_regions=load_from_ro_regions,
                )
```

+ `/site-packages/pyvex/lifting/lifter.py`: there is a lifter function here that call the libvex to get all the vex ir

```
def lift(
    self,
    data: LiftSource,
    bytes_offset: int | None = None,
    max_bytes: int | None = None,
    max_inst: int | None = None,
    opt_level: int | float = 1,
    traceflags: int | None = None,
    allow_arch_optimizations: bool | None = None,
    strict_block_end: bool | None = None,
    skip_stmts: bool = False,
    collect_data_refs: bool = False,
    cross_insn_opt: bool = True,
    load_from_ro_regions: bool = False,
    disasm: bool = False,
    dump_irsb: bool = False,
):
    """
    Wrapper around the `_lift` method on Lifters. Should not be overridden in child classes.

    :param data:                The bytes to lift as either a python string of bytes or a cffi buffer object.
    :param bytes_offset:        The offset into `data` to start lifting at.
    :param max_bytes:           The maximum number of bytes to lift. If set to None, no byte limit is used.
    :param max_inst:            The maximum number of instructions to lift. If set to None, no instruction limit is
                                used.
    :param opt_level:           The level of optimization to apply to the IR, 0-2. Most likely will be ignored in
                                any lifter other then LibVEX.
    :param traceflags:          The libVEX traceflags, controlling VEX debug prints. Most likely will be ignored in
                                any lifter other than LibVEX.
    :param allow_arch_optimizations:   Should the LibVEX lifter be allowed to perform lift-time preprocessing
                                optimizations (e.g., lookback ITSTATE optimization on THUMB) Most likely will be
                                ignored in any lifter other than LibVEX.
    :param strict_block_end:    Should the LibVEX arm-thumb split block at some instructions, for example CB{N}Z.
    :param skip_stmts:          Should the lifter skip transferring IRStmts from C to Python.
    :param collect_data_refs:   Should the LibVEX lifter collect data references in C.
    :param cross_insn_opt:      If cross-instruction-boundary optimizations are allowed or not.
    :param disasm:              Should the GymratLifter generate disassembly during lifting.
    :param dump_irsb:           Should the GymratLifter log the lifted IRSB.
    """
    irsb: IRSB = IRSB.empty_block(self.arch, self.addr)
    self.data = data
    self.bytes_offset = bytes_offset
    self.opt_level = opt_level
    self.traceflags = traceflags
    self.allow_arch_optimizations = allow_arch_optimizations
    self.strict_block_end = strict_block_end
    self.collect_data_refs = collect_data_refs
    self.max_inst = max_inst
    self.max_bytes = max_bytes
    self.skip_stmts = skip_stmts
    self.irsb = irsb
    self.cross_insn_opt = cross_insn_opt
    self.load_from_ro_regions = load_from_ro_regions
    self.disasm = disasm
    self.dump_irsb = dump_irsb
    self._lift()                # NOTE: in libvex.py and 
    return self.irsb
```

+ `/site-packages/pyvex/lifting/libvex.py`: call to the c lib to get the lift of ISA

```py
class LibVEXLifter(Lifter):
    __slots__ = ()

    REQUIRE_DATA_C = True

    @staticmethod
    def get_vex_log():
        return bytes(ffi.buffer(pvc.msg_buffer, pvc.msg_current_size)).decode() if pvc.msg_buffer != ffi.NULL else None

    def _lift(self):
        if TYPE_CHECKING:
            assert isinstance(self.irsb.arch, LibvexArch)
            assert isinstance(self.data, CLiftSource)
        try:
            _libvex_lock.acquire()

            pvc.log_level = log.getEffectiveLevel()
            vex_arch = getattr(pvc, self.irsb.arch.vex_arch, None)
            assert vex_arch is not None

            if self.bytes_offset is None:
                self.bytes_offset = 0

            if self.max_bytes is None or self.max_bytes > VEX_MAX_BYTES:
                max_bytes = VEX_MAX_BYTES
            else:
                max_bytes = self.max_bytes

            if self.max_inst is None or self.max_inst > VEX_MAX_INSTRUCTIONS:
                max_inst = VEX_MAX_INSTRUCTIONS
            else:
                max_inst = self.max_inst

            strict_block_end = self.strict_block_end
            if strict_block_end is None:
                strict_block_end = True

            collect_data_refs = 1 if self.collect_data_refs else 0
            if collect_data_refs != 0 and self.load_from_ro_regions:
                collect_data_refs |= 2  # the second bit stores load_from_ro_regions

            if self.cross_insn_opt:
                px_control = VexRegisterUpdates.VexRegUpdUnwindregsAtMemAccess
            else:
                px_control = VexRegisterUpdates.VexRegUpdLdAllregsAtEachInsn

            self.irsb.arch.vex_archinfo["hwcache_info"]["caches"] = ffi.NULL
            lift_r = pvc.vex_lift(
                vex_arch,
                self.irsb.arch.vex_archinfo,
                self.data + self.bytes_offset,
                self.irsb.addr,
                max_inst,
                max_bytes,
                self.opt_level,
                self.traceflags,
                self.allow_arch_optimizations,
                strict_block_end,
                collect_data_refs,
                px_control,
                self.bytes_offset,
            )
```

+ `/site-packages/pyvex/lifting/libvex.py`: call to the c lib to get the lift of ISA
```py
class LibVEXLifter(Lifter):
    __slots__ = ()

    REQUIRE_DATA_C = True

    @staticmethod
    def get_vex_log():
        return bytes(ffi.buffer(pvc.msg_buffer, pvc.msg_current_size)).decode() if pvc.msg_buffer != ffi.NULL else None

    def _lift(self):
        if TYPE_CHECKING:
            assert isinstance(self.irsb.arch, LibvexArch)
            assert isinstance(self.data, CLiftSource)
        try:
            _libvex_lock.acquire()

            pvc.log_level = log.getEffectiveLevel()
            vex_arch = getattr(pvc, self.irsb.arch.vex_arch, None)
            assert vex_arch is not None

            if self.bytes_offset is None:
                self.bytes_offset = 0

            if self.max_bytes is None or self.max_bytes > VEX_MAX_BYTES:
                max_bytes = VEX_MAX_BYTES
            else:
                max_bytes = self.max_bytes

            if self.max_inst is None or self.max_inst > VEX_MAX_INSTRUCTIONS:
                max_inst = VEX_MAX_INSTRUCTIONS
            else:
                max_inst = self.max_inst

            strict_block_end = self.strict_block_end
            if strict_block_end is None:
                strict_block_end = True

            collect_data_refs = 1 if self.collect_data_refs else 0
            if collect_data_refs != 0 and self.load_from_ro_regions:
                collect_data_refs |= 2  # the second bit stores load_from_ro_regions

            if self.cross_insn_opt:
                px_control = VexRegisterUpdates.VexRegUpdUnwindregsAtMemAccess
            else:
                px_control = VexRegisterUpdates.VexRegUpdLdAllregsAtEachInsn

            self.irsb.arch.vex_archinfo["hwcache_info"]["caches"] = ffi.NULL
            lift_r = pvc.vex_lift(
                vex_arch,
                self.irsb.arch.vex_archinfo,
                self.data + self.bytes_offset,
                self.irsb.addr,
                max_inst,
                max_bytes,
                self.opt_level,
                self.traceflags,
                self.allow_arch_optimizations,
                strict_block_end,
                collect_data_refs,
                px_control,
                self.bytes_offset,
            )
            log_str = self.get_vex_log()
            if lift_r == ffi.NULL:
                raise LiftingException("libvex: unknown error" if log_str is None else log_str)
            else:
                if log_str is not None:
                    log.debug(log_str)

            self.irsb._from_c(lift_r, skip_stmts=self.skip_stmts) # convert c vex to pyvex
```

- After get the lifted ir from lib c, the structure is like:
![lib c irsb structure](/commons/images/asm/clib_pyvex.png)
- We can see here about the convertion, every `c_stmt` has a tag that is mapped to a `stmt_class` in pyvex 
![c vex to pyvex](/commons/images/asm/cvex_to_pyvex.png)

# Core Classes

## `IRStmt` IR statements

```py
class IRStmt(VEXObject):
    """
    IR statements in VEX represents operations with side-effects.
    """

    tag: str | None = None
    tag_int = 0  # set automatically at bottom of file
    
    # ... ignore codes

    @staticmethod
    def _from_c(c_stmt):
        if c_stmt[0] == ffi.NULL:
            return None

        try:
            stmt_class = enum_to_stmt_class(c_stmt.tag)
        except KeyError:
            raise PyVEXError("Unknown/unsupported IRStmtTag %s.\n" % get_enum_from_int(c_stmt.tag))
        return stmt_class._from_c(c_stmt)
```
## Different Stmt Operations

```py
class NoOp(IRStmt):
    """
    A no-operation statement. It is usually the result of an IR optimization.
    """
class IMark(IRStmt):
    """
    An instruction mark. It marks the start of the statements that represent a single machine instruction (the end of
    those statements is marked by the next IMark or the end of the IRSB).  Contains the address and length of the
    instruction.
    """

    __slots__ = ["addr", "len", "delta"]

    tag = "Ist_IMark"
class AbiHint(IRStmt):
    """
    An ABI hint, provides specific information about this platform's ABI.
    """
class Put(IRStmt):
    """
    Write to a guest register, at a fixed offset in the guest state.
    """
class PutI(IRStmt):
    """
    Write to a guest register, at a non-fixed offset in the guest state.
    """
class WrTmp(IRStmt):
    """
    Assign a value to a temporary.  Note that SSA rules require each tmp is only assigned to once.  IR sanity checking
    will reject any block containing a temporary which is not assigned to exactly once.
    """
class Store(IRStmt):
    """
    Write a value to memory..
    """
class CAS(IRStmt):
    """
    an atomic compare-and-swap operation.
    """
class LLSC(IRStmt):
    """
    Either Load-Linked or Store-Conditional, depending on STOREDATA. If STOREDATA is NULL then this is a Load-Linked,
    else it is a Store-Conditional.
    """
class MBE(IRStmt):
    __slots__ = ["event"]

    tag = "Ist_MBE"
class Dirty(IRStmt):
    __slots__ = ["cee", "guard", "args", "tmp", "mFx", "mAddr", "mSize", "nFxState"]

    tag = "Ist_Dirty"
class Exit(IRStmt):
    """
    A conditional exit from the middle of an IRSB.
    """
class LoadG(IRStmt):
    """
    A guarded load.
    """
class StoreG(IRStmt):
    """
    A guarded store.
    """
```

##  `IRExpr` IR Expressions

```py

class IRExpr(VEXObject):
    """
    IR expressions in VEX represent operations without side effects.
    """

    __slots__ = []

    tag: str | None = None
    tag_int = 0  # set automatically at bottom of file

    def pp(self):
        print(str(self))

    def __str__(self):
        return self._pp_str()

    def _pp_str(self) -> str:
        raise NotImplementedError

    @property
    def child_expressions(self) -> list["IRExpr"]:
        """
        A list of all of the expressions that this expression ends up evaluating.
        """
        expressions = []
        for k in self.__slots__:
            v = getattr(self, k)
            if isinstance(v, IRExpr):
                expressions.append(v)
                expressions.extend(v.child_expressions)
        return expressions

    @property
    def constants(self):
        """
        A list of all of the constants that this expression ends up using.
        """
        constants = []
        for k in self.__slots__:
            v = getattr(self, k)
            if isinstance(v, IRExpr):
                constants.extend(v.constants)
            elif isinstance(v, IRConst):
                constants.append(v)
        return constants

    def result_size(self, tyenv):
        return get_type_size(self.result_type(tyenv))

    def result_type(self, tyenv):
        raise NotImplementedError()

    def replace_expression(self, replacements):
        """
        Replace child expressions in-place.

        :param Dict[IRExpr, IRExpr] replacements:  A mapping from expression-to-find to expression-to-replace-with
        :return:                    None
        """

        for k in self.__slots__:
            v = getattr(self, k)
            if isinstance(v, IRExpr) and v in replacements:
                setattr(self, k, replacements.get(v))
            elif isinstance(v, list):
                # Replace the instance in the list
                for i, expr_ in enumerate(v):
                    if isinstance(expr_, IRExpr) and expr_ in replacements:
                        v[i] = replacements.get(expr_)
            elif type(v) is tuple:
                # Rebuild the tuple
                _lst = []
                replaced = False
                for i, expr_ in enumerate(v):
                    if isinstance(expr_, IRExpr) and expr_ in replacements:
                        _lst.append(replacements.get(expr_))
                        replaced = True
                    else:
                        _lst.append(expr_)
                if replaced:
                    setattr(self, k, tuple(_lst))
            elif isinstance(v, IRExpr):
                v.replace_expression(replacements)

    @staticmethod
    def _from_c(c_expr) -> Optional["IRExpr"]:
        if c_expr == ffi.NULL or c_expr[0] == ffi.NULL:
            return None

        try:
            return enum_to_expr_class(c_expr.tag)._from_c(c_expr)
        except KeyError:
            raise PyVEXError("Unknown/unsupported IRExprTag %s\n" % get_enum_from_int(c_expr.tag))

```

## Different Expression Operations

```py
class Binder(IRExpr):
    """
    Used only in pattern matching within Vex. Should not be seen outside of Vex.
    """

class VECRET(IRExpr):
    tag = "Iex_VECRET"

class GSPTR(IRExpr):
    __slots__ = []

class GetI(IRExpr):
    """
    Read a guest register at a non-fixed offset in the guest state.
    """

class RdTmp(IRExpr):
    """
    Read the value held by a temporary.
    """

    __slots__ = ["_tmp"]

    tag = "Iex_RdTmp"

class Get(IRExpr):
    """
    Read a guest register, at a fixed offset in the guest state.
    """

class Qop(IRExpr):
    """
    A quaternary operation (4 arguments).
    """

    __slots__ = ["op", "args"]

    tag = "Iex_Qop"


class Triop(IRExpr):
    """
    A ternary operation (3 arguments)
    """

class Binop(IRExpr):
    """
    A binary operation (2 arguments).
    """

    __slots__ = ["_op", "op_int", "args"]

    tag = "Iex_Binop"

    def __init__(self, op, args, op_int=None):
        self.op_int = op_int
        self.args = args
        self._op = op if op is not None else None

    def _pp_str(self):
        return "{}({})".format(self.op[4:], ",".join(str(a) for a in self.args))

    @property
    def op(self):
        if self._op is None:
            self._op = get_enum_from_int(self.op_int)
        return self._op

    @property
    def child_expressions(self):
        expressions = sum((a.child_expressions for a in self.args), [])
        expressions.extend(self.args)
        return expressions

    @staticmethod
    def _from_c(c_expr):
        return Binop(
            None,
            [IRExpr._from_c(arg) for arg in [c_expr.Iex.Binop.arg1, c_expr.Iex.Binop.arg2]],
            op_int=c_expr.Iex.Binop.op,
        )

class Unop(IRExpr):
    """
    A unary operation (1 argument).
    """

    __slots__ = ["op", "args"]

    tag = "Iex_Unop"

class Load(IRExpr):
    """
    A load from memory.
    """

class Const(IRExpr):
    """
    A constant expression.
    """
class ITE(IRExpr):
    """
    An if-then-else expression.
    """

class CCall(IRExpr):
    """
    A call to a pure (no side-effects) helper C function.
    """
```

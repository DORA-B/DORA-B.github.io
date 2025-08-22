---
title: Reaching and Definitions Analysis Application
date: 2024-12-15
categories: [Assembly]
tags: [assembly,python,angr]     # TAG names should always be lowercase
published: false
---

[TOC]

# Basic Classes

## `Definition` class
Every node in the dep_graph is a `Definition` class: `/angr/knowledge_plugins/key_definitions/definition.py`

```py
class Definition(Generic[A]):
    """
    An atom definition.

    :ivar atom:     The atom being defined.
    :ivar codeloc:  Where this definition is created in the original binary code.
    :ivar dummy:    Tell whether the definition should be considered dummy or not. During simplification by AILment,
                    definitions marked as dummy will not be removed.
    :ivar tags:     A set of tags containing information about the definition gathered during analyses.
    """

    __slots__ = (
        "atom",
        "codeloc",
        "dummy",
        "tags",
        "_hash",
    )

    def __init__(self, atom: A, codeloc: CodeLocation, dummy: bool = False, tags: set[Tag] | None = None):
        self.atom: A = atom
        self.codeloc: CodeLocation = codeloc
        self.dummy: bool = dummy
        self.tags = tags or set()
```

## `LiveDefinitions` class

###  what are the Definitions in a depedency graph? 
When add new definitions into the graph, which come from `kill_and_add_definition` in class `SimEngineRDVEX`, `atom` may come from `Register`, `MemoryLocation`, or `Tmp`

```py

class LiveDefinitions:
    """
    A LiveDefinitions instance contains definitions and uses for register, stack, memory, and temporary variables,
    uncovered during the analysis.
    """
    # Methods ....

 def get_definitions(
     self, thing: A | Definition[A] | Iterable[A] | Iterable[Definition[A]] | MultiValues
 ) -> set[Definition[Atom]]:
     if isinstance(thing, MultiValues):
         defs = set()
         for vs in thing.values():
             for v in vs:
                 defs.update(LiveDefinitions.extract_defs_from_annotations(v.annotations))
         return defs
     elif isinstance(thing, Atom):
         pass
     elif isinstance(thing, Definition):
         thing = thing.atom
     else:
         defs = set()
         for atom2 in thing:
             defs |= self.get_definitions(atom2)
         return defs

    # Here can be seen that the type of Definitions in a graph is basically `Register`, `MemoryLocation`, or `Tmp`
     if isinstance(thing, Register):
         return self.get_register_definitions(thing.reg_offset, thing.size)
     elif isinstance(thing, MemoryLocation):
         if isinstance(thing.addr, SpOffset):
             return self.get_stack_definitions(thing.addr.offset, thing.size)
         elif isinstance(thing.addr, HeapAddress):
             return self.get_heap_definitions(thing.addr.value, size=thing.size)
         elif isinstance(thing.addr, int):
             return self.get_memory_definitions(thing.addr, thing.size)
         else:
             return set()
     elif isinstance(thing, Tmp):
         return self.get_tmp_definitions(thing.tmp_idx)
     else:
         defs = set()
         mv = self.others.get(thing, None)
         if mv is not None:
             defs |= self.get_definitions(mv)
         return defs
```

## `SimEngineRDVEX` class
###  The process that handles `stmt` and creates atoms

```py
class SimEngineRDVEX(
    SimEngineLightVEXMixin,
    SimEngineLight,
):  # pylint:disable=abstract-method
    """
    Implements the VEX execution engine for reaching definition analysis.
    """

    def _handle_WrTmp(self, stmt: pyvex.IRStmt.WrTmp):
        data: MultiValues = self._expr(stmt.data)

        tmp_atom = Tmp(stmt.tmp, self.tyenv.sizeof(stmt.tmp) // self.arch.byte_width)
        # if len(data.values) == 1 and 0 in data.values:
        #     data_v = data.one_value()
        #     if data_v is not None:
        #         # annotate data with its definition
        #         data = MultiValues(offset_to_values={
        #             0: {self.state.annotate_with_def(data_v, Definition(tmp_atom, self._codeloc()))
        #                 }
        #         })
        self.tmps[stmt.tmp] = data

        self.state.kill_and_add_definition(
            tmp_atom,
            data,
        )

    def _handle_WrTmpData(self, tmp: int, data):
        super()._handle_WrTmpData(tmp, data)
        self.state.kill_and_add_definition(Tmp(tmp, self.tyenv.sizeof(tmp)), self.tmps[tmp])
        def _store_core(
        self,
        addr: Iterable[int | claripy.ast.bv.BV],
        size: int,
        data: MultiValues,
        data_old: MultiValues | None = None,
        endness=None,
    ):
        if data_old is not None:
            data = data.merge(data_old)

        for a in addr:
            if self.state.is_top(a):
                l.debug("Memory address undefined, ins_addr = %#x.", self.ins_addr)
            else:
                tags: set[Tag] | None
                if isinstance(a, int):
                    atom = MemoryLocation(a, size)
                    tags = None
                elif self.state.is_stack_address(a):
                    offset = self.state.get_stack_offset(a)
                    if offset is None:
                        continue
                    atom = MemoryLocation(SpOffset(self.arch.bits, offset), size)
                    function_address = None  # we cannot get the function address in the middle of a store if a CFG
                    # does not exist. you should backpatch the function address later using
                    # the 'ins_addr' metadata entry.
                    tags = {
                        LocalVariableTag(
                            function=function_address,
                            metadata={"tagged_by": "SimEngineRDVEX._store_core", "ins_addr": self.ins_addr},
                        )
                    }

                elif self.state.is_heap_address(a):
                    offset = self.state.get_heap_offset(a)
                    if offset is not None:
                        atom = MemoryLocation(HeapAddress(offset), size)
                        tags = None
                    else:
                        continue

                elif isinstance(a, claripy.ast.BV):
                    addr_v = a.concrete_value
                    atom = MemoryLocation(addr_v, size)
                    tags = None

                else:
                    continue

                # different addresses are not killed by a subsequent iteration, because kill only removes entries
                # with same index and same size
                self.state.kill_and_add_definition(atom, data, tags=tags, endness=endness)

    def _handle_LoadG(self, stmt):
        guard = self._expr(stmt.guard)
        guard_v = guard.one_value()

        if claripy.is_true(guard_v):
            # FIXME: full conversion support
            if stmt.cvt.find("Ident") < 0:
                l.warning("Unsupported conversion %s in LoadG.", stmt.cvt)
            load_expr = pyvex.expr.Load(stmt.end, stmt.cvt_types[1], stmt.addr)
            wr_tmp_stmt = pyvex.stmt.WrTmp(stmt.dst, load_expr)
            self._handle_WrTmp(wr_tmp_stmt)
        elif claripy.is_false(guard_v):
            wr_tmp_stmt = pyvex.stmt.WrTmp(stmt.dst, stmt.alt)
            self._handle_WrTmp(wr_tmp_stmt)
        else:
            if stmt.cvt.find("Ident") < 0:
                l.warning("Unsupported conversion %s in LoadG.", stmt.cvt)
            load_expr = pyvex.expr.Load(stmt.end, stmt.cvt_types[1], stmt.addr)

            load_expr_v = self._expr(load_expr)
            alt_v = self._expr(stmt.alt)

            data = load_expr_v.merge(alt_v)
            self._handle_WrTmpData(stmt.dst, data)
```

## `Uses` class

After main `RDA`, the definitions relationships are needed for later, and the mappings methods include `(location, definitions)` or `(definition, locations)`

### Demonstrates how a definitions is related to location

```py
class Uses:
    """
    Describes uses (including the use location and the use expression) for definitions.
    """

    __slots__ = ("_uses_by_definition", "_uses_by_location")

    def __init__(
        self,
        uses_by_definition: DefaultChainMapCOW | None = None,
        uses_by_location: DefaultChainMapCOW | None = None,
    ):
        self._uses_by_definition: DefaultChainMapCOW["Definition", set[tuple[CodeLocation, Any | None]]] = (
            DefaultChainMapCOW(default_factory=set, collapse_threshold=25)
            if uses_by_definition is None
            else uses_by_definition
        )
        self._uses_by_location: DefaultChainMapCOW[CodeLocation, set[tuple["Definition", Any | None]]] = (
            DefaultChainMapCOW(default_factory=set, collapse_threshold=25)
            if uses_by_location is None
            else uses_by_location
        )

    def add_use(self, definition: "Definition", codeloc: CodeLocation, expr: Any | None = None):
        """
        Add a use for a given definition.

        :param definition:  The definition that is used.
        :param codeloc:     The code location where the use occurs.
        :param expr:        The expression that uses the specified definition at this location.
        """
        self._uses_by_definition = self._uses_by_definition.clean()
        self._uses_by_definition[definition].add((codeloc, expr))
        self._uses_by_location = self._uses_by_location.clean()
        self._uses_by_location[codeloc].add((definition, expr))
```

## `ATOM` class

This class contains the data that used to construct the `Definition`, so the types of it is needed when doing analysis.

```py

class Atom:
    """
    This class represents a data storage location manipulated by IR instructions.

    It could either be a Tmp (temporary variable), a Register, a MemoryLocation.
    """

    __slots__ = ("_hash", "size")

    def __init__(self, size):
        """
        :param size:  The size of the atom in bytes
        """
        self.size = size
        self._hash = None

    @staticmethod
    def from_ail_expr(expr: ailment.Expr.Expression, arch: Arch, full_reg: bool = False) -> "Register":
        if isinstance(expr, ailment.Expr.Register):
            if full_reg:
                reg_name = arch.translate_register_name(expr.reg_offset)
                return Register(arch.registers[reg_name][0], arch.registers[reg_name][1], arch)
            else:
                return Register(expr.reg_offset, expr.size, arch)
        raise TypeError(f"Expression type {type(expr)} is not yet supported")

    @staticmethod
    def from_argument(
        argument: SimFunctionArgument, arch: Arch, full_reg=False, sp: int | None = None
    ) -> Union["Register", "MemoryLocation"]:
        """
        Instanciate an `Atom` from a given argument.

        :param argument: The argument to create a new atom from.
        :param arch: The argument representing archinfo architecture for argument.
        :param full_reg: Whether to return an atom indicating the entire register if the argument only specifies a
                        slice of the register.
        :param sp:      The current stack offset. Optional. Only used when argument is a SimStackArg.
        """
        if isinstance(argument, SimRegArg):
            if full_reg:
                return Register(arch.registers[argument.reg_name][0], arch.registers[argument.reg_name][1], arch)
            else:
                return Register(arch.registers[argument.reg_name][0] + argument.reg_offset, argument.size, arch)
        elif isinstance(argument, SimStackArg):
            if sp is None:
                raise ValueError("You must provide a stack pointer to translate a SimStackArg")
            return MemoryLocation(
                SpOffset(arch.bits, argument.stack_offset + sp), argument.size, endness=arch.memory_endness
            )
        else:
            raise TypeError("Argument type %s is not yet supported." % type(argument))

    @staticmethod
    def reg(thing: str | RegisterOffset, size: int | None = None, arch: Arch | None = None) -> "Register":
        """
        Create a Register atom.

        :param thing:   The register offset (e.g., project.arch.registers["rax"][0]) or the register name (e.g., "rax").
        :param size:    Size of the register atom. Must be provided when creating the atom using a register offset.
        :param arch:    The architecture. Must be provided when creating the atom using a register name.
        :return:        The Register Atom object.
        """

        if isinstance(thing, str):
            if arch is None:
                raise ValueError(
                    "Cannot create a Register Atom by register name without having an architecture "
                    "specified through arch!"
                )
            if thing not in arch.registers:
                raise ValueError(f"Unknown register name {thing} for architecture {arch.name}")
            reg_offset, size_ = arch.registers[thing]
            if size is None:
                size = size_
        elif isinstance(thing, RegisterOffset):
            reg_offset = thing
            if size is None:
                raise ValueError("You must provide a size when specifying the register offset")
        else:
            raise TypeError(
                "Unsupported type of register. It must be a string (for register name) or an int (for "
                "register offset)"
            )
        return Register(reg_offset, size, arch=arch)

    register = reg

    @staticmethod
    def mem(addr: SpOffset | HeapAddress | int, size: int, endness: str | None = None) -> "MemoryLocation":
        """
        Create a MemoryLocation atom,

        :param addr:        The memory location. Can be an SpOffset for stack variables, an int for global memory
                            variables, or a HeapAddress for items on the heap.
        :param size:        Size of the atom.
        :param endness:     Optional, either "Iend_LE" or "Iend_BE".
        :return:            The MemoryLocation Atom object.
        """
        return MemoryLocation(addr, size, endness=endness)

    memory = mem

class GuardUse(Atom):
    """
    Implements a guard use.
    """

    __slots__ = ("target",)

    def __init__(self, target):
        super().__init__(0)
        self.target = target

    def __repr__(self):
        return "<Guard %#x>" % self.target

    def _identity(self):
        return (self.target,)


class ConstantSrc(Atom):
    """
    Represents a constant.
    """

    __slots__ = ("value",)

    def __init__(self, value: int, size: int):
        super().__init__(size)
        self.value: int = value

    def __repr__(self):
        return f"<Const {self.value}>"

    def _identity(self):
        return (self.value, self.size)


class Tmp(Atom):
    """
    Represents a variable used by the IR to store intermediate values.
    """

    __slots__ = ("tmp_idx",)

    def __init__(self, tmp_idx: int, size: int):
        super().__init__(size)
        self.tmp_idx = tmp_idx

    def __repr__(self):
        return "<Tmp %d>" % self.tmp_idx

    def _identity(self):
        return hash(("tmp", self.tmp_idx))


class Register(Atom):
    """
    Represents a given CPU register.

    As an IR abstracts the CPU design to target different architectures, registers are represented as a separated memory
    space.
    Thus a register is defined by its offset from the base of this memory and its size.

    :ivar int reg_offset:    The offset from the base to define its place in the memory bloc.
    :ivar int size:          The size, in number of bytes.
    """

    __slots__ = (
        "reg_offset",
        "arch",
    )

    def __init__(self, reg_offset: RegisterOffset, size: int, arch: Arch | None = None):
        super().__init__(size)

        self.reg_offset = reg_offset
        self.arch = arch


class MemoryLocation(Atom):
    """
    Represents a memory slice.

    It is characterized by its address and its size.
    """

    __slots__ = (
        "addr",
        "endness",
    )

    def __init__(self, addr: SpOffset | HeapAddress | int, size: int, endness: str | None = None):
        """
        :param int addr: The address of the beginning memory location slice.
        :param int size: The size of the represented memory location, in bytes.
        """


atom_kind_mapping = {
    AtomKind.REGISTER: Register,
    AtomKind.MEMORY: MemoryLocation,
    AtomKind.TMP: Tmp,
    AtomKind.GUARD: GuardUse,
    AtomKind.CONSTANT: ConstantSrc,
}

```

# Issues

## Issue 1

1. There is no nodes' atom type `MemoryLocation` after RDA, after analysing the below block:

```assembly
mov r1, r5,
mov r2, #0,
ldrb r3, [r1, #1]!,
add r2, r2, #1,
cmp r3, #0,
cmpne r3, #0x3d,
mov r6, r2,
bne #0x2c3a8
```
2. So why there is no such kind of type for `r3`, even though its value from `[r1, #1]`?
```py
atom_kind_mapping = {
    AtomKind.REGISTER: Register,
    AtomKind.MEMORY: MemoryLocation,
    AtomKind.TMP: Tmp,
    AtomKind.GUARD: GuardUse,
    AtomKind.CONSTANT: ConstantSrc,
}
```

For RDA to show a `MemoryLocation` atom corresponding to `[r1, #1]`, it must be able to concretely identify and track that memory address. If the analysis cannot determine a stable, well-defined memory address (for example, `r1` might be unknown or symbolic), it will not represent that load as a `MemoryLocation` atom in the graph.

**Why the Memory Location Doesn’t Appear:**

1. **Dynamic or Unknown Address:** If `r1` holds a value that RDA cannot resolve to a concrete memory address, RDA cannot create a `MemoryLocation` atom. Without a known, static address, the memory access remains a dynamic calculation rather than a definable memory location.
    
2. **No Prior Definition of that Memory Region:** RDA tracks definitions. If the memory location `[r1 + 1]` was never "defined" in a way RDA can recognize (e.g., writing known data to a known memory address), then RDA won't produce a `MemoryLocation` definition for it. The load just appears as a computation resulting in a temporary and then into `r3`.
    
3. **Atom Kinds Are Limited:** As you noted, atoms are strictly defined as one of the five kinds. If the memory is not concretely identified, it won’t appear as a `MemoryLocation`. Instead, you’ll likely see that `r3` depends on a `Tmp` that corresponds to the load operation's result, but no explicit `MemoryLocation` node is created for `[r1, #1]`.
    

If the analysis can concretely determine a stable memory address or offset, it will produce `MemoryLocation` atoms in the dependency graph. Stack-based memory references (like those involved in `push` and `pop` instructions) often resolve to known offsets relative to the stack pointer. Since the stack pointer (`sp`) is typically well-defined at function boundaries (RDA knows the initial SP and how it changes), references to memory through `sp` can often be concretized into `MemoryLocation` atoms.



# Applications

## Related predecessors of defined register `r1`

```py
# pick the last definition by code location
final_def = sorted(candidates, key=lambda d: (d.codeloc.ins_addr or 0, d.codeloc.stmt_idx or 0))[-1]
all_predecessors = dep_graph.find_all_predecessors(final_def)
```

![all predecessors of r1](/commons/images/asm/findallpredecessors.png)


## transitive closure of defined register `r1`

```py
closure_graph = dep_graph.transitive_closure(final_def)
```
![transitive graph's nodes of r1](/commons/images/asm/transitive_closure_r1.png)

And the sub graph that only for transtive closure of `r1` is: 

![transtive graph of r1](/commons/images/asm/dep_graph_r1.png)

## get all the tmp used during the `_run_on_node` function

In the process of `_analyze` for rda, will combine all the tmps with `self.all_uses`

```py
def _run_on_node(self, node, state: ReachingDefinitionsState):
  """

  :param node:    The current node.
  :param state:   The analysis state.
  :return:        A tuple: (reached fix-point, successor state)
  """
  state = engine.process(
      state,
      block=block,
      fail_fast=self._fail_fast,
      visited_blocks=self._visited_blocks,
      dep_graph=self._dep_graph,
      model=self.model,
  )

  self._node_iterations[block_key] += 1

  self.node_observe(node.addr, state, OP_AFTER)

  # update all definitions and all uses
  self.all_definitions |= state.all_definitions
  for use in [state.stack_uses, state.heap_uses, state.register_uses, state.memory_uses]:
      self.all_uses.merge(use)

  if self._track_tmps: # Tmps will be merged into `all_uses`
      # merge tmp uses to all_uses
      for tmp_idx, locs in state.tmp_uses.items():
          tmp_def = next(iter(state.tmps[tmp_idx]))
          for loc in locs:
              self.all_uses.add_use(tmp_def, loc)

  # drop definitions and uses because we will not need them anymore
  state.downsize()
```
 

![combining tmps into all_uses](/commons/images/asm/tmp_uses_in_rda.png)

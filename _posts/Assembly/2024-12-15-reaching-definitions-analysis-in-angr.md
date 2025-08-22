---
title: Reaching and Definitions Analysis in angr
date: 2024-12-15
categories: [Assembly]
tags: [assembly,python,angr]     # TAG names should always be lowercase
published: false
---

[TOC]

# Describe of the problem

1. A problem when want to analyze and construct a AST tree for semantic matching
2. Want to trace the register and get the whole process to define it 
3. Originally, use angr's simbolic engine to construct it, but some values of registers during the angr's symbolic execution is simplified, even use some state's option `angr.options.SIMPLIFY_EXPRS`, which aims to eliminate the automatic simplification, the information gotten from this process is still not enough.
4. Need to combine the Vex IR and RDA to get a proper AST.

# Pre-Settings

The block that is used to be analysed is

```python
"mov r1, r5",
"mov r2, #0",
"ldrb r3, [r1, #1]!",
"add r2, r2, #1",
"cmp r3, #0",
"cmpne r3, #0x3d",
"mov r6, r2",
"bne #0x2c3a8"
```

## Construct a block as `subject` to do the RDA
This step is to use a binary assembly block as a parameter to initialize a subject to do the symbolic execution.
```py
class ReachingDefinitionsAnalysis(
    ForwardAnalysis[ReachingDefinitionsState, NodeType, object, object], Analysis
):  # pylint:disable=abstract-method
    """
    ReachingDefinitionsAnalysis is a text-book implementation of a static data-flow analysis that works on either a
    function or a block. It supports both VEX and AIL. By registering observers to observation points, users may use
    this analysis to generate use-def chains, def-use chains, and reaching definitions, and perform other traditional
    data-flow analyses such as liveness analysis.

    * I've always wanted to find a better name for this analysis. Now I gave up and decided to live with this name for
      the foreseeable future (until a better name is proposed by someone else).
    * Aliasing is definitely a problem, and I forgot how aliasing is resolved in this implementation. I'll leave this
      as a post-graduation TODO.
    * Some more documentation and examples would be nice.
    """

    def __init__(
        self,
        subject: Subject | ailment.Block | Block | Function | str = None,
        func_graph=None,
        max_iterations=30,
        track_tmps=False,
        track_consts=True,
        observation_points: "Iterable[ObservationPoint]" = None,
        init_state: ReachingDefinitionsState = None,
        init_context=None,
        state_initializer: Optional["RDAStateInitializer"] = None,
        cc=None,
        function_handler: "Optional[FunctionHandler]" = None,
        observe_all=False,
        visited_blocks=None,
        dep_graph: DepGraph | bool | None = True,
        observe_callback=None,
        canonical_size=8,
        stack_pointer_tracker=None,
        use_callee_saved_regs_at_return=True,
        interfunction_level: int = 0,
        track_liveness: bool = True,
        func_addr: int | None = None,
        element_limit: int = 5,
        merge_into_tops: bool = True,
    ):
        """
        :param subject:                         The subject of the analysis: a function, or a single basic block
        :param func_graph:                      Alternative graph for function.graph.
        :param max_iterations:                  The maximum number of iterations before the analysis is terminated.
        :param track_tmps:                      Whether or not temporary variables should be taken into consideration
                                                during the analysis.
        :param iterable observation_points:     A collection of tuples of ("node"|"insn", ins_addr, OP_TYPE) defining
                                                where reaching definitions should be copied and stored. OP_TYPE can be
                                                OP_BEFORE or OP_AFTER.
        :param init_state:                      An optional initialization state. The analysis creates and works on a
                                                copy.
                                                Default to None: the analysis then initialize its own abstract state,
                                                based on the given <Subject>.
        :param init_context:                    If init_state is not given, this is used to initialize the context
                                                field of the initial state's CodeLocation. The only default-supported
                                                type which may go here is a tuple of integers, i.e. a callstack.
                                                Anything else requires a custom FunctionHandler.
        :param cc:                              Calling convention of the function.
        :param function_handler:                The function handler to update the analysis state and results on
                                                function calls.
        :param observe_all:                     Observe every statement, both before and after.
        :param visited_blocks:                  A set of previously visited blocks.
        :param dep_graph:                       An initial dependency graph to add the result of the analysis to. Set it
                                                to None to skip dependency graph generation.
        :param canonical_size:                  The sizes (in bytes) that objects with an UNKNOWN_SIZE are treated as
                                                for operations where sizes are necessary.
        :param dep_graph:                       Set this to True to generate a dependency graph for the subject. It will
                                                be available as `result.dep_graph`.
        :param interfunction_level:             The number of functions we should recurse into. This parameter is only
                                                used if function_handler is not provided.
        :param track_liveness:                  Whether to track liveness information. This can consume
                                                sizeable amounts of RAM on large functions. (e.g. ~15GB for a function
                                                with 4k nodes)
        :param merge_into_tops:                 Merge known values into TOP if TOP is present.
                                                If True: {TOP} V {0xabc} = {TOP}
                                                If False: {TOP} V {0xabc} = {TOP, 0xabc}
```
### Guest block's Vex IR use `opt_level=0`
```python
guestblock = guest_proj.factory.block(0, opt_level=0, cross_insn_opt=False)

IRSB {
   t0:Ity_I32 t1:Ity_I32 t2:Ity_I32 t3:Ity_I32 t4:Ity_I32 t5:Ity_I32 t6:Ity_I32 t7:Ity_I32 t8:Ity_I32 t9:Ity_I32 t10:Ity_I32 t11:Ity_I32 t12:Ity_I32 t13:Ity_I32 t14:Ity_I32 t15:Ity_I32 t16:Ity_I32 t17:Ity_I32 t18:Ity_I1 t19:Ity_I32 t20:Ity_I32 t21:Ity_I32 t22:Ity_I32 t23:Ity_I32 t24:Ity_I32 t25:Ity_I32 t26:Ity_I8 t27:Ity_I32 t28:Ity_I32 t29:Ity_I32 t30:Ity_I32 t31:Ity_I32 t32:Ity_I32 t33:Ity_I32 t34:Ity_I32 t35:Ity_I32 t36:Ity_I32 t37:Ity_I32 t38:Ity_I32 t39:Ity_I32 t40:Ity_I32 t41:Ity_I32 t42:Ity_I32 t43:Ity_I32 t44:Ity_I32 t45:Ity_I32 t46:Ity_I32 t47:Ity_I1 t48:Ity_I32

   00 | ------ IMark(0x0, 4, 0) ------
   01 | t1 = GET:I32(r5)
   02 | t0 = t1
   03 | t2 = t0
   04 | PUT(r1) = t2
   05 | PUT(pc) = 0x00000004
   06 | ------ IMark(0x4, 4, 0) ------
   07 | t3 = 0x00000000
   08 | t4 = t3
   09 | PUT(r2) = t4
   10 | PUT(pc) = 0x00000008
   11 | ------ IMark(0x8, 4, 0) ------
   12 | t24 = GET:I32(r1)
   13 | t23 = Add32(t24,0x00000001)
   14 | t5 = t23
   15 | t6 = GET:I32(r1)
   16 | t26 = LDle:I8(t5)
   17 | t25 = 8Uto32(t26)
   18 | t7 = t25
   19 | PUT(r3) = t7
   20 | PUT(r1) = t5
   21 | PUT(pc) = 0x0000000c
   22 | ------ IMark(0xc, 4, 0) ------
   23 | t8 = GET:I32(r2)
   24 | t9 = 0x00000001
   25 | t10 = Add32(t8,t9)
   26 | PUT(r2) = t10
   27 | PUT(pc) = 0x00000010
   28 | ------ IMark(0x10, 4, 0) ------
   29 | t11 = GET:I32(r3)
   30 | t12 = 0x00000000
   31 | t13 = 0x00000000
   32 | PUT(cc_op) = 0x00000002
   33 | PUT(cc_dep1) = t11
   34 | PUT(cc_dep2) = t12
   35 | PUT(cc_ndep) = t13
   36 | PUT(pc) = 0x00000014
   37 | ------ IMark(0x14, 4, 0) ------
   38 | t28 = GET:I32(cc_op)
   39 | t27 = Or32(t28,0x00000010)
   40 | t29 = GET:I32(cc_dep1)
   41 | t30 = GET:I32(cc_dep2)
   42 | t31 = GET:I32(cc_ndep)
   43 | t32 = armg_calculate_condition(t27,t29,t30,t31):Ity_I32
   44 | t14 = t32
   45 | t15 = GET:I32(r3)
   46 | t16 = 0x0000003d
   47 | t17 = 0x00000000
   48 | t18 = CmpNE32(t14,0x00000000)
   49 | t34 = GET:I32(cc_op)
   50 | t33 = ITE(t18,0x00000002,t34)
   51 | PUT(cc_op) = t33
   52 | t36 = GET:I32(cc_dep1)
   53 | t35 = ITE(t18,t15,t36)
   54 | PUT(cc_dep1) = t35
   55 | t38 = GET:I32(cc_dep2)
   56 | t37 = ITE(t18,t16,t38)
   57 | PUT(cc_dep2) = t37
   58 | t40 = GET:I32(cc_ndep)
   59 | t39 = ITE(t18,t17,t40)
   60 | PUT(cc_ndep) = t39
   61 | PUT(pc) = 0x00000018
   62 | ------ IMark(0x18, 4, 0) ------
   63 | t20 = GET:I32(r2)
   64 | t19 = t20
   65 | t21 = t19
   66 | PUT(r6) = t21
   67 | PUT(pc) = 0x0000001c
   68 | ------ IMark(0x1c, 4, 0) ------
   69 | t42 = GET:I32(cc_op)
   70 | t41 = Or32(t42,0x00000010)
   71 | t43 = GET:I32(cc_dep1)
   72 | t44 = GET:I32(cc_dep2)
   73 | t45 = GET:I32(cc_ndep)
   74 | t46 = armg_calculate_condition(t41,t43,t44,t45):Ity_I32
   75 | t22 = t46
   76 | t47 = 32to1(t22)
   77 | if (t47) { PUT(pc) = 0x8; Ijk_Boring }
   78 | PUT(pc) = 0x00000020
   79 | t48 = GET:I32(pc)
   NEXT: PUT(pc) = t48; Ijk_Boring
}
```
## A discrepancy will discuss later
![the graph generated after the RDA](/commons/images/asm/dep_graph.png)
In this image, the temp's index can reach up to 56, but when we get the block's vex if from `guest_block`, it only has a max index `48`.

The answer will be exist later, because the RDA process use the `opt_level=1`

# RDA Step by step

If `dep_graph=True`, then the result of the RDA is to get a dependance graph `dep_graph`, which belongs to class `DepGraph`
Here is a call stack when doing the analysis process of stmts
![call stack of the stmt analysis](/commons/images/asm/call_stack.png)
```py
class DepGraph:
    """
    The representation of a dependency graph: a directed graph, where nodes are definitions, and edges represent uses.

    Mostly a wrapper around a <networkx.DiGraph>.
    """

    def __init__(self, graph: Optional["networkx.DiGraph[Definition]"] = None):
        """
        :param graph: A graph where nodes are definitions, and edges represent uses.
        """
        # Used for memoization of the `transitive_closure` method.
        self._transitive_closures: dict = {}

        if graph and not all(map(_is_definition, graph.nodes)):
            raise TypeError("In a DepGraph, nodes need to be <%s>s." % Definition.__name__)

        self._graph: "networkx.DiGraph[Definition]" = graph if graph is not None else networkx.DiGraph()
```

## Two process that is important for understanding

`SimEngineRDVEX` is designed to perform reaching definitions analysis on VEX IR (the low-level, lift-to-IR form of the binary code), while `SimEngineRDAIL` operates on `AIL` (angr’s higher-level, decompiled-like IR). They serve different abstraction layers of the code, which is why both are needed depending on whether the analysis runs on raw VEX IR or on a more processed, AIL-based representation.
```py
class SimEngineRDVEX(
    SimEngineLightVEXMixin,
    SimEngineLight,
):  # pylint:disable=abstract-method
    """
    Implements the VEX execution engine for reaching definition analysis.
    """

    state: ReachingDefinitionsState

    def __init__(self, project, functions=None, function_handler=None):
        super().__init__()
        self.project = project
        self.functions: Optional["FunctionManager"] = functions
        self._function_handler: Optional["FunctionHandler"] = function_handler
        self._visited_blocks = None
        self._dep_graph = None

        self.state: ReachingDefinitionsState
class SimEngineRDAIL(
    SimEngineLightAILMixin,
    SimEngineLight,
):  # pylint:disable=abstract-method
    arch: archinfo.Arch
    state: ReachingDefinitionsState

    def __init__(
        self,
        project,
        function_handler: FunctionHandler | None = None,
        stack_pointer_tracker=None,
        use_callee_saved_regs_at_return=True,
        bp_as_gpr: bool = False,
    ):
        super().__init__()
        self.project = project
        self._function_handler = function_handler
        self._visited_blocks = None
        self._dep_graph = None
        self._stack_pointer_tracker = stack_pointer_tracker
        self._use_callee_saved_regs_at_return = use_callee_saved_regs_at_return
        self.bp_as_gpr = bp_as_gpr

        self._stmt_handlers = {
            ailment.Stmt.Assignment: self._ail_handle_Assignment,
            ailment.Stmt.Store: self._ail_handle_Store,
            ailment.Stmt.Jump: self._ail_handle_Jump,
            ailment.Stmt.ConditionalJump: self._ail_handle_ConditionalJump,
            ailment.Stmt.Call: self._ail_handle_Call,
            ailment.Stmt.Return: self._ail_handle_Return,
            ailment.Stmt.DirtyStatement: self._ail_handle_DirtyStatement,
            ailment.Stmt.Label: ...,
        }

        self._expr_handlers = {
            claripy.ast.BV: self._ail_handle_BV,
            ailment.Expr.Tmp: self._ail_handle_Tmp,
            ailment.Stmt.Call: self._ail_handle_CallExpr,
            ailment.Expr.Register: self._ail_handle_Register,
            ailment.Expr.Load: self._ail_handle_Load,
            ailment.Expr.Convert: self._ail_handle_Convert,
            ailment.Expr.Reinterpret: self._ail_handle_Reinterpret,
            ailment.Expr.ITE: self._ail_handle_ITE,
            ailment.Expr.UnaryOp: self._ail_handle_UnaryOp,
            ailment.Expr.BinaryOp: self._ail_handle_BinaryOp,
            ailment.Expr.Const: self._ail_handle_Const,
            ailment.Expr.StackBaseOffset: self._ail_handle_StackBaseOffset,
            ailment.Expr.DirtyExpression: self._ail_handle_DirtyExpression,
        }
```
Both of the two has a function `process`, which is used to analyse IR stmt later

```py
self._engine_vex = SimEngineRDVEX(
   self.project,
   functions=self.kb.functions,
   function_handler=self._function_handler,
)
self._engine_ail = SimEngineRDAIL(
   self.project,
   function_handler=self._function_handler,
   stack_pointer_tracker=stack_pointer_tracker,
   use_callee_saved_regs_at_return=self._use_callee_saved_regs_at_return,
   bp_as_gpr=bp_as_gpr,
)
```


## Entry of Analysis

```py
# in class ReachingDefinitionsAnalysis's init function, use from the inherited ForwardAnalysis
self._analyze()
# implement of _analyze()
def _analyze(self) -> None:
   """
   The main analysis routine.

   :return: None
   """

   self._pre_analysis() # NOTE:  pass for rda, but not for another analysis tools like cfg...

   if self._graph_visitor is None:
       # There is no base graph that we can rely on. The analysis itself should generate successors for the
       # current job.
       # An example is the CFG recovery.

       self._analysis_core_baremetal() 

   else:
       # We have a base graph to follow. Just handle the current job.

       self._analysis_core_graph()  # NOTE: Entry when there is previous knowledge, we have the block, so it is not None

   self._post_analysis()
```

## Analysis in ForwardAnalysis

```py
def _analysis_core_graph(self) -> None:
     while not self.should_abort:
         self._intra_analysis()

         n: NodeType = self._graph_visitor.next_node()

         if n is None:
             break

         job_state = self._get_and_update_input_state(n)
         if job_state is None:
             job_state = self._initial_abstract_state(n)

         changed, output_state = self._run_on_node(n, job_state) # NOTE: IT will use the RDA class's _run_on_node to do the analysis

         if changed is False:
             # no change is detected
             self._output_state[self._node_key(n)] = output_state
             continue
         if changed is True:
             # changes detected

             # output state of node n is input state for successors to node n
             self._add_input_state(n, output_state)

             # revisit all its successors
             self._graph_visitor.revisit_successors(n, include_self=False)
         else:
             # the change of states are determined during state merging (_add_input_state()) instead of during
             # simulated execution (_run_on_node()).

             if self._node_key(n) not in self._output_state:
                 reached_fixedpoint = False
             else:
                 # is the output state the same as the old one?
                 reached_fixedpoint = self._compare_states(n, self._output_state[self._node_key(n)], output_state)
             self._output_state[self._node_key(n)] = output_state

             if not reached_fixedpoint:
                 successors_to_visit = self._add_input_state(n, output_state)
                 # revisit all successors in the `successors_to_visit` list
                 for succ in successors_to_visit:
                     self._graph_visitor.revisit_node(succ)
```

## Run on Node (here is the block) function in ReachingDefinitionsAnalysis class

```py
def _run_on_node(self, node, state: ReachingDefinitionsState):
   """

   :param node:    The current node.
   :param state:   The analysis state.
   :return:        A tuple: (reached fix-point, successor state)
   """

   self._visited_blocks.add(node)

   engine: SimEngineLight

   if isinstance(node, ailment.Block):
       block = node
       block_key = (node.addr, node.idx)
       engine = self._engine_ail
   elif isinstance(node, (Block, CodeNode)):
       block = self.project.factory.block(node.addr, node.size, opt_level=1, cross_insn_opt=False) # NOTE: The opt level = 1 
       engine = self._engine_vex # NOTE: choose Vex engine to do the analysis class 
       block_key = node.addr
   elif isinstance(node, CFGNode):
       if node.is_simprocedure or node.is_syscall:
           return False, state.copy(discard_tmpdefs=True)
       block = node.block
       engine = self._engine_vex
       block_key = node.addr
   else:
       l.warning("Unsupported node type %s.", node.__class__)
       return False, state.copy(discard_tmpdefs=True)

   state = state.copy(discard_tmpdefs=True)
   self.node_observe(node.addr, state.copy(), OP_BEFORE)

   if self.subject.type == SubjectType.Function:
       node_parents = [
           CodeLocation(pred.addr, 0, block_idx=pred.idx if isinstance(pred, ailment.Block) else None)
           for pred in self._graph_visitor.predecessors(node)
       ]
       if node.addr == self.subject.content.addr:
           node_parents += [ExternalCodeLocation()]
       self.model.at_new_block(
           CodeLocation(block.addr, 0, block_idx=block.idx if isinstance(block, ailment.Block) else None),
           node_parents,
       )

   state = engine.process( # NOTE: Main processs to analyse the stmts, happens in SimEngineRDVEX
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

   if self._track_tmps:
       # merge tmp uses to all_uses
       for tmp_idx, locs in state.tmp_uses.items():
           tmp_def = next(iter(state.tmps[tmp_idx]))
           for loc in locs:
               self.all_uses.add_use(tmp_def, loc)

   # drop definitions and uses because we will not need them anymore
   state.downsize()

   if self._node_iterations[block_key] < self._max_iterations:
       return None, state
   else:
       return False, state
```


## Process stmt in SimEngineLightVEXMixin (super class of SimEngineRDVEX)

+ Processing stmts in vex engine

```py
class SimEngineRDVEX(
    SimEngineLightVEXMixin,
    SimEngineLight,
): 
    """
    Implements the VEX execution engine for reaching definition analysis.
    """

    state: ReachingDefinitionsState

    def __init__(self, project, functions=None, function_handler=None):
        super().__init__()
        self.project = project
        self.functions: Optional["FunctionManager"] = functions
        self._function_handler: Optional["FunctionHandler"] = function_handler
        self._visited_blocks = None
        self._dep_graph = None

        self.state: ReachingDefinitionsState

class SimEngineLightVEXMixin(SimEngineLightMixin):
    """
    A mixin for doing static analysis on VEX
    """

    def _process(self, state, successors, *args, block, whitelist=None, **kwargs):  # pylint:disable=arguments-differ
        # initialize local variables
        self.tmps = {}
        self.block = block
        self.state = state

        if state is not None:
            self.arch: archinfo.Arch = state.arch

        self.tyenv = block.vex.tyenv

        self._process_Stmt(whitelist=whitelist) # NOTE: process stmts in block vex statements

        self.stmt_idx = None
        self.ins_addr = None

    def _process_Stmt(self, whitelist=None):
        if whitelist is not None:
            # optimize whitelist lookups
            whitelist = set(whitelist)

        for stmt_idx, stmt in enumerate(self.block.vex.statements):
            if whitelist is not None and stmt_idx not in whitelist:
                continue
            self.stmt_idx = stmt_idx

            if type(stmt) is pyvex.IRStmt.IMark:
                # Note that we cannot skip IMarks as they are used later to trigger observation events
                # The bug caused by skipping IMarks is reported at https://github.com/angr/angr/pull/1150
                self.ins_addr = stmt.addr + stmt.delta

            self._handle_Stmt(stmt)

        self._process_block_end()
    def _process_block_end(self):
        # handle calls to another function
        # Note that without global information, we cannot handle cases where we *jump* to another function (jumpkind ==
        # "Ijk_Boring"). Users are supposed to overwrite this method, detect these cases with the help of global
        # information (such as CFG or symbol addresses), and handle them accordingly.
        if self.block.vex.jumpkind == "Ijk_Call":
            self.stmt_idx = DEFAULT_STATEMENT
            handler = "_handle_function"
            if hasattr(self, handler):
                func_addr = (
                    self.block.vex.next if isinstance(self.block.vex.next, int) else self._expr(self.block.vex.next)
                )
                if func_addr is None and self.l is not None:
                    self.l.debug("Cannot determine the callee address at %#x.", self.block.addr)
                getattr(self, handler)(func_addr)
            else:
                if self.l is not None:
                    self.l.warning("Function handler not implemented.")

```

+ During the process, it will choose correct handler to deal with different stmt
  
```py
    #
    # Statement handlers
    #

    def _handle_Stmt(self, stmt):
        handler = "_handle_%s" % type(stmt).__name__
        if hasattr(self, handler):
            getattr(self, handler)(stmt)
        elif type(stmt).__name__ not in ("IMark", "AbiHint"):
            if self.l is not None:
                self.l.error("Unsupported statement type %s.", type(stmt).__name__)

    # synchronize with function _handle_WrTmpData()
    def _handle_WrTmp(self, stmt):
        data = self._expr(stmt.data)
        if data is None:
            return

        self.tmps[stmt.tmp] = data

    # invoked by LoadG
    def _handle_WrTmpData(self, tmp, data):
        if data is None:
            return
        self.tmps[tmp] = data
    def _handle_Dirty(self, stmt):
        if self.l is not None:
            self.l.error("Unimplemented Dirty node for current architecture.")

    def _handle_Put(self, stmt):
        raise NotImplementedError("Please implement the Put handler with your own logic.")

    def _handle_Store(self, stmt):
        raise NotImplementedError("Please implement the Store handler with your own logic.")

    def _handle_StoreG(self, stmt):
        raise NotImplementedError("Please implement the StoreG handler with your own logic.")

    def _handle_LLSC(self, stmt: pyvex.IRStmt.LLSC):
        raise NotImplementedError("Please implement the LLSC handler with your own logic.")

    #
    # Expression handlers
    #

    def _expr(self, expr):
        handler = "_handle_%s" % type(expr).__name__
        if hasattr(self, handler):
            return getattr(self, handler)(expr)
        elif self.l is not None:
            self.l.error("Unsupported expression type %s.", type(expr).__name__)
        return None
```

+ Example: Handle a Put, if there is a def, then there should be an edge in the dependancy graph

```py
# e.g. PUT(rsp) = t2, t2 might include multiple values
def _handle_Put(self, stmt):
    reg_offset: int = stmt.offset
    size: int = stmt.data.result_size(self.tyenv) // 8
    reg = Register(reg_offset, size, self.arch)
    data = self._expr(stmt.data)

    # special handling for references to heap or stack variables
    if data.count() == 1:
        for d in next(iter(data.values())):
            if self.state.is_heap_address(d):
                heap_offset = self.state.get_heap_offset(d)
                if heap_offset is not None:
                    self.state.add_heap_use(heap_offset, 1)
            elif self.state.is_stack_address(d):
                stack_offset = self.state.get_stack_offset(d)
                if stack_offset is not None:
                    self.state.add_stack_use(stack_offset, 1)

    if self.state.exit_observed and reg_offset == self.arch.sp_offset:
        return
    self.state.kill_and_add_definition(reg, data) # NOTE: in reaching and definition class

def kill_and_add_definition(
    self,
    atom: Atom,
    data: MultiValues,
    dummy=False,
    tags: set[Tag] = None,
    endness=None,  # XXX destroy
    annotated: bool = False,
    uses: set[Definition[A]] | None = None,
    override_codeloc: CodeLocation | None = None,
) -> tuple[MultiValues | None, set[Definition[A]]]:
    codeloc = override_codeloc or self.codeloc
    existing_defs = self.live_definitions.get_definitions(atom)
    mv = self.live_definitions.kill_and_add_definition(
        atom, codeloc, data, dummy=dummy, tags=tags, endness=endness, annotated=annotated
    )

    if mv is not None:
        defs = set(LiveDefinitions.extract_defs_from_mv(mv))
        self.all_definitions |= defs

        if self._dep_graph is not None:
            stack_use = {u for u in self.codeloc_uses if isinstance(u.atom, MemoryLocation) and u.atom.is_on_stack}

            sp_offset = self.arch.sp_offset
            bp_offset = self.arch.bp_offset

            values = set()
            for vs in mv.values():
                for v in vs:
                    values.add(v)

            if uses is None:
                uses = self.codeloc_uses
            for used in uses:
                # sp is always used as a stack pointer, and we do not track dependencies against stack pointers.
                # bp is sometimes used as a base pointer. we recognize such cases by checking if there is a use to
                # the stack variable.
                #
                # There are two cases for which it is superfluous to report a dependency on (a use of) stack/base
                # pointers:
                # - The `Definition` *uses* a `MemoryLocation` pointing to the stack;
                # - The `Definition` *is* a `MemoryLocation` pointing to the stack.
                is_using_spbp_while_memory_address_on_stack_is_used = (
                    isinstance(used.atom, Register)
                    and used.atom.reg_offset in (sp_offset, bp_offset)
                    and len(stack_use) > 0
                )
                is_using_spbp_to_define_memory_location_on_stack = (
                    isinstance(atom, MemoryLocation)
                    and (
                        atom.is_on_stack
                        or (isinstance(atom.addr, claripy.ast.Base) and self.is_stack_address(atom.addr))
                    )
                    and isinstance(used.atom, Register)
                    and used.atom.reg_offset in (sp_offset, bp_offset)
                )

                if not (
                    is_using_spbp_while_memory_address_on_stack_is_used
                    or is_using_spbp_to_define_memory_location_on_stack
                ):
                    # Moderately confusing misnomers. This is an edge from a def to a use, since the
                    # "uses" are actually the definitions that we're using and the "definition" is the
                    # new definition; i.e. The def that the old def is used to construct so this is
                    # really a graph where nodes are defs and edges are uses.
                    self._dep_graph.add_node(used)
                    for def_ in defs:
                        if not def_.dummy:
                            self._dep_graph.add_edge(used, def_)
                    self._dep_graph.add_dependencies_for_concrete_pointers_of(
                        values,
                        used,
                        self.analysis.project.kb.cfgs.get_most_accurate(),
                        self.analysis.project.loader,
                    )
    else:
        defs = set()

    for def_ in existing_defs:
        self.analysis.model.kill_def(def_)
    for def_ in defs:
        self.analysis.model.add_def(def_)

    return mv, defs
```
+ Handle `ITE`

```py
# CAUTION: experimental
def _handle_ITE(self, expr: pyvex.IRExpr.ITE):
    cond = self._expr(expr.cond)
    cond_v = cond.one_value()
    iftrue = self._expr(expr.iftrue)
    iffalse = self._expr(expr.iffalse)

    if claripy.is_true(cond_v):
        return iftrue
    elif claripy.is_false(cond_v):
        return iffalse
    else:
        data = iftrue.merge(iffalse)
        return data
```

# After using the `opt_level=1`

```log
IRSB {
   t0:Ity_I32 t1:Ity_I32 t2:Ity_I32 t3:Ity_I32 t4:Ity_I32 t5:Ity_I32 t6:Ity_I32 t7:Ity_I32 t8:Ity_I32 t9:Ity_I32 t10:Ity_I32 t11:Ity_I32 t12:Ity_I32 t13:Ity_I32 t14:Ity_I32 t15:Ity_I32 t16:Ity_I32 t17:Ity_I32 t18:Ity_I1 t19:Ity_I32 t20:Ity_I32 t21:Ity_I32 t22:Ity_I32 t23:Ity_I32 t24:Ity_I32 t25:Ity_I32 t26:Ity_I8 t27:Ity_I32 t28:Ity_I32 t29:Ity_I32 t30:Ity_I32 t31:Ity_I32 t32:Ity_I32 t33:Ity_I32 t34:Ity_I32 t35:Ity_I32 t36:Ity_I32 t37:Ity_I32 t38:Ity_I32 t39:Ity_I32 t40:Ity_I32 t41:Ity_I32 t42:Ity_I32 t43:Ity_I32 t44:Ity_I32 t45:Ity_I32 t46:Ity_I32 t47:Ity_I1 t48:Ity_I32 t49:Ity_I32 t50:Ity_I32 t51:Ity_I1 t52:Ity_I32 t53:Ity_I32 t54:Ity_I32 t55:Ity_I1 t56:Ity_I1

   00 | ------ IMark(0x0, 4, 0) ------
   01 | t1 = GET:I32(r5)
   02 | PUT(r1) = t1
   03 | PUT(pc) = 0x00000004
   04 | ------ IMark(0x4, 4, 0) ------
   05 | PUT(r2) = 0x00000000
   06 | PUT(pc) = 0x00000008
   07 | ------ IMark(0x8, 4, 0) ------
   08 | t24 = GET:I32(r1)
   09 | t23 = Add32(t24,0x00000001)
   10 | t26 = LDle:I8(t23)
   11 | t49 = 8Uto32(t26)
   12 | PUT(r3) = t49
   13 | PUT(r1) = t23
   14 | PUT(pc) = 0x0000000c
   15 | ------ IMark(0xc, 4, 0) ------
   16 | t8 = GET:I32(r2)
   17 | t10 = Add32(t8,0x00000001)
   18 | PUT(r2) = t10
   19 | PUT(pc) = 0x00000010
   20 | ------ IMark(0x10, 4, 0) ------
   21 | t11 = GET:I32(r3)
   22 | PUT(pc) = 0x00000014
   23 | ------ IMark(0x14, 4, 0) ------
   24 | t51 = CmpNE32(t11,0x00000000)
   25 | t15 = GET:I32(r3)
   26 | PUT(cc_op) = 0x00000002
   27 | t52 = ITE(t51,t15,t11)
   28 | PUT(cc_dep1) = t52
   29 | t53 = ITE(t51,0x0000003d,0x00000000)
   30 | PUT(cc_dep2) = t53
   31 | PUT(cc_ndep) = 0x00000000
   32 | PUT(pc) = 0x00000018
   33 | ------ IMark(0x18, 4, 0) ------
   34 | t20 = GET:I32(r2)
   35 | PUT(r6) = t20
   36 | PUT(pc) = 0x0000001c
   37 | ------ IMark(0x1c, 4, 0) ------
   38 | t55 = CmpNE32(t52,t53)
   39 | t54 = 1Uto32(t55)
   40 | t56 = 32to1(t54)
   41 | if (t56) { PUT(pc) = 0x8; Ijk_Boring }
   NEXT: PUT(pc) = 0x00000020; Ijk_Boring
}
```

## The numebr of const in opt_level = 1, is 16

![temps in opt level = 1](/commons/images/asm/tmp_in_optlevel1.png)

+ In `opt_level=0`, the tmp indices are from `0` to `48`, total is `49`.
+ In `opt_level=0`, some new tmp indices are from `49` to `56`, total is `16`.

So, during lifting, some tmp variables may be created for intermediate calculations and then optimized away if opt_level is greater than zero. Even at `opt_level=0`, certain intermediate steps might not appear in the final IR, leaving "gaps" in the tmp numbering.

## Differences between opt levels
What’s Happening in the Example:

+ **opt_level = 0 (No Optimization)**:
The IR shows every micro-operation. Assignments like `t0 = t1`, `t2 = t0`, and re-reads of registers appear as separate statements. This is a very verbose representation. Essentially, you see all the intermediate steps that VEX generates to break down the original instructions into a simple, generic RISC-like form. For instance:

    + `mov r1, r5` might appear as multiple steps: load `r5` into a temporary, then another temporary, and then write it to `r1`.
    + `ldrb r3, [r1, #1]!` shows multiple temporaries handling pointer arithmetic and loading, along with sign or zero extensions.
The result is a long list of temporaries (`tX` variables) and PUT statements to registers and the program counter.

+ opt_level = 1 (Some Optimization):
Here, VEX tries to fold unnecessary assignments and remove extra temporaries. As a result:

    + Simple moves (`mov r1, r5`) become a single PUT operation rather than a chain of `WrTmp` steps.
    + Operations like add and ldrb are handled in fewer statements, more closely resembling the original instructions.
    + Many redundant temporaries are eliminated, and conditional logic (`cmp`, `cmpne`) is streamlined into fewer IR statements.

The IR is still low-level compared to the original assembly instructions, but it’s more concise and easier to read. Assignments to registers are more directly and fewer intermediate steps.

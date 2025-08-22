---
title: Debugging Symbolic Execution Errors, Handling Memory Address Constraints in Symbolic States
date: 2024-10-19
categories: [Assembly]
tags: [assembly,python,angr]     # TAG names should always be lowercase
published: false
---

# The description of the promblem and solution
During the development of the symbolic verification tool, I met an error for a specific guest block, and I will walk through the issue I faced while performing symbolic execution on a guest block, how I diagnosed the problem, and the steps I took to resolve it.

This error is, I tried to ignore this error state and put the error state to the active state directly, however, this does not work, because there will be 2 states after running and does not align with the host's side, so I need to find a better way to find out what is the root reason for this issue.
```log
[MAPPIGN LIVE-IN REGISTERS FOR SYMBOLIC EXECUTION] info: Register Mapping Solution 1:{'pc': 'ebp', 'r3': 'ebx'}
[SYMBOLIC EXECUTION ANALYSIS] info: ## Init guest machine state ...
[SYMBOLIC EXECUTION ANALYSIS] info: ## Run guest machine ...
[SYMBOLIC EXECUTION ANALYSIS] msg: == after step: defaultdict(<class 'list'>, {'active': [<SimState @ 0x4>], 'stashed': [], 'pruned': [], 'unsat': [], 'errored': [], 'deadended': [], 'unconstrained': []}), type(active): <class 'list'>
[SYMBOLIC EXECUTION ANALYSIS] msg: == state: <SimState @ 0x4> , ip: <BV32 0x4>, == constraints before: []
[SYMBOLIC EXECUTION ANALYSIS] msg: == constraints after: []
[SYMBOLIC EXECUTION ANALYSIS] msg: == after filter: defaultdict(<class 'list'>, {'active': [<SimState @ 0x4>], 'stashed': [], 'pruned': [], 'unsat': [], 'errored': [], 'deadended': [], 'unconstrained': []})
[SYMBOLIC EXECUTION ANALYSIS] msg: == after step: defaultdict(<class 'list'>, {'active': [<SimState @ 0x8>], 'stashed': [], 'pruned': [], 'unsat': [], 'errored': [], 'deadended': [], 'unconstrained': []}), type(active): <class 'list'>
[SYMBOLIC EXECUTION ANALYSIS] msg: == state: <SimState @ 0x8> , ip: <BV32 0x8>, == constraints before: []
[SYMBOLIC EXECUTION ANALYSIS] msg: == constraints after: []
[SYMBOLIC EXECUTION ANALYSIS] msg: == after filter: defaultdict(<class 'list'>, {'active': [<SimState @ 0x8>], 'stashed': [], 'pruned': [], 'unsat': [], 'errored': [], 'deadended': [], 'unconstrained': []})
[SYMBOLIC EXECUTION ANALYSIS] info: read addr: <BV32 0x10 + (0xfffffffd + r3_5_32 << 0x2)>, size: 4 , condition: <Bool ((if 0xfffffffd + r3_5_32 >= 0x1e then 0x1 else 0x0) & ~(0#31 .. (if 0xfffffffd + r3_5_32 == 0x1e then 1 else 0))) != 0x1>
[SYMBOLIC EXECUTION ANALYSIS] info: state constraints before: [<Bool shadow_pc_39_32 == 0x8>]
[SYMBOLIC EXECUTION ANALYSIS] info: state constraints after: [<Bool shadow_pc_39_32 == 0x8>]
[SYMBOLIC EXECUTION ANALYSIS] msg: == after step: defaultdict(<class 'list'>, {'active': [<SimState @ 0x8>], 'stashed': [], 'pruned': [], 'unsat': [], 'errored': [], 'deadended': [], 'unconstrained': []}), type(active): <class 'list'>
[SYMBOLIC EXECUTION ANALYSIS] msg: == state: <SimState @ 0x8> , ip: <BV32 0x8>, == constraints before: []
[SYMBOLIC EXECUTION ANALYSIS] msg: == constraints after: []
[SYMBOLIC EXECUTION ANALYSIS] error: == Error: [<State errored with "Unable to concretize address for load with the provided strategies.">]
 May caused by assign a symbolic value to the pc, will continue run till end
```
## Three versions of test code blocks that I use
I used three versions of the test cases to localize the issue:
```json
{
    "arm": {
        "hex_code": [
            "AzBD4g==",
            "HgBT4w==",
            "A/Gflw=="
        ],
        "instructions": [
            "sub r3, r3, #3",
            "cmp r3, #0x1e",
            "ldrls pc, [pc, r3, lsl #2]"
        ]
    },
    "x86": {
        "hex_code": [
            "g+sD",
            "g/se",
            "D0NsnQA="
        ],
        "instructions": [
            "subl    $3, %ebx",
            "cmpl    $0x1e, %ebx",
            "cmovae (%ebp,%ebx,4), %ebp"
        ]
    },
    "valid": false
},
{
    "arm": {
        "hex_code": [
            "AzBD4g==",
            "HgBT4w==",
            "A+Gflw=="
        ],
        "instructions": [
            "sub r3, r3, #3",
            "cmp r3, #0x1e",
            "ldrls lr, [pc, r3, lsl #2]"
        ]
    },
    "x86": {
        "hex_code": [
            "g+sD",
            "g/se",
            "D0NsnQA="
        ],
        "instructions": [
            "subl    $3, %ebx",
            "cmpl    $0x1e, %ebx",
            "cmovae (%ebp,%ebx,4), %ebp"
        ]
    },
    "valid": false
},
{
    "arm": {
        "hex_code": [
            "AzBD4g==",
            "HgBT4w==",
            "AuCflQ=="
        ],
        "instructions": [
            "sub r3, r3, #3",
            "cmp r3, #0x1e",
            "ldrls lr, [pc, #2]"
        ]
    },
    "x86": {
        "hex_code": [
            "g+sD",
            "g/se",
            "D0NtAg=="
        ],
        "instructions": [
            "subl    $3, %ebx",
            "cmpl    $0x1e, %ebx",
            "cmovae 2(%ebp), %ebp"
        ]
    },
    "valid": false
}
```

## The Problem: Erroneous State Errors During Symbolic Execution
While executing a guest block symbolically, I noticed that the execution state consistently errored out and was placed into the error stash. This behavior was unexpected and hindered the progress of the symbolic execution.

- Guest Block: 
```
0x0:    sub     r3, r3, #3
0x4:    cmp     r3, #0x1e
0x8:    ldrls   pc, [pc, r3, lsl #2]
IRSB {
   t0:Ity_I32 t1:Ity_I32 t2:Ity_I32 t3:Ity_I32 t4:Ity_I32 t5:Ity_I32 t6:Ity_I32 t7:Ity_I32 t8:Ity_I32 t9:Ity_I32 t10:Ity_I32 t11:Ity_I32 t12:Ity_I32 t13:Ity_I32 t14:Ity_I32 t15:Ity_I32 t16:Ity_I32 t17:Ity_I32 t18:Ity_I32 t19:Ity_I32 t20:Ity_I1 t21:Ity_I32 t22:Ity_I32 t23:Ity_I1 t24:Ity_I1 t25:Ity_I32 t26:Ity_I32 t27:Ity_I32
   00 | ------ IMark(0x0, 4, 0) ------
   01 | t0 = GET:I32(r3)
   02 | t1 = 0x00000003
   03 | t2 = Sub32(t0,t1)
   04 | PUT(r3) = t2
   05 | PUT(pc) = 0x00000004
   06 | ------ IMark(0x4, 4, 0) ------
   07 | t3 = GET:I32(r3)
   08 | t4 = 0x0000001e
   09 | t5 = 0x00000000
   10 | PUT(cc_op) = 0x00000002
   11 | PUT(cc_dep1) = t3
   12 | PUT(cc_dep2) = t4
   13 | PUT(cc_ndep) = t5
   14 | PUT(pc) = 0x00000008
   15 | ------ IMark(0x8, 4, 0) ------
   16 | t11 = GET:I32(cc_op)
   17 | t10 = Or32(t11,0x00000090)
   18 | t12 = GET:I32(cc_dep1)
   19 | t13 = GET:I32(cc_dep2)
   20 | t14 = GET:I32(cc_ndep)
   21 | t15 = armg_calculate_condition(t10,t12,t13,t14):Ity_I32
   22 | t6 = t15
   23 | t18 = GET:I32(r3)
   24 | t17 = Shl32(t18,0x02)
   25 | t16 = Add32(0x00000010,t17)
   26 | t7 = t16
   27 | t8 = 0x00000010
   28 | t19 = GET:I32(pc)
   29 | t20 = CmpNE32(t6,0x00000000)
   30 | t9 = if (t20) ILGop_Ident32(LDle(t7)) else t19
   31 | t22 = GET:I32(pc)
   32 | t23 = CmpNE32(t6,0x00000000)
   33 | t21 = ITE(t23,t9,t22)
   34 | PUT(pc) = t21
   35 | t25 = Xor32(t6,0x00000001)
   36 | t24 = 32to1(t25)
   37 | if (t24) { PUT(pc) = 0xc; Ijk_Boring }
   38 | t26 = GET:I32(pc)
   39 | PUT(pc) = t26
   40 | t27 = GET:I32(pc)
   NEXT: PUT(pc) = t27; Ijk_Boring
}
```
- Host Block: 
```
0x0:    sub     ebx, 3
0x3:    cmp     ebx, 0x1e
0x6:    cmovae  ebp, dword ptr [ebp + ebx*4]
IRSB {
   t0:Ity_I32 t1:Ity_I32 t2:Ity_I32 t3:Ity_I32 t4:Ity_I32 t5:Ity_I32 t6:Ity_I32 t7:Ity_I32 t8:Ity_I32 t9:Ity_I32 t10:Ity_I32 t11:Ity_I32 t12:Ity_I32 t13:Ity_I32 t14:Ity_I32 t15:Ity_I1 t16:Ity_I32 t17:Ity_I32 t18:Ity_I32 t19:Ity_I32 t20:Ity_I32 t21:Ity_I32

   00 | ------ IMark(0x0, 3, 0) ------
   01 | t2 = GET:I32(ebx)
   02 | t1 = 0x00000003
   03 | t0 = Sub32(t2,t1)
   04 | PUT(cc_op) = 0x00000006
   05 | PUT(cc_dep1) = t2
   06 | PUT(cc_dep2) = t1
   07 | PUT(cc_ndep) = 0x00000000
   08 | PUT(ebx) = t0
   09 | PUT(eip) = 0x00000003
   10 | ------ IMark(0x3, 3, 0) ------
   11 | t5 = GET:I32(ebx)
   12 | t4 = 0x0000001e
   13 | t3 = Sub32(t5,t4)
   14 | PUT(cc_op) = 0x00000006
   15 | PUT(cc_dep1) = t5
   16 | PUT(cc_dep2) = t4
   17 | PUT(cc_ndep) = 0x00000000
   18 | PUT(eip) = 0x00000006
   19 | ------ IMark(0x6, 5, 0) ------
   20 | t12 = GET:I32(ebx)
   21 | t11 = Shl32(t12,0x02)
   22 | t13 = GET:I32(ebp)
   23 | t10 = Add32(t13,t11)
   24 | t9 = Add32(t10,0x00000000)
   25 | t8 = t9
   26 | t6 = LDle:I32(t8)
   27 | t7 = GET:I32(ebp)
   28 | t16 = GET:I32(cc_op)
   29 | t17 = GET:I32(cc_dep1)
   30 | t18 = GET:I32(cc_dep2)
   31 | t19 = GET:I32(cc_ndep)
   32 | t20 = x86g_calculate_condition(0x00000003,t16,t17,t18,t19):Ity_I32
   33 | t15 = 32to1(t20)
   34 | t14 = ITE(t15,t6,t7)
   35 | PUT(ebp) = t14
   36 | PUT(eip) = 0x0000000b
   37 | t21 = GET:I32(eip)
   NEXT: PUT(eip) = t21; Ijk_Boring
}
```
Here should be noticed that in the guest side, there is a instruction `ldrls`, which means it will ldr from the memory adress based on the comparison of `r3` and the bounds of JumpTable.

## Initial Hypothesis: Symbolic Value in Program Counter
My first thought was that the error occurred because a symbolic value was being moved into the program counter (PC). Since the PC dictates the flow of execution, having a symbolic value could lead to undefined behavior or errors in the execution state.
I also add the concretization step for the memory value when I do the initialization of the memory of the address, however, after this the state will go to the error stash once again.
```python
  def guest_handle_memory_read(self, state: angr.SimState):
      addr = state.inspect.mem_read_address
      size = state.inspect.mem_read_length
      cond = state.inspect.mem_read_condition
      # Not Suitable for Pre-Read Modifications: If you need to initialize uninitialized memory or modify the read before it occurs, this approach value_expr won't help.
      # value_expr = state.inspect.mem_read_expr # need to specially handle the uninitialized memory value for read

      addrin = self.get_current_memory_address(state, addr)
      mloc = self.find_location(state, addrin)

      if mloc == None:
          # allocate a new memory location
          name = self.create_location_name()
          # Use a symbolic value for uninitialized memory
          val = state.solver.BVS(name, size * 8)
          self.initialize_locations.append({addrin:val})
          ################################################################
          # No need to concretize the memory region for the new name           
          code_end = self._code_addr + self._code_size
          state.solver.add(val > code_end)
          state.solver.add((val & 0b11) == 0)  # Ensures val is a multiple of 4
          concrete_values = state.solver.eval_upto(val, 10, cast_to=int)
          # Ensure state is still satisfiable
          if not state.solver.satisfiable():
              raise GuestMemoryConcretizationError("State became unsatisfiable after adding constraints.")
          state.add_constraints(val == concrete_values[0])
          ################################################################
```
Further more, to test this hypothesis, I modified the code to prevent symbolic values from being moved into the PC. However, the state continued to error out, indicating that my initial assumption might not be the root cause.

## Further Investigation: Memory Moves into Registers
```asm
"sub r3, r3, #3",
"cmp r3, #0x1e",
"ldrls lr, [pc, r3, lsl #2]"
```
Next, I examined other blocks where memory values were moved into registers, such as the link register (`lr`). I observed that even in these cases, the state errored out similarly. This suggested that the issue was not isolated to the PC but might be related to moving memory values into registers in general.

## Experiment: Removing Register Operations
To isolate the problem further, I removed operations involving the `r3` register in the final instructions of the block. Interestingly, after this change, the state remained active and did not error out. This pointed towards the `r3` register's involvement in causing the error.

## Diagnosing the Root Cause: Erroneous Concretization of Memory Address Values
Based on the observations, I hypothesized that the error stemmed from incorrectly concretizing the memory address value associated with `r3`. Specifically, the value of `r3` might be the inverse of what it should be during execution.

## Understanding the Role of r3 in Execution
In the context of the execution:

- Expected Behavior: When comparing the value of `r3` against the bounds of a jump table, the value should be less than the upper bound to ensure valid access.
- Erroneous Behavior: If, during concretization, `r3` holds a value greater than the bounds, accessing the jump table would be invalid, leading to state errors.
  
This mismatch suggested that the concretization process might be assigning incorrect or unintended values to `r3`, causing the symbolic execution to fail.

## Solution: Adding Memory Address Constraints to Extra Constraints
To verify my hypothesis, I examined the symbolic values of the registers and memory addresses involved. The analysis confirmed that `r3` was indeed receiving incorrect values due to a lack of proper constraints during concretization.

### Implementing the Fix
When reading from memory in symbolic execution, it's crucial to ensure that the memory addresses are within valid ranges. To enforce this, I added the necessary constraints on the memory address to the solver's extra constraints during the satisfiability check.

Here's how I adjusted the code:

```python
# Construct the list of extra constraints
extra_constraints = [naddr == caddr]

# If there is a condition on the memory address, include it
if cond is not None:
    extra_constraints.append(cond)

# Check if the constraints are satisfiable under the current state
is_satisfiable = state.solver.satisfiable(extra_constraints=extra_constraints)

if not is_satisfiable:
    raise ValidationError("Constraint naddr == caddr is unsatisfiable with the current state.")
else:
    # Add the constraints to the state to ensure correct concretization
    for cod in extra_constraints:
        # state.add_constraints(*extra_constraints)
        state.add_constraints(cod)
```
### Explanation:
- **Constructing `extra_constraints`**: I created a list containing the essential constraint naddr == caddr, ensuring that the next address matches the current address.
- **Including the Memory Condition**: If there was a condition (cond) related to the memory address validity, I appended it to `extra_constraints`.
- **Checking Satisfiability**: Before adding the constraints to the state, I checked if they were satisfiable. If not, a ValidationError was raised, indicating an unsolvable state.
- **Adding Constraints to the State**: If the constraints were satisfiable, I added them to the state using state.add_constraints(*`extra_constraints`). This step ensured that during concretization, r3 and other relevant registers would receive values consistent with the constraints.
- **Why I add the constraint one by one**: Because it is not accepted in angr to add a whole list.

### Outcome

Successful Symbolic Execution, it means at least both guest and host side has no issues with the state number.
```log
INFO     | 2024-10-19 01:29:50,039 | VerificationLogger | [SYMBOLIC EXECUTION ANALYSIS] info: ## Validate the states ...
[SYMBOLIC EXECUTION ANALYSIS] info: ## Number of fini states to validate:1
INFO     | 2024-10-19 01:30:08,518 | VerificationLogger | [SYMBOLIC EXECUTION ANALYSIS] info: ## Number of fini states to validate:1
[SYMBOLIC EXECUTION ANALYSIS] error: Inequivalent reg values: lr(<BV32 if ~((if 0x21 <= r3_5_32 || r3_5_32[31:2] == 0x0 && r3_5_32[1:0] <= 2 then 0xfffffffe else 0xffffffff) | (0x0 .. (if r3_5_32 == 0x21 then 1 else 0))) == 0x1 then lr_1_32 else mem0_40_32>) vs ebp(<BV32 if 0x21 <= r3_5_32 || r3_5_32[31:2] == 0x0 && r3_5_32[1:0] <= 2 then mem0_40_32 else 0x8 + shadow_pc_39_32>)
ERROR    | 2024-10-19 01:30:20,518 | VerificationLogger | [SYMBOLIC EXECUTION ANALYSIS] error: Inequivalent reg values: lr(<BV32 if ~((if 0x21 <= r3_5_32 || r3_5_32[31:2] == 0x0 && r3_5_32[1:0] <= 2 then 0xfffffffe else 0xffffffff) | (0x0 .. (if r3_5_32 == 0x21 then 1 else 0))) == 0x1 then lr_1_32 else mem0_40_32>) vs ebp(<BV32 if 0x21 <= r3_5_32 || r3_5_32[31:2] == 0x0 && r3_5_32[1:0] <= 2 then mem0_40_32 else 0x8 + shadow_pc_39_32>)
[SYMBOLIC EXECUTION ANALYSIS] error: Inequivalent reg values: lr(<BV32 if ~((if 0x21 <= r3_5_32 || r3_5_32[31:2] == 0x0 && r3_5_32[1:0] <= 2 then 0xfffffffe else 0xffffffff) | (0x0 .. (if r3_5_32 == 0x21 then 1 else 0))) == 0x1 then lr_1_32 else mem0_40_32>) vs ebx(<BV32 0xfffffffd + r3_5_32>)
ERROR    | 2024-10-19 01:30:20,521 | VerificationLogger | [SYMBOLIC EXECUTION ANALYSIS] error: Inequivalent reg values: lr(<BV32 if ~((if 0x21 <= r3_5_32 || r3_5_32[31:2] == 0x0 && r3_5_32[1:0] <= 2 then 0xfffffffe else 0xffffffff) | (0x0 .. (if r3_5_32 == 0x21 then 1 else 0))) == 0x1 then lr_1_32 else mem0_40_32>) vs ebx(<BV32 0xfffffffd + r3_5_32>)
[SYMBOLIC EXECUTION ANALYSIS] error: ## Fail translation validation for state0
ERROR    | 2024-10-19 01:30:20,522 | VerificationLogger | [SYMBOLIC EXECUTION ANALYSIS] error: ## Fail translation validation for state0
Block 1 Equivalent Verified: False
```

# During the debug process

Actually I tried lots of steps to localize the trigger of the error, because I do not know where the state will become errored and be added into the error state.

> Unable to concretize address for load with the provided strategies.

I followed the breakpoint and observed how an active state becomes an error state.

In `angr.storage.memory_mixins._address_concretization_mixin`
if the state is failed to fetch, it will get a empty successor' state. (is_empty = True)

```python
angr/state_plugins/inspect.py
    def _set_inspect_attrs(self, **kwargs):
        for k, v in kwargs.items():
            if k not in inspect_attributes:
                raise ValueError(f"Invalid inspect attribute {k} passed in. Should be one of: {inspect_attributes}")
            # l.debug("... setting %s", k)
            setattr(self, k, v)
```

How to process a failure state:

```python
/angr/engines/failure.py
class SimEngineFailure(SuccessorsMixin, ProcedureMixin):
    def process_successors(self, successors, **kwargs):
        state = self.state
        jumpkind = state.history.parent.jumpkind if state.history and state.history.parent else None

        if jumpkind in ("Ijk_EmFail", "Ijk_MapFail") or (jumpkind is not None and jumpkind.startswith("Ijk_Sig")):
            raise AngrExitError("Cannot execute following jumpkind %s" % jumpkind)

        if jumpkind == "Ijk_Exit":
            from ..procedures import SIM_PROCEDURES

            l.debug("Execution terminated at %#x", state.addr)
            terminator = SIM_PROCEDURES["stubs"]["PathTerminator"](project=self.project)
            return self.process_procedure(state, successors, terminator, **kwargs)

        return super().process_successors(successors, **kwargs)
```

Then the successor of the failure state will be processed, return a failed status

```python
  def process_successors(self, successors, **kwargs):
      """
      Implement this function to fill out the SimSuccessors object with the results of stepping state.

      In order to implement a model where multiple mixins can potentially handle a request, a mixin may implement
      this method and then perform a super() call if it wants to pass on handling to the next mixin.

      Keep in mind python's method resolution order when composing multiple classes implementing this method.
      In short: left-to-right, depth-first, but deferring any base classes which are shared by multiple subclasses
      (the merge point of a diamond pattern in the inheritance graph) until the last point where they would be
      encountered in this depth-first search. For example, if you have classes A, B(A), C(B), D(A), E(C, D), then the
      method resolution order will be E, C, B, D, A.

      :param state:           The state to manipulate
      :param successors:      The successors object to fill out
      :param kwargs:          Any extra arguments. Do not fail if you are passed unexpected arguments.
      """
      successors.processed = False  # mark failure
```

All the things happen before we set the initial guest memory.
Currently, the guest memory address, which finds the error state.

## Attatchment

The method that I used, that want to record the memory address when the memory concretization fails and I concretize it again, global's get the last symbolic address only record the last address that is being concretized:

```python
 if isinstance(err, angr.errors.SimMemoryAddressError):
      # [patch 2], may exist some instruction changes the pc value:
      # e.g. ldrls	pc, [pc, r3, lsl #2]
      # and pc is loaded from the memory
      # generally the last one
      addr = s.globals.get('last_symbolic_address', None)
      if addr is None:
          # Unable to retrieve the address
          # Handle this case appropriately
          continue
      if s.solver.symbolic(addr):
          # Try to concretize the addres
          possible_addrs = s.solver.eval_upto(addr, 10, cast_to=int)
          if possible_addrs:
              for concrete_addr in possible_addrs:
                  new_state = s.copy()
                  # Constrain the address to the concrete value
                  new_state.add_constraints(addr == concrete_addr)
                  # Adjust the PC if necessary
                  if addr == s.regs.pc:
                      new_state.regs.pc = concrete_addr
                  simgr.active.append(new_state)
                  break
          else:
              # Cannot concretize address, discard the state
              simgr.drop(stash='errored')
          # Clear the stored address
          s.globals['last_symbolic_address'] = None
          continue
      else:
          # PC is concrete, re-add the state
          simgr.active.append(s)
          # Clear the stored address
          s.globals['last_symbolic_address'] = None
      continue
  else: 
      continue
```

# Be Methodical, Not Mythical
Rather than guessing which component is at fault, adopt a systematic approach:

- Isolate Variables: Focus on one variable or component at a time.
- Test Hypotheses: Make a prediction about what's wrong and test it directly.
- Collect Evidence: Use logging and debugging tools to gather data before jumping to conclusions.
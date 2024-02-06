List Assign with different strategies in Python
===============================================

Here is a code block

```python
def run_infer(self, g_blocks: List[TraceBlock], max_o_num: int = THREADS_NUM) -> Iterable[B_PAIRS]:
    self.model.eval()
    _g_blocks = [*g_blocks]
    _default_inst = ArmInst(opcode='mov', destination='reg1', source='reg0')
    _trace_inst = TraceInst.init_from_inst(_default_inst)
    _default_block = TraceBlock([_trace_inst], line_idx=-1)
    diff = self.batch_size - len(_g_blocks)
```

Questions:

+ What is this kind of copy?
  + A shallow copy of the list `g_blocks`
  + If a deep copy is wanted, should use `copy.deepcopy()` instead.
+ If it is a shallow copy, why not directly use `_g_blocks = g_blocks`?
  + Using `_g_blocks = g_blocks` directly links `_g_blocks `and `g_blocks` to the same list object. This means any changes made to the list through `_g_blocks` will also reflect in g_blocks, and vice versa, because both variables point to the same memory location.
  + In contrast, `_g_blocks = [*g_blocks]` creates a new list object with the same elements. Here's the difference:
    + Direct Assignment (`_g_blocks = g_blocks`):
      + Both `_g_blocks` and `g_blocks` reference the same list.
      + Changes to the list via either variable will be visible through the other.
      + Example: Adding an item to `_g_blocks` will also show up when accessing `g_blocks`.
    + Shallow Copy (`_g_blocks = [*g_blocks]`):
    + A new list object is created and assigned to `_g_blocks`.
    + Both `_g_blocks` and `g_blocks` contain the same elements, but they are independent lists.
    + Changes to the structure of the list (like adding or removing items) in `_g_blocks` won't affect `g_blocks`. However, changes to the elements themselves will reflect in both lists if those elements are mutable objects.

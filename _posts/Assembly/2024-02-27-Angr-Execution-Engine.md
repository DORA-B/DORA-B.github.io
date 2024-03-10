---
title: Angr Execution Engine
date: 2024-02-27
categories: [Assembly]
tags: [assembly,python,angr]     # TAG names should always be lowercase
---
# Simulation Managers

The most important interrface in angr is the SimulationManager, which allows you to control symbolic execution over groups of states simultaneously, applying search strategies to explore a program’s state space.

## Stepping

Most basic capability of a simulation is to step forward all states in a given stash by one basic block.

```py
import angr
proj = angr.Project('examples/fauxware/fauxware', auto_load_libs=False)
state = proj.factory.entry_state()
simgr = proj.factory.simgr(state)
simgr.active
[<SimState @ 0x400580>]
simgr.step()
simgr.active
[<SimState @ 0x400540>]
```

When a state encounters a symbolic branch condition, both of the successor states appear in the stash, and you can step both of them in sync.
If only want to step until there's nothing left to step, without caring about controlling analysis, use `.run()`

```py
# Step until the first symbolic branch
>>> while len(simgr.active) == 1:
...    simgr.step()

>>> simgr
<SimulationManager with 2 active>
>>> simgr.active
[<SimState @ 0x400692>, <SimState @ 0x400699>]

# Step until everything terminates, 3 deadended states.
>>> simgr.run()
>>> simgr
<SimulationManager with 3 deadended>
```

When a state fails to produce any successors during execution, for example, because it reached an `exit` syscall, it is removed from the active stash and placed in the `deadended` stash.

## Stash Management

To move states between stashes, use `.move()`, which takes `from_stash`, `to_stash`, and `filter_func` (optional, default is to move everything). For example, let’s move everything that has a certain string in its output:



```python
simgr.move(from_stash='deadended', to_stash='authenticated', filter_func=lambda s: b'Welcome' in s.posix.dumps(1))
simgr
# <SimulationManager with 2 authenticated, 1 deadended>
```

Create a new stash named “authenticated” just by asking for states to be moved to it. All the states in this stash have “Welcome” in their stdout, which is a fine metric for now.

Each stash is just a list, and you can index into or iterate over the list to access each of the individual states, but there are some alternate methods to access the states too. If you prepend the name of a stash with `one_`, you will be given the first state in the stash. If you prepend the name of a stash with `mp_`, you will be given a [mulpyplexed](https://github.com/zardus/mulpyplexer) version of the stash.

```python
for s in simgr.deadended + simgr.authenticated:
    print(hex(s.addr))
0x1000030
0x1000078
0x1000078

simgr.one_deadended
<SimState @ 0x1000030>
simgr.mp_authenticated
MP([<SimState @ 0x1000078>, <SimState @ 0x1000078>])
simgr.mp_authenticated.posix.dumps(0)
MP(['\x00\x00\x00\x00\x00\x00\x00\x00\x00SOSNEAKY\x00',
    '\x00\x00\x00\x00\x00\x00\x00\x00\x00S\x80\x80\x80\x80@\x80@\x00'])
```

```txt
Here's a breakdown of what posix.dumps(0) is doing in your example:

posix: Refers to the POSIX environment of the simulation state. This environment simulates aspects of a UNIX-like operating system, allowing the analysis of how a binary interacts with files, the standard input/output, and other OS-level features.

dumps(0): This method is called on the posix attribute to "dump" or read out the contents of a file descriptor. The argument 0 specifies which file descriptor to read from. In POSIX-compliant operating systems, file descriptor 0 is traditionally assigned to standard input (stdin), 1 to standard output (stdout), and 2 to standard error (stderr). Thus, dumps(0) attempts to read the contents that would have been read from standard input in the simulated execution.
```

Of course, `step`, `run`, and any other method that operates on a single stash of paths can take a `stash` argument, specifying which stash to operate on.



### Stash types

Stashes that will be used to categorize some special kinds of states.

| Stash         | Description                                                                                                                                                                                                                                                                                                                                                                                                         |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| active        | This stash contains the states that will be stepped by default, unless an alternate stash is specified.                                                                                                                                                                                                                                                                                                             |
| deadended     | A state goes to the deadended stash when it cannot continue the execution for some reason, including no more valid instructions, unsat state of all of its successors, or an invalid instruction pointer.                                                                                                                                                                                                           |
| pruned        | When using `LAZY_SOLVES`, states are not checked for satisfiability unless absolutely necessary. When a state is found to be unsat in the presence of `LAZY_SOLVES`, the state hierarchy is traversed to identify when, in its history, it initially became unsat. All states that are descendants of that point (which will also be unsat, since a state cannot become un-unsat) are pruned and put in this stash. |
| unconstrained | If the `save_unconstrained` option is provided to the SimulationManager constructor, states that are determined to be unconstrained (i.e., with the instruction pointer controlled by user data or some other source of symbolic data) are placed here.                                                                                                                                                             |
| unsat         | If the `save_unsat` option is provided to the SimulationManager constructor, states that are determined to be unsatisfiable (i.e., they have constraints that are contradictory, like the input having to be both “AAAA” and “BBBB” at the same time) are placed here.                                                                                                                                              |

There is another list of states that is not a stash: `errored`. If, during execution, an error is raised, then the state will be wrapped in an `ErrorRecord` object, which contains the state and the error it raised, and then the record will be inserted into `errored`. You can get at the state as it was at the beginning of the execution tick that caused the error with `record.state`, you can see the error that was raised with `record.error`, and you can launch a debug shell at the site of the error with `record.debug()`.

## Exploration Techniques

The archetypical example of why you would want an exploration technique is to modify the pattern in which the state space of the program is explored - the default “step everything at once” strategy is effectively breadth-first search, but with an exploration technique you could implement, for example, depth-first search.

To use an exploration technique, call `simgr.use_technique(tech)`, where tech is an instance of an ExplorationTechnique subclass. angr’s built-in exploration techniques can be found under `angr.exploration_techniques`.

Within `angr`, exploration techniques are strategies that guide the symbolic execution or other forms of analysis by defining how the analysis explores the state space of a program. The `angr.exploration_techniques` module provides several built-in techniques that can be applied to control the exploration process.


### Overview of some of the built-in ones ([`SimulationManager API`](https://docs.angr.io/en/latest/api.html#angr.sim_manager.SimulationManager),[`ExplorationTechnique API`](https://docs.angr.io/en/latest/api.html#angr.exploration_techniques.ExplorationTechnique "angr.exploration_techniques.ExplorationTechnique"))

- *DFS*: Depth first search, as mentioned earlier. Keeps only one state active at once, putting the rest in the `deferred` stash until it deadends or errors.

- *Explorer*: This technique implements the `.explore()` functionality, allowing you to search for and avoid addresses.

- *LengthLimiter*: Puts a cap on the maximum length of the path a state goes through.

- *LoopSeer*: Uses a reasonable approximation of loop counting to discard states that appear to be going through a loop too many times, putting them in a `spinning` stash and pulling them out again if we run out of otherwise viable states.

- *ManualMergepoint*: Marks an address in the program as a merge point, so states that reach that address will be briefly held, and any other states that reach that same point within a timeout will be merged together.

- *MemoryWatcher*: Monitors how much memory is free/available on the system between simgr steps and stops exploration if it gets too low.

- *Oppologist*: The “operation apologist” is an especially fun gadget - if this technique is enabled and angr encounters an unsupported instruction, for example a bizzare and foreign floating point SIMD op, it will concretize all the inputs to that instruction and emulate the single instruction using the unicorn engine, allowing execution to continue.

- *Spiller*: When there are too many states active, this technique can dump some of them to disk in order to keep memory consumption low.

- *Threading*: Adds thread-level parallelism to the stepping process. This doesn’t help much because of Python’s global interpreter locks, but if you have a program whose analysis spends a lot of time in angr’s native-code dependencies (unicorn, z3, libvex) you can seem some gains.

- *Tracer*: An exploration technique that causes execution to follow a dynamic trace recorded from some other source. The [dynamic tracer repository](https://github.com/angr/tracer) has some tools to generate those traces.

- *Veritesting*: An implementation of a [CMU paper](https://users.ece.cmu.edu/~dbrumley/pdf/Avgerinos%20et%20al._2014_Enhancing%20Symbolic%20Execution%20with%20Veritesting.pdf) on automatically identifying useful merge points. This is so useful, you can enable it automatically with `veritesting=True` in the SimulationManager constructor! Note that it frequenly doesn’t play nice with other techniques due to the invasive way it implements static symbolic execution.

### Using Exploration Techniques

To use an exploration technique in `angr`, you typically:

1. Instantiate an `angr.Project`.
2. Create a simulation manager (`simgr`) for the project.
3. Instantiate the exploration technique with any necessary parameters.
4. Apply the technique to the simulation manager using `simgr.use_technique(tech)`.

### Example: Using `Explorer` Technique

One common exploration technique is `Explorer`, which searches for a path to a specific address or set of addresses, avoiding others. Here's a basic example of how to use it:

```python
import angr
from angr.exploration_techniques import Explorer

# Load the binary into an angr project
proj = angr.Project("/path/to/binary", auto_load_libs=False)

# Create an initial state and a simulation manager for the project
initial_state = proj.factory.entry_state()
simgr = proj.factory.simgr(initial_state)

# Instantiate the Explorer exploration technique
# Here, we're looking for a path to 'find' address and avoiding 'avoid' address
find_addr = 0x1000  # Hypothetical address to find
avoid_addr = 0x2000  # Hypothetical address to avoid
explorer = Explorer(find=find_addr, avoid=avoid_addr)

# Use the exploration technique with the simulation manager
simgr.use_technique(explorer)

# Run the simulation manager
simgr.run()

# Check if we found the state
if simgr.found:
    found_state = simgr.found[0]
    print("Found a state that reaches the target address!")
    # Optionally, examine the found state
    # e.g., print(found_state.posix.dumps(0)) to print stdin of the found state
else:
    print("No state found that reaches the target address.")

```

### More on Exploration Techniques

`angr` includes various other exploration techniques, each tailored for different analysis goals, such as:

- `DFS` (Depth-First Search): Prioritizes depth in the exploration, exploring as far down a path as possible before backtracking.
- `LoopSeer`: Detects and handles loops by limiting the number of times a loop can be executed, useful for avoiding infinite loops during analysis.
- `Threading`: Allows parallel exploration of states, potentially speeding up the analysis.

### Custom Exploration Techniques

You can also define custom exploration techniques by subclassing `angr.exploration_techniques.ExplorationTechnique`. This involves implementing methods to hook into the simulation manager's stepping process, allowing for highly customized analysis behaviors.

### Conclusion

Exploration techniques in `angr` offer powerful ways to guide the analysis process towards specific goals, making `angr` a versatile tool for a wide range of binary analysis tasks. Whether using built-in techniques or developing custom ones, they provide an essential mechanism for controlling how `angr` explores the vast state space of complex binaries.

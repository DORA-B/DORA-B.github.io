---
title: Angr Execution Pipeline
date: 2024-02-28
categories: [Assembly]
tags: [assembly,python,angr]     # TAG names should always be lowercase
published: false
---
Angr Simulation Manager
===

Reference1: [Blog from csdn](https://blog.csdn.net/Palmer9/article/details/123919530?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-1-123919530-blog-79467268.235^v43^pc_blog_bottom_relevance_base5&spm=1001.2101.3001.4242.2&utm_relevant_index=4)
Reference2: Angr Source Code.

## Simulation Manager in Angr

```py
class SimulationManager:
    """
    The Simulation Manager is the future future.

    Simulation managers allow you to wrangle multiple states in a slick way. States are organized into "stashes", which
    you can step forward, filter, merge, and move around as you wish. This allows you to, for example, step two
    different stashes of states at different rates, then merge them together.

    Stashes can be accessed as attributes (i.e. .active).
    A mulpyplexed stash can be retrieved by prepending the name with `mp_`, e.g. `.mp_active`.
    A single state from the stash can be retrieved by prepending the name with `one_`, e.g. `.one_active`.

    Note that you shouldn't usually be constructing SimulationManagers directly - there is a convenient shortcut for
    creating them in ``Project.factory``: see :class:`angr.factory.AngrObjectFactory`.

    The most important methods you should look at are ``step``, ``explore``, and ``use_technique``.

    :param project:         A Project instance.
    :type project:          angr.project.Project
    :param stashes:         A dictionary to use as the stash store.
    :param active_states:   Active states to seed the "active" stash with.
    :param hierarchy:       A StateHierarchy object to use to track the relationships between states.
    :param resilience:      A set of errors to catch during stepping to put a state in the ``errore`` list.
                            You may also provide the values False, None (default), or True to catch, respectively,
                            no errors, all angr-specific errors, and a set of many common errors.
    :param save_unsat:      Set to True in order to introduce unsatisfiable states into the ``unsat`` stash instead
                            of discarding them immediately.
    :param auto_drop:       A set of stash names which should be treated as garbage chutes.
    :param completion_mode: A function describing how multiple exploration techniques with the ``complete``
                            hook set will interact. By default, the builtin function ``any``.
    :param techniques:      A list of techniques that should be pre-set to use with this manager.
    :param suggestions:     Whether to automatically install the Suggestions exploration technique. Default True.

    :ivar errored:          Not a stash, but a list of ErrorRecords. Whenever a step raises an exception that we catch,
                            the state and some information about the error are placed in this list. You can adjust the
                            list of caught exceptions with the `resilience` parameter.
    :ivar stashes:          All the stashes on this instance, as a dictionary.
    :ivar completion_mode:  A function describing how multiple exploration techniques with the ``complete`` hook set
                            will interact. By default, the builtin function ``any``.
    """

    ALL = "_ALL"
    DROP = "_DROP"

    _integral_stashes: Tuple[str] = ("active", "stashed", "pruned", "unsat", "errored", "deadended", "unconstrained")
```

## Simulation Manager in Angr, `Step` method

+ step method

```py
    def step(
        self,
        stash="active",
        target_stash=None,
        n=None,
        selector_func=None,
        step_func=None,
        error_list=None,
        successor_func=None,
        until=None,
        filter_func=None,
        **run_args,
    ):
        """
        Step a stash of states forward and categorize the successors appropriately.

        The parameters to this function allow you to control everything about the stepping and
        categorization process.

        :param stash:           The name of the stash to step (default: 'active')
        :param target_stash:    The name of the stash to put the results in (default: same as ``stash``)
        :param error_list:      The list to put ErroredState objects in (default: ``self.errored``)
        :param selector_func:   If provided, should be a function that takes a state and returns a
                                boolean. If True, the state will be stepped. Otherwise, it will be
                                kept as-is.
        :param step_func:       If provided, should be a function that takes a SimulationManager and
                                returns a SimulationManager. Will be called with the SimulationManager
                                at every step. Note that this function should not actually perform any
                                stepping - it is meant to be a maintenance function called after each step.
        :param successor_func:  If provided, should be a function that takes a state and return its successors.
                                Otherwise, project.factory.successors will be used.
        :param filter_func:     If provided, should be a function that takes a state and return the name
                                of the stash, to which the state should be moved.
        :param until:           (DEPRECATED) If provided, should be a function that takes a SimulationManager and
                                returns True or False. Stepping will terminate when it is True.
        :param n:               (DEPRECATED) The number of times to step (default: 1 if "until" is not provided)

        Additionally, you can pass in any of the following keyword args for project.factory.successors:

        :param jumpkind:        The jumpkind of the previous exit
        :param addr:            An address to execute at instead of the state's ip.
        :param stmt_whitelist:  A list of stmt indexes to which to confine execution.
        :param last_stmt:       A statement index at which to stop execution.
        :param thumb:           Whether the block should be lifted in ARM's THUMB mode.
        :param backup_state:    A state to read bytes from instead of using project memory.
        :param opt_level:       The VEX optimization level to use.
        :param insn_bytes:      A string of bytes to use for the block instead of the project.
        :param size:            The maximum size of the block, in bytes.
        :param num_inst:        The maximum number of instructions.
        :param traceflags:      traceflags to be passed to VEX. Default: 0

        :returns:           The simulation manager, for chaining.
        :rtype:             SimulationManager
        """
        l.info("Stepping %s of %s", stash, self)
        # 8<----------------- Compatibility layer -----------------
        if n is not None or until is not None:
            if once("simgr_step_n_until"):
                print(
                    "\x1b[31;1mDeprecation warning: the use of `n` and `until` arguments is deprecated. "
                    "Consider using simgr.run() with the same arguments if you want to specify "
                    "a number of steps or an additional condition on when to stop the execution.\x1b[0m"
                )
            return self.run(
                stash,
                n,
                until,
                selector_func=selector_func,
                step_func=step_func,
                successor_func=successor_func,
                filter_func=filter_func,
                **run_args,
            )
        # ------------------ Compatibility layer ---------------->8
        bucket = defaultdict(list)
        target_stash = target_stash or stash
        error_list = error_list if error_list is not None else self._errored

        for state in self._fetch_states(stash=stash):
            goto = self.filter(state, filter_func=filter_func)
            if isinstance(goto, tuple):
                goto, state = goto

            if goto not in (None, stash):
                bucket[goto].append(state)
                continue

            if not self.selector(state, selector_func=selector_func):
                bucket[stash].append(state)
                continue

            pre_errored = len(error_list)

            successors = self.step_state(state, successor_func=successor_func, error_list=error_list, **run_args)
            # handle degenerate stepping cases here. desired behavior:
            # if a step produced only unsat states, always add them to the unsat stash since this usually indicates bugs
            # if a step produced sat states and save_unsat is False, drop the unsats
            # if a step produced no successors, period, add the original state to deadended

            # first check if anything happened besides unsat. that gates all this behavior
            if not any(v for k, v in successors.items() if k != "unsat") and len(error_list) == pre_errored:
                # then check if there were some unsats
                if successors.get("unsat", []):
                    # only unsats. current setup is acceptable.
                    pass
                else:
                    # no unsats. we've deadended.
                    bucket["deadended"].append(state)
                    continue
            else:
                # there were sat states. it's okay to drop the unsat ones if the user said so.
                if not self._save_unsat:
                    successors.pop("unsat", None)

            for to_stash, successor_states in successors.items():
                bucket[to_stash or target_stash].extend(successor_states)

        self._clear_states(stash=stash)
        for to_stash, states in bucket.items():
            for state in states:
                if self._hierarchy:
                    self._hierarchy.add_state(state)
            self._store_states(to_stash or target_stash, states)

        if step_func is not None:
            return step_func(self)
        return self
```

+ `step_state` method

```py

def step_state(self, state, successor_func=None, error_list=None, **run_args):
    """
    Don't use this function manually - it is meant to interface with exploration techniques.
    """
    error_list = error_list if error_list is not None else self._errored
    try:
        successors = self.successors(state, successor_func=successor_func, **run_args)
        stashes = {
            None: successors.flat_successors,
            "unsat": successors.unsat_successors,
            "unconstrained": successors.unconstrained_successors,
        }

    except (SimUnsatError, claripy.UnsatError) as e:
        if LAZY_SOLVES not in state.options:
            error_list.append(ErrorRecord(state, e, sys.exc_info()[2]))
            stashes = {}
        else:
            stashes = {"pruned": [state]}

        if self._hierarchy:
            self._hierarchy.unreachable_state(state)
            self._hierarchy.simplify()

    except claripy.ClaripySolverInterruptError as e:
        resource_event(state, e)
        stashes = {"interrupted": [state]}

    except tuple(self._resilience) as e:
        error_list.append(ErrorRecord(state, e, sys.exc_info()[2]))
        stashes = {}

    return stashes
```


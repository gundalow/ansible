
.. contents:: Topics

Ansible execution begins with the script: `bin/ansible`. This script is sym-linked to other names, such as ansible-playbook and ansible-pull.

When executed, this script looks at the first parameter passed to it. On Linux/Unix systems, this parameter is the name of the script itself. By examining this value, we can determine which CLI class we should load and execute.

Ansible CLI Classes:

- AdHocCLI (the plain "ansible" command)
- PlaybookCLI (when run with "ansible-playbook")
- PullCLI (when run with "ansible-pull")
- DocCLI (when run with "ansible-doc")
- GalaxyCLI (when run with "ansible-galaxy")
- VaultCLI (when run with "ansible-vault")
- ConsoleCLI (when run with "ansible-console")


Each of these CLI classes is a sub-class of the CLI class found in `lib/ansible/cli/__init__.py`. This base class defines some common methods to all CLI classes, such as option parsing and loading the Ansible configuration file.

The entry point for the CLI classes is the `run()` method, which takes no arguments. In this document, we will cover how the PlaybookCLI class works. In many ways, the AdHocCLI and PullCLI classes function in a similar way to the PlaybookCLI class, however the other classes are very different and will not be covered here.

How Ansible Works
#################

When `run()` is called from the PlaybookCLI class, Ansible creates several objects which are singletons across the life of the run:

1. DataLoader() (`lib/ansible/parsing/dataloader.py`)
    .. literalinclude:: ../../../../lib/ansible/cli/__init__.py
       :language: python
       :start-after: STARTDOC: architecture_deep_dive:ID:cli_data_loader
       :end-before: ENDDOC: architecture_deep_dive:ID:cli_data_loader
2. Inventory ('lib/ansible/inventory/__init__.py`)
    .. literalinclude:: ../../../../lib/ansible/cli/__init__.py
       :language: python
       :start-after: STARTDOC: architecture_deep_dive:ID:cli_inventory
       :end-before: ENDDOC: architecture_deep_dive:ID:cli_inventory
3. VariableManager() (`lib/ansible/vars/__init__.py`)
    .. literalinclude:: ../../../../lib/ansible/cli/__init__.py
       :language: python
       :start-after: STARTDOC: architecture_deep_dive:ID:cli_variable_manager
       :end-before: ENDDOC: architecture_deep_dive:ID:cli_variable_manager
4. PlaybookExecutor ('lib/ansible/executor/playbook_executor.py`)


DataLoader
----------

The DataLoader class does the following jobs:

- Load YAML and JSON files in a consistent manner.
- Caching files which have been previously loaded.
- Manage decrypting vault-encrypted files.

The DataLoader class can parse YAML or JSON from an in memory string, or from a given path.

One final use of the DataLoader class is as an abstraction to make it easy for us to unit test and mock common file operations. Many methods defined in this class simply wrap `os` and `os.path` calls, however it is easier to mock them via this class than it is in the Python modules themselves. In our unit tests, we use the DictDataLoader class (found in `test/units/mock/loader.py`, which sub-classes DataLoader) to allow us to specify file content during testing.

VariableManager
---------------

The VariableManager codifies the variable precedence rules for Ansible. It allows Ansible to generate a "view" of variables, based on a set of given objects: the current play, the current host, and the current task. Any one of these may be omitted. Usually, vars are fetched with the following combinations of objects:

- (play,) (early on during playbook parsing time)
- (play, host) (some inventory specific parsing functions)
- (play, host, task) (task validation and execution)

The VariableManager also manages loading vars files for host and group vars, roles (`defaults/main.yml` and `vars/main.yml`), and the vars_files specified in any plays.

Finally, the VariableManager generates what we call "magic" variables, which are internal representations of Ansible state or other variables which cannot be defined anywhere else.

Inventory
---------

The Inventory class is responsible for loading and parsing the inventory from the specified source. We will cover the Inventory class and related objects later.

PlaybookExecutor
----------------

The PlaybookExecutor is (as the name implies) the main class used to execute the specified playbooks. This class creates the TaskQueueManager (more on that later) and when invoked via its `run()` method starts a nested loop::

    for playbook in playbooks:
        for play in playbook:
            ...

Ansible, at its core, is a series of nested loops. The outer-most of these is the loop over all playbooks given as arguments to the `ansible-playbook` command. The second loop iterates over each play in a playbook, as any given playbook can consist of one or more plays.

Within the second loop, PlaybookExecutor does the following tasks:

1. Sets the current base directory (basedir) to the directory containing the playbook.
2. Clears the inventory restrictions (more on that later).
3. Prompts for any variables, as specified in the vars_prompt for the current play.
4. Post-validates the play.

At this point, if a CLI option like `--list-tasks` or --lists-hosts` was specified, the current play is simply appended to a list for use later. Otherwise, we move on to actually running the play. PlaybookExecutor then does the following:

1. Updates internal records of failed an unreachable hosts (mainly done only after the first play in the list).
2. Uses the `serial` keyword to split the list of hosts into "batches". This is handled by the `_get_serialized_batches(play)` method in PlaybookExecutor.
3. Begins to loop over each "batch".

As noted above, we have now hit the third nested loop in the chain of Ansible execution::

    for playbook in playbooks:
        for play in playbook:
            for batch in batches:
                ...

By default there is only one batch, which contains all hosts for the play as specified by the `hosts:` keyword.

For each batch, PlaybookExecutor:

1. Sets the inventory restriction, so all future calls to `Inventory.get_hosts()` return the hosts in this batch.
2. Calls `TaskQueueManager.run()` (recall that the TQM was created in the PlaybookExecutor `__init__()`).
3. The TQM `run()` call returns an integer value, which is a bitmask representing several possible success and/or failure states. Certain failure states cause the PlaybookExecutor to break out of the playbook loop, represented by the `break_play` variable. This may also be triggered if the number of failed and unreachable hosts is equal to the total batch size.
4. Updates the counters used for previously failed and unreachable hosts for the next pass.

Upon failure, the PlaybookExecutor may (if Ansible is so configured), generate and write a "retry" file. This is partially handled by `_generate_retry_inventory()` in PlaybookExecutor.

Finally, `PlaybookExecutor.run()` returns the last result returned from `TaskQueueManager.run()`.

TaskQueueManager
----------------

In its original conception, the TaskQueueManager (or TQM for short) was meant to be the class which managed the multiprocessing aspects of Ansibles executor engine, but it could just as easily be called the PlayExecutor instead as its role has evolved.

The TQM has quite a few responsibilities today:

1. Contains the dictionaries used to keep track of the failed/unreachable hosts.
2. Contains the dictionaries used to keep track of listening and notified handlers.
3. Contains the stats object (`AggregateStats`) used to display the summary of results at the conclusion of a play's run.
4. Contains the lists of passwords used for things like vault-encrytped files.
5. Manages the list of active callback plugins loaded and in-use, as well as the currently set "stdout" callback plugin (the only one allowed to write things to the terminal). The TQM also defines the `send_callback()` method, which is used by dependent classes to send callbacks.
6. Manages the multiprocessing Queue object used by the workers to send results back to the main process.
7. Manages a lock file used by connection plugins via the PlayContext (more on this later).
8. Manages the slots used when starting worker processes.

Most of these things are created during the TQMs `__init__()`, however some things like the worker slots and callbacks are handled later.

As noted in the PlaybookExecutor section, the main entry point for the TQM is its `run()` method which takes a single argument - the current play being run.

When run, the TQM does the following:

1. Loads callbacks using the `load_callbacks()` method, if they had not been loaded previously.
2. Compiles the handlers in the play (which gets all of the handlers from the play and any roles referenced in the play). This information is then used by the `_initialize_notified_handlers()` call, which sets up the data structures mentioned above.
3. Calculates the maximum number of worker processes which should be started. This value is normally based on the `--forks` parameter (or ansible.cfg setting), but may be limited by the total number of hosts, or the size of the current batch.
4. Creates the PlayContext object (more on this later).
5. Creates the PlayIterator object (more on this later too).
6. As the TQM may be used by more than one play, but the iterator only lives for the life of one play, the TQM makes sure the PlayIterator knows which hosts have previously failed by calling `PlayIterator.mark_host_failed()` for all failed hosts. It also zeros out its list of currently failed hosts so we can hosts which specifically failed during this play.
7. If `--start-at-task` was specified, we check here to see if we've found the specified task. If not, this entire play has been skipped and Ansible will move on to the next play to find the requested task.
8. Loads the strategy plugin and runs it.
9. Updates the failed hosts data structure (used by PlaybookExecutor, as noted earlier).
10. Calls cleanup methods for TQM and the loaded strategy.
11. Returns the result from the strategy.

Strategy Plugins
----------------

In Ansible, strategy plugins were created to control how tasks were queued to remote systems. The base class for strategies (StrategyBase, found in `lib/ansible/plugins/strategy/__init__.py`), handles several common tasks:

1. Creates a background thread to pull results off the `final_q` (defined in the TQM above).
2. Defines the `_queue_task()` method, which handles starting up a worker process to run a task on a host (or blocking until a worker slot is available).
3. Reading results off the result queue and processing the results.
4. Executing any handlers which may have been notified.
5. Processing dynamically included task files and roles.
6. Provides common methods to add a host or group to the current inventory (used when processing results).
7. Handles running `meta` tasks, which are special internal tasks.

By far, the most important (and longest) process in StrategyBase is `_process_pending_results()`. This method dequeues any pending results which may have arrived and processes them. The wrapper method `_wait_on_pending_results()` also uses `_process_pending_results()` to read all outstanding results. A call to `_queue_task()` increments the count of outstanding requests, while `_process_pending_results()` decrements it, so `_wait_on_pending_results()` loops until the counter reaches zero. The use of strategy plugins occurs in single-threaded code, so the count will never increment while `_wait_on_pending_results()` is looping.

This method executes a `while True:` loop, which does the following (at a high level):

1. Acquires a lock for the results list (because the reading thread may be inserting results concurrently in the background thread), and attempts to `pop()` an item off the list. If this fails, we break out of the loop.
2. The result is a TaskResult object, so we use the string values for the task UUID and host name to look up the correct (original) objects.
3. We make a copy of the original task, and update its attributes based on the post-validated attributes contained within the TaskResult object (we do this to save having to re-template things in the main worker process).
4. Some TaskResults are received during loops from the TaskExecutor (more on that later), so if this result is one of these we fire off an appropriate callback and skip back to the top of the loop.
5. If the task was registering a result, we save the result using the VariableManager and the requested variable name (after cleaning the data of some internal keys we don't want exposed).
6. We then use an if/elif/else branch based on whether the task was successful, failed, or skipped, or if the host was unreachable via the requested connection method.
7. After this if/elif/else, we decrement the pending results counter and remove the host from the blocked list (which is used by some strategy plugins, such as the `free` strategy).
8. The current result is appended to a running list of processed results.
9. If the current task came from a role, and the task was not skipped, we set a flag on the given role (stored in the ROLE_CACHE dictionary) indicating that this role had a task run. This is used later to prevent roles from executing more than once.
10. If this method was invoked with a maximum number of passes set, check to see if we've exceeded that count and break out of the loop if so.
11. Finally, the above list of processed results is returned.

When a task fails:

1. We check for `ignore_errors`. If it is True, we just increment a couple of stats saying the task was 'OK' and/or 'changed'.
2. If we're not ignoring errors, we check to see if this is a "run once" task.
   - If yes, we make sure we fail every host in the current group, because a failed run-once task is a fatal failure for all hosts.
   - If not, we just fail the current host.
3. In all cases, the callback for failed tasks is sent.

For any failures, the failed host is added to the corresponding TQM dictionary, and `mark_host_failed` is called via the PlayIterator. This is necessary so that the PlayIterator can transition state to the rescue or always portion of a block, or to finish iterating over tasks completely.

When a host is unreachable:

1. The host is added to the unreachable hosts list.
2. The corresponding callback is sent.
3. The corresponding stats value is updated.

When a task is skipped:

1. The corresponding callback is sent.
2. The corresponding stats value is updated.

When a task is successful, quite a bit more happens. First, whether or not this task contained a loop, we generate a list of task results. By default, with no loop, this list will contain a single item, which is the dictionary data `_result` from the original TaskResult. When there is a loop, this list comes from `TaskResult._result['results']` instead.

For each result in this list (`result_items`), a loop does the following:

1. If the task is notifying a handler (and was 'changed'), we attempt to find the handler based on the name and/or listener string. If found, we append this host to the proper list contained within the TQM dictionary `_notified_handlers`.
2. If the task was `add_host`, we create a new host via the Inventory object.
3. If the task was `add_group`, we create a new group via the Inventory object.
4. If the result contains `ansible_facts` as a key, we use the VariableManager to save the returned facts in the correct location, depending on the module name and (when the task used `delegate_to:`) the host.
5. If the result contains `ansible_stats` as a key, we use the stats object in the TQM to update some statistics.

Once the sub-result loop is complete:

1. If the top-level result dictionary contains the key `diff`, the diff data (for files) is shown.
2. If this was not a dynamic include (for tasks or roles), the corresponding stats are updated.
3. The corresponding callback is sent.

The last important piece of StrategyBase is the `_queue_task()` method. This method originally did place tasks in a multiprocessing Queue object, but currently this method handles starting a WorkerProcess (defined in `lib/ansible/executor/process/worker.py`) to avoid limitations of serializing complex objects. The overall flow of this method is:

1. Creates a lock if necessary, which is used later during module formatting.
2. Creates a `SharedPluginLoaderObj` class, which was originally used to send PluginLoader updates to forked processes (but may now be unnecessary).
3. Attempts to find an open slot in the TQM workers list. If not slot is available, it will do a short sleep (to avoid a tight spin) and try again.
4. Increments the pending results counter.

Strategy Plugin Responsibilities
--------------------------------

As with many things, strategy plugins override the `run()` method. Strategy plugins built on top of StrategyBase are responsible for quite a few things on the controller side in this method:

1. Use the PlayIterator and Inventory objects to ensure all hosts (which have not failed) receieve every task in the order they are listed in the play.
2. Use `_queue_task()` to queue a task for a given host.
3. Use either `_wait_for_results()` or `_process_pending_results()` to fetch outstanding results.
4. Handle the processing and loading of dynamic includes for tasks, roles, etc.
5. Skipping tasks which come from a role which has already run (and does not allow duplicates).
6. Running `meta` tasks, which do not run through the full executor engine.
7. Finally, return the super `run()` method when done (`return super(StrategyModule, self).run(iterator, play_context, result)`).

PlayContext
-----------

The PlayContext is used to consolidate configuration settings for many aspects of a plays execution, including connection variables, become variables, and some other flags which may be set (such as verbosity).

The PlayContext also manages the precedence of these settings: Play < CLI Options < Task Parameters. This precedence is maintained by calling the following methods in order:

* `set_options(options)`
* `set_play(play)`
* `set_task_and_variable_override(task, variables, templar)`

The first two of these are pretty simple, however `set_task_and_variable_override()` does several complex things.

1. Makes a copy of itself. This is done for safety, and to make it a bit easier to use this method at the point of the executor engine which uses it. All operations after this point modify the copy, not the `self` object.
2. For all attributes on the task object passed in as a parameter, if there's a matching attribute on PlayContext we copy the value over.
3. If the task is using `delegate_to:`, we check the variables dictionary passed in for `ansible_delegated_vars` (created by VariableManager). These delegated variables are used to further override certain connection settings in the PlayContext.
4. The `MAGIC_VARIABLE_MAPPING` dictionary is used to update attributes based on possible variable name aliases. This uses the delegated vars from above first, and if those are not present it will look in the variables dictionary passed as a parameter.
5. Deals with legacy `sudo` and `su` variables and turns them into the corresponding `become` value.
6. Makes sure that, if we were using the `local` connection before taking overrides into account, we double-check to make sure the proper connection type and user is being used.
7. Sets become defaults, which will initialize values if they were not set.
8. Deals with "check-mode" options (currently including a deprecated option).
9. Returns the copied object from step 1.

Finally, the PlayContext manages formatting commands based on its internal become variables. This is handled in the `make_become_cmd()` method.

PlayIterator
------------

The PlayIterator is essentially a finite-state machine. Its primary job is to iterate over a list of blocks and the tasks contained within, which may have optional `rescue` and `always` portions. It uses states to determine which portion of the block from which the next task will come. In a block with no failures or nested blocks, the states will transition as follows::

    ITERATING_SETUP -> ITERATING_TASKS -> ITERATING_ALWAYS -> ITERATING_COMPLETE

If there is a failure in the setup or tasks portion of the block, the transitions will be::

    ITERATING_SETUP -> ITERATING_TASKS -> ITERATING_RESCUE -> ITERATING_ALWAYS -> ITERATING_COMPLETE

The PlayIterator maintains a dictionary, which uses host names for keys and `HostState` objects for values. Each `HostState` object contains:

- A copy of the list of blocks from the compiled play (more on this later).
- An index for each block portion (block/rescue/always), indicating which task the host is currently on.
- The current run state. These states are defined in PlayIterator (the `ITERATING_*` values shown above).
- The current failure state, which is a bitmask of possible values.
- The current dependency chain of roles.
- Three entries for tracking child state. These are used when a block section contains a nested block.
- A flag indicating whether the host has a `setup` result pending.
- A flag indicating whether the current state involved executing a `rescue` block.
- A flag indicating whether a task was found when `--start-at-task` was used from the command line.

The `HostState` object doesn't do much else beyond defining some helper methods, though the `copy()` call is used to safe guard the state when iterating with `peek=True`. Using the copied HostState, the PlayIterator can modify the state without prematurely advancing the host to the next task.

The PlayIterator class has multiple critical methods:

- `__init__`, which compiles the current play down to a list of blocks and creates the dictionary of states. It also handles searching the blocks for a task if `--start-at-task` was specified.
- `get_next_task_for_host`, which fetches the next task for a given host. `_get_next_task_from_state` does most of the work for this method.
- `mark_host_failed`, which sets the `fail_state` flag for the current host state. The `_set_failed_state` method does most of the work for this method.
- `is_failed`, which determines if the host is considered failed, based on its current block position and `fail_state` flag. The `_check_failed_state` method does most of the work here.
- `add_tasks`, which is used to insert blocks from dynamic includes into the state for a given host. The `_insert_tasks_into_state` method does most of the work for this.

The common thread here is that each of the main methods has a helper. The reason for this is because of nested blocks, which create recursive states. For each main method, the helper method may be called recursively due to the child states.

The `get_next_task_for_host` is the main method of PlayIterator. This method always returns a `(state, task)` tuple. If the given host is done iterating, the task will be set to `None`. This method:

1. Uses `get_host_state` to get a copy of the current state for the given host.
2. As an early exit, the state is checked for `ITERATING_COMPLETE`, and if so the `(state, None)` tuple is returned.
3. The helper `_get_next_task_from_state` is called to advance the state to the next task.
4. If we're not peeking at the next task, we save the modified state back to the dictionary of host states to make the advancement permanent.
5. The found task and state are returned as the tuple `(state, task)`.

This is quite simple, as `_get_next_task_from_state` does the heavy lifting here. The transitions between `ITERATING_TASKS`, `ITERATING_RESCUE`, and `ITERATING_ALWAYS` are very similar. This method functions as a `while True:` loop, which is broken out the first time a valid task is found, a failed state is found, or the `ITERATING_COMPLETE` state is hit. Within this loop:

1. The current block is fetched, based on the `state.cur_block` value.
2. If the `run_state` is `ITERATING_SETUP`:

   1. We check to see if the host is waiting for a pending setup task.
   2. If so, we clear this flag and move to `ITERATING_TASKS` by incrementing and/or reseting state counters.
   3. If not, we determine the gathering method and figure out if this host needs to gather facts, and set the task to the setup task.
3. If the `run_state` is `ITERATING_TASKS`:

   1. Clear the `pending_setup` flag if it's set.
   2. If the `tasks_child_state` is set:

      1. Call `_get_next_task_from_state` on the child state.
      2. If the child state has a failed state, we use the `_set_failed_state` helper to fail the current (parent) state and zero out the child state.
      3. If the child state returned `None` for a task, or if the child state reached `ITERATING_COMPLETE`, we also zero out the child state but instead of failing we continue back to the top of the while loop to try and find the next task from the advanced state.
   3. When there's no child state:

      1. If the current state is failed, advance the state to `ITERATING_RESCUE` and continue back to the top of the loop.
      2. If the current task index went past the end of the task list for this portion of the block, advance the state to `ITERATING_ALWAYS` and continue back to the top of the loop.
      3. Get the task from the current block section. If this task is a `Block` object, create a child state (starting in `ITERATING_TASKS`, because a child state will never run setup). We also clear the task so we don't break out of the loop and instead on the next pass will iterate into the child state. Otherwise, if the task is just a `Task`, we'll break out of the loop.
4. If the `run_state` is `ITERATING_RESCUE`, we pretty much do exactly the same thing as above, with two differences:

   1. We advance to `ITERATING_ALWAYS` on failures, or if we run out of tasks.
   2. If we did run out of tasks, it means we performed a rescue, so we reset the `fail_state` and set the `did_rescue` flag to True. This flag is used later to make sure we don't consider a state failed if it has done a rescue.
5. If the `run_state` is `ITERATING_ALWAYS, we again do the same thing as `ITERATING_TASKS`, with two exceptions:

   1. We advance to `ITERATING_COMPLETE` on failures, or if we run out of tasks.
   2. If we did run out of tasks, we advance the `cur_block` counter and reset all of the other state counters. If this block was an "end of role" block (and this role had a task run), we set a flag to use later to prevent roles from running more than once.
6. If the `run_state` is `ITERATING_COMPLETE`, we again return the `(state, None)` tuple.
7. If the task value here is not `None`, we break out of the loop and return the `(state, task)` tuple.

The `mark_host_failed` and `_set_failed_state` work similarly. Starting in the main function:

1. Uses `get_host_state` to get a copy of the current state for the given host.
2. Calls `_set_failed_state` to set the failure state.
3. Saves the modified state back to the state dictionary.
4. Adds the host to the `removed_hosts` list.

And in `_set_failed_state`:

1. If `run_state` is `ITERATING_SETUP`:

   1. Add `FAILED_SETUP` to the `fail_state` bitmask.
   2. Set the `run_state` to `ITERATING_COMPLETE`.
2. If `run_state` is `ITERATING_TASKS`:

   1. If there's a child state here, set the child state to the value returned by a recursive call to `_set_failed_state`.
   2. Otherwise, add `FAILED_TASKS` to the `fail_state` bitmask.

      1. If there's a rescue block, set the `run_state` to `ITERATING_RESCUE`.
      2. Otherwise we set the `run_state` to `ITERATING_COMPLETE`.
3. If the `run_state` is `ITERATING_RESCUE` or `ITERATING_ALWAYS`, we do the same thing as above except for the different failure values and the state to which we advance.
4. We return the newly modified state.

We noted above that each HostState object contains a copy of the block list. The reason we do this is because each host may (via dynamically included tasks or roles), end up with a different list of blocks upon which to iterate. This comes into effect when `add_tasks`/`_insert_tasks_into_state` are used. Again, these functions work similarly to the above, where we can recursively dive into child states. The main thing to note here is that the `Block` objects contained in each HostState are the same objects, even if the lists are different. So, as we insert new blocks into these lists, we need to make a copy of the target block we're inserting into so we don't impact other hosts. Here's an example from the `ITERATING_TASKS` section of `_insert_tasks_into_state`::

    if state.run_state == self.ITERATING_TASKS:
        if state.tasks_child_state:
            state.tasks_child_state = self._insert_tasks_into_state(state.tasks_child_state, task_list)
        else:
            target_block = state._blocks[state.cur_block].copy(exclude_parent=True)
            before = target_block.block[:state.cur_regular_task]
            after  = target_block.block[state.cur_regular_task:]
            target_block.block = before + task_list + after
            state._blocks[state.cur_block] = target_block

Worker Process
--------------

As noted above, when a strategy queues a task, Ansible creates a `WorkerProcess` to handle it. This class is a subclass of `multiprocessing.Process`, so we simply override the `run()` method.

When Ansible 2.0 was first being written, we originally started all workers at once and passed them things over a multiprocessing `Queue`. However, we quickly ran into speed and memory issues doing so, so we now create workers on the fly and they receive all shared-memory objects via the `__init__` call of `WorkerProcess`. The other thing `__init__` does is to create a copy of the stdin when a TTY is in use, or to make sure stdin is pointed at `/dev/null` when there is no TTY.

The implementation of `run()` is very simple, as it basically just creates a `TaskExecutor` object and immediately calls its `run()` method to execute the task. If successful, the resulting `TaskResult` object is put on on the shared queue (`rslt_q`) and the worker exits. If an exception was raised, we try and do a little special handling depending on the error, and put a custom-made `TaskResult` on the queue for the main thread to process.

Task Executor
-------------

As noted above, the `TaskExecutor` is the main classed used in the forked worker process. The main entry point is (of course) the `run()` method, which does the following:

1. Creates the loop items list using the `_get_loop_items()` method. If current task has no loop, the value will be `None`. If an error is raised here, the error is saved and deferred to later due to the fact that the task may be skipped via a conditional.
2. If there is a list of items to process from the list:
   1. `TaskExecutor` calls the `_run_loop()` method. This method calls `_execute()` in a loop and returns a list of results, which is then embedded in a dictionary result to become the `result['results']` value mentioned in the section on `_process_pending_results()` above.
   2. Each result is also checked to see if any sub-result was changed and/or failed, which determines whether the task overall was changed and/or failed.
   3. If the items list was empty (which can happen due to conditional filtering in `_get_loop_items()`), an appropriate result is generated.
3. If there is no list of items, the `_execute()` method is called directly.

Before we get to `_execute()`, we'll look at the chain of execution when looping. The call to `_run_loop` does the following:

1. Creates a list to keep track of results.
2. Determines the loop variable name. By default, this is `item` but can be set via the loop control object. We also do a check here to see if the loop variable name already exists in the variable dictionary and issue a warning to the user if so.
3. Items are squashed. This typically occurs for packaging modules so they can be executed in one pass instead of multiple, but it possible to configure other modules to be "squashable". This will be changing in 2.5.
4. Items are looped over:

   1. Add the item to the variable dictionary, using the loop variable name determined above.
   2. If there is a pause configured in the loop control and this is not the first pass, we pause for the configured amount of time.
   3. Create a copy of the Task object. Because later on we call `post_validate` on the task (which modifies the task in-place) and we're looping, we need a clean copy for each pass.
   4. Also create a copy of the PlayContext object for the same reason.
   5. Swaps the copied Task and PlayContext objects, call `_execute()`, and then swap the objects back.
   6. The result from `_execute()` above is updated with information about the loop item as well as a flag indicating it's a loop result, and a special TaskResult is sent back via the result queue to trigger a per-item callback (see the above flow for `_process_pending_results()` to see where this is used).
   7. The result is appended to the result list, and we remove the loop variable from the variable dictionary.
5. Finally, the list of results is returned.

The `_execute()` method is the main method used in `TaskExecutor`. This method:

1. Uses the internal variables (set in `__init__`) or those passed in via the method call (which happens when looping over items) and creates a `Templar` object for templating things later.
2. A flag is created to determine if there was an error validating the PlayContext object. We do this as the task may be skipped due to a conditional later, in which case this deferred error can be ignored.
3. The PlayContext is updated via the `set_task_and_variable_overrides()` mentioned above, and the object is post-validated (meaning all variables are finalized into values) using the `Templar` created above. We also use the variable dictionary to set any special variables the PlayContext may care about via the `PlayContext.update_vars()` method.
4. Evaluate conditionals. This will loop over the list of conditionals on the task, and if any evaluate to `False` the task will be skipped. If errors are raised here, we may return one of the deferred errors instead (which happened first and thus take precedence).
5. If the task was not skipped and we have any deferred errors, we re-raise them as they are fatal at this point.
6. If the task is a dynamic include (a task file or a role), we stop here and return a special result containing the variables from the include, which will be processed as described above in the `_process_pending_results()` section.
7. If this is a regular task, we now post-validate it using the `Templar` created above. This means any attributes containing variables will be templated to contain their final values. As noted above, this is done in-place, which is why we create a copy of the task object before calling `_execute()`.
8. The connection plugin is loaded.
9. The action plugin is loaded.
10. The `omit` value is pulled out of the variable dictionary, and any task attributes which equal this value are filtered out.
11. If using the do/until task syntax, we setup the number of loops, etc. to use. By default, there will be one loop with no pause so the task is executed at least once.
12. We make a copy of the variables, in case we need to update them with the value from a `register` on a task or some other reason.
13. Begin looping over the number of retries:

   1. Call the `run()` method of the loaded action plugin (referred to as the `handler`).
   2. If namespaced facts are enabled (in Ansible 2.4+), we move any returned facts to the special facts namespace.
   3. If the result contains an `rc` (return code) value and it is non-zero, we set the `failed` flag on the result to `True`.
   4. If the task was not skipped, we call the helper methods `_evaluate_failed_when_result` and `_evaluate_changed_when_result` to modify the result (if the user has specified `changed_when` or `failed_when` on the task).
   5. If this task is using a do/until loop, we evaluate the `until` conditional here to see if it has been satisfied. If not, start over at step #1 above. Whether we've succeeded or failed, a few extra flags are updated on the result to reflect the results of the do/until loop, in case this was the last retry attempt. Another per-item result callback is triggered here on a retry, similar to the per-item callback triggered in the item loop above.

14. We again save the `register` value into the vars and move namespaced facts (if necessary).
15. If this task is notifying a handler, we set a special internal variable (`_ansible_notify`) in the result dictionary with the values from the task.
16. If any delegated vars exist in the variable dictionary, we also add them to the dictionary result in the special variable named `_ansible_delegated_vars` for use in `_process_pending_results`.
17. Finally, we return the result dictionary.

Action Plugins
--------------

Action plugins create a layer between the Ansible controller and the remote system. All action plugins are sub-classes of `ActionBase` (defined in `lib/ansible/plugins/action/__init__.py`), which defines many methods necessary for running commands on remote systems. This makes sense when you consider that action plugins are very closely tied to connection plugins (discussed below).

The `ActionBase` class does the following:

1. Provides helper methods for remote file systems. For example:
   1. Determining if a remote file or directory exists (`_remote_file_exists`).
   2. Managing remote temp directories (`_make_tmp_path`, `_early_needs_tmp_path`, and `_remove_tmp_path`).
2. Using the module formatting methods to compile the module we will send to the remote system.
3. Creating the environment string, which is prepended to the remote command Ansible runs to execute the module.
4. Copying files (via the connection plugin) and managing remote permissions to ensure the files are accessible/executable as needed:
   1. `_transfer_file` - copies a file to the remote system.
   2. `_transfer_data` - which creates a temp file and uses `_transfer_file` to move it to the remote system.
   3. `_fixup_perms` (deprecated) and `_fixup_perms2`, used to ensure the permissions on the remote file/directory are correct.
   4. `_remote_chown`, `_remote_chmod`, and `_remote_set_user_facl` - used to ensure the correct ownership and/or accessibility of the remote file.
   5. `_remote_expand_user` - used to expand the `~` shell variable to a full path.
5. Fetching information about remote files:
   1. `_execute_remote_stat` - uses `_execute_module` (more on that below) to run the `stat` module on a remote path.
   2. `_remote_checksum` - uses `_execute_remote_stat` to get a remote checksum in a consistent and platform-agnostic way.
6. Inserts some special module arguments based on internal settings via `_update_module_args`.
7. Allows for executing raw commands on remote systems via `_low_level_execute_command`.
8. Defines a common method for running modules via `_execute_module`.
9. Defines a common method for parsing the data returned from a module execution into a Python data structure (`_parse_returned_data`).

Most of these methods are basic helpers, with `_low_level_execute_command` and `_execute_module` (which uses the former) being the main methods used.

Following the standard Ansible plugin convention, all action plugins derived from `ActionBase` override the `run()` method. The `normal` action plugin is the default, when an action plugin is not found matching the name of the module being executed (this is handled in the `TaskExecutor` during action plugin loading time, as mentioned above). Within their `run()` method, derived action plugins will eventually use `_execute_module` to run a module on a remote system. It is designed to be flexible though, so it is just as easy to run the original task delivered via `TaskExecutor` or any other module with any args. The `_execute_module` method:

1. First determines if any of the internal data has been overridden by parameters.
2. Builds the module using `_configure_module()`, which returns the module style, remote shebang, module data, and module path.
3. If pipelining is disabled, we check to see if we need a remote temp directory and make one if so. We also build the remote file name and upload path for the module.
4. If the `module_style` value returned by `_configure_module` is a special value ('old', 'non_native_want_json', 'binary'), we create an arguments file containing the module arguments and upload it in a format based on the `module_style` value.
5. The environment string is built using `_compute_environment_string()`.
6. A list of remote files to upload is built, to keep track of things we need to manage permissions on and possibly delete later.
7. If this task is an `async` task:
   1. Use `_configure_module` to build the `async_wrapper` module.
   2. Upload the above module package.
   3. Build the command string to execute it later.
8. Otherwise:
   1. If pipelining is enabled, we set the `in_data` value to the `module_data` returned by `_configure_module` above, otherwise the remote command is set to the remote path created earlier.   2. In certain situations, we may need to manually remove the temp directory later, so that is calculated next.
   3. The final command to run on the remote system is built using the associated connection plugin's shell plugin (more on this later).
9. Permissions are updated on all remote files in the list we created earlier using `_fixup_perms2`.
10. `_low_level_execute_command` is called to actually run the command we built on the target system.
11. The resulting data string is parsed using `_parse_returned_data` into a Python dictionary. This data is expected to be JSON, so this method mainly handles stripping out extraneous data from the module and turning the JSON it sent on stdout into this data. We also clean out some internal variables using `_remove_internal_keys()`.
12. If there is a temp directory to delete, it is deleted here.
13. The stdout and stderr values are split into lines and stored back into the result dictionary.
14. The result dictionary is returned.

As noted above, `_low_level_execute_command` is used to do the actual running of the module command on the remote system. However, this method is also used directly when we're doing something on the target system outside of Python and also serves as the basis of execution for the `raw` and `script` action/modules. This method:

1. Determines if the remote user is the same as the `become_user`. If not, it wraps the command with the proper become syntax using `make_become_cmd()` from the PlayContext (discussed above).
2. If the remote command allows a specific executable to be used (typically a shell command), we wrap the original command using `shlex.quote()` and prepend the remote executable.
3. We use the `exec_command()` method of the associated connection plugin to run the command on the target system.
4. The stdout and stderr values are massaged to ensure they're unicode text internally.
5. If no `rc` (return code) was set, we assume it was successful and set it to `0`.
6. The "success message" (used to determine if a `become` method was successful) is stripped from the output to avoid JSON parsing issues later.
7. The tuple `(rc, stdout, stdout_lines, stderr)` is returned.

Some good examples of action plugins using other modules to do work are the `template` and `assemble` action plugins.

Some good examples of action plugins not using modules at all (and never touching a remote system) are the `debug` and `add_host` action plugins.

Module Formatter
----------------

As noted above, the module formatter code is responsible for compiling the requested module (and any Ansible-provided dependencies) for transfer to the target system for execution. This code lives in `lib/ansible/executor/module_common.py` and, unlike many areas of Ansible, is entirely made of individual functions rather than a class. Another minor function of the formatting code is to ensure the correct "shebang" (`#!`) is used for the module code, where applicable.

This code deals with modules written in Python (as we ship them) as well as any other language (including Powershell &tm;). Because of this, there are essentially two main pathways the formatter may take.

1. For Python modules, the formatter will create a Python script with an embedded zip file contained within. Upon execution, this wrapper script writes the zip to a temporary file and imports the code contained within (which is the actual module and dependencies).
2. For all other modules, the formatter will determine the "style" of the module and try to find the requested module. For some styles (for example, with Powershell modules), the formatter will do simple string replacement on special strings to insert Ansible-provided common code. The result is a single large text file.

.. note::
   We use a wrapper script for Python modules due to the fact that Python 2.6 does not provide good support for executing zip files via the CLI python executable. We may revisit the use of this wrapper once Ansible no longer officially supports Python 2.6.

The main entry point to compile modules is the `modify_module()` method. This method:

1. Opens the file path to read the contents of the module file.
2. Calls `_find_module_utils()`, which handles finding all Ansible-dependencies regardless of the module type. This method also returns the data/style/shebang tuple which `modify_module()` will return at the end.
3. If the module is binary, we simply return the values above immediately. Otherwise, we attempt to create a shebang if none was found or we make sure the shebang is in bytes format.
4. Finally, the data/style/shebang are returned.


Connection Plugins
------------------

Connection plugins provide an abstraction between the Ansible controller and the target system. The controller plugin API is pretty simple:

1. `exec_command` - Run an arbitrary command on the target system.
2. `put_file` - Transfer a file to the target system from the controller.
3. `fetch_file` - Retrieve a file from the target system to the controller.
4. `_connect` - Create the connection to the target system. This is generally not called directly, and instead we use a decorator named `@ensure_connect` for the other API methods below to ensure the connection is established before doing anything else. The `_connected` flag should be set to `True` upon a successful connection.
5. `close` - Terminates the connection. This generally sets the `_connected` flag to `False`.

Each of these methods is a Python `@abstractmethod` on the base class `ConnectionBase`, so all derived classes must override them. However, in many cases `_connect` and `close` essentially do nothing (the `local` and `chroot` connection plugins are good examples of this).

Shell Plugins
-------------


Playbook Parsing and Classes
============================

Field Attributes
----------------

The Base Class
--------------

Mixin Classes - Conditional, Taggable and Become
------------------------------------------------

Parsing
-------

Loading Objects
---------------


Inventory
=========

The Inventory Object
--------------------

Host Objects
------------

Group Objects
-------------


Templar and the Templating System
=================================

Templar
-------

AnsibleJ2Template
-----------------

AnsibleJ2Vars
-------------

Safe Eval
---------

The Plugin System and Other Plugin Types
========================================

PluginLoader
------------

Cache Plugins
-------------

Callback Plugins
----------------

Filter and Test Plugins
-----------------------

Lookup Plugins
--------------

Vars Plugins
------------


Module Utilities and Common Code
================================

basic.py
--------

urls.py
-------


Miscellaneous Pieces and Helpers
================================

Ansible Error Classes
---------------------

Ansible Constants
-----------------


Utilities
=========

Display
-------

Unsafe Proxy
------------


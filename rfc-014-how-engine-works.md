# Workflow orchestration

# Principles

## Branches

Decider implements the methods below to decide what to do: 

    function handle() { ... }
    
    function onEvent(event) { ... }
    
    function onStart(work) { ... }
    
    function onSuccess(work, output) { ... }
    
    function onFailure(work, error) { ... }
    
    function onTimeout(work) { ... }

`handle` is the main method - others are optional and let Decider react to some notifications:

- `onEvent(event)`: is called each time an event is sent to an instance of this workflow
- `onStart(work)`: is called each time a `work` is started on a worker
- `onSuccess(work, output)`: is called each time a worker completed `work` , `output` is the return value
- `onFailure(work, error)`: is called each time a worker threw an `error` when running `work`
- `onTimeout(work)`: is called each time a worker has started `work` but did not complete nor fail within a timeout duration. Note: it does not mean that `work` will not eventually fail or succeed.

Branches are created:

- `handle`: *once*  on workflow start
- `onEvent`: *each time*  an event occured
- `onStart`,  `onSuccess`, `onFailure`, `onTimeout`: *each time*  a work is started/successful/failed/timed out

On Decider, each branch execution must be mono-threaded, and stops 

- when reaching a non-completed work
- or when reaching end.

So as soon as a branch implementation contains at least one work, this branch will be executed more than once.

## Example

Lets's look at this simple example:

    function handle()
    {
        [task1, task2, task3]->execute();
    		task4->execute();
    }

Execution history:

1. (Worker) branch 0 (`handle`) execution ends with decision to schedule `task1` , `task2` , `task3`. (Engine) schedule `task1` , `task2` , `task3`.
2. (Worker) first task (eg. `task2`) start execution. (Engine) this triggers creation of a new branch 1 (`onStart(task2)`) and schedule decision of branch 1. 
3. (Worker) `task1`  starts and (Engine) triggers creation of a new branch 2 (`onStart(task1)`) and schedule decision of branch 2.  Being ongoing, decision process waits.
4. (Worker) `task3`  starts and (Engine) triggers creation of a new branch 3 (`onStart(task3)`) and schedule decision of branch 3.  Being ongoing, decision process waits.
5. (Worker) branch 1 execution ends with no decision. (Engine) Scheduled decisions on branches 2 and 3 are sent together to Worker.
6. (Worker) branches 2 and 3 executions ends with no decision. (Engine) Decision process waits.
7. (Worker) `task3` is successful. (Engine) This triggers creation of a new branch 4 (`onSuccess(task3)`) and schedule decision of branches 4 and 0 (as `task3` belongs to this branch). Scheduled branches are sent to Worker.
8. (Worker) `task1` is successful. (Engine) This triggers creation of a new branch 5  (`onSuccess(task1)`) and schedule decision of branches 5 and 0 (as `task1`  belongs to this branch). Being ongoing, decision process waits.
9. (Worker)`task2` is successful. (Engine) This triggers creation of a new branch 6  (`onSuccess(task2)`) and schedule decision of branches 6 and 0 (as `task2`  belongs to this branch). As the decision is ongoing, nothing more is done.
10. (Worker) execution of branches 4 and 0 executions end with no decision. Scheduled branches 5, 0, 6, 0 are sent to Worker.
11. (Worker) execution of branches 5, 0, 6, 0 ends with a decision to schedule `task4`. (Engine) schedule  `task4.`
12. (Worker)`task4`  starts and (Engine) triggers creation of a new branch 7 (`onStart(task4)`) and schedule decision of branch 7. Scheduled branches are sent to Worker.
13. (Worker) execution of branch 7 ends with no decision. Decision process waits.
14. (Worker)`task4`  is successful. (Engine) This triggers creation of a new branch 8 (`onSuccess(task4)`) and schedule decision of branch 8 and 0 (as `task4`  belongs to this branch). Scheduled branches are sent to Worker.
15. (Worker) execution of branches 8 and 0 end with a `setOutput` decision. Engine decides to terminate this instance.

![](DecisionExample-2868e598-095d-4d23-93bd-d1a8a3338204.png)

# Engine Data  Structure

## `id`

id of this workflow instance

## `workflow_name`

Name of the class describing the decider

## `programming_language`

Programming language of the decider

## `branches`

Individual branches are stored with input values in `branches`:

    [
      %{
    		 type: "handle"|"onEvent"|"onStart"|"onSuccess"|"onFailure"|"onTimeout",
    	   event_name: (optional) <string>,
    	   event_input: (optional) <jsonified>,
    		 task_name: (optional) <string>,
    		 task_input: (optional) <jsonified>,
    		 task_output: (optional) <jsonified>,
    		 error_name: (optional) <string>,
    		 error_input: (optional) <jsonified>,
    		 output: (optional) <jsonified>,
    	},
      ...
    ]

Where:

- `type`: what triggered the branch
    - if "handle", then no additional attribute needed,
    - if "onEvent", then `event_name`, `event_input` describes the triggering event,
    - if "onStart", then `task_name`, `task_input` describes the triggering task,
    - if "onSuccess", then `task_name`, `task_input` describes the triggering task, and `task_output` the output,
    - if "onFailure", then `task_name`, `task_input` describes the triggering task, and `error_name`, `error_input` describes the error,
    - if "onTimeout", then `task_name`, `task_input` describes the triggering task.
- `output` is the return value of this branch.

## `executions`

The `executions`  array stores - in chronological order - the keys of branches already executed and currently being executed.

    [ 0, 1, 2, 0, 3, 4, ...]

In this document `index` refers specifically to this array. For a branch `n` we define `start_index` as the index of first appearance of `n` in `executions` array. With the example above, `execution_index_at_start` is `4`  for branch `3`, and `1` for branch `1` .

## `statuses`

Status of all (known) works  are stored in `statuses` 

    [
      %{
         branch_key: <integer>
         position: <string>
         hash: <string>
    		 status: "scheduled" | "successful" | (next) "failed"
    	   completion_index: (optional) <integer>
         output: (optional) <jsonified>
    		 (next) error_name: (optional) <string>
    		 (next) error_input: (optional) <jsonified>
    	},
      ...
    ]

where

- `branch_key`: work's branch key in `branches` array (corresponding branch is `branches[branch_key]`)
- `position`: work's position on this branch (as provided by decider when scheduling)
- `hash`: hash identifying the work (is unique / (name, input, branch, position, sync))
- `completion_index`:  index of first branch for which this work was completed (should be a `onCompleted` branch)
- `output`:  return value of the completed work

## `waits`

Array of running waiting processes (used only for restoring engine state)

    [
      %{
         hash: <string>
         timestamp: <integer>
    		 event: (optional) <string>
         (next) type: "wait" | "while"
      }
    ]

where:

- `hash` identifies the wait task within the workflow
- `timestamp`: timestamp of end of this wait task (`= begin + duration` if defined as a duration)
- `event`: name of event if any

## `properties`

Properties are all mutable variables inside workflow implementation. These variables can be modified during a branch execution.

`properties` is defined as such:

- `properties(0)` = initial workflow properties
- `properties(1)` = properties at end of 1st executed branch (when `index` is 0)
- `properties(n)` = properties at end of nth executed branch (when `index` is n-1)

![](properties-9061d1a3-0b9e-4883-8fe5-e907b3b0944b.png)

Note : to optimize memory usage (as properties can be quite big, as well as number of branch executions), we should store execution index and  properties only when properties change :

    [
      %{key:0, values: p0},
    	%{key:4, values: p1},
      %{key:9, values: p2}
    ]

such as properties(0) = p0, properties(1) = p0, properties(2) = p0, properties(3) = p0, properties(4) = p1, properties(5) = p1, ...

**Properties when starting a branch**

Properties when starting a branch `b`  are those at the end of the executed branch, ie. branch with index `start_index(b) - 1`. That can be rewritten as: given a branch `b` , properties when starting are `properties(start_index(b))` .

**Properties after work completion**

Given a work `w` on a branch `b`  after completion decider will execute at least two branches: 

- a new  `onSuccess`  branch that will start with `properties(index)`
- the `b`  branch itself
    - properties at start: `properties(start_index(b))`
    - properties after a succesful `work->execute()`: `properties(work.completion_index)`
    - properties after a succesful `parallel(workA, workB)->execute()`: `properties(max(workA.completion_index, workB.completion_index))`,
    - properties after a dispatch `work->dispatch()`, properties are unchanged.

## `buffer`

The `buffer` array stores what happens before a decision is triggered.

    [
    	%{timestamp: ..., type:"`onEvent`"|`"onStart`"|"`onFail`ure"|"`on`Success"|"`onTimeout`", data: ...},
    	...
    ]

Why use a buffer? 

- use case 1: a workflow beginning with a `wait(event)`. After the beginning of the workflow, during the decision duration, this `event` is fired. Without a buffer, this event will not release the `wait` task, as this task does not exist yet.
- use case 2: a workflow including a `wait().minutes(5)`. If for any reason (eg. worker off), the decision takes 10 minutes to execute. Without a buffer, this `wait` task will be released *after*  other instructions received in the meantime - that could lead to unexpected results.

Note: usually `timestamp`  is the timestamp at which the action is received by the engine. *There are some exceptions in the case of actions directly initiated by decisions*:

- `onStart` of wait tasks
- `onSuccess` of wait tasks already finished as soon as scheduled

For those, the `timestamp` value is the engine timestamp when the current decision was triggered.

# Engine Operation

## Starting a Workflow

`branches` 

    []

`executions`

    []

`ongoing_index` 

    null

`statuses`

    []

`properties` 

    [ %{key:0, values:<jsonified initial properties>} ]

`buffer` 

    [ %{timestamp: <current>, type: "handle", data: nil} ]

## Receiving decisions from Deciders

- received queued message
    - for each decision:
        - if `schedule_task` with data  `branch_key, position, hash, task_name, task_input, task_timeout, uuid`
            - create entry for this task in `statuses`
                - push `%{branch_key: branch_key,  position: position, hash: hash}` into `statuses`
            - send this task to workers
        - if `schedule_wait` with data  `branch_key, position, hash, event_name, task_timeout, uuid`
            - create entry for this task in `statuses`
                - push `%{branch_key: branch_key, position: position, hash: hash}` into `statuses`
            - add new action `onStart` in `buffer`
                - push `%{timestamp: ..., type:"onStart, data: ...}` to `buffer` where `timestamp` is the engine timestamp when the current decision was triggered.
            - if already terminated, add new action `onCompleted` in `buffer`
                - by event, if any, search for first (in time) element in `buffer` matching `event_name` => `byevent_timestamp`

                        Enum.filter(buffer, &(&1.type === "OnEvent" and &1.data.event_name === wait.event)) |> Enum.sort(&(&1.timestamp <= &2.timestamp)) |> List.first

                - by time, `bytime_timeout = task_timeout` (if < current timestamp)
                - if terminated
                    - push `%{timestamp: ..., type:"onCompleted, data: ...}` to `buffer` according to first occuring `byevent_timestamp` or `bytime_timeout`
            - if not terminated, start wait process  (from time of current decision triggering) and add entry to `waits` array
        - if `update_properties` with data `branch_index`, `branch_properties`
            - update `properties`  array
                - push `%{index: branch_index, properties: branch_properties}`  to `properties`
        - if `set_output`
            - update `output` for current branch in `branches`
- If at least one decision
    - **Try to trigger a decision**
    - save `states`
- acknowledge queued message.

## Receiving actions from Workers `onEvent`, `onStart`, `onFailure`, `onSuccess`, `onTimeout`

- received queued message
    - push this action into `buffer` with timestamp of reception
    - if `onEvent`:
        - check if this event terminates a wait(Event), if yes add an `onSuccess` action into `buffer`
        - **(next)** check if this event prolongs a waitWhite(Event)
- **Try to trigger a decision**
- save `states`
- acknowledge queued message.

## Try to trigger a decision

- if `decision_ongoing == false` and `decision_mode == run` and `length(buffer)>0`
    - **prepare decision**
    - **trigger decision**

## Prepare Decision

- `ongoing_index = length(executions)`
- for each `action` in `buffer`, by timestamp and reception order:
    - plan executions of new branches:
        - `branch_key = length(branches)`
        - `branch_index = length(executions)`
        - push `%{type: :..., data:...}` to `branches` array
        - push `branch_key`  to `executions`
    - if type `onCompleted`, store result and execute corresponding branch:
        - update `statuses` with `completion_index = branch_index` and `output`
        - if task is synchronous, push `work.branch_key`  to `executions`
- `buffer = []`

## Trigger Decision

`decision_ongoing = true` 

Send to worker the following data:

- `uuid`  <string>
- `workflow_name`  <string>
- `programming_language` <string>
- `properties` [%{properties: <jsonified>, key: <integer>}...]
- `executions` [<integer>]
- `ongoing_index` <integer>
- `ongoing_branches`:  [%{branch: %{ <branch> }, key: <integer>}...]

    branches with key in `ongoing_executions = Enum.take(executions, ongoing_index - length(executions))`  

        ongoing_`branche`s = branches
        	|> Enum.with_index()
        	|> Enum.filter(fn({branch,key})-> Enum.member?(`ongoing_executions`, key) end)
        	|> Enum.map(fn({branch, key}) -> %{branch: branch, key: key} end)

- `ongoing_statuses`: [%{ <status> }...]

    status of works in `ongoing_executions`

        ongoing_statuses = `statuses`
        	|> Enum.filter(&(Enum.member?(`ongoing_executions`, &1.branch_key)))

# Restoring Engine

Current instance state is saved on a table `States` with

- `id` <integer>
- `custom_id` <string>
- `workflow_name` <string>
- `programming_language` <string>
- `branches` <json>
- `executions` <json>
- `statuses` <json>
- `properties` <json>
- `buffer` <json>
- `ongoing_index` <integer>
- `decision_mode`
- `decision_ongoing`
- `decision_current_index`

    To restore Engine, you should be able to restore those states + rebuild the living processes:

- waiting process: start wait process for each element in `waits` that is not already successful.

# Worker

![](scripts-4f9ceaab-aafa-4175-9e6d-6e95ff4721d6.png)

## Distributing jobs

For each `uuid`, a worker runs a process where all data related to this decision is maintained. Scripts will request those data through `[http://localhost:4001](http://localhost:4001/env/{envId}/lang/{lang}/job)` :

- `/env/{envId)/lang/{lang}/job`  will return next available job
    - for a decision

            `{action: "DecisionScheduled", uuid: <uuid>, name: <flow_name> }` 

    - for a task

            `{action: "TaskScheduled", uuid: <uuid>, name: <task_name>, input: <task_input>, hash:<task_hash>}` 

    - **(next)** if none

            {action: "None"}

    Note: programming_language is not specified when received as queues are language-specific. 

**Endpoints** 

- `get /decisions/{uuid}/branch`
    - if `(ongoing_index < length(executions))`, then return next branch to execute (in `executions` from `ongoing_index` to end):

            `{branch: <branch>, properties: <values>}`

        where <values> are initial properties of <branch> (`Enum.find(ongoing_branches, &(&1["key"] == ongoing_index))`) .

    - if `(ongoing_index >= length(executions))`  then return

            {}

         and send stored decisions to engine

- `post /decisions/{uuid}/branch` with body `%{properties: <jsonified>, output (optional): <jsonified>}` is sent at branch execution's end.
    - worker will check if properties changed from `properties(ongoing_index)`, and if yes,
        - will schedule a "updateProperties" decision with key = ongoing_index
        - add a `%{values: new_properties, key: ongoing_index}` in  `properties` list.
    - if `output` is present, worker will schedule a "setOutput" decision with branch_key = executions(ongoing_index)
    - `ongoing_index++`
- `post /decisions/{uuid}/execute` with an array of works as body: `[ %{name: , input: , position: , timeout: , sync: , type: , event: }, ... ]`. Worker response:
    - For each work
        - if position is unknown in this branch, this is a new work:
            - add a new status in `statuses` list
            - add a "scheduleTask" / "scheduleFlow" decision with branch_key = ongoing_index and
            - Add a status "scheduled" to `loop_result`
        - if position is known but hash is different, then decider implementation was modified, stops and returns a status: "modified"
        - if position is known and hash is the same: the work has already been scheduled, worker checks if it's now completed
            - if `(status == "completed") AND (completion_index <= ongoing_index)`then  return "completed", output and completion_index
            - or else return a status "scheduled"
    - if at least one of works has been modified (different hash from name/input/position/sync), worker response:

            `{status: "modified"}`

    - if at least one of works is not completed, worker response:

            `{status: "scheduled"}`

    - if all works are completed, worker response:

            `{status: "completed", properties: <jsonified>, outputs: [<jsonified>]}`

        where properties = properties(max[completion_index..]) and outputs = [output..]

## Task jobs

**Data sent from engine to worker**

- `uuid`
- `task_name`
- `task_input`

## Endpoints

- `post /tasks/{uuid}`
    - with body `{status: "completed", output: <jsonified>}` after task completion. A `taskCompleted` action is sent to Engine with `output`
    - with body `{status: "failed", error: {code: , name: , message: , stacktrace: }}`  after task failure. A `taskFailed` action is sent to Engine with `output`

# Scripts

Decider launches branch execution based on these data.

- When reaching an `execute(task)`,
    - if `task` is unknown, then Decider decides to schedule this task and leaves this branch
    - if `task` is already scheduled but not completed, Decider leaves this branch
    - if `task` is completed, `output` is returned and execution continues
- When reaching an `execute(taskA, taskB)`:
    - for each `task`
        - if `task` is unknown, then Decider decides to schedule this task and continues
        - if `task` is already scheduled but not completed, Decider continues
        - if `task` is completed, `output` is returned and execution continues
    - if all tasks are completed, return `[output...]` and continue
    - or else leaves branch
- When reaching an `dispatch(task)`
    - if `task` is unknown, then Decider decides to schedule this task and continues
    - if `task` is already scheduled but not completed, Decider continues
    - if `task` is completed, Decider continues
- 

## Decision endpoints:

`**give_branch(uuid)**`

Give the first ongoing_branch to the script and delete the given branch on the list. If there is no branch to give, it is the signal to terminate the decision and to send a `CompleteDecision` message to the engine.

    def handle_call(:give_branch, _from, state = %{decision_scheduled: decision_scheduled, name: name, ongoing_branches: ongoing_branches, properties: properties}) do
            case Enum.empty?(ongoing_branches) do
                true ->
                    # Send CompleteDecision
    								# Return an empty array 
                    {:reply, [], state}
                false ->
                    branch_to_execute = List.first(ongoing_branches)
                    branch_to_send = %{
                        name: name,
                        properties: get_properties_for_decision(properties, branch_to_execute["start_index"]),
                        type: branch_to_execute["type"],
                        data: branch_to_execute["data"]
                    }
                    {:reply, branch_to_send, %{state | ongoing_branches: List.delete(ongoing_branches, branch_to_execute)}}
            end
        end

**`execute(uuid, works)`**

will execute each ongoing branch one by one.
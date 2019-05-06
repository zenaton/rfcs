# Zenaton Scheduling Syntax

## Problem

Which syntax is proposed to users for defining recurrent tasks/workflows ?

## Inspirations

- Java: http://www.quartz-scheduler.org
- Laravel: https://laravel.com/docs/master/scheduling
- Node: https://github.com/agenda/agenda
- Airflow : https://airflow.apache.org/scheduler.html
- Prefect: https://docs.prefect.io/guide/core_concepts/schedules.html#overview
...

## Discussion

### Preamble
A previous version of this RFC proposed a *static* version of a scheduler - ie. something like [Laravel scheduler](https://laravel.com/docs/master/scheduling). But the discussion converged towards an approach where task/workflows are scheduled *dynamically/programmaticaly*.

For such a choice, the state of scheduling is not maintained anymore within a well-identified file. So we must provide a clear syntax to schedule tasks/workflows but also to update/cancel a scheduling.

This RFC was a bit complex to write as I want to be sure - as far as possible - that our choices are future-proof. And this subject is at the crossroads of many others:
- new syntax for task/workflow to avoid "modifiedDeciderException" unwanted behavior - https://github.com/zenaton/rfcs/blob/task_new_syntax/task_new_syntax.md
- new syntax for wait (good proposal from PY) - https://github.com/zenaton/rfcs/blob/major-php-lib-overhaul/rfc-000-major-php-library-overhaul.md
- new syntax to allow microservices (situation where we do not know the task/workflow implementation when dispatching it) - in particular how to set 'id' ? We need to set it at dispatch if we want to request state at any time
- feature request from users to programatically requests state of a task / workflow
- how to set task/workflow settings such as 'schedule to finished' timeout - you need to set this sort of timeout during dispatch as it includes the time spent in queues
- how to delay a task like in https://laravel.com/docs/5.8/queues#delayed-dispatching

### Dispatch and Execute
When you think about it, `dispatch` and `execute` actions are different beasts:

- `dispatch` can be used everywhere (inside and outside workflows)
- it's fondamentally an instruction for a distributed, asynchronous processing
- this instruction describes the intention of the user: in practice, this execution can succeed, failed, never be completed, retried, (skipped/canceled)... The processing history of this instruction can so be complex
- local execution does not make sense really
- delayed or recurrent `dispatch` DO make sense
- return value is certainly *not* the result of the execution

At the opposite,
- `execute` can only be used inside a workflow
- it's fondamentally an helper for processing workflows
- this instruction describes the intention to proceed the workflow sequentially
- local execution can make sense (if a workflow is processed locally)
- delayed or recurrent `execute` do NOT make sense
- return value (if any) should be those of the task/workflow
- there is an underlying dispatch for an execute, with its own execution history

### Id
An id provided by the user (`custom_id` for us) means that the user wants later access to this object. Eg. to be able to send an event to a workflow, or also to know the status of a dispatched task or workflow (request formulated multiple times by users : "how to i know the status of execution?"). From the discussion above, it means that the `id` refers to the task/workflow dispatched (the underlying dispatch for `execute`), that - again - can have a complex history execution. 

this `id` serving a reference, *it should be unique* (at least per task or workflow and per environnement). 

A clean implementation implies that the `id` is provided when dispatching in order to be able to assess the success or not. In case of microservices, the `id` can not be read from class implementation and MUST be provided at dispatch. 
 
### Options
Today, no option are provided, but when looking to similar service, we see that adding options to dispatching is inevitable:
- eg. `schedule_to_finished_timeout` 
- `child policy` cf. https://docs.aws.amazon.com/amazonswf/latest/developerguide/swf-dev-adv-child-workflows.html

Some of those options make sense even before the instruction reachs an Agent, this implies we should provide them at dispatch. 

### Syntax Constraints

`Execute` syntax constraints:
- should be usable without knowing task implementation 
- should be able to set:
    * class name
    * arbitrary number of arguments
    * options such as `id`
- as we want to return an arbitrary value - we can not use method chaining without having a special closing method.


`Dispatch` syntax constraints:
- should be usable without knowing task implementation 
- should be able to set:
    * class name
    * arbitrary number of arguments
    * options such as `id`, or `timeout`
- should be able do set delay
- should be able to set scheduling (recurrent execution)
- if we want to return a useful value - such as a object - we can not use method chaining without having a special closing method.

## Proposal
This proposal is a target: we may need intermediate steps if it's to much to do at once.

### Client API
The Agent in client mode should be replaced by (or proposed with) an API (eg. at https://agent.zenaton.com). The rationals behind that are:
- Easiness for users. Installing the agent on web servers has been an underestimated pain - especially for people using docker (a lot of them) or serverless (more and more).
- Necessity to have a value returned in some cases:  eg. when trying to create a dispatch with an already existing id.

This implies to have an always on and scalable API - I would tend toward a serverless implementation. 

### Syntax For Execute

Inspired by PY proposed syntax for Wait  
`{$name}::with($args...)->getReturn();`  
OR equivalent for microservices  
`Task::name($name)->with($args...)->getReturn();`  

Example:  
`SendWelcomeEmail::with($user, $from)->getReturn();`  
OR equivalent for microservices  
`Task::name("SendWelcomeEmail")->with(user, $from)->getReturn();`  

By using a final `getReturn` method, we are free to use chaining methods to define additional options :   
`Task::name("SendWelcomeEmail")->with(user, $from)->id($user->id)->getReturn();`


### Syntax For Wait
*I'm not sure about the `get` method, suggestion welcome*

Wait for a duration  
`Wait::for()->...->get();`

Wait for a datetime  
`Wait::until()->...->get()`

Wait for an event  
`Wait::event($eventName)->getEvent();`

Wait for an event, with a maximal duration  
`Wait::event($eventName)->for()->...->getEvent();`  

Wait for an event, up to a datetime
`Wait::event($eventName)->until()->...->getEvent();`  

Examples:  
- `Wait::for()->hours(3)->minutes(30)->get();`  
- `Wait::until()->monday()->at("8:00")->get();` 

### Syntax For Recurrent Scheduling of a task / workflow
 
I propose a unique syntax for dispatching a task/ workflow, with or without a delay, recurrent or not. To handle all those cases, we use the `schedule` word:

`{$name}::with($args)->schedule();`  
OR equivalent for microservices  
`Task::name($name)->with($args...)->schedule();`

Derivatives:  
`{$name}::with($args)->delay()->...->schedule();`  
`{$name}::with($args)->repeat()->...->schedule();`  

Same syntax for `Workflow::name(...`  

Examples:  
`SendWelcomeEmail::with($user)->schedule();`  
`SendWelcomeEmail::with($user)->delay()->for()->hours(2)->schedule();`  
`SendWelcomeEmail::with($user)->delay()->until()->monday()->at('8:00')->schedule();`  
`SendWeeklyReport::with($user)->repeat()->cron('* * * * *')->schedule();`  
`SendWeeklyReport::with($user)->repeat()->forever()->schedule();`  

OR equivalent for microservices 
`Task::name('SendWeeklyReport')->with($user)->repeat()->cron('* * * * *')->schedule();`  

Important: when the agent API will be available, the result of `schedule` method could be a `Scheduled` object

### Scheduled object
-----

** This part needs to be matured **

`->schedule()` returns an Scheduled instance containing all info relative to the scheduled task/workflow:

- task or workflow name
- arguments
- id 
- time type (immediate, delayed, repeated) and details
- datetime of scheduling
- history details (*format to define, it should include all retries* - *for a recurrent scheduling, provide details of each iteration*)

Note: 
- A `Scheduled` object can be obtained through `Scheduled::whereTask($name)->whereId($id)->find();`
- cancelling a task / workflow schedule can be done through : `Scheduled::whereTask($name)->whereId($id)->cancel();` (or `Scheduled::whereWorkflow($name)->whereId($id)->cancel();`) 
- we can imagine that `Scheduled::repeat()->get();` will provide a view of all recurrent tasks / workflows

Warning: there is an ambiguity in the chosen vocabulary:
- 'scheduled' points to to the user intention (eg. the email to be sent in 30 minutes)
- while internally 'scheduled' means "sent to queues for processing")
It can be very misleading in our code.

### Options Precedence
It can be useful (for backward compatibility or for more elegant syntax) to be able to define options inside task or workflow.

The following rules will apply to determinate value of an option or `id` at scheduling:
- the value provided at scheduling (if any) is used
- if none and task/workflow implementation is known, then value provided in task/workflow (if any) is used
- if none, then no value is used (default being managed by server)

Note that some options (such as maxProcessingTime) will be used by the Agent. For them also the value provided in task/workflow (if any) is used if no value was provided at scheduling => The Agent must not confond default value provided by server with value provided at scheduling (especially if task implementation in unknown at scheduling).

For sake of simplicity (user point of view - not ours !), I suggest that all options can be define at scheduling AND within task or workflow with a common syntax.

Example:  
`SendWelcomeEmail::with($user)->options(['maxProcessingTime' => 300])->schedule();`  
or  
`public function getMaxProcessingTime() { return 300; }`

Note: I hesitate with `SendWelcomeEmail::with($user)->maxProcessingTime(300)->schedule();`  

## Short Term

To avoid to make too much changes, a first implementation of recurrent executions could be:

`(new $name($args...))->repeat()->cron('* * * * *')->schedule();`

With a graphical dashboard of all recurrent scheduling, where the user can manually cancel a recurrent scheduling

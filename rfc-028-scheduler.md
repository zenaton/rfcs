# Zenaton Scheduler

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

### Prealable
A solution to manage a scheduler is to set everything in a UI. (like easycron.com) - but the philosophy of Zenaton is to provide everything by code.

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
An id provided by the user (`custom_id` for us) means that the user wants to access to this object. It can be to be able to send an event to a workflow, but also to know the status of a dispatched task or workflow (request formulated multiple times by users : "how to i know the status of execution?"). From the discussion above, it means that the `id` refers to the task/workflow dispatched (the underlying dispatch for `execute`), that - again - can have a complex history execution. 

this `id` serving a reference, *it should be unique* (at least per task or workflow and per environnement). 

A clean implementation implies that the `id` is provided when dispatching in order to be able to assess the success or not. In case of microservices, the `id` can not be read from class implementation and MUST be provided at dispatch.  
### Options
Today, no option are provided, but when looking to similar service, we see that adding options to dispatching is inevitable, eg.:
- `skippable` to tell if the task can be skipped
- `schedule_to_finished_timeout` 
- `child policy` cf. https://docs.aws.amazon.com/amazonswf/latest/developerguide/swf-dev-adv-child-workflows.html

Some of those options make sense even before the instruction reachs an Agent, this implis we MUST provide them at dispatch. 

### Syntax Constraints

`Execute` syntax constraints:
- should be able to ignore task implementation 
- should be able to set:
    * class name
    * arbitrary number of arguments
    * options such as `id`, or `skippable` 
- should return an arbitrary value (as result in a worklow)


`Dispatch` syntax constraints:
- should be able to ignore task implementation 
- should be able to set:
    * class name
    * arbitrary number of arguments
    * options such as `id`, or `timeout`
- should be able do set delay
- should be able to set scheduling

## Proposal
This proposal is a target: we may need intermediate steps if it's to much to do at once.

### Client API
The Agent in client mode should be replaced by (or proposed with) an API (eg. at https://agent.zenaton.com). The rationals behind that are:
- Easiness for users. Installing the agent on web servers has been an underestimated pain - especially for people using docker (a lot of them) or serverless (more and more).
- Necessity to have a value returned in some cases:  eg. when trying to create a dispatch with an already existing id.

This implies to have an always on and scalable API - I would tend toward a serverless implementation. 

### Proposed Syntax For Wait

Inspired by PY proposed syntax for Wait (except that we suppose to be able to interpret a string)

`Wait::forever();`  // should be avoided  
`Wait::for('3 hours');`  
`Wait::until('Monday')`  
`Wait::event(OrderPaidEvent::class)->forever();`  // should be avoided  
`Wait::event(OrderPaidEvent::class)->for('3 hours');`  
`Wait::event(OrderPaidEvent::class)->until('Friday');`  

Note that a wait instruction do not have / need `id` or `$options`

### Proposed Syntax For Execute

Inspired by PY proposed syntax for Wait.

`Execute::task($name, $args, $options);`  

Example: 

`Execute::task('SendWelcomeEmail, $user);`

`Execute::task('SendWelcomeEmail, $user, ['id' => $user->id, 'skippable' => true]);` ($args can be an array OR a unique value)


### Proposed Syntax For dispatch

We propose a unique syntax for dispatching a task/ workflow, a delayed task / workflow or a recurrent task / workflow. To handle all those cases, we use the `schedule` word:

`Schedule::task($name, $args, $options)->now();`  
`Schedule::task($name, $args, $options)->delay('30 minutes);`  
`Schedule::task($name, $args, $options)->cron('* * * * *');`
`Schedule::task($name, $args, $options)->forever();`

same syntax for `Schedule::workflow(...`  

Example:

`Schedule::task('SendWelcomeEmail, $user)->delay('30 minutes);`

### Schedule object
-----

`Schedule::task` and `Schedule::workflow` returns an Schedule instance containing all  info  relative to this scheduled task/workflow

while `Execute::task` return the (future) value of the task. The underlying `Schedule` object must be requested, through
`Schedule::whereTask('SendMail')->whereId($id)->find();` 


## Additional Possible Features

### Local Scheduling

Ability to define a local scheduler for current agent (it may be directly processed by the agent itself)

This is the opposite of https://laravel.com/docs/5.8/scheduling#running-tasks-on-one-server

We may use:
````php
->onAgent();            | Run on this agent                  |
->onAgent('foobar');    | Run on the agent with name 'foobar'|
````


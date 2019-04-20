# My RFC Title

## Summary

This RFC describe how our scheduler will work

## Problem

Which syntax is proposed to users for defining recurrent tasks/workflows? 

## Inspirations

- Java: http://www.quartz-scheduler.org
- Laravel: https://laravel.com/docs/master/scheduling
- Node: https://github.com/agenda/agenda
- Airflow : https://airflow.apache.org/scheduler.html
- ...

## Discussion

Note: a solution to manage a dispatcher is to set everything in a UI. (like easycron.com) - but the philosphy of Zenaton is to provide everything by code - this has a lot of advantages:
- it can be versioned
- it allow an easy management of different environments (dev, staging,  production)
- it consistent with developer habits
- it's a trend
So we discuss here how to do it with code.

From a usage perspective, it seems dangerous that doing `dispatch(task).everyMinute()` twice (eg. from n distinct agents), would dispatching n `task` every minute. => A scheduler seems in essence a singleton. And actually it is his purpose to keep in a single place the definition of recurrent jobs. 

Also, once I have defined a set of recurring tasks/workflows - how can I update it consistently from different servers ? => it seem the most obvious way is to reset the settings and replace it entirely by the new ones.

For this same reason, it seems reasonnable to have an explicit command to set the scheduler.

## Proposal

### Commands
Settin the scheduler will be done through the command
```
zenaton schedule
```
In case of global maintenance, we may want to stop the scheduling
```
zenaton unschedule
```
(for local maintenance, a simple `unlisten` is enough).

then zenaton should be able to find one (and only one) class implementing `Zenaton\Interfaces\SchedulerInterface` and use it to set the scheduler. Eg.

### Scheduler definition

Having the ability to setup a cron syntax is the simplest first version:

```php
<?php

use Zenaton\Interfaces\SchedulerInterface;
use App\Tasks\TaskA;
use App\Workflows\WorkflowA;

class Scheduler implements SchedulerInterface
{
    public function schedule()
    {
        TaskA::schedule(data...)->cron('* * * * *');

        WorkflowA::schedule(data...)->cron('* * * * *');
    }
}
```
We have used `TaskA::schedule(data...)` syntax instead of `(new TaskA(data...))->schedule()` as we know that it should be the final syntax for dispatching also (cf. https://github.com/zenaton/rfcs/blob/task_new_syntax/task_new_syntax.md). But we may need to update agent and librairies to handle that (TBD).

### Helpers (WIP - TO BE IMPROVED)

Then we could have helpers similar to Laravel:

|Method                 | Meaning|
|-----------------------|--------|
|->everySeconde();      | Run the task every second|
|->everySecondes();     | Run the task every second|
|->everySecondes(30);   | Run the task every 30 seconds|
|->everyMinute();       | Run the task every minute|
|->everyMinutes(2);     | Run the task every 2 minutes|
|->everyMinutes(5, 30); | Run the task every 5 minutes at 30 seconds|
|->hourly();            | Run the task every hour|
|->hourly(2);           | Run the task every 2 hours|
|->hourly(2, 30, 10);   | Run the task every 2 hours at 30 minutes and 10 seconds|
|->daily();             | Run the task every day at midnight|
|->daily(2);            | Run the task every 2 day at midnight|
|->daily(2, 10, 0, 30); | Run the task every 2 day at '10h0m30s'|
|->weekly();            | Run the task every week|
|->weeklyOn(1, '8:00'); | Run the task every week on Monday at 8:00|
|->monthly();           | Run the task every month|
|->monthlyOn(4, '15:00');| Run the task every month on the 4th at 15:00|
|->quarterly();         | Run the task every quarter|
|->yearly();            | Run the task every year|
|->timezone('America/New_York');| Set the timezone|


## Additional Possible Features

### Local Scheduling
Ability to define a local scheduler for current agent (it may be directly processed by the agent itself)

This is the opposite of https://laravel.com/docs/5.8/scheduling#running-tasks-on-one-server

We may use:
````php
->onAgent();            | Run on this agent                  |
->onAgent('foobar');    | Run on the agent with name 'foobar'|
````

### Env variable
Define which class I want to use (if more than one) through a cli option `--definition` or a env variable such as `ZENATON_SCHEDULER_DEFINITION`

### Always Running
How to schedule a task/workflow to have it always running (ie. repeat itself after completion)?

We may use:
````php
->TaskA::schedule(data...)->always(); 
->WorkflowA::schedule(data...)->always(); 
````
In this case, the Task will be rescheduled as soon as completed. In case of a workflow, we should implement in a way that the instance never stops, but reset its history between 2 completions.

### Completion Check
Sometimes we need to ensure that previous processing is completed, before running a new one. 

Eg. synchronizing a database with Algolia (true case!): each minute I request the dbb and launch the synchro - if it is not ended after 1 minute, then the data that end up to Algolia can be outdated at data (m+1) can be replaced by data(m)

This is equivalent to https://laravel.com/docs/5.8/scheduling#preventing-task-overlaps

We may use `->withoutOverlapping()` to not dispatch the new task/workflow while previous one is still running or `->queuedIfOverlapping()` to dispatch the new task/workflow only after previous one is completed.

### Programmatic Scheduling
How to programatically set or update scheduling? Not sure it's actually a good idea, as it may render very difficult to know what is running at a given time. We may need an actual use case to think about that.




# New Syntax

## Problem

Current syntax has issues:
- only adapted to monolith (Laravel-inspired), where code is the same on all servers
- internal properties are included in modification checks, leading to false positive `ModifiedDeciderException`
- do not include new features such as 
  - scheduling
  - delays
  - workflow state introspection 

## Proposal

After *a lot* of iteration, I converged toward the syntax below.

## Scheduling Tasks

### Scheduling for immediate processing

`Schedule::task('SendMail')->args(...)->now();`

`...` are task's arguments.

### Scheduling delayed for a duration

`Schedule::task('SendMail')->args(...)->delayed()->for($duration);`

$duration is an integer (seconds). We may provide a `Duration` class, to allow
syntax such as `Duration::hours(2)->minutes(10)`.

### Scheduling delayed until a date time:  

`Schedule::task('SendMail')->args(...)->delayed()->until($date);`

$date is an DateTime. We may provide a `Date` class, to allow
syntax such as `Date::monday()->at('8:00')`.

### Scheduling repeated ...:  

`Schedule::task('SendMail')->args(...)->repeated($each);`

$each is a cron expression. We may provide a `Each` class to allow
syntax such as `Each::day()->at('8:00')`.

Reapeated task or workflow are a bit difficult to handle conceptually.
Thinking them as always running workflows with task/sub-workflows may help. 

## Scheduling Workflows

Same syntax that tasks, except that we use `Schedule::workflow`, for example:

`Schedule::workflow('SendMailSerie')->args(...)->now();`

## Scheduling Options

Today, no option are provided, but when looking to similar service, we see that adding options to dispatching is unavoidable:
- eg. `schedule_to_finished_timeout` 
- `child_policy` cf. https://docs.aws.amazon.com/amazonswf/latest/developerguide/swf-dev-adv-child-workflows.html

Some of those options make sense even before the instruction reachs an Agent, this implies we should provide them at scheduling. 

It can be useful (for backward compatibility or for more elegant syntax) to be able to define options inside task or workflow.

The following rules will apply to determinate value of an option at scheduling:
- the value provided at scheduling (if any) is used
- if none and task/workflow implementation is known, then value provided in task/workflow (if any) is used
- if none, then no value is used (default being managed by server)

Note that some options (such as `maxProcessingTime`) will be used by the Agent. For them also the value provided in task/workflow (if any) is used if no value was provided at scheduling => The Agent must not confond default value provided by server with value provided at scheduling (especially if task implementation in unknown at scheduling).

For sake of simplicity (user point of view - not ours !), I suggest that all options can be define at scheduling AND within task or workflow with a common syntax.

`Schedule::workflow('SendMailSerie')->args(...)->options($options)->now();`

or inside task or workflow
```
public function options()
{
    return $options;
}
``` 
With (example) `$options = [Task::MaxProcessingTime => 300]`

We could also offer the options to set default values:  
`Schedule::workflow('SendMailSerie')->setDefaultOptions($options);`

## Scheduling's Output

Any schedule will synchronously output a `Task` or `Workflow` object with:
- name
- id (provided by sdk)
- args
- tags
- delay (default:0)
- repeated (default:none)

`id` (our `schedule_id`) is the (unique) most interesting value, as user may store is for future usage. 

## Tagging

When scheduling, we can assign a tag to a task or a workflow:

`Schedule::task('SendMail')->args(...)->tag($tag)->now();`

$tag is a string, eg. 'id:1781b8c7'. We may add multiples tags, by repeating `tag` method.

Note: user does not have anymore the ability to provide an `id` - this is replaced by tags.

Note: when tagging a repeated workflow:  
`Schedule::task('SendMail')->args(...)->tag($tag)->repeated();`  
the tag is applied to the repeatable, *not* the underlying instances.

## Selecting

### By Id

We can find a task or a workflow by its `id` (our `schedule_id`)

- select task by Id  
    `Task::whereId($id)`  

- select workflow by Id  
    `Workflow::whereId($id)` 

Output will be a unique instance.

### By name

We can select instances of tasks or workflows by name: 

- select task by name  
    `Task::whereName($name)`  

- select workflow by name  
    `Workflow::whereName($name)` 

Output will be a collection of instances.

### By Tags

We can select instances of tasks or workflows by tags: 

- select task by tag  
    `Tasks::hasTag($tag)`  

- select workflow by tag   
    `Workflow::hasTag($tag)`

Output will be a collection of instances.

### Combined

Except `whereId()`, other selection criteria can be mixed, for example:

- select tasks by name and tag  
    `Tasks::whereName($name)->hasTag($tag)`  
- select by multiple tags (AND)  
    `Tasks::hasTag($tag1)->hasTag($tag2)`
- select by tag (OR)  
    `Tasks::hasTag($tag1)->orTag($tag2)`

## Commands on selection

On selection of tasks or workflows, we can
- `id()` retrieve `id` (our `schedule_id`)
- `history()` retrieve processing history
- `skip()` skip processing (useful only for tasks within workflows)
- `cancel()` cancelling  (useful only for delayed and repeated)

On selection of workflows only:
- `pause()` pause selected instances
- `resume()` resume selected instances
- `terminate()` terminate selected instances
- `send($event)` send $event to selected instances

From a task and workflow implementation:
- `$this->id()` provides own `id` (our `schedule_id`)
- `$this->history()` retrieve own history
- `$this->skip()` skip own processing (useful only for tasks or workflows within workflows)

From a workflow implementation:
- `$this->pause()` pause itself
- `$this->resume()` resume itself (useless?)
- `$this->terminate()` terminate itself
- `$this->send($event)` send event to itself 

### History command (Tasks and Workflows)

On a selection of tasks or workflows, we can retrieve processing history, eg.
    - name
    - id
    - args
    - tags
    - delay (default:0)
    - repeated (default:none)
    - *history*

As a user, we can use this to display a status of processing.

### Cancel command (Tasks and Workflows)

Cancelling tasks are best effort (possible only when not yet sent to queues, and for repeated).

## Inside a workflow

#### Scheduling in workflow

All scheduling syntax should be working inside a workflow. It implies that, inside a workflow, we should manage scheduling idempotency.

#### Executing

For "executing" jobs, ie. by waiting the output to continue workflow execution, we do:

`Execute::task('SendMail')->args(...);`

and for (sub-)workflow:

`Execute::workflow('SendMail')->args(...);`

There is no way to delay with this syntax, as user should use provided `Wait` class.

#### Waiting

Wait forever  
`Wait::forever();`

Wait for a duration  
`Wait::for($duration);`

Wait for a datetime  
`Wait::until($date);`

Wait for an event  
`Wait:event($eventName)->forever();`

Wait for an event, with a maximal duration  
`Wait:event($eventName)->for($duration);`

Wait for an event, up to a date time  
`Wait:event($eventName)->until($date);`

Note: $duration and $date are used as for delayed tasks.

#### Commands in workflow

All acting syntax should be working inside a workflow. It implies that, inside a workflow, we should manage acting idempotency.



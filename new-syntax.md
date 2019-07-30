# New Syntax

## Problem

Current syntax has issues:
- only adapted to monolith (Laravel-inspired), where code is the same on all servers
- internal properties are included in modification checks, leading to false positive `ModifiedDeciderException`
- do not include new features such as 
  - scheduling
  - delays
  - workflow state introspection 

# Dispatching

## Task or Workflow?

We consider that - from the orchestrator point of view - there should be no way to know if a job will be processed as a task or as another (sub)workflow. That's why, the syntax should not make any distinction between both.

Something to be done inside a workflow is always called a job. Zenaton will know if it's a task of a workflow only when the name will be resolved by an Agent (so it's not possible to have task and workflow with same name).

We do not make any assumption neither on the language used, that's why only the name and the data are transfered. A good practice for the user should be to avoid language-specific data structure.

** I BELIEVE THAT OUR CURRENT DEVELOPMENTS DO NOT ALLOW THIS ABOVE
SO `job` BELOW SHOULD BE UNDERSTAND AS `task` or `workflow` UNDIFFERENTLY **

## Dispatching for immediate processing

```php
Dispatch::job($name, Array $data = []);
```

`$name` is a string and `$data` is an array representing job's arguments (eg. `new $name(...$data)`, ie. `[]` means no argument, `[1,2]`means 2 arguments `1` and `2`). To simplify, we will just write `$data` below, instead of `Array $data = []`).

## Delayed Dispatching (duration)

```php
Dispatch::in($duration)->job($name, $data);
```

$duration is an integer (seconds). We may provide a `Duration` class, to allow syntax such as  `Duration::hours(2)->minutes(10)`.

## Delayed Dispatching (date time)

```php
Dispatch::at($date)->job($name, $data);
```

$date is an DateTime. We may provide a `Date` class, to allow
syntax such as `Date::monday()->at('8:00')`.

## Dispatch's Tagging

When dispatching, we can assign one tag or more:

```php
Dispatch::tag($tag)->job($name, $data);
```

`$tag` is a string, eg. `'id:1781b8c7'`. We may add multiples tags eg. `->tag($tag1, $tag2, $tag3)`.

Tags will be mainly use to target jobs to apply an instruction or request a status.

Note: user does not have anymore the ability to provide an `id` - this is replaced by tags.

We could also offer the options to set default tags for a job:

```php
Dispatch::defaultTag($tag)->job($name);
```

## Dispatch's Options

When looking to similar service, we see that adding options to dispatching is unavoidable, eg.
- `dispatch_to_start_timeout`
 
Theoritically, none of these options should be dependant of being a task or a workflow - if you find a conter-example - please tell me (Gilles).

So,
```php
Dispatch::options($options)->...;
```
With (example) `$options = [Job::DispatchToStartTimeout => 300]`

Later, we could offer the options to set default values for a job:

```php
Dispatch::defaultOptions($options)->job($name);
```

## Job Options
Other options can be defined specifically in workflow ot task implementation.

Example for workflows:
- `child_policy` cf. https://docs.aws.amazon.com/amazonswf/latest/developerguide/swf-dev-adv-child-workflows.html

Example for tasks:
- `maxProcessingTime`

These options will be defined inside jobs by
```php
public function options()
{
    return $options;
}
``` 
With:
- (workflow example) `$options = [Workflow::ChildPolicy => "TERMINATE"]`
- (task example) `$options = [Task::MaxProcessingTime => 600]` 

## Dispatch's Output

Any dispatch will synchronously output a `Job` object with:
- id (provided by sdk)
- name
- data
- tags
- options
- delay (default:0) // buffer ?

`id` (our `intent_id`) is the (unique) most interesting value, as user may store is for future usage. 

## Selecting Jobs

Different filters are provided to filter jobs:

- by Id   `Job::whereId($id)->...`  
- by name `Job::whereName($name)->...`  
- by tag: `Job::hasTag($tag)->...`  

When applied, this targets a collection (containing 0 or more element)

Notes:
- `$name` could also be a regular expression.
- `hasTag($tag1, $tag2)->...` operates as a logical AND operator
- `$tag` could be a regular expression (allowing logical OR).
- filters can be mixed, for example `Job::whereName($name)->hasTag($tag)` will select jobs by name and tag

## Applying Commands On Jobs Selection

On a selection of jobs, we can
- `id()` retrieve `id`s (our `intent_id`s)
- `parent()` select parent (if exists) 
- `history()` retrieve processing histories
- `skip()` skip processing (makes sense only if a parent exists)

From a task and workflow implementation:
- `$this->id()` provides own `id` (our `schedule_id`)
- `$this->parent()` select parent (if exists)
- `$this->history()` retrieve own history
- `$this->skip()` skip own processing (useful only for jobs within workflows)

On selection of workflows only:
- `pause()` pause selected instances
- `resume()` resume selected instances
- `terminate()` terminate selected instances
- `send($eventName, $eventData)` send event to selected instances

From a workflow implementation:
- `$this->pause()` pause itself
- `$this->resume()` resume itself (useless?)
- `$this->terminate()` terminate itself
- `$this->send($eventName, $eventData)` send event to itself 

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

# Inside a workflow

## Commands

All commands syntax should be working inside a workflow. It implies that, inside a workflow, we should manage commands idempotency.

## Dispatching

All dispatching syntax should be working inside a workflow. It implies that, inside a workflow, we should manage dispatching idempotency.

## Scheduling

All scheduling syntax should be working inside a workflow. It implies that, inside a workflow, we should manage scheduling idempotency.

## Executing

For "executing" jobs, ie. by waiting the output to continue workflow execution, we do:

```php
Execute::job($name, $data);
```

- there is no way to delay or repeat an execution.
- *tags* can be used, with same syntax than for dispatching.
- *options* can be used, with same syntax than for dispatching.

## Time Waiting

Waiting forever

```php
Wait::forever();
```

Waiting for a duration  

```php
Wait::for($duration);
```

Waiting for a datetime  

```php
Wait::until($date);
```

Note 1: with above syntax, the return value is always `null`.

Note 2: `$duration` and `$date` are the same as used for delayed tasks. 

## Events

Events are explicitly *sent* to workflows: 
- for a choregraphy, they can be sent to all workflows:
`Workflow::send($eventName, $eventData)`. 
- for an orchestration, they can be sent to a specific workflow:
`Workflow::whereId($id)->send($eventName, $eventData)`.

Within a workflow, receiving an event `$eventName` will:
1) trigger exit of an ongoing `Wait` instruction specific to this event
2) trigger execution of an `on{$eventName}` method, if it exists
3) do nothing if not 1) or 2)

### Waiting for an event

Waiting for an event without time limitation (without restriction on $eventData)  
```php
Wait::event($eventName)->forever();
```

Waiting for an event, with a maximal duration (without restriction on $eventData)  
```php
Wait::event($eventName)->for($duration);
```

Waiting for an event, up to a date time (without restriction on $eventData)  
```php
Wait::event($eventName)->until($date);
```

Waiting multiple events (AND)
```php
Wait::event($event1Name, $event2Name, ...)->forever();
```

Output of the wait event:
```php
[$eventName, $eventData] = Wait::event($eventName)->for($duration);

[[$event1Name, $event1data], [$event2Name, $event2data]] = Wait::event($event1Name, $event2Name)->for($duration);
```

If the `Wait` terminates without event, then `$name` and `$data` will be `null`

Later we could introduce an `EventFilter` class such as `Wait::event($eventFilter)` to more specialized filter such as OR operand on name,or filter on data.

### Reacting on Event

We propose to deprecate the `onEvent` method, as this implementations has some drawbacks

- it's quite verbose as it always begins with a test of event's name
- as it does not allow event auto-discovery, it may generates useless decision if an event is not implemented

Workflows can now implement a method `on{EventName}` to decide what to do when receiving an event:

```php
public function on{EventName} ($eventName, $eventData) {}
````

For example: `public function onInvoicePaid('invoicePaid', $data)` will be called only when receiving a 'invoicePaid' event. It is more readable and allows also to ask a decision only when the listener is implemented.

## Dispatched job management

### Waiting for job completion

We propose to introduce a new feature inspired from Durable Functions from Azure.

As discussed below, this new feature allow a more natural implementation of parallel processings. With

```php
$job = Dispatch::job($name, $data);
```
then
```php
$output = Wait::completion($job)->forever();
```
will go through only when `$job` processing is succesfully completed. 

### Parallel processings
Waiting for a parallel completion becomes:
```php
$job1 = Dispatch::job($name1, $data1);
$job2 = Dispatch::job($name2, $data2);
$job3 = Dispatch::job($name3, $data3);

[$output1, $output2, $output3] = Wait::completion($job1, $job2, $job3)->forever();
```

### Reacting on Completion

We propose to deprecate the `onSuccess` method, as this implementations has same drawbacks than `onEvent`.

Workflows can now implement a method `on{jobName}Completion($job)` to decide what to do when a job is completed:

```php
public function on{JobName}Completion ($job) {}
```

For example: `public function onPayInvoiceCompletion($job)` will be called only when a `payInvoice` job is completed. `$job` will be an object containing all information relative to $job processing including the output.

### Reacting on Failure

We propose to deprecate the `onFailure` method, as this implementations has same drawbacks than `onEvent`.

Workflows can now implement a method `on{jobName}Failure($job)` to decide what to do when a job is failed:

```php
public function on{JobName}Failure ($job) {}
````

For example: `public function onPayInvoiceFailure($job)` will be called only when a `payInvoice` job is failed (after automatic retry). `$job` will be an object containing all information relative to $job processing including the error.

## Executing API from a 3rd Party Service

Here syntax is given in node.js.

### From an HTTP API

With `const { execute, schedule } = require("zenaton");`

Sync request
```javascript
await execute.verb(url, body);
```

Async request
```javascript
await schedule.verb(url, body);
```

Examples:
```javascript
await execute.post('https://slack.com/api/chat.postMessage', {...});

await schedule.post('https://slack.com/api/chat.postMessage', {...});
```

### From a SDK

With `const { execute, schedule } = require("zenaton");`

Sync request

```javascript
await execute.task('service:method', $data);
```

Async request

```javascript
await schedule.task('service:method', $data);
```

Examples:

```javascript
await execute.task('aws.s3:getObject', $data);
```

```javascript
await schedule.task('aws.ses:SendRawEmail', $data);
```

```javascript
await execute.task('slack:web.chat.postMessage', $data);
```

## Listening Events from a 3rd Party Service

As events come from a third party, we should provide a way for workflows to 
describe which event they listen.

### From inside a workflow

#### Listening all events
```javascript
await this.listen('service');
```

#### Listening a specific event
```javascript
await this.listen('service:event_name');
```

#### Listening a specific event with specific data
```javascript
await this.listen('service:event_name', {email: 'foo@bar.com'});
```

#### Listening a specific event but only related to an object created by a HTTP API

```javascript
await execute.listen('service:event_name').post(url, body);
```

Example:
```javascript
await execute
        .listen('slack:reaction_added')
        .post('https://slack.com/api/chat.postMessage', body);
```

#### Listening a specific event but only related to an object created by a SDK

```javascript
await execute.listen('service:event_name').task('service:method', $data);
```

Example:
```javascript
await execute
        .listen('slack:reaction_added')
        .task('slack:web.chat.postMessage', $data);
```

### From outside a workflow

Same syntax, but begining with a selection, eg.
```javascript
await Workflow.whereName('Retention').listen('service:');
await Workflow.whereName('Retention').listen('service:event_name');
await Workflow.whereName('Retention').listen('service:event_name', {email: 'foo@bar.com'});
```

## Waiting Events from a 3rd Party Service

With `const { wait } = require("zenaton");`

Waiting all events
```javascript
await wait.event('service:').forever();
```

Waiting a specific event
```javascript
await wait.event('service:event_name').forever();
```

Waiting a specific event with specific data
```javascript
await wait.event('service:event_name', {email: 'foo@bar.com'}).forever();
```

## Example

Post a message and remove any reaction to it for 1 hour:

```javascript
const { Workflow, schedule, execute, wait } = require("zenaton");

module.exports = Workflow("PostSlackMessageWithoutReaction", {
  async handle() {
    await dispatch
        .listen('slack:reaction_added')
        .post('https://slack.com/api/chat.postMessage', {
            channel: 'C1234567890',
            text: 'try to react ont this!)'
        });
    await wait.for(3600);
  },
  onSlackReactionAdded(data) {
      await execute.post('https://slack.com/api/reactions.remove', {timestamp: data.timestamp});
  }
})
```


# Scheduling

### Syntax

We use the word scheduling for a request of reccurent dispatching:

```php
Schedule::each($cron)->job($name, $data);
```
$cron is a cron expression. We may provide a `Each` class to allow
syntax such as `Each::day()->at('8:00')`.

### Schedule's tags

Tags can be provided and will be applied to future dispatched jobs.

### Schedule's options

Options can be provided and will be applied to future dispatched jobs.

### Schedule's delay

Not possible yet.

### Schedule's Output

Any scheduling will synchronously output a `Schedule` object with:
- id (provided by sdk)
- name
- data
- tags
- options
- active (default:true)

`id` (our `intent_id`) is the (unique) most interesting value, as user may store is for future usage. 

### Schedule's Selecting 

Different filters can be applied and mixed:
- `Schedule::whereName($name)->`
- `Schedule::hasTag($name)->`
- `Schedule::whereId($id)->`
- `Schedule::isActive(true)->`


### Schedule's Commands

- `->cancel()`
- `->pause()` (active = false)
- `->resume()` (active = true)
- `->get()`

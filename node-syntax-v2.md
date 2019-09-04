# Node.js Syntax v2

Current syntax has drawbacks:

- _only adapted to monolith_, where code is the same on all servers
- internal properties are included in modification checks, leading to false positives `ModifiedDeciderException`
- parallel processings and events management are currently somehow inefficient
- impossible to mix programming languages
- does not include new features such as
  - workflow state introspection
  - multiple clients
  - 3rd party APIs usage

Our proposed syntax attempts to be very complete to avoid any more changes in the future. Not all features will be implemented initially.

## Task or Workflow?

We consider that - from the orchestrator point of view - there should be no way to know if a job will be processed as a task or as another (sub)workflow. That's why, the syntax should not make any distinction between them.

Something to be done inside a workflow is always called a job. Zenaton will know if it's a task of a workflow only when the name will be resolved by an Agent (so it's not possible to have task and workflow with same name).

We do not make any assumptions about the language used, that's why only the name and the data are transfered. A good practice for the user should be to avoid language-specific data structure.

** `job` BELOW SHOULD BE UNDERSTAND AS `task` or `workflow` UNDIFFERENTLY **

# From Client

Clients are not singletons and can be instantiated and named.

```javascript
client = new Client(appId, apiToken, appEnv, (name = ""));
```

An instance can be retrieved individualy by name:

```javascript
client = Client.get((name = ""));
```

or all clients:

```javascript
clients = Client.all();
```

## Dispatching Task or Workflow

Dispatching a Task or a Workflow is done through a client

### Dispatching for immediate processing

```javascript
client.dispatch.job(name, ...input);
```

`name` is a string and `data` is an array representing job's arguments (use equivalent to `new name(...data)`, ie. `[]` means no argument, `[1,2]`means 2 arguments `1` and `2`).

### Delayed Dispatching (duration)

```javascript
client.dispatch.in(duration).job(name, ...input);
```

`duration` is an integer (seconds). We provide a `Duration` helper class, to allow syntax such as `Duration.hours(2).minutes(10)`.

### Delayed Dispatching (date time)

```javascript
client.dispatch.at(date).job(name, ...input);
```

`date` is an DateTime. We provide a `DateTime` helper class, to allow syntax such as `DateTime.monday().at('8:00')`.

### Dispatch's Tagging

When dispatching, we can assign one tag or more:

```javascript
client.dispatch.tag(...tags).job(name, ...input);
```

`tags` will be an array of one (`.tag(tag1)`) or more (`.tag(tag1, tag2, tag3)`) strings.

Users will use tags to target jobs in order to apply commands or request status.

Note: the user no longer has the ability to provide an `id` ("custom id") - this is replaced by tags.

### Dispatch's Options

When looking at similar services, we see that adding options to dispatching is unavoidable, eg.

- `dispatchToStartTimeout`
- `dispatchToCompleteTimeout`

Theoritically, none of these options should be dependant on being a task or a workflow - (if you find a counter-example - please tell Gilles).

```javascript
client.dispatch.options(...options). ...;
```

Example: `options = { dispatchToStartTimeout: 300 }`

### Dispatch's Output

Any dispatch will synchronously output a `TaskContext`, `WorkflowContext` object
with status `dispatched` - see below

### TaskContext

An updated version of TaskContext is obtained:

- when dispatching a task
- when retrieving a task (eg. `client.select.task().withId(id).find())
- in `this.context`when processing a task

Format:

- `id` (provided by sdk) (our intent id)
- `task` { name, version, input}
  - `name` (canonical name) cf. versioning below
  - `version` (default: 0) - integer representing version
  - `input` (default: []) - array representing input
- `dispatch` { in, at, tags, options }
  - `in` (default: undefined)
  - `at` (default: undefined)
  - `tags` (default: [])
  - `options` (default: [])
- `createdAt` (initial client.dispatch timestamp)
- `status`
- `retryIndex` (optional) - current retry index when processing
- `output` (optional)
- `history` (optional)
- `appId`
- `appEnv`

Possible status for task: `dispatched`, `queued`, `processing`, `failed`, `completed`, `skipped`

### WorkflowContext

An updated version of WorkflowContext is obtained:

- when dispatching a workflow
- when retrieving a workflow (eg. `client.select.workflow().withId(id).find())
- in `this.context`when processing a workflow

Format:

- `id` (provided by sdk) (our intent id)
- `workflow` { name, version, input}
  - `name` (canonical name) cf. versioning below
  - `version` (default: 0) - integer representing version
  - `input` (default: []) - array representing input
- `dispatch` { in, at, tags, options }
  - `in` (default: undefined)
  - `at` (default: undefined)
  - `tags` (default: [])
  - `options` (default: [])
- `createdAt` (initial client.dispatch timestamp)
- `status`
- `output` (optional)
- `properties` (current workflow properties)
- `history` (optional)
- `appId`
- `appEnv`

Possible status for workflow: `dispatched`, `running`, `paused`, `failed`, `completed`, `skipped`

## Commands on Tasks or Workflows

Job (task or workflow) selection is done through `client.select`.

- by name: `client.select.job(name).`
- by id: `client.select.job().withId(id).`
- by tag: `client.select.job().withTag(...tags).`

these filters can be chained:
eg. `client.select.workflow('paymentProcess').withTag({email: 'foo@bar.com'})`

Special methods can modify filters:

- `.parent`: select parent of current selection (if any)
- `.childs`: select childs of current selection

Commands can be applied:

- `.get()`: retrieve array of `TaskContext` or `WorkflowContext` (pagination to be defined as parameters) (history is not provided for batch request)
- `.find()`: retrieve first `TaskContext` or `WorkflowContext`

On workflows only:

- `.pause()` pause selected instances
- `.resume()` resume selected instances
- `.terminate(output)` terminate selected instances
- `.send(eventName, ...eventData)` send event to instances

On tasks only:

- `.skip(output)` skip task in parent workflow (returning output)

## Scheduling

### Scheduling Syntax

We use the word `schedule` for a request of recurrent dispatching:

```javascript
client.schedule(cron).job(name, ...data);
```

`cron` is a cron expression. We will provide a `Each` class to allow
syntax such as `client.schedule(Each.day.at('8:00')).job(name, ...data);`.

### Schedule's tags

```javascript
client
  .schedule(cron)
  .tag(...tags)
  .job(name, ...data);
```

Provided tags are applied to Schedule object (not future dispatched jobs).

### Schedule's options

```javascript
client
  .schedule(cron, ...options)
  .tag(...tags)
  .job(name, ...data);
```

There are specific options related to scheduling, for example to indicate if we should prevent to dispatch a recurrent job if previous iteration is not completed.

### Schedule's Output

Any schedule will synchronously output a `ScheduleContext` object with `status` equals `running` (see below)

### ScheduleContext

An updated version of WorkflowContext is obtained:

- when scheduling a task or a workflow
- when retrieving a schedule (eg. `client.select.schedule.workflow().withId(id).find())`)

Format:

- `id` (provided by sdk) (our intent id)
- `workflow` or `task` { name, version, input}
  - `name` (canonical name) cf. versioning below
  - `version` (default: 0) - integer representing version
  - `input` (default: []) - array representing input
- `schedule` { cron, tags, options}
  - `cron`
  - `tags` (default: [])
  - `options` (default: [])
- `createdAt` (initial client.schedule timestamp)
- `status`
- `history`(optional)
- `appId`
- `appEnv`

Status for schedules: `running`, `paused`

### Commands on Schedules

Selection of schedules can be done through:

- `client.select.schedule.workflow(name).`
- `client.select.schedule.withId(id).`
- `client.select.schedule.withTag(...tags).`
- `client.select.schedule.withStatus(status).`

these filters can be chained:
eg. `client.select.schedule.withTag(...tags).withStatus('paused').`

Commands can be applied:

- `.pause()`: pause scheduling
- `.resume()`: resume scheduling
- `.delete()`: remove scheduling
- `.get()`: retrieve array of `ScheduleContext` (pagination to be defined)
- `.find()`: retrieve first `ScheduleContext`

# Task

## Task definition

Tasks are defined by:

```javascript
const { task } = require("zenaton");

task(
  name,
  function(...input) {
    /* task definition */
  },
  options
);
```

`options`can be an object or a function of (...input) returning an object

Example : `{maxProcessingTime: 300}`

## Task version

Task definition follow a convention: `mytask_v3` will be understood as an implementation of `mytask` version `3`.

When dispatching a task inside a worklow, both syntax with or without version can be used:

- `this.dispatch.task('mytask', input)` will use last known version
- `this.dispatch.task('mytask_v3', input)` will use version `3``

Versioning tasks can be useful when its signature evolves.

# Workflow

## Definition

Worflows are defined by:

```javascript
const { workflow } = require("zenaton");

workflow(
  name,
  function(...input) {
    /* workflow definition */
  },
  options
);
```

`options`can be an object or a function of (...input) returning an object

Example : `{childPolicy: 'terminate'}`

Note: `onEvent` is deprecated if favor of an explicit declaration inside the workflow processing, see below.

## Workflow version

A workflow definition follows a convention: `myworkflow_v3` will be understood as an implementation of `myworkflow` version `3`.

When dispatching a task inside a worklow, both syntax with or without version can be used:

- `this.dispatch.workflow('myworkflow', input)` will use last known version
- `this.dispatch.workflow('myworkflow_v3', input)` will use version `3``

Same for selecting workflows. Versioning workflow is needed as soon as you have different version running concurrently.

## Properties

In the function describing the function, the user can set properties (for example `this.email = email`);
This can be useful as those properties are provided in current `WorkflowContext` and so can be inspected from outside.

The only restriction for naming those properties is to avoid naming collision with methods
provided by Zenaton (`execute`, `select`, `dispatch`, `onEvent`, `now`, ...).

## Commands

All commands syntax should be working inside a workflow. It implies that, inside a workflow, we should manage commands idempotency.

Inside a workflow `client` can be replaced by `this`. It means the command is applied to current app/env. For example :

```javascript
this.select
  .workflow("processPayment")
  .withId("244DR3")
  .send("completed");
```

will be possible from inside a workflow. Also, commands that can be applied to a selection of workflow, can also be applied directly to `this`.

For examples:

```javascript
this.send("completed");
```

will send an event `"completed"` to itself.

```javascript
this.terminate();
```

will immediatly terminate this workflow.

## Helpers

Due to constraints applied to workflows, it's not possible to use function with side effects. As helpers, we proposed most common functions that can be used:

- `this.now()` will behave as a usable `Date.now()`
- `this.random()`will behave as a usable `Math.random()`
- `this.console.log` will behave as a usable `console.log` function
- `this.console.dir` will behave as a usable `console.dir` function
- `this.console.error` will behave as a usable `console.error` function
- `this.console.table` will behave as a usable `console.table` function

## Dispatching

All dispatching syntax should be working inside a workflow. It implies that, inside a workflow, we should manage dispatching idempotency.

Inside a workflow `client` can be replaced by `this`. It means the command is applied to current app/env.

For example :

```javascript
this.dispatch.workflow("processPayment", { cart });
```

will now be possible inside a workflow to dispatch another workflow.

## Scheduling

All scheduling syntax should be working inside a workflow. It implies that, inside a workflow, we should manage scheduling idempotency.

Inside a workflow `client` can be replaced by `this`. It means the command is applied to current app/env.

## Executing

For "executing" jobs, ie. by waiting the output to continue workflow execution, we do:

```javascript
this.execute.job(name, ...input);
```

Notes:

- there is no way to delay or repeat an execution.
- tags can be used, with same syntax than for dispatching.
- options can be used, with same syntax than for dispatching.

## Time Waiting

Waiting forever

```javascript
this.wait.forever();
```

Waiting for a duration

```javascript
this.wait.for(duration);
```

Waiting for a datetime

```javascript
this.wait.until(datetime);
```

Notes:

- using the above syntax, the return value is always `null`.
- `duration`: a duration in seconds (We can use `Duration` helper)
- `datetime`: a timestamp (We can use `DateTime` helper)

## Events

Within a workflow, receiving an event `eventName` will:

1. trigger execution of a listener of `eventName`, if it exists
2. trigger exit of an ongoing `Wait` instruction on this event

Do nothing if not 1) neither 2)

### Waiting for an event

Waiting for an event without time limitation

```javascript
this.wait.event(eventName).forever();
```

Waiting for an event, with a maximal duration

```javascript
this.wait.event(eventName).for(duration);
```

Waiting for an event, up to a date time

```javascript
this.wait.event(eventName).until(date);
```

Waiting multiple events (AND)

```javascript
this.wait.event(event1Name, event2Name, ...).forever();
```

Waiting for an event with specific data

```javascript
this.wait.event([eventName, eventDataFilter]).forever();
```

Here `eventDataFilter` is an object containing criteria on `eventData`. For exemple `{ email: "gilles@zenaton.com"}` will wait only for events with `eventData.email === "gilles@zenaton.com"`.

Output of the wait event:

```javascript
eventData = this.wait.event(eventName).for(duration);

[event1data, event2data] = Wait.event(event1Name, event2Name).for(duration);
```

If `this.wait` completes without event, then `eventData` will be `null`.
If `this.wait` completes with an event, then `eventData` can be `undefined` but never `null`.

### Reacting on Event

We propose to deprecate the `onEvent` method, as this implementations has some drawbacks

- it's quite verbose as it always begins with a test of an event's name
- as it does not allow event auto-discovery, it may generates a useless decision if an event is not implemented
- it's not consistent with planned 3rd party API behavior

At any moment, an instance can subscribe to an event to decide what to do when receiving an event.
So at any moment inside the main function, a user can subscribe to an event:

```javascript
await this.onEvent(eventName, function(...data) {
  // what I do after recieving this event.
});
```

If a user wants to listen event only with specific data:

```javascript
await this.onEvent([eventName, eventDataFilter], function(...data) {
  // what I do after recieving an event with name eventName, with right data
  // (eg. eventDataFilter {email: 'gilles@zenaton.com'} than eventData.email should be 'gilles@zenaton.com')
});
```

## Parallel

We introduce a new way to manage parallel processing.

For that, we introduce a new way to wait for asynchronous jobs:

If

```javascript
job = this.dispatch.job(name, data);
```

then

```javascript
output = this.wait.completion(job).forever();
```

will go through only when `job` processing is successfully completed.

Waiting for a parallel completion becomes:

```javascript
job1 = this.dispatch.job(name1, data1);
job2 = this.dispatch.job(name2, data2);
job3 = this.dispatch.job(name3, data3);

[output1, output2, output3] = this.wait.completion(job1, job2, job3).forever();
```

Note: `output = this.wait.completion(job).minutes(5)` will wait for only 5 minutes. If `job` is not processed during that delay, output will be `null`.

## Automatically processing HTTP calls as tasks

with `verb`being `get`, `post`, `put`, `delete`:

```javascript
await this.execute.verb(url, body, headers);
```

or

```javascript
await this.dispatch.verb(url, body, headers);
```

Examples:

```javascript
await this.execute.post('https://slack.com/api/chat.postMessage', {...}, { Content-type: 'application/json', Authorization });

await this.dispatch.post('https://slack.com/api/chat.postMessage', {...}, { Content-type: 'application/json', Authorization });
```

## Processing API from a 3rd Party Service

For 3rd party API supported by Zenaton (supporting authentification and webhooks as event), a special syntax would be provided

### If 3rd party provides a Rest API

```javascript
await this.execute.verb("service:method", data);
```

or

```javascript
await this.dispatch.verb("service:method", data);
```

Examples:

```javascript
await this.execute.post("slack:web.chat.postMessage", data);
```

Note: if user has connected Zenaton to more than 1 service, he should provided a serviceId

```javascript
await this.execute.verb("serviceId@service:method", data);
```

### If 3rd party provides a full SDK

```javascript
await this.execute.task("service:method", data);
```

or

```javascript
await this.dispatch.task("service:method", data);
```

Examples:

```javascript
await this.execute.task("aws.s3:getObject", data);
await this.dispatch.task("aws.ses:SendRawEmail", data);
await this.execute.task("slack:web.chat.postMessage", data);
```

As above, if user has connected Zenaton to more than 1 service, he should provided a serviceId

## Reacting to events from a 3rd Party Service

Events usually comes from 3rd parties through webhooks. Zenaton handles those webhooks and transforms
them into events that a user can use without worrying to manage a server to receive webhooks.

```javascript
await this.onEvent("service:eventName", func);
```

Note: if a user has connected Zenaton to more than 1 service, he should provided a serviceId

If user wants to react to an event only if this event comes with specific data:

```javascript
await this.onEvent([eventName, eventDataFilter], func);
```

Example: `this.onEvent(['slack:reaction_added', {timestamp: 126342563}]` will react only on event with this specific timestamp.

## Waiting Events from a 3rd Party Service

Waiting for an event from a 3rd party

```javascript
await this.wait.event("service:eventName").forever();
```

## Example

Post a message and remove any reaction to it for 1 hour:

```javascript
const { workflow } = require("zenaton");

module.exports = workflow("PostSlackMessageWithoutReaction", function() {
    const response = await this.execute.post('slack:chat.postMessage', {
            channel: 'C1234567890',
            text: 'try to react ont this!)'
    });
    this.onEvent(['slack:reaction_added', {timestamp: response.message.ts}], () => {
            await this.execute.post('slack:reactions.remove', {timestamp: data.timestamp});
    })
    await this.wait.for(3600);
})
```

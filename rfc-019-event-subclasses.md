# Problem Recognizing Events' Subclasses

## Summary

Subclasses of awaited events are not properly handled by Zenaton

## Problem

As Florian Jourda pointed out today via Crisp, and as some of us know for some times now, Zenaton is not recognizing event subclasses properly. That means that if a Wait task is waiting for an instance of the  `MyEvent` class and receives an instance of the `MyChildEvent` class (where `MyChildEvent` is a subclass of `MyEvent`), the "wait for event" condition will not be satisfied and the Wait task will evaluate to `None`.

That is obviously a problem because if users are trying to write workflows where they subclass a base event, it will simply not work. For example users may use two event classes `OfferAccepted` and `OfferDeclined` both subclassing `OfferAnswered` to modelize the decision of a customer regarding on offer in a marketplace.
 

###  How to reproduce the problem?

This is Python code, but the same logic can be applied to any language.

Create a `MyChildEvent` class that subclasses `MyEvent` event in the examples:
```
class MyChildEvent:
    pass
```

Then, in the `launch_wait_event.py` file, send a `MyChildEvent`instead of `MyEvent:

`WaitEventWorkflow.where_id(event_id).send_event(event=MyChildEvent())`

The `WaitEventWorkflow` will execute `TaskB` instead of `TaskA`.

The same behavior can be observed in the EventWorkflow by dong this in the `launch_event.py` file:

`WaitEventWorkflow.where_id(event_id).send_event(event=MyChildEvent())`
 
## Proposal

Updates has to be done on the engine, agent and clients sides. This includes, but is not limited to:

- Client librairies should send classes, not just Event class name as string.

Example: 
```php
<?php

use Zenaton\Interfaces\EventInterface;
use Zenaton\Interfaces\WorkflowInterface;
use Zenaton\Tasks\Wait;
use Zenaton\Traits\Zenatonable;

class OfferAnswered implements EventInterface
{
}

class OfferAccepted extends OfferAnswered
{
}

class OfferDeclined extends OfferAnswered 
{
}

class WaitEventWorkflow implements WorkflowInterface
{
    use Zenatonable;

    protected $id;

    public function __construct($id)
    {
        $this->id = $id;
    }

    public function handle()
    {
        $event = (new Wait(OfferAnswered::class))->execute();

        if ($event instanceof  OfferAccepted) {
            (new TaskA())->execute();
        } else {
            (new TaskB())->execute();
        }
    }

    public function getId()
    {
        return $this->id;
    }
}
```

- The `sendEvent` function of the `client` should send the whole inheritance structure of the event class, in order to recognize if the event class is a child of the awaited event class.

Example:
```php
# Mock the event you need to send
$event = new OfferAccepted();
# Used in many OOP languages to get infos on class
$class = new ReflectionClass($event);

# First add the child event
$matchEvents[] = get_class($event);

while ($parent = $class->getParentClass()) {
    $matchEvents[] = $parent->getName();
    $class = $parent;
```

`$matchEvents` will be a list of classname. The agent need to give the instructions and the engine need to be ready to receive those events to terminate a wait and send back this information to libraries when received an event.

- The Agent and Engine have to handle string, as well as arrays of string (because an event class can inherit from several classes).


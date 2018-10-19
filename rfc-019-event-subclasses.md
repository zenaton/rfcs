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

WaitEventWorkflow.where_id(event_id).send_event(event=MyChildEvent())`
 
## Proposal

First, I was thinking something was wrong in the implementation of the Wait class, where the event type is checked, but this part seems OK. Furthermore it seems that the bug is also present on the plain EventWorkflow, which does not involve the Wait Task.
I have a few leads on what might cause the problem but I am not sure yet. Let's discuss it together. I will also update this RFC if I find a real lead of this behavior's cause.
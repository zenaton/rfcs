rfc-017-go-user-interface-change.md


# New Zenaton-go interface

## Summary

I would like to make some ergonomic changes to the zenaton-go interface.

## Problem

Virtually all these issues apply to tasks as well, but I'll talk here only about the workflows.

In go (as in javascript actually) there are two ways to define workflows, a simpler way, where you just provide a handle function, and a more complicated way that allows you to provide optional methods (like OnEvent(), and ID()). I will focus here on the more complicated way of defining a workflow as anything that would work here, should work pretty easily with the simple workflow)


Currently a workflow is defined like this (from `workflows/event.go` in the examples):

```go
var EventWorkflow = workflow.NewCustom("EventWorkflow", &Event{State: true})

type Event struct {
	IDstr string
	State bool
}

func (e *Event) Handle() (interface{}, error) {
	tasks.A.New().Execute()
	if e.State {
		tasks.B.New().Execute()
	} else {
		tasks.C.New().Execute()
	}
	return nil, nil
}

func (e *Event) Init(id string) {
	e.IDstr = id
}

func (e *Event) OnEvent(eventName string, eventData interface{}) {
	if eventName == "MyEvent" {
		e.State = false
	}
}

func (e *Event) ID() string {
	return e.IDstr
}
```

And this is how you use that workflow (from `event/main.go`):
```
func main() {

	workflows.EventWorkflow.New("testID").Dispatch()

	time.Sleep(4 * time.Second)

	workflows.EventWorkflow.WhereID("testID").Send("MyEvent", nil)
}
```

Here are my problems:

1. There are a lot of problems with the Init function, but I will focus on one main one and add more if there is sufficient pushback on this point. When you call `workflows.EventWorkflow.New(id)` the id is then passed to the Init function. Here we have compromised on type safety. If the type that is passed into `EventWorkflow.New()` doesn't match the type defined in the `Init()` function, there will be a runtime panic. Ideally you would able to utilize go's static typing to catch type issues at compile time.

2. When you define your type (`type Event struct {…}`) with a handle method, that type is not the actual workflow. So you have to have two different names `EventWorkflow` which is the workflow Definition, and `Event` which is the type that is used as the basis for the workflow Definition. Ideally, you would just have something called `EventWorkflow`. (note: In javascript you don't have this issue, as you can just pass an anonomous object into the workflow constructer. But in go you need to have a named type if you want to define methods.)

3. There are two types related to workflows that a user needs to keep track of, a workflow Definition, and a workflow Instance. Here `workflow.NewCustom(…)` returns the Definition (this is analogous to the WorkflowClass returned by the `Workflow()` function in javascript). Then you can use `SomeWorkflow.New(…)` to get an Instance, which you can then `Dispatch()`. In php for example, you call Dispatch directly on your class that you define. There are less entities you have to keep track of. Ideally it would be the same in go. (There is a similar problem in node actually)

4. `EventWorkflow` is a global variable here. That's not the worst thing, but it's a bit of a soft anti-pattern. I do that here because by having this assignment in a global variable, we call workflow.NewCustom when the application first starts, before even the main function is called (because of the order in which go executes things). `NewCustom()` needs to be called before the main function, as it sets up the workflow with the workflowManager. But the user doesn't see this, so they don't really understand why EventWorkflow must be a global variable. Without reading the documentation, it's just a mysterious requirement.

5. The last problem isn't really visible in the examples as printed above. But it's related to point (3). The agent must use both workflow Instances and workflow Definitions. And there are some exported methods on those types that the agent must use that the user should actually never use, or at least they don't need to use them. We essentially have API polution, where the user should only use a subset of the exported methods/types. I've tried my best to reduce the amount dangerous/unnecessary exported methods, but there are still some hanging arround, partly because of these two types that the user must use.

## Proposal

I propose to make the definition of a workflow look like this:
```
const EventWorkflowName = "EventWorkflow"

func init() {
	workflow.Register(EventWorkflowName, &EventWorkflow{})
}

type EventWorkflow struct {
	IDstr string
	State bool
}

func NewEventWorkflow(id string) *EventWorkflow {
	return &EventWorkflow{
		IDstr: id,
	}
}

func (e *EventWorkflow) Handle() (interface{}, error) {
	zenaton.Execute(tasks.NewTaskA())

	if e.State {
		zenaton.Execute(tasks.NewTaskB())
	} else {
		zenaton.Execute(tasks.NewTaskC())
	}

	return nil, nil
}

func (e *EventWorkflow) OnEvent(eventName string, eventData interface{}) {
	if eventName == "MyEvent" {
		e.State = false
	}
}

func (e *EventWorkflow) ID() string {
	return e.IDstr
}
```

And dispatching a workflow and sending an event would look like this:
```
func main() {

	zenaton.Dispatch(workflows.NewEventWorkflow("test id"))

	time.Sleep(4 * time.Second)

	workflow.WhereID(workflows.EventWorkflowName, "test id").Send("MyEvent", nil)
}
```

Here's how it addresses my points:
1. Here the user calls their own constructor function (`NewEventWorkflow()`) directly, so they get to maintain type safety! It's slightly more verbose, but I think the extra type safty is a big win.

2. Here you get to make your custom type be the actual `EventWorkflow`.

3. the user never has to use a workflow Instance, or a workflow Definition. They just call `workflow.Dispatch()` passing in their workflow. They don't have to reason about the differences between an Instance and Definition, or try to figure out how they should be using/producing each one.

4. Instead of having our workflow be a global variable, we call `workflow.Register(EventWorkflowName, &EventWorkflow{})` in the `init()` function. In go, the init() function is called before main, so it serves the same function as the global variable. As we don't need to store a workflow Definition for later use (see point (3)), we don't need to have the workflow stored as a variable. Instead it's just written as a type (closer to how it is in php rather than the avascript). As we now use `workflow.Register()` it's much more clear why it needs to be called before main. We're registering the workflow (with zenaton).

5. Because we don't have workflow Definitions and workflow Instances (and their associated methods), there is a dramatically reduced set of public methods and types. This should make the library much more intuitive for the user. In the background we may have to use workflow Definitions and Instances, but at least the user won't need to handle these.

Possible issues:
* This is slightly more verbose. I think it's worth the extra clarity.
* The interface would become more different than the other language libraries. I feel that it's a much bigger priority to make the libraries more ergonomic than all the same.
* In referencing a workflow when sending an event, you must use it's name instead of calling a method on the workflow. This is the biggest inconvenience I can see.


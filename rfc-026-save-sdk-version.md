# Save SDK version of workflow/task start

## Summary

Save which version of the language SDK was used to start a workflow/task.

See also [#24](https://github.com/zenaton/rfcs/pull/24).

## Problem

The NodeJS SDK is about to ship a new version which behaves differently than the previous ones (asynchronous vs synchronous). But by necessity this new version of the SDK will have to handle old workflows that might have been started a long time ago with a previous version of the SDK.

This requires the NodeJS SDK to use clever code paths to accomodate both old and new workflows and tasks. The split would be based on the version of the SDK that was used to start them.

Problem is: currently this information is not associated with workflows or tasks and is not available to workers.

## Proposal

Update the whole pipeline to have, besides the programming language, the SDK version used to start the workflow/task be sent all the way to the engine, saved to the database, and then being sent back to the workers.

SDK (client) -> Agent -> Engine -> Agent -> SDK (worker)

The worker - but also the agent - can then use this information to decide which code paths is better suited to handle the workflow/task.

## Deployment

* Update all the SDKs to add a `library_version` in the body (query string? headers?) of the HTTP request sent to the agent to start workflows and single tasks.

* Update the agent to push `library_version` to the engine.

* Have the engine save `library_version` with the rest of workflows/tasks data.

* Have the engine forward `library_version` to the agent.

* Have the agent forward `library_version` to the script that will handle the decision/worker.

* For the Javascript part of the agent and the NodeJS SDK only: switch code path depending on the `library_version` to handle both synchronous and asynchronous workflows/tasks.

* Possibly update the monitoring to display `library_version`, since in the upcoming update we already display `programming_language`.

## Additional thoughts

In [a previous rfc](https://github.com/zenaton/rfcs/pull/24) I proposed to add the following HTTP headers in all calls between SDK and agent, to be future proof:

* Call from SDK to agent:
  * `X-Zenaton-SDK-Version`
  * `X-Zenaton-SDK-Language`
  * `X-Zenaton-Minimum-Agent-Version`

* Call from agent to SDK:
  * `X-Zenaton-Agent-Version`
  * `X-Zenaton-Agent-Boot-Language`

Now I realize that:

* `X-Zenaton-SDK-Version` is exactly what's being discussed in this rfc.

* `X-Zenaton-SDK-Language` is already set in the HTTP body as `programming_language`.

* `X-Zenaton-Minimum-Agent-Version` is what's discussed in the other rfc.

* `X-Zenaton-Agent-Version` and `X-Zenaton-Agent-Boot-Language` does not make much sense since the agent is using the SDK, not calling it.

### Save Agent version

On a related note, I'm thinking that we should also send the version of the agent that was used to start the workflow/task. Imagine a scenario where a web app uses the Zenaton agent v2.0.0, and the workers are using v1.0.0. Do we have something to handle this?

I think we should also save in the database the version of the agent that was used to start a workflow/task, and when a worker's agent received work to do, it should check if its own version is at least equal or greather than, but never lower than the version of the agent that was used by the web app. Otherwise it might mean that the worker's agent does not know the correct code paths to handle the work.

In the end for each workflow/task, we should save:

* The programming language.
* The version of the SDK that started it.
* The version of the agent that started it.
* Possibly the minimum version of the agent the SDK was expecting (discussed in other rfc).

i.e:
* Javascript
* SDK 0.5.0
* Agent 1.0.0
* Minimum agent 0.8.0

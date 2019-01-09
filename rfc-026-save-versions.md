# Save versions on workflow/task start

## Vocabulary

In this rfc the term _versions_ designates:

* The language SDK version (like the NodeJS npm package version). Like `0.5.1`.
* The folder name corresponding to a code path, when a SDK undergo a major overhaul. Like `v1` or `kebab` or whatever convention is in use.
* The Agent version. Like `0.6.7`.

This rfc is about saving **all of those versions**.

## Summary

Save which versions were used to start a workflow/task.

See also [#24](https://github.com/zenaton/rfcs/pull/24).

## Problem

As a user, you want to be able to use the latest version of a language SDK to start new workflows and single tasks, since it allows you to benefit from the last improvements.

But your workers, running this latest version of a language SDK, have to be able to execute very old workflows and tasks, which might be expecting the code paths and logic of an older SDK: the one they were used to be started in the first place.

For example, the NodeJS SDK is about to ship a new version which behaves differently than the previous ones (asynchronous vs synchronous). But by necessity this new version of the SDK will have to handle old workflows that might have been started a long time ago with a previous version of the SDK.

This requires the NodeJS SDK to use clever code paths to accomodate both old and new workflows and tasks. The split would be based on the initial code path that was used to start them, asynchronous (new) or synchronous (old).

Problem is: there is no way to know *a priori* for the Agent which version should be used by a task or a decision. This version is actually the default version used by the library when the task or the workflow was dispatched. So, this information should be attached to the task or the workflow execution.

More generaly, we lack data on the circumstances surrounding a workflow/task being started, so it would be very useful to record which version of a language SDK and which version of the Agent has been used. It would allow us to have a better understanding of what's in the wild.

## Proposal

When a workflow/task is started:

* Have the language SDK version and the folder name version be sent in the HTTP call to the Agent, just like the programming language is already sent.

* Have the Agent pass those informations to the Engine, adding its own version in the process.

* Have the Engine store all these informations as metadata for the workflow/task.

* When the Engine contacts a Worker to trigger a decision or an execution, have him send over all those informations.

* When a Worker starts a slave in any language, have him inject those informations.

## Deployment

Names for the fields:

* `initial-library-version`
* `code-path-version`
* `initial-agent-version`

On all SDKs:

* Add all three fields in the body of the HTTP request sent to the agent to start workflows and single tasks.

On the Agent:

* On the way up, push all three fields to the Engine.
* On the way down, send all three fields to the slave which will process the decision/execution.

On the Engine:

* Save all three fields with the rest of workflows/tasks data.
* On decision/execution trigger, forward all three fields to the worker.

On the NodeJS SDK:

* Switch code path depending on the `code-path-version` to handle both synchronous and asynchronous workflows/tasks. If no `code-path-version` is given, assume that it's an old workflow started with the old synchronous code path.

## Retrocompatibility

### Engine and Elixir part of the Agent

Engine and Elixir part of the Agent can just ignore the absence of the versions. Engine won't save them and forward them to the Workers. Agent won't send them when it starts a workflow, or forward them to the slave processes.

Absence of the version fields just denotes old workflows that can live their life without them, or workflows started by languages that does not have multiple code paths for the moment and does not need this policy.

### Agent various languages code and SDKs

Right now this change is necessary only for the NodeJS SDK and Javascript part of the Agent, which are going through a major overhaul. Implementing it in other SDKs can wait.

The new Javascript asynchronous stack (Agent and SDK) cannot and musn't be released until what's being described in this rfc has been deployed in production.

Use of the new asynchronous NodeJS SDK will need to be made in conjunction with the new Agent as well. This necessity will be enforced by a strong _breaking changes_ communication on release, disclaiming that the use of this novelty dictates that all deployed Agents must be updated, and all Javascript boot files must use the newly published Zenaton NodeJS SDK.

The _minimal agent version_ policy discussed in rfc [#24](https://github.com/zenaton/rfcs/pull/24) will also prevent a user to download the new asynchronous NodeJS SDK, and use it with an old synchronous version of the agent.

The new Javascript stack will behave as follows:

* All new workflows/tasks using the new asynchronous stack must be started with all the version fields associated to them.

* If a Javascript slave started by the Agent does not receive the version fields, it will assume the job to execute was started on the old synchronous stack, and will behave accordingly.

* If a Javascript slave started by the Agent does receive the version fields, it will check the `code-path-version` field to know on which stack the job was started (logically it will always be on the new asynchronous stack), and will behave accordingly. To be future proof, if the `code-path-version` is unknown, the Agent should throw an error inviting the user to perform an update.

## Additional thoughts

It would be nice to update the monitoring to display the initial SDK and Agent versions (possibly in *debug* mode), since in the upcoming update we already display `programming_language`.

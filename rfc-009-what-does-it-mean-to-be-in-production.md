# What does it mean to be in production?

## Problem

Today, we do not really know when Zenaton is used in production!

- it's useful to understand usage, potentially to differentiate infrastructure.
- it's useful to invoice them (if we don't want users to pay during development)

Today, we do not have any way to distinguish between production, staging or development,
except with Zenaton's env name, but we can not rely on them.

## Proposal

There are different options. Up to now none of them seems convincing:

Add some constraints to non-production environment, such as:
- limit execution rate (throttling)
- limit the number of monthly tasks
- limit the duration of workflows
...

# Monorepo

## Summary

Break up our current monorepo into separate repositories per project

## Problem

Most CI/CD tools make the assumption that a single repo corresponds to a single
project. This causes the following problems:
- Useless builds on CircleCI. Example: worker tests are run for a website change
- Builds take longer. Example: I need to wait for the queue to be clear for the
  tests related to a change I made to be run
- Too many build notifications on Slack.
- Exhausting our montly cap of builds. We have a limited number of builds and
    minutes on CircleCI.
- Cloning the whole repo takes time. Example: When provisionning new machines or
    running the automated builds
- Makes it hard to setup automatic deploys to staging. Example: Should a website
  commit trigger a build and release of a new worker version?

While having everything in a single repo can make things easier to keep track
of, I think still can safely split the repositories, as we would still be able
to:
- Receive email/GitHub notifications for new branches, PRs, issues, etc.
- Be able to link issues and PRs from different repos together
- Receive notifications on Slack (we have a #github channel for these)

## Proposal

Currently our `core` repo contains 3 distinct applications:
- the website
- the agent (worker + cli)
- the engine

I suggest we start by extracting the engine, then the website into their
respective private repositories on GitHub.

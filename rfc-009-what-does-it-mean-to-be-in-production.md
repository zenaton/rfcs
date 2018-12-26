# What does it mean to be in production?

## Problem

Today, we do not really know when Zenaton is used in production!

- it's useful to understand usage, potentially to differentiate infrastructure.
- it's useful to invoice them (if we don't want users to pay during development)

Today, we do not have any way to distinguish between production, staging or development,
except with Zenaton's env name, but we can not rely on them.

## Proposal

### Pricing 

#### Free
- max 100 tasks/s, 100k tasks /month
- 3 environments
- Error Handling
- 1 week retention data

#### 0.130$ / hour (only during execution)
- max 1000 tasks/s, 10M tasks /month
- 30 environments
- Alerts & Error Handling
- Tasks, Workflow and Infrastructure Monitoring
- Workflow Orchestration
- 90 days Retention Data

#### Contact us
- Dedicated Infrastructure
- 1 Year Retention Data
- Developer Guidance & Support
- 99.99% SLA

### Explanation

the proposed pricing is based on the cumulated duration of task managed by the Agent, so if Agent manage some time execution 30% of the time, the cost for the corresponding agent will be: 0.3 (activity ratio) * 0.130$ * 24h * 31 (for a month with 31 days) = 29$

Notes: 
- What you pay is not sensitive to the # of agents: if you have 2 agents, this can impact the price (theoretically up to x2) but only if you need it, if 1 agent would be enough for your workload then the price will stay the same
- What you pay is not sensitive to the maximal # of parallel executions you authorize : if you have a setting of 5, this can impact the price (theoretically up to x5), but only if you really need them.
- Decisions are included in activity time calculation
- If you forget unused agent, then you do not pay for them :)
- Developer costs are based on usage :)
- the limit of tasks/second is mainly here to avoid the flooding of the infrastructure when launching tasks/workflows.

*As a teaser, we could add a credit 30$ to new account, to let them use all features :)*

## Conclusions
With this proposal, there is no production mode :) You pay or you don't pay. If you pay, then you pay also dor your development activity, but normally very few. 

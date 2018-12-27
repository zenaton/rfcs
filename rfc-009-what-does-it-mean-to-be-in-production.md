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
- max 100k tasks /month
- max 100 tasks /second
- max 3 environments
- Error Handling

#### 0.130$ / hour (only during execution, min. 10$/month)
- max 10M tasks /month
- max 1000 tasks /second
- max 30 environments
- Alerts & Error Handling
- Monitoring
- 90 days Retention Data
- Email/Slack support

#### Contact us
- Dedicated Infrastructure
- Unlimited environments
- 1 Year Retention Data
- Developer Guidance & Support
- Phone support
- 99.99% SLA

### Explanation

the proposed pricing is based on the cumulated duration of tasks managed by the Agent, so if Agent manage some time execution 30% of the time, the cost for the corresponding agent will be: 0.3 (activity ratio) * 0.130$ * 24h * 31 (for a month with 31 days) = 29$

Notes: 
- What you pay is capped by your infrastructure: eg. with two workers, you'll pay max of 2*31*24*.130 = $193 (probably much less). If you allow N executions in parallel, this theoretical cap is N times bigger. 
- If you parallelize your tasks by having more workers, or more tasks per worker, your tasks will be terminated sooner, but you won't pay more. Idem, you can have a large worker or many small workers - this should not impact the invoice a lot.
- Decisions are NOT included in activity time calculation
- Unused Agent do not cost
- Developer costs should stay tiny
- the limit of tasks/second is mainly here to avoid the flooding of the infrastructure when launching tasks or workflows.
- customers with few large tasks (airflow use cases) will pay as a lot of smaller tasks (if similar execution time).
- min. amount is here to avoid having customers switching back and forth to free plan, and also to pay for long-running workflows.

*As a teaser, we could add a 30$ credit to each new account, to taste monitoring and alerts features :)*

## Conclusions
With this proposal, there is no production mode :) Everyone is in production, even developers.

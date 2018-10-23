# Key Metrics

## Summary

Define useful key metrics for Zenaton

## Problem

We need to define what metrics are interesting for Zenaton.

This RFC can contains business metrics, technical metrics, operational metrics, etc.
This RFC will also describe where the defined metrics should be displayed.

## Proposal

For now we only have on place to display metrics: the backoffice.
Until we have too many metrics serving different purposes to be displayed
in separate places, we will display all metrics in the backoffice.

The Nova backoffice has many locations where it can display metrics.

- Dashboard: The main screen when you go into the backoffice. It is best suited for
generate purpose metrics.
- Resource index: The listing screen when you click on a resource name in the left side menu bar.
It is best suited for resource related metrics that are trifling enough to be displayed
on the main dashboard.
- Resource details: The details screen displayed when you click the eye icon to display a resource's
details. It is best suited to display metrics related to a specific resource.

Nova also have support for different types of metrics:
- Value: Single value displayed, with a comparison relative to the previous timeframe.
- Trend: Line chart.
- Partition: Pie chart.

The following is a table about all key metrics useful for Zenaton, their display location and
a description.

### Metrics related to teams and users

| Name | Type | Location | Description |
| ---  | ---  | ---      | ---         |
| `LostPayingTeams` | `Trend` | `Dashboard` | The number of teams having a paid subscription that ended during the current month. |
| `NewPayingTeams` | `Trend` | `Dashboard` | The number of teams having a paid subscription. |
| `NewTeamsRegistered` | `Trend` | `Dashboard` | The number of teams registered per day. |
| `NewUsersActivated` | `Trend` | `Dashboard` | The number of users activated per day. |
| `NewUsersRegistered` | `Trend` | `Dashboard` | The number of users registered per day. |
| `PayingTeams` | `Trend` | `Dashboard` | The total number of teams having a paid subscription. |
| `TeamsRegistered` | `Trend` | `Dashboard` | The total number of teams registered. |
| `UsersRegistered` | `Trend` | `Dashboard` | The total number of users registered. |
| `UsersActivated` | `Trend` | `Dashboard` | The total number of users activated. |

### Metrics related to workflows and tasks

| Name | Type | Location | Description |
| ---  | ---  | ---      | ---         |
| `AverageTaskSize` | `Value` | `Team Details` | The average size of tasks executed by the team. |
| `AverageWorkflowCompletionTime` | `Value` | `Team Details` | The average execution time of workflows. |
| `AverageWorkflowSize` | `Value` | `Team Details` | The average size of tasks executed by the team. |
| `MaxTaskSize` | `Value` | `Team Details` | The maximum size of tasks executed by the team. |
| `MaxWorkflowCompletionTime` | `Value` | `Team Details` | The max execution time of workflows. |
| `MaxWorkflowSize` | `Value` | `Team Details` | The maximum size of workflows executed by the team. |
| `MinTaskSize` | `Value` | `Team Details` | The minimum size of tasks executed by the team. |
| `MinWorkflowSize` | `Value` | `Team Details` | The minimum size of workflows executed by the team. |
| `MinWorkflowCompletionTime` | `Value` | `Team Details` | The average execution time of workflows. |
| `CurrentlyRunningWorkflows` | `Value` | `Dashboard` + `Team Details` | The number of workflows currently in a running state. On `Dashboard`, it will display the total number, all teams included. On `Team Details`, it will display the number of workflows running for the team. |
| `TasksExecutedPerDay` | `Trend` | `Dashboard` + `Team details` | The number of tasks executed per day. On `Dashboard`, it will show the total number, all teams included. On `Team details`, it will show the number of tasks executed by the team. |
| `WorkflowsStartedPerDay` | `Trend` | `Dashboard` + `Team details` | The number of workflows started per day. On `Dashboard`, it will show the total number, all teams included. On `Team details`, it will show the number of workflows started by the team. |

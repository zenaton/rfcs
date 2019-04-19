# My RFC Title

## Summary

This RFC describe how our scheduler will work

## Problem

Which syntax is proposed to users for defining recurrent tasks/workflows? 

## Inspirations

- Java: http://www.quartz-scheduler.org
- Laravel: https://laravel.com/docs/master/scheduling
- Node: https://github.com/agenda/agenda
- Airflow : https://airflow.apache.org/scheduler.html
- 
- ...

## Discussion

Note: a solution to manage a dispatcher is to set everything in a UI. (like easycron.com) - but the philosphy of Zenaton is to provide everything by code - this has a lot of advantages:
- it can be versioned
- it allow an easy management of different environments (dev, staging,  production)
- it consistent with developer habits
- it's a trend
So we discuss here how to do it with code.

### Pitfals

From a usage perspective, it seems dangerous that doing `dispatch(task).everyMinute()` twice (eg. from n distinct agents), would dispatching n `task` every minute. => A scheduler seems in essence a singleton. And actually it is his purpose to keep in a single place the definition of recurrent jobs. 

Also, once I have defined a set of recurring tasks/workflows - how can I update it consistently from different servers ? => it seem the most obvious way is to reset the settings and replace it entirely by the new ones.

For this same reason, it seems reasonnable to have an explicit command to set the scheduler.

## Proposal



# Retry Task Syntax

## Summary

This RFC propose to describe how automatic retry of task will work

## Problem

As a user, we want to be able to define a way to automatically retry failed task

## Inspiration

- [Laravel](https://laravel.com/docs/master/queues#dealing-with-failed-jobs)
- [Kue](https://github.com/Automattic/kue#failure-backoff)
- [Sidekiq](https://github.com/mperham/sidekiq/wiki/Error-Handling#automatic-job-retry)
- [Celery](https://celery.readthedocs.io/en/latest/userguide/tasks.html#retrying)


## Discussion
It's quite clear for me that the parameters for retrying should be in the task definition:
- deciding what to do depends on the task
- we do not need thos parameters before the first execution really

So the main question I habe is:
- should this retry be programmatic
- should this retry be handled throug parameters?

### Programmatic proposal
Basically, a method `onFailure` would allow to decide what to do:
```php
public function onFailure()
{
    $this->retry();
}
```
That may imply to add a new method `retry` method to be able to differentiate a retry from a simple `dispatch`. To be really useful, we should probably provide a context as a parameter, such as the # of retrial, and also to add a way to delay the execution, such as
```php
public function onFailure($nbRetry)
{
    $this->wait(10*$nbRetry)->retry();
}
```
### Parametric proposal
We could an optional method 
```php
public function getRetryStrategy;
```
With values
- `none` for no automatic retries (no parameters)
- `simple` for simple retries (parameters: max retry, max delai)
- `exponential` for exponential backoff (parameters: max retry, max delay, curvature)

and a
```php
public function getRetryParameters;
```
that will return an array defining the retry strategy based on imposed mathematical formula.

### Retry Delay function
We could provide an optional retry delay method, such as
```php
public function onFailureRetryDelay(n);
```
where `n` is the number of attempts up to now (0 for first execution) and the return value is
- `false` if user does not want to retry (default)
- an integer representing the time in seconds to wait before retrying

As example, we can implement a simple [Exponential Backoff](https://dzone.com/articles/understanding-retry-pattern-with-exponential-back) or a simple retry with delay and maximum try attempts.

## Proposal

- the `onFailure` method seems to generic for me and implying some unneeded additional syntax. It can also encourage deviant behavior such as add more code within this method (note that it's still possible with `onFailureRetryDelay` nevertheless but less obvious)
- the `retryStrategy` way wanted to be simpler but is more complex and rigid due to the necessity to handle parameters. Also the user can not define his own strategy.

Due to its simplicity, I propose to use a `onFailureRetryDelay(n)` method. And to provide simple example of uses.

WARNING: if an exception occurs within the `onFailureRetryDelay` method, it should be identified as so a generate an exception to the retry procedure.

## Comments
When a job (task or decision) is automatically retried, its status stay one of
- scheduled 
- processing
- failed or completed

(timed-out is not a status but an info related to processing)

We will add specific info related to retry such as
- previous_attempt_id (undefined for the first time)
- current_attempt_nb (0 for the first time)

It means also that a job is not considered as failed until its retry policy has expired. 

Nevertheless a nice feature would be to be able to manually retry the job when it's in "waiting to retry" mode (which should be considered as a warning - like timeout)

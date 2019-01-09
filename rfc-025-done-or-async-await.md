# Done or async/await

## Summary

How to migrate user using the NodeJS SDK from callback based tasks to promise based task.

_I initially wrote this RFC to prove it was not possible to handle both callbacks and promises at the same time. But it turns out it is. :-)_

## Problem

With the NodeJS SDK, one must use the `done()` callback to notify that a task has finished business.

But in the NodeJS world, the days of the callbacks are long gone, and pretty much everyone uses promises nowadays. Promises are state machines with three states: pending, and resolved or rejected.

A bit of history:

* Callbacks were there from the start.
* Vanilla promises (then/catch) were introduced in NodeJS 0.12 (February 2015).
* Async/await (which makes manipulating promises more natural) was introduced in NodeJS 7.10 (May 2017).

Actual LTS maintenance versions are 6.X until April 2019 and 8.X until January 2020. Active LTS is 10.X, with next being 12.X, programmed for Novembre 2019.

With vanilla promises you chain promises like that:

```javascript
someFunctionAsync()
    .then((result) => {
        console.log(result);
    })
    .catch((err) => {
        console.error(err);
    });
```

With async/await you can do it like that:
```javascript
// The scoping function must be marked as 'async'
// which means you can't do that at module level
try {
    const result = await someFunctionAsync();
    console.log(result);
} catch (err) {
    console.error(err);
}
```

The point is: it would be nice to allow users to use promises instead of the `done()` callback to notify the end of their tasks.

Instead of this:
```javascript
Task("foobar", function (done) {
    setTimeout(function () {
        done();
    }, 1000);
});
```

You'd do that:
```javascript
// 'async' is optional here since we're not using 'await'
// But it would be a good practice to clarify that the function is asynchronous
Task("foobar", /* async */ function () {
    return new Promise((resolve) => {
        setTimeout(function () {
            resolve();
        }, 1000);
    });
});
```

Problem: how do we manage to keep handling tasks already running in the wild with `done`, and new updated tasks using promises.

I've identified three use cases to cover:

```javascript
// Current documented solution with 'done'
Task("TaskDone", function (done) { // => undefined
    setTimeout(function () {
        done(null, 42);
    }, 1000);
});
```

```javascript
// New solution with a promise
Task("TaskAsync", async function () { // => Promise<Number>
    return new Promise((resolve/*, reject */) => {
        setTimeout(function () {
            resolve(42);
        }, 1000);
    });
});
```

```javascript
// Solution using 'done' inside an async function
Task("TaskAsyncPlusDone", async function (done) { // => Promise<undefined>
    const result = await databaseCallAsync();

    done(null, result);
    
    // implicit return
    // return undefined;
});
```

The root of the problem lies in the last case: users who run tasks which uses `done()`, but also return a promise.

This code is totally possible with the NodeJS SDK currently deployed. The function is asynchronous and Node's engine will wait for `databaseCallAsync` to resolve before executing the rest of the function, and calling `done()`. The return statement is implicit but it will return a promise, initially pending, which will later resolve to `undefined` if no error is thrown.

And here is the simplified code that would handle both the use of `done()` and promise.

```javascript
let doneWasCalled = false;

try {
    // Important: there is no 'await' in front of 'this.handle'
    // We do not wait for an eventual promise to resolve/reject
    const result = this.handle(this.done);
    if (result instanceof Promise) {
        const unwrappedResult = await result;

        if (!doneWasCalled) {
            completeBranch(unwrappedResult);
        }
    }
} catch (err) {
    failBranch(err);
}

function done(err, result) {
    doneWasCalled = true;

    if (err) {
        failBranch(err);
        return;
    }

    completeBranch(result);
}
```

The code will work with `TaskDone`: it sees that `handle()` doesn't return a `Promise` instance. So it will wait for `done()` to be called, which will happen eventually.

The code will work for `TaskAsync`: it sees that `handle()` does return a `Promise` instance, so it will wait for its resolution. Once the promise resolved, it will notice that `done()` was never called, so it will complete the branch.

The code will work with `TaskAsyncPlusDone`: it sees that `handle()` does return a `Promise` instance, so it will wait for its resolution. Once the promise resolves, it will notice that in the meantime `done()` was already called.

Only exception would be a very vicious user who would write this:

```javascript
Task("TaskAsync", async function (done) { // => Promise<Number>
    return new Promise((resolve/*, reject */) => {
        setTimeout(function () {
            resolve(42);
            done(null, "Something completely different");
        }, 1000);
    });
});
```

Here we use a do-it-yourself promise. A promise which we control so we decide when it's resolved/rejected.

What we do here is that we resolve the promise, which should mark the end of the function, and **then** we call `done()` with a completely different return value.

I see absolutely zero reason why a user would write code like this, today or tomorrow, unless they have a death wish because that's ground for a lot of very nasty bugs.

## Proposal

Implementing the code above, additionaly adding new warnings when `done()` is called:

```javascript
function done(err, result) {
    console.warn("Use of 'done' to complete tasks is deprecated. Please return a promise instead.");
    /* ... */
}
```

## Additional thoughts

While we maintain the possibility to use `done()`, users will need to have their tasks always return a promise, even if they perform no asynchronous operation.

```javascript
// Like that...
Task("TaskSync", async function () { // <- Note the 'async' which forces a promise
    return 42;
});

// Or like that...
Task("TaskSync", function () {
    return Promise.resolve(42);
});
```

In the next major versions of the Agent and NodeJS SDK that will follow the asynchronous one (or possibly later) it would be interesting to remove deprecation and totally forbid usage of the `done()` callback, throwing when it is called.

This will allow users to write synchronous tasks, which is impossible as long as we maintain the `done()` callback, because tasks not returning a promise are expected to call it to mark completion.

```javascript
// No promise returned, no 'async' keyword
Task("TaskSync", function () {
    return 42;
});
```

To better understand, you have to remember that the new asynchronous NodeJS stack always executes tasks handles in an asynchronous context anyway.

```javascript
await task.handle();
```

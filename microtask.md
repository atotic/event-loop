Promise spec:
https://promisesaplus.com/#point-34
2.2.4: onFulfilled or onRejected must not be called until the execution context stack contains only platform code. [3.1].
 This can be implemented with either a “macro-task” mechanism such as setTimeout or setImmediate, or with a “micro-task” mechanism such as MutationObserver or process.nextTick. Since the promise implementation is considered platform code, it may itself contain a task-scheduling queue or “trampoline” in which the handlers are called.
ECMAScript promise spec: https://tc39.github.io/ecma262/#sec-promise-objects


Promise

https://html.spec.whatwg.org/#microtask-queue

Each event loop has a microtask queue.

2 kinds of microtasks: solitary callback, compound microtasks. This spec only covers solitary.

Microtask can be moved to a regular task queue if it spins the event loop.

Algorithms:

perform a microtask checkpoint:


```
let MicrotaskQueue = function(eventLoop) {
    this.eventLoop = eventLoop;
}

MicrotaskQueue.prototype = {

    microtasks: [],

    queueTask: task => {
        this.microtasks.push(task);
    },

    getOldestTask: () => {
        return this.microtasks.shift();
    },

    performCheckpoint: () => {
        if (this.performing)
            return;
        this.performing = this.true;

        while (let task = this.getOldestTask()) {

            this.eventLoop.currentlyRunningTask = task;

            task.run();

            this.eventLoop.currentlyRunningTask = null;
        }

        // For each environment settings object whose responsible event loop
        // is this event loop, notify about rejected promises on that
        // environment settings object.
    }

}
```

When is MicrotaskQueue.performCheckpoint() called?

1) https://html.spec.whatwg.org/#clean-up-after-running-script
   If the JavaScript execution context stack is now empty, perform a microtask checkpoint.
2) https://html.spec.whatwg.org/#event-loop-processing-model
   Runs after task, before rendering
   Step ... execute task
   Step 6: Microtasks: Perform a microtask checkpoint.
   Step 7: Update the rendering
3) https://heycam.github.io/webidl/#es-invoking-callback-functions
   Invoking callback functions:
   Invoking callback performs "Clean up after running script with relevant settings"

Microtasks are queued using: https://html.spec.whatwg.org/#enqueuejob(queuename,-job,-arguments):queue-a-microtask

8.1.3.7.1 EnqueueJob(queueName, job, arguments)


ECMAScript promise spec: https://tc39.github.io/ecma262/#sec-promise-objects

ECMAScript defines EnqueueJob method for executing promise callbacks.

EnqueueJob is defined in https://html.spec.whatwg.org/#enqueuejob(queuename,-job,-arguments):queue-a-microtask


Firefox:

XPCJSRuntime::BeforeProcessTask
Promise::PerformMicroTaskCheckpoint

Bugs:
- PerformMicroTaskCheckpoint called from XPCJSRuntime::BeforeProcessTask
  it should happen after task

FF run & build
./mach build
./mach run

FF microtask bug:
https://bugzilla.mozilla.org/show_bug.cgi?id=1193394

1) Promise runs immediately after a callback
rAF
Promise.resolve().then(
var p = new Promise( function(fulfill, reject) {
        fulfill();
        // log("promise A fulfill");
    })
  promise.resolve()
    assert
2) Promise resolved inside a promise is run immediately
3) Inside event loop, promise runs after a task, and before rAF
4) What happens with alerts

Firefox rAF
schedule:
nsGlobalWindow::RequestAnimationFrame
  mDoc->ScheduleFrameRequestCallback
    nsRefreshDriver::ScheduleFrameRequestCallbacks
      mFrameRequestCallbackDocs.AppendElement(aDocument);
dispatch:
nsRefreshDriver::Tick
  nsRefreshDriver::RunFrameRequestCallbacks

Firefox promise dispatch
dispatches from:
Promise::PerformMicroTaskCheckpoint
called from:
nsGlobalWindow::RunTimeoutHandler
XPCJSRuntime::BeforeProcessTask

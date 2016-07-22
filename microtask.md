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
Questions:
- does this mean "notify about rejected promises" is not a task?


# event-loop
https://html.spec.whatwg.org/multipage/webappapis.html#processing-model-8

Browsing content has a corresponding WindowProxy object.

Browsing contect can have:
creator browsing context: (one responsible for creation)
    it can be parent
    it can be opener
    it can be empty

idle tasks

what answers am I trying to find out today?
- clarify pseudo-code: layout & style recomputation
- events (aka tasks): clarify handling
- observers
- callbacks
- microtasks

paper strategy:
- pile on the facts, then edit for clarity

# Event loop explainer

## What is the event loop?

The event loop is the mastermind that orchestrates:

* **what** JavaScript code gets executed

* **when** does it run

* when do layout and style get updated

* paint: when do DOM changes get rendered

It is formally specified in whatwg's [HTML standard](https://html.spec.whatwg.org/multipage/webappapis.html#event-loops).

The current specification is incomplete. The event loop behavior differs between browsers. Further work is being done to clarify the issues:

* [Rendering Processing Model](https://docs.google.com/document/d/1Mw6qNw8UAEfW96CXaXRVYPPZjqQS3YdK7v57wFttAhs/edit?pref=2&pli=1#)

* [Discussion on event order](https://github.com/w3c/pointerevents/issues/9)

This document is a developer-oriented description of what event loop does. It tries to be readable, and might skip over edge cases for clarity.

## Event loop description

This is what the spec says:

    eventLoop = {
        taskQueues: {
            events: [], // UI events from native GUI framework
            parser: [], // HTML parser
            callbacks: [], // setTimeout, requestIdleTask
            resources: [], // image loading
            domManipulation[]
        },

        microtaskQueue: [
        ],

        nextTask: function() {
            // Spec says:
            // "Select the oldest task on one of the event loop's task queues"
            // Which gives browser implementers freedom to implement
            // task queues
            for (let q of taskQueues)
                if (q.length > 0)
                    return q.shift();
            return null;
        },

        executeMicrotasks: function() {
            if (scriptExecuting)
                return;
            let microtasks = this.microtaskQueue;
            this.microtaskQueue = [];
            for (let t of microtasks)
                t.execute();
        },

        needsRendering: function() {
            return vSyncTime() && needsDomRerender();
        },

        render: function() {
            resizeSteps();
            scrollSteps();
            mediaQuerySteps();
            cssAnimationSteps();
            fullscreenRenderingSteps();

            animationFrameCallbackSteps();

            intersectionObserverSteps();

            while (resizeObserverSteps()) {
                updateStyle();
                updateLayout();
            }
            paint();
        }
    }

    while(true) {
        task = eventLoop.nextTask();
        if (task) {
            task.execute();
        }
        eventLoop.executeMicrotasks();
        if (eventLoop.needsRendering())
            eventLoop.render();
    }

## Chrome implementation of the event loop

### Events

DOM has 250 event types. See https://cs.chromium.org/chromium/src/out/Debug/gen/blink/core/EventTypeNames.h

To find out how any particular event is handled, follow the code search links in code searh.

Events are dispatched by following methods:

#### 1. DOMWindowEventQueue

Triggered by a timer

Sample events: window.storage, window.hashchange, document.selectionchange

#### 2. ScriptedAnimationController

Triggered by BeginMainFrame function that's called by Frame.

Also manages requestAnimationFrame requests

sample events: Animation.finish, Animation.cancel, CSSAnimation.animationstart, CSSAnimation.animationiteration(CSSAnimation)

#### 3. Custom dispatch

Triggers vary: OS events, timers, document/element lifecycle events.

Custom dispatch event do not pass through queues, they are fired
directly.

There are no globally applicable delivery guarantees for custom
events. Specific events might have event-specific guarantees
about ordering.

#### 4. Microtask queue

Triggered most often by EndOfTaskRunner.didProcessTask().

Tasks are run by TaskQueueManager. They are used internally by other dispatchers to schedule execution.

Microtask queue executes whenever task completes.

sample events: Image.onerror, Image.onload

Microtasks also contain Promise callbacks

### [Timers](https://developer.mozilla.org/en-US/Add-ons/Code_snippets/Timers)

#### [requestIdleCallback](https://w3c.github.io/requestidlecallback/#the-requestidlecallback-methodsetTimeout)

Triggered by internal timer when browser is idle.

#### [requestAnimationFrame](https://html.spec.whatwg.org/multipage/webappapis.html#animation-frames)

Triggered by ScriptedAnimationController, that also handles events.

#### [Timers](https://html.spec.whatwg.org/multipage/webappapis.html#timers:dom-setinterval): setTimeout, setInterval

Triggered by WebTaskRunner, which runs on TaskQueue primitive.

### Observers

Observers watch for changes, and report all observations at once.

There are two ways of watching for changes:

1. Push: trap changes when they happen, and push to observation queue.

2. Poll: poll for changes when it is time to broadcast.

#### [MutationObserver](https://dom.spec.whatwg.org/#mutation-observers)

Push-based.

Observations broadcast is placed on microtask queue.

#### [IntersectionObserver](http://rawgit.com/WICG/IntersectionObserver/master/index.html)

Poll-based.

Observations poll on layout, broadcast via 100ms timeout.

### Promises

Completed promises run callbacks after completion.

Callbacks are placed on the microtask queue.

## Developer takeaways

Now we understand the spec, and how Chrome implements it.

Event loop does not say much about when events are dispatched:

1. Events on the same queue are dispatched in order.

2. Events can be dispatched directly, bypassing the event loop task queues.

3. Microtasks get executed immediately after a task.

4. Render part of the loop gets executed on vSync, and delivers events in the following order:

    1. 'resize' event

    2. 'scroll' event

    3. mediaquery listeners

    4. 'CSSAnimation' events

    5. Observers

    6. rAF

## What really happens

We've build a test rig, 'shell.html' that
## Multiple event loops and their interaction

## Examples

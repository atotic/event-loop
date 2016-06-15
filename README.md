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

It is formally specified in whatwg's [HTML standard](https://html.spec.whatwg.org/#event-loops).

The current specification is incomplete. The event loop behavior differs between browsers. Further work is being done to clarify the issues:

* [Rendering Processing Model](https://docs.google.com/document/d/1Mw6qNw8UAEfW96CXaXRVYPPZjqQS3YdK7v57wFttAhs/edit?pref=2&pli=1#)

This document is a developer-oriented description of what event loop does. It tries to be readable, and might skip over edge cases for clarity.

## Why is the event loop interesting?

Event loop is in charge.
All the application code is scheduled for execution by the event loop.

Event loop is interesing because it influences performance and correctness of your app.

If your event loop takes more than 16ms, animations might skip a frame. If your code is split into multiple event handlers, Promise callbacks, timeouts, observers, in what order will the code execute? Do you really need to `setTimeout(_=>{doIt();}, 1)` because things were not executing in the right order?

Understanding the event loop will help you understand when your code gets executed, and its performance implications.

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
            // microtasks are:
            // - completed promises
        ],

        executeMicrotasks: function() {
            let microtasks = this.microtaskQueue;
            this.microtaskQueue = [];
            for (let t of microtasks)
                t.execute();
        },
        needsPaint: function() {
            return vSyncTime() && needsDomRepaint();
        },

        paintPipeline: function() {
            resizeSteps();
            scrollSteps();
            mediaQuerySteps();
            cssAnimationSteps();
            fullscreenRenderingSteps();

            animationFrameCallbackSteps();

            intersectionObserverSteps();

            updateStyle();
            updateLayout();
            paint();
        }
    }

    while(true) {
        task = eventLoop.nextTask();
        if (task) {
            task.execute();
        }
        eventLoop.executeMicrotasks();
        if (eventLoop.needsPaint())
            eventLoop.paintPipeline();
    }

This is what really happens in Chrome:

    eventLoop = {
        taskQueues: {
            events: [], // UI events from native GUI framework
            parser: [], // HTML parser
            callbacks: [], // setTimeout, requestIdleTask
            resources: [], // image loading
            domManipulation[]
        },

        microtasks: [
            // microtasks are:
            // - completed promises
        ],

        executeMicrotasks: function() {
            let microtasks = this.microtasks;
            this.microtasks = [];
            for (let t of microtasks)
                t.execute();
        },
        needsPaint: function() {
            return vSyncTime() && needsDomRepaint();
        },

        paintPipeline: function() {
            mediaQuerySteps();
            resizeSteps();
            scrollSteps();
            cssAnimationSteps();
            fullscreenRenderingSteps();

            animationFrameCallbackSteps();

            intersectionObserverSteps();

            updateStyle();
            updateLayout();
            paint();
        }
    }

    while(true) {
        task = eventLoop.nextTask();
        if (task) {
            task.execute();
        }
        eventLoop.executeMicrotasks();
        if (eventLoop.needsPaint())
            eventLoop.paintPipeline();
    }

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

#### 3. ad-hoc dispatch

Triggers vary: mouse events, keyboard events, timers.

Many events get dispatched immediately, without pausing in a queue.

#### 4. Microtask queue

Triggered most often by EndOfTaskRunner.didProcessTask().
Tasks are run by TaskQueueManager.
What are tasks? They are used internally by other dispatchers to schedule
execution.


### [Timers](https://developer.mozilla.org/en-US/Add-ons/Code_snippets/Timers)

#### [requestIdleCallback](https://w3c.github.io/requestidlecallback/#the-requestidlecallback-methodsetTimeout)

Triggered by internal timer when browser is idle.

#### requestAnimationFrame

Triggered by ScriptedAnimationController, that also handles events.

#### setTimeout, setInterval

Triggered by WebTaskRunner, which runs on TaskQueue primitive.

### Observers

Observers watch for changes, and report all observations at once.

There are two ways of watching for changes:

1. Push: trap changes when they happen, and push to observation queue.

2. Poll: poll for changes when it is time to broadcast.

#### MutationObserver

Push-based.

Observations broadcast is placed on microtask queue.

#### IntersectionObserver

Poll-based.

Observations broadcast via timeout.

### Promises

Completed promises run callbacks after completion.

Callbacks are placed on the microtask queue.

## Multiple event loops and their interaction

## Examples


# Event loop explainer

## What is the event loop?

The event loop is the mastermind that orchestrates:

* **what** JavaScript code gets executed

* **when** does it run

* when do layout and style get updated

* render: when do DOM changes get rendered

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
            // Which gives browser implementers lots of freedom
            // Queues can have different priorities, etc.
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
            return vSyncTime() && (needsDomRerender() || hasEventLoopEventsToDispatch());
        },

        render: function() {
            dispatchPendingUIEvents();
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
            if (updatedLayout) {
              scheduleAsyncHoverUpdate();
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



## Close reading of the spec

Now we understand the spec.

Event loop does not say much about when events are dispatched:

1. Events on the same queue are dispatched in order.

2. Events can be dispatched directly, bypassing the event loop task queues.

3. Microtasks get executed immediately after a task.

4. Render part of the loop gets executed on vSync, and delivers events in the following order:
    1. dispatch pending UI events

    2. 'resize' event

    3. 'scroll' event

    4. mediaquery listeners

    5. 'CSSAnimation' events

    6. Observers

    7. rAF

## Pending UI events

There are two classes of UI events:

1. Discrete - those that aren't continuous (eg. mousedown, mouseup,
   touchstart, touchend)

2. Continuous - mousewheel, wheel, mousemove, pointermove, touchmove.

Continuous events may be coalesced (updating positions, magnitude) with
matching events (that haven't yet been dispatched) while being held in the
UI event task queue.

Ordering of discrete and continuous events must be preserved. Discrete events
should be dispatched right as soon as possible when received from the
hardware. Continuous events can be held and dispatched in the render part
of the event loop. If a discrete event is received, all continuous events
in the task queue should run immediately to prevent the discrete event
from being delayed.

## Asynchronously Update Hover State

After a layout has occurred the user agent will update the hover state of the
current node since during a layout the position may have shifted from the
previous hover state. If the hover state changes the CSS hover rule will
be updated as well mouseout, mouseleave, mouseenter, mouseover will be
dispatched to the relevant nodes.

## What really happens

We've built a [test page]( https://rawgit.com/atotic/event-loop/master/shell.html). The page simultaneously generates (listed in order):

* 2 requestAnimationFrames

* 2 setTimeout(0)

* 2 chained promises

* CSSAnimation

* scroll event

* resize event

We compare when these callbacks get executed to what is expected by the
spec. Here are the results from major browsers. Spoiler: they are all different.

### Chrome 51

    301.66 script start
    302.91 script end
    303.31 promise 0
    303.86 promise 1
    305.43 timeout 0
    305.83 timeout 1
    316.21 scroll
    316.62 matchMedia
    316.92 resize
    317.29 animationstart
    317.62 rAF 0
    318 rAF 0 promise
    318.31 rAF 1
    17ms

Chrome behavior almost matches the spec. Failure:

* scroll event fires before resize event. It looks like Chrome puts scroll and resize on the same task queue.

Chrome's conformance was expected, as spec was written with large
input by the Chrome team :)

### Firefox 47

    322 script start
    323.47 animationstart
    324.58 script end
    326.2 scroll
    327.96 matchMedia
    330.03 resize
    338.39 promise 0
    339.13 promise 1
    339.94 timeout 0
    341.11 timeout 1
    356.52 rAF 0
    357.22 rAF 1
    362.27 rAF 0 promise
    40ms

Firefox diverges from the spec.

* scroll event fires before resize event.

* Promises are not on a microtask queue, but a different task queue. Instead of executing immediately after `scriptEnd`, they execute after
`resize`, and after `rAF 1` instead of `rAF 0`.

* Timeout fires in the middle of the render part of the loop. According to spec, it should fire either before, or after, but not in the middle.

* CSSAnimation event is delivered immediately, and not inside the
`render` block.

### Safari 9.1.2

    328.17 script start
    329.32 script end
    329.53 promise 0
    329.96 scroll
    330.24 timeout 0
    330.38 timeout 1
    330.5 promise 1
    330.81 matchMedia
    332.57 animationstart
    332.88 resize
    344.67 rAF 0
    344.95 rAF 0 promise
    345.09 rAF 1
    17ms

Safari diverges from the spec:

* scroll event fires before resize event.

* Chained Promise does not execute immediately, it happens after timeout.

* timeout fires in the middle of `render` block, between `scroll` and `matchMedia`.

### Microsoft Edge XXXX

    510.09 script start
    512.08 script end
    512.4 promise 0
    512.69 promise 1
    512.89 timeout 0
    513.06 timeout 1
    513.3 animationstart
    526.24 scroll
    528.1 rAF 0
    528.44 rAF 0 promise
    528.67 rAF 1
    528.93 matchMedia
    529.17 resize
    19ms

Microsoft diverges from the spec. It does well with Promises and timeout,
but mostly ignores `render` event ordering from the spec, sprinking
different events all over.

## Conclusion

There are significant differences between browsers' implementation of the event loop. The spec is not helpful in determining what actually happens.

It will be hard for developers to try to understand when is their
arbitrary code going to get executed. If order of execution is important,
be careful from the start: architect your ordered callbacks using only
primitives whose behavior is well understood.

Here are a few rules of thumb:

* callbacks of the same type always execute in order requested. Callbacks of different types execute in unspecified order. Therefore, you should not mix callback types in your callback chain.

* rAF always fire before rendering, no more than 60 times/sec. If your callbacks are manipulating DOM, use rAF, so that you do not disturb layout unless painting.

* timeout(0) fires "in the next task loop". Use sparingly, only if you need high-frequency callbacks that are not tied to drawing.

* Promises fire "sooner than timeout".

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

#### 5. Main Thread Event Queue

Implementation of the UI event task queue. Events are coalesced in this queue.
This queue also handles requesting the vSync when continuous events have
been placed in this task queue.


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

## Multiple event loops and their interaction

## Examples

# event-loop
https://html.spec.whatwg.org/multipage/webappapis.html#processing-model-8

Browsing content has a corresponding WindowProxy object.

Browsing contect can have:
creator browsing context: (one responsible for creation)
    it can be parent
    it can be opener
    it can be empty

# Event loop explainer

## What is the event loop?

The event loop is the mastermind that orchestrates:

* **what** JavaScript code gets executed

* **when** does it run

* when do layout and style get updated

* paint: when do DOM changes get rendered

It is formally specified in whatwg's [HTML standard](https://html.spec.whatwg.org/#event-loops).

The current specification is incomplete, and behavior differs between browsers. Further work is being done to clarify the issues:

* [Rendering Processing Model](https://docs.google.com/document/d/1Mw6qNw8UAEfW96CXaXRVYPPZjqQS3YdK7v57wFttAhs/edit?pref=2&pli=1#)

This document is a developer-oriented description of what event loop does. It tries to be readable, and might skip over edge cases for clarity (unlike a formal spec).

## Why is the event loop interesting?

Event loop is interesing because it influences performance and correctness of your app.

All application code gets executed by the event loop. At the end of the loop, the browser might paint.

If your event loop takes more than 16ms, animations might skip a frame. If your code is split into multiple event handlers, Promise callbacks, timeouts, observers, in what order will the code execute? Do you really need to `setTimeout(_=>{doIt();}, 1)` because things were not executing in the right order?

Understanding the event loop will help you understand when your code gets executed, and its performance implications.

## Event loop description

Event loop has a task queue, and a microtask queue.

eventLoop = {
    tasks: [],
    microtasks: [],
    executeMicrotasks: function() {
        let microtasks = this.microtasks;
        this.microtasks = [];
        for (let t of microtasks)
            t.execute();
    }
}

while(true) {
    task = eventLoop.tasks.shift();
    task.execute();
    eventLoop.executeMicrotasks();
    if (eventLoop.needDrawing()) {
        resizeSteps();
        scrollSteps();
        mediaQuerySteps();
        cssAnimationSteps();
        fullscreenRenderingSteps();
        animationFrameCallbackSteps();
        intersectionObserverSteps();
        updateStyleAndLayout();
    }
}

### What

## Multiple event loops and their interaction

## Examples

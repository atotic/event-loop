### Moving to animation tasks model

5) Run the resize steps

If doc’s viewport has had its width or height changed (e.g. as a result of the user resizing the browser window, or changing the page zoom scale factor, or an iframe element’s dimensions are changed) since the last time these steps were run, fire an event named resize at the Window object associated with doc.

6) Run the scroll steps
For each item target in doc’s pending scroll event targets, in the order they were added to the list, run these substeps:

If target is a Document, fire an event named scroll that bubbles at target.
Otherwise, fire an event named scroll at target.
Empty doc’s pending scroll event targets.
Fires an event, first target gets notified first

7) Evaluate media queries and report changes

For each MediaQueryList object target that has doc as its document, in the order they were created, oldest first, run these substeps:

If target’s matches state has changed since the last time these steps were run, dispatch a new event to target using the MediaQueryList interface, with its type attribute initialized to change, its isTrusted attribute initialized to true, its media attribute initialized to target’s media, and its matches attribute initialized to target’s matches state.

8) run CSS animations and send events

Not specified.

9)  run the fullscreen rendering steps

Not specified.

10) run the animation frame callbacks

For each entry in callbacks, in order: invoke the callback, passing now as the only argument, and if an exception is thrown, report the exception.

11) run the update intersection observations steps

Let observer list be a list of all IntersectionObservers whose root is in the DOM tree of document.
For each observer in observer list:


### Can, and should, this be refactored?



##
Existing event loop spec##
  <p>An <span>event loop</span> must continually run through the following steps for as long as it
  exists:</p>

  <ol>

   <!-- lots of places in the spec refer to "step 1" -->

   <li id="step1">

    <p>Select the oldest <span data-x="concept-task">task</span> on one of the <span>event
    loop</span>'s <span data-x="task queue">task queues</span>, if any, ignoring, in the case of a
    <span>browsing context</span> <span>event loop</span>, tasks whose associated
    <code>Document</code>s are not <span>fully active</span>. The user agent may pick any <span>task
    queue</span>. If there is no task to select, then jump to the <i>microtasks</i> step below.</p>

   </li>

   <!-- note: reference to 'previous step' below -->

   <li><p>Set the <span>event loop</span>'s <span>currently running task</span> to the <span
   data-x="concept-task">task</span> selected in the previous step.</p></li>

   <li><p><i>Run</i>: Run the selected <span data-x="concept-task">task</span>.</p></li>

   <li><p>Set the <span>event loop</span>'s <span>currently running task</span> back to
   null.</p></li>

   <li><p>Remove the task that was run in the <i>run</i> step above from its <span>task
   queue</span>.</p></li>

   <li><p><i>Microtasks</i>: <span>Perform a microtask checkpoint</span>.</p></li>

   <li>

    <p><i>Update the rendering</i>: If this <span>event loop</span> is a <span>browsing
    context</span> <span>event loop</span> (as opposed to a <a href="#workers">worker</a>
    <span>event loop</span>), then run the following substeps.</p>

    <ol>

     <li><p>Let <var>now</var> be the value that would be returned by the <code>Performance</code>
     object's <code data-x="dom-Performance-now">now()</code> method. <ref spec=HRT></p>

     <li>

      <p>Let <var>docs</var> be the list of <code>Document</code> objects associated with the
      <span>event loop</span> in question, sorted arbitrarily except that the following conditions
      must be met:</p>

      <ul>

       <li><p>Any <code>Document</code> <var>B</var> that is <span>nested through</span> a
       <code>Document</code> <var>A</var> must be listed after <var>A</var> in the list.</p></li>

       <li><p>If there are two documents <var>A</var> and <var>B</var> whose <span
       data-x="concept-document-bc">browsing contexts</span> are both <span data-x="nested browsing
       context">nested browsing contexts</span> and their <span data-x="browsing context
       container">browsing context containers</span> are both elements in the same
       <code>Document</code> <var>C</var>, then the order of <var>A</var> and <var>B</var> in the
       list must match the relative <span>tree order</span> of their respective <span
       data-x="browsing context container">browsing context containers</span> in
       <var>C</var>.</p></li>

      </ul>

      <p>In the steps below that iterate over <var>docs</var>, each <code>Document</code> must be
      processed in the order it is found in the list.</p>

     </li>

     <li>

      <p>If there are <span data-x="top-level browsing context">top-level browsing contexts</span>
      <var>B</var> that the user agent believes would not benefit from having their rendering
      updated at this time, then remove from <var>docs</var> all <code>Document</code> objects whose
      <span data-x="concept-document-bc">browsing context</span>'s <span>top-level browsing
      context</span> is in <var>B</var>.</p>

      <div class="note">
       <p>Whether a <span>top-level browsing context</span> would benefit from having its rendering
       updated depends on various factors, such as the update frequency. For example, if the browser
       is attempting to achieve a 60Hz refresh rate, then these steps are only necessary every 60th
       of a second (about 16.7ms). If the browser finds that a <span>top-level browsing
       context</span> is not able to sustain this rate, it might drop to a more sustainable 30Hz for
       that set of <code>Document</code>s, rather than occasionally dropping frames. (This
       specification does not mandate any particular model for when to update the rendering.)
       Similarly, if a <span>top-level browsing context</span> is in the background, the user agent
       might decide to drop that page to a much slower 4Hz, or even less.</p>

       <p>Another example of why a browser might skip updating the rendering is to ensure certain
       <span data-x="concept-task">tasks</span> are executed immediately after each other, with only
       <span data-x="perform a microtask checkpoint">microtask checkpoints</span> interleaved (and
       without, e.g., <span data-x="run the animation frame callbacks">animation frame
       callbacks</span> interleaved). For example, a user agent might wish to coalesce timer
       callbacks together, with no intermediate rendering updates.</p>
      </div>

     </li>

     <li>

      <p>If there are a <span data-x="nested browsing context">nested browsing contexts</span>
      <var>B</var> that the user agent believes would not benefit from having their rendering
      updated at this time, then remove from <var>docs</var> all <code>Document</code> objects whose
      <span data-x="concept-document-bc">browsing context</span> is in <var>B</var>.</p>

      <p class="note">As with <span data-x="top-level browsing context">top-level browsing
      contexts</span>, a variety of factors can influence whether it is profitable for a browser to
      update the rendering of <span data-x="nested browsing context">nested browsing
      contexts</span>. For example, a user agent might wish to spend less resources rendering
      third-party content, especially if it is not currently visible to the user or if resources are
      constrained. In such cases, the browser could decide to update the rendering for such content
      infrequently or never.</p>

     </li>

     <li><p>For each <span>fully active</span> <code>Document</code> in <var>docs</var>, <span>run the resize
     steps</span> for that <code>Document</code>, passing in <var>now</var> as the timestamp. <ref
     spec="CSSOMVIEW"/></p></li>

     <li><p>For each <span>fully active</span> <code>Document</code> in <var>docs</var>, <span>run the scroll
     steps</span> for that <code>Document</code>, passing in <var>now</var> as the timestamp. <ref
     spec="CSSOMVIEW"/></p></li>

     <li><p>For each <span>fully active</span> <code>Document</code> in <var>docs</var>, <span>evaluate media queries
     and report changes</span> for that <code>Document</code>, passing in <var>now</var> as the timestamp. <ref
     spec="CSSOMVIEW"/></p></li>

     <li><p>For each <span>fully active</span> <code>Document</code> in <var>docs</var>, <dfn>run CSS animations and send
     events</dfn> for that <code>Document</code>, passing in <var>now</var> as the timestamp. <ref
     spec="CSSANIMATIONS"/></p></li>

     <li><p>For each <span>fully active</span> <code>Document</code> in <var>docs</var>, <dfn>run the fullscreen rendering
     steps</dfn> for that <code>Document</code>, passing in <var>now</var> as the timestamp. <ref
     spec="FULLSCREEN"/></p></li>

     <li><p>For each <span>fully active</span> <code>Document</code> in <var>docs</var>, <span>run the animation frame
     callbacks</span> for that <code>Document</code>, passing in <var>now</var> as the
     timestamp.</p></li>

     <li><p>For each <span>fully active</span> <code>Document</code> in <var>docs</var>, <span>run the update
     intersection observations steps</span> for that <code>Document</code>, passing in <var>now</var> as the
     timestamp. <ref spec="INTERSECTIONOBSERVER"/></p></li>

     <li><p>For each <span>fully active</span> <code>Document</code> in <var>docs</var>, update the
     rendering or user interface of that <code>Document</code> and its <span
     data-x="concept-document-bc">browsing context</span> to reflect the current state.</p></li>

    </ol>

   </li>

   <li><p>If this is a <a href="#workers">worker</a> <span>event loop</span> (i.e. one running for a
   <code>WorkerGlobalScope</code>), but there are no <span data-x="concept-task">tasks</span> in the
   <span>event loop</span>'s <span data-x="task queue">task queues</span> and the
   <code>WorkerGlobalScope</code> object's <span
   data-x="dom-WorkerGlobalScope-closing">closing</span> flag is true, then destroy the <span>event
   loop</span>, aborting these steps, resuming the <span>run a worker</span> steps described in the
   <a href="#workers">Web workers</a> section below.</p></li>

   <li><p>Return to the <a href="#step1">first step</a> of the <span>event loop</span>.</p></li>

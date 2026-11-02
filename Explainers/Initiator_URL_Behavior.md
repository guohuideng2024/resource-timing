# initiatorUrl in Resource Timing: Behavior Description

**Status:** Draft for discussion, November 2026

**Author:** Guohui Deng (GitHub: [guohuideng2024](https://github.com/guohuideng2024))

## Links

- Spec discussion: <https://github.com/w3c/resource-timing/issues/263>
- Explainer: <https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/ResourceTimingInitiatorInfo/explainer.md>
- Chrome Status: <https://chromestatus.com/feature/4876586949804032>
- Chromium bug: <https://issuetracker.google.com/issues/40919714>
- Web Platform Tests: <https://github.com/web-platform-tests/wpt/tree/master/resource-timing/tentative/initiator-url>
- Prior work: <https://chromestatus.com/feature/6197847657021440> and the related discussion in <https://github.com/w3c/resource-timing/issues/380>

## 1. Introduction

For background, see the
[explainer](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/ResourceTimingInitiatorInfo/explainer.md).
The proposed API is:

```webidl
partial interface PerformanceResourceTiming {
  readonly attribute DOMString initiatorUrl;
};
```

This document defines what `initiatorUrl` is for each resource.

## 2. Key concept: the initiating script

When a resource is requested while script is running, the initiator is **not**
the script that contains the function that made the request. Instead, it is
the script that started the current chain of execution. This document calls it
the *initiating script*.

- When a classic script or module script is evaluated, it becomes the
  initiating script for everything it runs synchronously, including calls into
  functions defined in other scripts.
- An inline script is identified by the URL of its document.
- When script schedules a callback, the callback keeps the initiating script of
  the code that scheduled it. This applies to timers, animation frame and idle
  callbacks, `scheduler.postTask()`, promise reactions, microtasks, event
  listeners and event handler attributes, and message handlers.

For example, suppose `page.html` contains an inline script that calls
`setTimeout(loadImage, 0)`, and `loadImage()` is defined in `lib.js`. The image
fetched by `loadImage()` reports `page.html` as its initiator, because the
inline script in `page.html` scheduled the callback. `lib.js` only provides the
function.

This makes the reported initiator stable: it points at the code that decided
to load the resource, rather than at a shared helper library.

## 3. Behavior

### 3.1 Resources requested by the document's markup

Resources requested by the HTML parser have the document's URL as their
initiator. This includes:

- `<script src>`, including `async`, `defer` and `type="module"` scripts.
- `<link rel="preload">` and `<link rel="stylesheet">`.
- `<img>` and `<iframe>`.
- Resources referenced from an inline `<style>` element, such as a background
  image or an `@font-face` font.

Tests: `static-resource.html`.

### 3.2 Resources requested by script

A resource requested while script is running has the initiating script's URL
as its initiator (section 2). This applies whether the request is made through
an API such as `fetch()`, or by inserting an element such as `<img>`, `<link>`,
`<script>` or `<iframe>`.

- A resource requested by an inline script reports the document's URL.
- A resource requested by an external classic script reports that script's URL.
- Calling a function defined in another script does not change the initiator.

Tests: `inline-script.html`, `external-script.html`,
`external-script-in-posted-task.html`.

### 3.3 Resources requested in callbacks

A resource requested in a callback reports the initiating script of the code
that scheduled or registered the callback. This holds even when the callback
function itself is defined in another script. Covered callbacks:

- `setTimeout()` and `setInterval()`.
- `requestAnimationFrame()` and `requestIdleCallback()`.
- `scheduler.postTask()`.
- Promise reactions and `queueMicrotask()`.
- Event listeners added with `addEventListener()`, and event handler
  attributes such as `onload`. When an event handler attribute is overwritten,
  the handler that runs determines the initiator.
- `XMLHttpRequest` event handlers.
- `MessagePort` message handlers. The initiator is determined by the receiver
  that registered the handler, not by the sender of the message (for example,
  an iframe).

Tests: `set-timeout.html`, `set-interval.html`,
`request-animation-frame.html`, `request-idle-callback.html`,
`post-task.html`, `promise.html`, `queue-microtask.html`,
`event-handler.html`, `event-handler-overwritten.html`,
`xml-http-request.html`, `iframe-post-message.html`.

### 3.4 Stylesheets

- Resources referenced by an external stylesheet, such as `@import` rules,
  `@font-face` fonts and `url()` images, report the stylesheet's URL.
- Resources referenced by an inline `<style>` element report the document's
  URL (section 3.1).
- The stylesheet itself follows the rules above: a stylesheet in markup reports
  the document, and a stylesheet inserted by script reports the initiating
  script.

Tests: `stylesheet-static.html`, `stylesheet-dynamic.html`.

### 3.5 Module scripts

- A module loaded by the document reports the document's URL. This includes a
  `<script type="module" src>` element in markup, a static `import` in an
  inline module script, a dynamic `import()` in an inline script, and a module
  script element added by an inline script (directly or from a callback it
  scheduled).
- A module requested by another module reports the importing module's URL.
  This applies to static imports, dynamic `import()`, and module script
  elements that the importing module adds to the document. It does not matter
  whether the importing module was itself loaded statically or dynamically.
- A dynamic `import()` in an external classic script reports that classic
  script's URL.

Tests: `module-script-from-document.html`,
`module-script-from-static-importer.html`,
`module-script-from-dynamic-importer.html`.

### 3.6 Workers

- The script of a dedicated worker, classic or module, reports the initiating
  script of the code that called `new Worker()`. For example, it is the
  document if an inline script started the worker, an external script if that
  script started it, and the document if an inline script scheduled a
  `setTimeout()` callback that started it.
- Inside a dedicated or shared worker, resources requested by the worker's
  code report the worker script's URL. This includes `fetch()`,
  `importScripts()`, callbacks (timers, promise reactions,
  `scheduler.postTask()`) and message handlers.
- Modules imported by a module worker, statically or dynamically, report the
  worker script's URL.
- The top-level scripts of shared workers and service workers do not produce a
  Resource Timing entry today, so they have no `initiatorUrl`.

Tests: `dedicated-worker-script.html`, `direct.any.js`, `set-timeout.any.js`,
`set-interval.any.js`, `promise.any.js`, `post-task.any.js`,
`queue-microtask.any.js`, `post-message-worker.html`,
`post-message-shared-worker.html`, `module-script-from-worker.html`.

## 4. When initiatorUrl is empty

`initiatorUrl` is the empty string when:

1. **There is no initiator.** For example, the main document of a top-level
   navigation.
2. **The resource was not fetched from the network.** Resources served from the
   cache or handled by a service worker still have Resource Timing entries, but
   they are not part of the network dependency tree, so their `initiatorUrl` is
   intentionally empty. This also keeps the cost low for fast loads.
3. **The browser does not report the initiator for that kind of request.**
   Implementations may cover only part of the cases above and improve over
   time; partial data is still useful.

## 5. Not covered

- **`document.write()`.** Reporting an initiator for resources inserted by
  `document.write()` is not planned, because the API is deprecated. The current
  tests for it should be removed.

## 6. Security and privacy

`initiatorUrl` only reveals which URL caused a resource to be fetched. Like
other detailed Resource Timing information, it is exposed only when the
resource passes a CORS check; `Timing-Allow-Origin` alone does not expose it.
See the explainer for the full self-review questionnaire.

## 7. Alternatives considered

- **A numeric ID for each resource.** Consumers would need the IDs of all
  resources to interpret the data, which makes it harder to use.
- **A reference to the initiator's `PerformanceResourceTiming` entry.** The
  initiator's entry may already have been dropped from the buffer when the
  dependent resource is reported.

A URL is simple to consume. If the same URL was fetched several times, the
initiating instance is the most recent fetch of that URL.

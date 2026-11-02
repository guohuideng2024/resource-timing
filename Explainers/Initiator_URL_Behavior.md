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

## 2. Status

The behavior described here is what Chromium implements, behind the
`ResourceTimingInitiator` runtime flag. It is a proposal for discussion and
may change.

Some cases may be hard to implement in other browsers. For such a case, we can
change the behavior described here. As a last resort, the spec could allow
more than one value for that case, but this is highly undesirable because it
breaks interoperability.

## 3. Behavior

All the tests are in
[resource-timing/tentative/initiator-url/](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/).

In the examples, a comment such as `// a.png: page.html` means that the
`initiatorUrl` of `a.png` is the URL of `page.html`. File names stand for
absolute URLs.

### 3.1 Resources in HTML markup

Resources requested by the HTML parser have the document's URL as their
initiator. This includes:

- `<script src>`, including `async`, `defer` and `type="module"` scripts.
- `<link rel="preload">` and `<link rel="stylesheet">`.
- `<img>` and `<iframe>`.
- Resources referenced from an inline `<style>` element, such as a background
  image or an `@font-face` font.

Example:

```html
<!-- page.html -->
<script src="a.js"></script>          <!-- a.js: page.html -->
<link rel="stylesheet" href="s.css">  <!-- s.css: page.html -->
<img src="i.png">                     <!-- i.png: page.html -->
<style>
  body { background: url(bg.png); }   /* bg.png: page.html */
</style>
```

Tests:

- [static-resource.html](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/static-resource.html)

### 3.2 Resources referenced by CSS

- Resources referenced by an external stylesheet, such as `@import` rules,
  `@font-face` fonts and `url()` images, report the stylesheet's URL. This is
  the same whether the stylesheet is in the markup or inserted by script.
- Resources referenced by an inline `<style>` element report the document's
  URL (section 3.1).
- The stylesheet itself follows the rules for markup (section 3.1) or for
  JavaScript (section 3.3), depending on how it was inserted.

Example:

```css
/* s.css */
@import "other.css";                               /* other.css: s.css */
@font-face { font-family: f; src: url(f.ttf); }    /* f.ttf: s.css */
body { background: url(bg.png); }                  /* bg.png: s.css */
```

```html
<!-- page.html -->
<script>
  const link = document.createElement("link");
  link.rel = "stylesheet";
  link.href = "s.css";
  document.head.appendChild(link);  // s.css: page.html
                                    // resources in s.css: s.css
</script>
```

Tests:

- [stylesheet-static.html](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/stylesheet-static.html)
- [stylesheet-dynamic.html](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/stylesheet-dynamic.html)

### 3.3 Resources requested by JavaScript

When a resource is requested while script is running, the initiator is **not**
the script that contains the function that made the request. Instead, it is
the script that started the current chain of execution. This document calls it
the *initiating script*. An inline script is identified by the URL of its
document.

This applies whether the request is made through an API such as `fetch()`, or
by inserting an element such as `<img>`, `<link>`, `<script>` or `<iframe>`.

#### 3.3.1 Synchronous execution

When a classic script is evaluated, it becomes the initiating script for
everything it runs synchronously, including calls into functions defined in
other scripts.

- A resource requested by an inline script reports the document's URL.
- A resource requested by an external classic script reports that script's URL.
- Calling a function defined in another script does not change the initiator.

Example:

```html
<!-- page.html -->
<script src="lib.js"></script>     <!-- lib.js defines loadImage() -->
<script>
  fetch("a.json");                 // a.json: page.html
  loadImage("b.png");              // b.png: page.html, not lib.js
</script>
<script src="loader.js"></script>  <!-- loader.js runs fetch("c.json") -->
                                   <!-- c.json: loader.js -->
```

Tests:

- [inline-script.html](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/inline-script.html)
- [external-script.html](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/external-script.html)

#### 3.3.2 Callbacks

A callback keeps the initiating script of the code that scheduled or
registered it. This holds even when the callback function itself is defined in
another script. Covered callbacks:

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

Example:

```html
<!-- page.html -->
<script src="lib.js"></script>  <!-- lib.js defines loadImage() -->
<script>
  setTimeout(() => loadImage("a.png"), 0);   // a.png: page.html
  scheduler.postTask(() => fetch("b.json")); // b.json: page.html
  Promise.resolve().then(() => fetch("c.json"));
                                             // c.json: page.html
  window.addEventListener("load", () => fetch("d.json"));
                                             // d.json: page.html
</script>
```

Tests:

- [set-timeout.html](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/set-timeout.html)
- [set-interval.html](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/set-interval.html)
- [request-animation-frame.html](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/request-animation-frame.html)
- [request-idle-callback.html](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/request-idle-callback.html)
- [post-task.html](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/post-task.html)
- [external-script-in-posted-task.html](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/external-script-in-posted-task.html)
- [promise.html](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/promise.html)
- [queue-microtask.html](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/queue-microtask.html)
- [event-handler.html](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/event-handler.html)
- [event-handler-overwritten.html](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/event-handler-overwritten.html)
- [xml-http-request.html](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/xml-http-request.html)
- [iframe-post-message.html](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/iframe-post-message.html)

### 3.4 Module scripts

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

Example:

```html
<!-- page.html -->
<script type="module">
  import "./a.mjs";   // a.mjs: page.html
  import("./b.mjs");  // b.mjs: page.html
</script>
<script src="classic.js"></script>
```

```js
// a.mjs
import "./c.mjs";     // c.mjs: a.mjs
import("./d.mjs");    // d.mjs: a.mjs
const script = document.createElement("script");
script.type = "module";
script.src = "./e.mjs";
document.head.appendChild(script);  // e.mjs: a.mjs
```

```js
// classic.js (a classic script)
import("./f.mjs");    // f.mjs: classic.js
```

Tests:

- [module-script-from-document.html](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/module-script-from-document.html)
- [module-script-from-static-importer.html](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/module-script-from-static-importer.html)
- [module-script-from-dynamic-importer.html](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/module-script-from-dynamic-importer.html)

### 3.5 Workers

#### 3.5.1 Worker scripts

The script of a dedicated worker, classic or module, follows section 3.3: it
reports the initiating script of the code that called `new Worker()`. For
example, it is the document if an inline script started the worker, an
external script if that script started it, and the document if an inline
script scheduled a `setTimeout()` callback that started it.

The top-level scripts of shared workers and service workers do not produce a
Resource Timing entry today, so they have no `initiatorUrl`.

Example:

```html
<!-- page.html -->
<script>
  new Worker("w1.js");                           // w1.js: page.html
  setTimeout(() => new Worker("w2.js"), 0);      // w2.js: page.html
</script>
<script src="starter.js"></script>  <!-- starter.js runs new Worker("w3.js") -->
                                    <!-- w3.js: starter.js -->
```

Tests:

- [dedicated-worker-script.html](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/dedicated-worker-script.html)

#### 3.5.2 Resources requested inside a worker

- Inside a dedicated or shared worker, resources requested by the worker's
  code report the worker script's URL. This includes `fetch()`,
  `importScripts()`, callbacks (timers, promise reactions,
  `scheduler.postTask()`, microtasks) and message handlers.
- Modules imported by a module worker, statically or dynamically, report the
  worker script's URL.

Example:

```js
// worker.js
importScripts("helper.js");             // helper.js: worker.js
fetch("a.json");                        // a.json: worker.js
setTimeout(() => fetch("b.json"), 0);   // b.json: worker.js
onmessage = () => fetch("c.json");      // c.json: worker.js
```

Tests:

- [direct.any.js](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/direct.any.js)
- [set-timeout.any.js](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/set-timeout.any.js)
- [set-interval.any.js](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/set-interval.any.js)
- [promise.any.js](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/promise.any.js)
- [post-task.any.js](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/post-task.any.js)
- [queue-microtask.any.js](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/queue-microtask.any.js)
- [post-message-worker.html](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/post-message-worker.html)
- [post-message-shared-worker.html](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/post-message-shared-worker.html)
- [module-script-from-worker.html](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/module-script-from-worker.html)

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

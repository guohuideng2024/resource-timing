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

### 3.1 Coverage

Resources can be fetched in many ways, and defining and implementing an
initiator for every one of them is not practical. The goal is to cover
enough cases for `initiatorUrl` to be useful, not every case. As the
explainer notes, partial dependency information is still useful, so an
implementation can cover some cases first and add more later.

The rest of this section describes the cases covered so far, which are the
common ways resources are fetched. Coverage may be extended in the future.
For resources that are not covered, `initiatorUrl` is the empty string (see
[section 4](#4-when-initiatorurl-is-empty) of this document).

Chromium plans to run an origin trial, so this behavior is not final and may
change based on feedback.

All the tests are in
[resource-timing/tentative/initiator-url/](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/). **In the
examples, a comment `<resource>: <initiatorUrl>` gives the expected
`initiatorUrl` of the resource referenced on the next line.**

### 3.2 Resources in HTML markup

Resources referenced by elements in the document's HTML source have the
document's URL as their initiator. This includes:

- `<script src>`, including `async`, `defer` and `type="module"` scripts.
- `<link rel="preload">` and `<link rel="stylesheet">`.
- `<img>` and `<iframe>`.
- Resources referenced from an inline `<style>` element, such as a background
  image or an `@font-face` font.

Examples:

From [`tentative/initiator-url/static-resource.html`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/static-resource.html):

```html
<!-- initiatorUrl of every resource below: tentative/initiator-url/static-resource.html -->
<script src="../../resources/empty.js?blocking"></script>
<link rel="preload" href="../../resources/display_paragraph.js" as="script">
<script async src="../../resources/empty.js?async"></script>
<script defer src="../../resources/empty.js?deferred"></script>
<script type="module" src="../../resources/empty.js?module"></script>
<link rel="stylesheet" href="../../resources/empty_style.css?link">
<style>
body {
  background-image: url("/images/blue.png?inline-style");
  font-family: remoteFont, sans-serif;
}
@font-face {
  font-family: remoteFont;
  src: url("/fonts/Ahem.ttf?initiator-html");
}
</style>
<img src="/images/blue.png?using-Img-tag"
     alt="Sample Image for testing initiator Attribute">
<iframe src="../../resources/green.html"></iframe>
```

### 3.3 Resources referenced by CSS

- Resources referenced by an external stylesheet, such as `@import` rules,
  `@font-face` fonts and `url()` images, report the stylesheet's URL. This is
  the same whether the stylesheet is in the markup or inserted by script.
- Resources referenced by an inline `<style>` element report the document's
  URL ([section 3.2](#32-resources-in-html-markup)).
- The stylesheet itself follows the rules for markup ([section 3.2](#32-resources-in-html-markup)) or for
  JavaScript ([section 3.4](#34-resources-requested-by-javascript)), depending on how it was inserted.

Examples:

From [`tentative/initiator-url/stylesheet-static.html`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/stylesheet-static.html):

```html
<link rel="stylesheet" href="/resource-timing/resources/nested-initiator.css">
```

From [`resources/nested-initiator.css`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/resources/nested-initiator.css):

```css
/* initiatorUrl of every resource below: resources/nested-initiator.css */
@import "resource_timing_test0.css?label=initiator";
@font-face {
    font-family: remoteFont;
    src: url('/fonts/Ahem.ttf?label=initiator');
}
ol {
    font-family: remoteFont;
    list-style-image: url('blue.png?label=initiator');
}
```

From [`tentative/initiator-url/stylesheet-dynamic.html`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/stylesheet-dynamic.html):

```js
const link = document.createElement("link");
link.rel = "stylesheet";
// nested-dynamic-initiator.css: tentative/initiator-url/stylesheet-dynamic.html
link.href = "/resource-timing/resources/nested-dynamic-initiator.css";
// initiatorUrl of the resources in resources/nested-dynamic-initiator.css: resources/nested-dynamic-initiator.css
document.head.appendChild(link);
```

### 3.4 Resources requested by JavaScript

When a resource is requested while script is running, the initiator is **not**
the script that contains the function that made the request. Instead, it is
the script that started the current chain of execution. This document calls it
the *initiating script*.

This applies whether the request is made through an API such as `fetch()`, or
by inserting an element such as `<img>`, `<link>`, `<script>` or `<iframe>`.

#### 3.4.1 Synchronous execution

When a classic script is evaluated, it becomes the initiating script for
everything it runs synchronously, including calls into functions defined in
other scripts.

- A resource requested by an inline script reports the document's URL.
- A resource requested by an external classic script reports that script's URL.
- Calling a function defined in another script does not change the initiator.

Examples:

From [`tentative/initiator-url/inline-script.html`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/inline-script.html):

```js
const label = "no_cache_inline_script";
const img = document.createElement("img");
// blue.png?no_cache_inline_script: tentative/initiator-url/inline-script.html
img.src = "/images/blue.png?" + label;
document.body.appendChild(img);
```

From [`tentative/initiator-url/external-script.html`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/external-script.html):

```html
<script src="../../resources/loading-resources.js"></script>
```

From [`resources/loading-resources.js`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/resources/loading-resources.js):

```js
var img = document.createElement("img");
// blue.png?no_cache: resources/loading-resources.js
img.src = "/images/blue.png?no_cache";
document.body.appendChild(img);
```

#### 3.4.2 Callbacks

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

Examples:

From [`resources/loading-resource-lib.js`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/resources/loading-resource-lib.js):

```js
async function load_image(label, img_element, image_to_load = "blue.png") {
  const url = "/images/" + image_to_load + "?" + label;
  const response = await fetch(url);
```

From [`tentative/initiator-url/set-timeout.html`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/set-timeout.html):

```js
const label = "initiator_url_settimeout";
// blue.png?initiator_url_settimeout: tentative/initiator-url/set-timeout.html
setTimeout(function() {load_image(label, img);}, 0);
```

From [`tentative/initiator-url/event-handler.html`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/event-handler.html):

```js
// blue.png?initiator_url_eventhandler: tentative/initiator-url/event-handler.html
frame.addEventListener("load", () => {
  load_image(label, img1, "blue.png");
});
// red.png?initiator_url_eventhandler: tentative/initiator-url/event-handler.html
frame.onload = () => {
  load_image(label, img2, "red.png");
};
const label = "initiator_url_eventhandler";
```

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

Examples:

From [`tentative/initiator-url/module-script-from-document.html`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/module-script-from-document.html):

```html
<!-- module-script-imported.js?label=document-src: tentative/initiator-url/module-script-from-document.html -->
<script type="module" src="/resource-timing/resources/module-script-imported.js?label=document-src"></script>
<script type="module">
// module-script-imported.js?label=document-static: tentative/initiator-url/module-script-from-document.html
import "/resource-timing/resources/module-script-imported.js?label=document-static";
</script>
<script>
// module-script-imported.js?label=document-dynamic: tentative/initiator-url/module-script-from-document.html
import("/resource-timing/resources/module-script-imported.js?label=document-dynamic");
```

From [`resources/module-script-importer-module-dynamic.js`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/resources/module-script-importer-module-dynamic.js):

```js
// module-script-imported.js?label=dynamic-module-importer-static: resources/module-script-importer-module-dynamic.js
import './module-script-imported.js?label=dynamic-module-importer-static';
// module-script-imported.js?label=dynamic-module-importer-dynamic: resources/module-script-importer-module-dynamic.js
import('./module-script-imported.js?label=dynamic-module-importer-dynamic');
```

From [`resources/module-script-importer-classic.js`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/resources/module-script-importer-classic.js):

```js
// module-script-imported.js?label=classic-importer-dynamic: resources/module-script-importer-classic.js
import('/resource-timing/resources/module-script-imported.js?label=classic-importer-dynamic');
```

### 3.6 Workers

#### 3.6.1 Worker scripts

The script of a dedicated worker, classic or module, follows [section 3.4](#34-resources-requested-by-javascript): it
reports the initiating script of the code that called `new Worker()`. For
example, it is the document if an inline script started the worker, an
external script if that script started it, and the document if an inline
script scheduled a `setTimeout()` callback that started it.

The top-level scripts of shared workers and service workers do not produce a
Resource Timing entry today, so they have no `initiatorUrl`.

Examples:

From [`tentative/initiator-url/dedicated-worker-script.html`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/dedicated-worker-script.html):

```js
// initiator-url-worker.js?label=classic-worker-from-document: tentative/initiator-url/dedicated-worker-script.html
new Worker(scriptUrl("classic-worker-from-document"));
// initiator-url-worker.js?label=module-worker-from-document: tentative/initiator-url/dedicated-worker-script.html
new Worker(scriptUrl("module-worker-from-document"), {type: "module"});
// initiatorUrl of the workers started by startDedicatedWorkers() in resources/initiator-url-dedicated-worker-start.js: tentative/initiator-url/dedicated-worker-script.html
setTimeout(startDedicatedWorkers, 0);
```

From [`resources/initiator-url-dedicated-worker-start.js`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/resources/initiator-url-dedicated-worker-start.js):

```js
// initiator-url-worker.js?label=classic-worker-from-js: resources/initiator-url-dedicated-worker-start.js
new Worker(workerScriptUrl('classic-worker-from-js'));
// initiator-url-worker.js?label=module-worker-from-js: resources/initiator-url-dedicated-worker-start.js
new Worker(workerScriptUrl('module-worker-from-js'), {type: 'module'});
function startDedicatedWorkers() {
  // initiator-url-worker.js?label=classic-worker-from-setTimeout: tentative/initiator-url/dedicated-worker-script.html
  new Worker(workerScriptUrl('classic-worker-from-setTimeout'));
  // initiator-url-worker.js?label=module-worker-from-setTimeout: tentative/initiator-url/dedicated-worker-script.html
  new Worker(workerScriptUrl('module-worker-from-setTimeout'),
             {type: 'module'});
}
```

#### 3.6.2 Resources requested inside a worker

- Inside a dedicated or shared worker, resources requested by the worker's
  code report the worker script's URL. This includes `fetch()`,
  `importScripts()`, callbacks (timers, promise reactions,
  `scheduler.postTask()`, microtasks) and message handlers.
- Modules imported by a module worker, statically or dynamically, report the
  worker script's URL.

Examples:

From [`tentative/initiator-url/direct.any.js`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/direct.any.js):

```js
// loading-resource-lib.js?direct_any: tentative/initiator-url/direct.any.worker.js
importScripts("../../resources/loading-resource-lib.js?direct_any")
const label = "initiator_url_direct_worker";
const imageResource = "/images/blue.png?"+label;
// blue.png?initiator_url_direct_worker: tentative/initiator-url/direct.any.worker.js
fetch_in_function(imageResource);
```

From [`tentative/initiator-url/set-timeout.any.js`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/tentative/initiator-url/set-timeout.any.js):

```js
const label = "initiator_url_settimeout_worker";
// blue.png?initiator_url_settimeout_worker: tentative/initiator-url/set-timeout.any.worker.js
setTimeout(function(){fetch_in_function(resource)}, 10);
```

From [`resources/message-handler-in-worker.js`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/resources/message-handler-in-worker.js):

```js
const label = "initiator_url_message_handler_worker";
// blue.png?initiator_url_message_handler_worker: resources/message-handler-in-worker.js
self.onmessage = async function (event) {
  fetch_in_function(resource);
```

From [`resources/module-script-worker.js`](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/web_tests/external/wpt/resource-timing/resources/module-script-worker.js):

```js
// module-script-imported.js?label=worker-importer-static: resources/module-script-worker.js
import './module-script-imported.js?label=worker-importer-static';
```

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

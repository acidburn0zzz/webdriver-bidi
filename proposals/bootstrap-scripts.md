# WebDriver BiDi Bootstrap Scripts

## Overview

This document presents Bootstrap Scripts, a new feature based on the [Bidirectional WebDriver Protocol](./core.md).

A bootstrap script is a function that runs once whenever a new script execution context is created and is guaranteed to run before any other script in that context. It runs in the same script context as a page, worker, or service worker so that it can inspect and manipulate variables in these contexts to do things like interact with the DOM or polyfill APIs. A messaging channel is provided so that bootstrap scripts can communicate with the WebDriver client.

## Motivation

WebDriver already has the ability to inject script into the page and get a result, but there is no easy way to listen for events generated by the page or receive ongoing notifications. There is also currently no way to ensure the injected scripts runs before any other scripts on the page.

A simple example might be using a bootstrap script to register a "DOMContentLoaded" event listener on the page, and then firing a notification to the WebDriver client when the "DOMContentLoaded" event occurs. This would signal to the client that the test is ready to proceed.

Expanding on the above example, a single-page app may load some UI content asynchronously and the "DOMContentLoaded" or "load" events may not be a good enough indication that the app is in a steady state and ready to test. If the app had a way to signal to test code that the UI is fully loaded, then the test code could listen for this event before proceeding with the test. This would be more reliable and efficient than either timeouts or polling the DOM. A bidirectional WebDriver protocol makes this sort of messaging possible.

Another use case is adding instrumentation or shims before a page starts running. A bootstrap script could wrap the console.log function with a custom implementation that forwards its arguments to the WebDriver client. This would be a simple but effective way to add logging to a WebDriver test. Similarly, a bootstrap script could create a [PerformanceObserver](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceObserver) and use the messaging channel to forward performance entries to the WebDriver client.

## Registering Bootstrap Scripts

The [bidirectional WebDriver protocol](./core.md) defines 3 types of script contexts: "document", "worker", and "serviceWorker". Every script context has a unique ID and the WebDriver client can use the getScriptContexts command to get a list of script contexts for a given target. The client may also subscribe to the scriptContextCreated and scriptContextClosed events to find out when a script context is created or closed.

The main purpose of a bootstrap script is to run before any other scripts in the same script context. To do anything with a script context, the WebDriver client normally needs to know its ID. The WebDriver client first learns of a context's ID through a scriptContextCreated event. However, due to the asynchronous nature of events, it is possible (and even likely) that this context will start running script before the WebDriver client has a chance to inject a bootstrap script. Therefore, the WebDriver client needs some way to register a bootstrap script to run in a script context before it even knows the script context's ID.

A solution would be to not use IDs at all, but to use match patterns. Instead of the client stating "Run this bootstrap script on scriptContext #4" (which is impractical for the reason stated above), the client would say "Run this bootstrap script on all script contexts belonging to example.com" or "Run this bootstrap script for service workers runing example.com/sw.js". This is similar in principle to setting a conditional breakpoint. The client declares where they want the bootstrap script to run, and the browser will check every new script context to see if it matches the user's conditions. Here's what the API call to register a bootstrap script would look like:

```javascript
{
    id: 99,
    method: "registerBootstrapScript",
    params: {
        match: [
            { type: "document", urlPattern: "http://example.com/*" }
        ],
        script: "... script text to execute here ..."
    }
}
```

The "match" parameter is an array of rules describing which contexts a script should execute in and the "script" parameter is the JS script to run. When this API is called, the script text is persisted on the WebDriver server. Whenever a new script context is created, the server checks if the script context matches any of the given rules, and if so, it executes the bootstrap script in that context. In the above example, the client is registering some script to run on "document" script contexts that belong to a browsing context who's URL matches "http://example.com/*". The API returns a unique identifier for the registration:

```javascript
{
    id: 99,
    result: { bootstrapScriptId: "<ID>" }
}
```

Note that this ID represents the _registration_ and not a particular instance of the script. Many instances of the script may run in many different contexts as a result of the registration, or the script may not run at all if no contexts match.

Later, when the client is done with the bootstrap script, they can unregister it:

```javascript
{
    id: 100,
    method: "unregisterBootstrapScript",
    params: { bootstrapScriptId: "<ID>" }
}
```

The server would remove the persisted script from memory and stop injecting it into new script contexts. However, instances of the bootstrap script would continue to run in any existing contexts where they are already running.

### Match Patterns

A match pattern is an object with two properties:

- type - enum
    - "document": A script context with access to the DOM.
    - "worker": A web worker context.
    - "serviceWorker": A service worker context.
- urlPattern - A URL regex string to match.
    - When used with type "document", the current URL of the associated browsing context is checked.
    - When used with either "worker" or "serviceWorker", the URL of the initial script is checked.

#### Examples

Match a service worker with a specific URL:

```javascript
{ type: "serviceWorker", "urlPattern": "https://mdn.github.io/sw-test/sw.js" }
```

Match any frame on the bing.com domain:

```javascript
{ type: "document", "urlPattern": "*://bing.com/*" }
```

## Messaging

Bootstrap scripts should allow bidi communication with the WebDriver client code. This has a number of potential uses such as:

- Coordination/timing between test automation code and test page.
- Requesting the client to perform operations that the page doesn't have access to.
- Forwarding DOM events to the client.


To send a message from the WebDriver client to a bootstrap script, we need a way to identify the bootstrap script. However, in the previous section, we defined a bootstrapScriptId as an ID for a *registration*, not a particular instance of a bootstrap script. Once a bootstrap script is registered, it can be invoked in potentially many contexts, and the user may want to message only some of these contexts. For example, the user might register a bootstrap script to run on all "documents" to hook up some event listeners and add some instrumentation, and then they might want to message just one instance of the bootstrap script at a time to perform some operation on the corresponding page. The bootstrapScriptId isn't suitable in this case since it doesn't uniquely identify a single instance of a bootstrap script running in a single script context.

To enable this scenario, the user needs a way to know when a bootstrap script has been matched with a particular script context. Then, they can use the bootstrapScriptId, plus the scriptContextId to route a message to just the one script context they want. We can provide this information with an event:

```javascript
{
    method: "bootstrapScriptExecuted",
    params: { bootstrapScriptId: "<ID>", scriptContextId: "<ID>" }
}
```

The user can subscribe to this event to track when a bootstrap script actually executes and how to address a message to it. In this scenario, we assume the user is using other APIs such as browsingContextAttached/scriptContextCreated so they have a general idea of what each script context is.

### WebDriver client-side

Now that we have a way to identify a particular instance of a bootstrap script on the WebDriver client side, we need a couple APIs for sending and receiving messages:

Command for sending

```javascript
{
    "method": "postMessageToBootstrapScript",
    "params": {
        bootstrapScriptId: "<ID>",
        scriptContext: "<ID>",
        data: /* Arbitrary JSON data */
    }
}
```

Event for receiving

```javascript
{
    "method": "messageReceivedFromBootstrapScript",
    "params": {
        bootstrapScriptId: "<ID>",
        scriptContext: "<ID>",
        data: /* Arbitrary JSON data */
    }
}
```

Message data is an arbitrary JSON blob that will be deserialized on the bootstrap script side.

### Bootstrap script-side

To allow a bootstrap script to send and receive messages from the client side, we can expose a [MessagePort](https://developer.mozilla.org/en-US/docs/Web/API/MessagePort) object. The script can subscribe to the MessagePort's onmessage event which will fire whenever the client calls postMessageToBootstrapScript. The script can call postMessage() which will fire a messageReceivedFromBootstrapScript event on the client side.

The question then is how to expose the MessagePort object to the bootstrap script. Bootstrap scripts run in the same context as page script, so exposing the port as a global is not a good idea. This would give untrusted page script access to the port object.

One solution would be to require all bootstrap scripts to be a function that accepts the port as a parameter:

```javascript
function myBootstrapScript(port) {
    // ... Do something with the port.
}
```

The browser would call the function, passing in a MessagePort object. This would scope the port variable so that it is only visible inside the function. Of course, the user could proceed to expose the port to the page somehow, but this would at least be an explicit choice by the user.

Another option is to avoid running bootstraps scripts in the same context as page script, and run them in some kind of "isolated" context instead; Something similar to a WebExtension [content script](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/Content_scripts#Content_script_environment), which has access to a clean version of the DOM, but has its own global scope. This would make it possible to expose the MessortPort (and possibly other priviledged APIs) to the bootstrap script as globals. However, this kind of functionality may not be available in all browsers and doesn't appear to be standardized outside of WebExtensions. This also makes it difficult for the bootstrap script to manipulate the page's view of the DOM, which could be a useful feature.

## Examples

Below are some end-to-end WebDriver example using bootstrap scripts with messaging. The client-side code is using a hypothetical JavaScript library that supports bidi WebDriver using async/await syntax.

### Record navigation performance

This example creates a PerformanceObserver before a page starts loading, and uses the message port to continuosly send performance entries to the WebDriver client as they happen.

```javascript
// Bootstrap strip that observes navigation performance entries and forwards them to the client.
function bootstrapScript(port) {
    const observer = new PerformanceObserver((list, obj) => {
        for (let entry of list.getEntries()) {
            port.postMessage({ performanceEntry: entry });
        }
    });
    observer.observe({ entryTypes: ["navigation"] });
}

// Register the bootstrap script to run on all pages.
const { bootstrapScriptId } = await driver.registerBootstrapScript({
    match: [ { type: "document" } ],
    script: bootstrapScript.toString() // Stringify the bootstrap function for sending to the remote end.
});

// Listen for messages from the bootstrap script.
driver.on("messageReceivedFromBootstrapScript", params => {
    const entry = params.data.performanceEntry;
    if (entry) {
        // Log some fields we're interested in.
        console.log("domContentLoadedEventStart:", entry.domContentLoadedEventStart);
        console.log("domContentLoadedEventEnd:", entry.domContentLoadedEventEnd);
        console.log("domComplete:", entry.domComplete);
        console.log("loadEventStart:", entry.loadEventStart);
        console.log("loadEventEnd:", entry.loadEventEnd);
    }
});
```

### Fail fast on uncaught JavaScript errors

This example listens for uncaught JavaScript exceptions on the page, and signals the client to stop the test early and show the exception information for debugging purposes.

```javascript
// Bootstrap script to report JS errors and unhandled Promise rejections to the client.
function bootstrapScript(port) {
    window.addEventListener("error", e => {
        port.postMessage({ error: e.error.toString() });
    });
    window.addEventListener("unhandledrejection", e => {
        port.postMessage({ error: e.reason.toString() });
    });
}

// Helper function to await an unexpected script error.
async function scriptErrorOccurred(driver) {
    return new Promise((resolve, reject) => {
        driver.on("messageReceivedFromBootstrapScript", params => {
            if (params.data.error) {
                reject(params.data.error);
            }
        });
    });
}

async function runTest(driver) {
    // ... Navigate to a page and run some tests ...
}

try {
    // Start a WebDriver session.
    const driver = new WebDriver();

    // Register the bootstrap script to run on all pages.
    const { bootstrapScriptId } = await driver.registerBootstrapScript({
        match: [ { type: "document" } ],
        script: bootstrapScript.toString() // Stringify the bootstrap function for sending to the remote end.
    });

    // Wait for either the test to complete successfully or an error to occur.
    await Promise.race([scriptErrorOccurred(driver), runTest(driver)]);
} catch (e) {
    // An unexpected error occurred. Log to the console.
    console.log(e);
} finally {
    // In either case, close the session.
    await driver.close();
}

```

### Intercept console.log calls

This example wraps the page's console.log method with a custom method that sends the arguments up to the client and then forwards to the original console.log method. Note that this is not a complete solution for adding logging to bidi WebDriver since it covers only console.log calls from JS and ignores other sources of console messages. Logging deserves its own set of new WebDriver APIs. However, this example is useful to illustrate how bootstap scripts empower users to add at least some of this functionality to WebDriver themselves.

```javascript
function bootstrapFunc(port) {
    // Wrap the console.log method...
    const originalLog = console.log.bind(console);
    console.log = function customLog(...args) {
        // Notify client. Args are serialized as JSON.
        port.postMessage({ msg: "consoleLog", args: args });
        // Pass-thru to real console.log method.
        originalLog(...args);
    }

    // Also wrap console.error and console.warn...
}

// Register our bootstrap script to run on the test page.
const { bootstrapScriptId } = await driver.registerBootstrapScript({
    match: [
        { type: "document", urlPattern: "http://foo.com" }
    ],
    script: bootstrapFunc.toString()
});

// Listen for messages from the bootstrap script.
driver.on("messageReceivedFromBootstrapScript", params => {
    // Object passed to port.postMessage is deserialized and available in the .data property.
    if (params.data.msg === "consoleLog") {
        // Do something with this log entry like save it to a test log file.
        writeToFile(JSON.stringify(params.data.args));
    }
});
```

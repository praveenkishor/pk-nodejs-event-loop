# Node.js Event Loop

A learning reference for understanding how Node.js executes synchronous JavaScript and schedules asynchronous callbacks.

## Architecture diagram

![Node.js Event Loop Architecture Flowchart](Node.js%20Event%20Loop%20Architecture%20Flowchart.png)

## Core concepts

- **V8:** The JavaScript engine that compiles and executes JavaScript.
- **Call stack:** Tracks active function calls. Synchronous code executes before queued callbacks can run.
- **libuv:** Supports the Node.js event loop and asynchronous I/O.
- **Microtasks:** Promise handlers and callbacks scheduled with `queueMicrotask()` run after the current JavaScript execution completes, before the next timer callback in the example below.
- **Event loop:** Coordinates callback execution across phases such as timers, poll (I/O), check (`setImmediate()`), and close callbacks.

The diagram is a simplified overview. Node.js uses libuv rather than browser Web APIs, and `process.nextTick()` has its own queue rather than sharing the Promise microtask queue.

## Example

Save this code as `example.js`:

```js
console.log('Start');

setTimeout(() => {
  console.log('Timeout');
}, 0);

Promise.resolve().then(() => {
  console.log('Microtask');
});

console.log('End');
```

With Node.js installed, run:

```sh
node example.js
```

Expected output:

```text
Start
End
Microtask
Timeout
```

`Start` and `End` execute synchronously. The Promise handler runs next, followed by the timer callback. A zero-millisecond timer schedules a callback for a later opportunity; it does not execute immediately.

## Project contents

- `README.md` — explanation and runnable example.
- `Node.js Event Loop Architecture Flowchart.png` — visual overview.

No dependencies or package installation are required for the example.

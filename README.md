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

The diagram shows how Node.js executes synchronous JavaScript and coordinates asynchronous work using V8, the Call Stack, libuv, queues, microtasks, and the Event Loop.

## Core idea
Think of Node.js as having two cooperating parts:

`V8` executes JavaScript. It owns the JavaScript execution machinery, including the `Call Stack` and JavaScript objects such as Promises.

`Node.js + libuv` coordinate asynchronous operations such as timers and many I/O operations. When asynchronous work becomes ready, Node.js eventually gets the corresponding callback executed by V8.
A useful simplified flow is:

`JavaScript → V8 → Call Stack → async work → libuv/Node → callback queues → Event Loop → Call Stack → V8 executes callback`

Microtasks such as Promise handlers get special priority at appropriate checkpoints.

1. JavaScript enters V8
Suppose you run:
console.log("Start");

setTimeout(() => {
  console.log("Timeout");
}, 0);

Promise.resolve().then(() => {
  console.log("Promise");
});

console.log("End");

V8 parses and compiles the JavaScript and begins executing it.
The Call Stack keeps track of which JavaScript functions are currently executing.
So the first operation is roughly:
Call Stack
┌─────────────────┐
│ console.log()   │
│ main script     │
└─────────────────┘

It prints:
Start

Once console.log() finishes, its frame is removed from the stack.
2. setTimeout() does not block the Call Stack
Next:
setTimeout(callback, 0);

A common misunderstanding is that 0 means:
Execute the callback immediately.

It doesn't.
The timer is registered through Node.js's timer/event-loop machinery, which is built around libuv.
Conceptually:
V8 / Call Stack
      │
      │ register timer
      ▼
Node.js / libuv
      │
      │ timer becomes ready
      ▼
Timers processing

Node.js does not sit on the Call Stack waiting for the timer.
Execution continues.
3. Promise creates a microtask
Now:
Promise.resolve().then(() => {
  console.log("Promise");
});

The Promise is already fulfilled, but its .then() handler still does not execute synchronously.
Instead, the handler is scheduled as a microtask.
Conceptually:
Microtask Queue
┌──────────────────────┐
│ Promise .then(...)   │
└──────────────────────┘

This distinction is extremely important.
Promise handlers don't go into the same place as timer callbacks.
4. Synchronous JavaScript continues first
The program then reaches:
console.log("End");

It executes immediately because synchronous JavaScript currently has control of the Call Stack.
Output so far:
Start
End

Eventually the top-level script finishes and the stack becomes empty.
5. Microtasks get processed
At the appropriate microtask checkpoint, Node/V8 processes pending microtasks.
Our queue contains:
Microtask Queue
┌──────────────────────┐
│ Promise .then(...)   │
└──────────────────────┘

The Promise callback gets executed by V8:
() => {
  console.log("Promise");
}

So now:
Start
End
Promise

An important rule is that the microtask queues are drained before Node continues on to the next relevant event-loop work.
6. Node's special process.nextTick() queue
Node.js has another mechanism:
process.nextTick(() => {
  console.log("nextTick");
});

This is Node-specific and is even more urgent than the normal Promise/queueMicrotask() queue.
A simplified priority model is:
Current JavaScript
       ↓
process.nextTick callbacks
       ↓
Promise / queueMicrotask microtasks
       ↓
continue event-loop work

So don't think of process.nextTick() as simply another Promise microtask.
7. The Event Loop processes phases
Once synchronous execution and the relevant microtasks are finished, Node can continue through its event-loop phases.
A simplified representation is:
              ┌─────────────┐
              │   Timers    │
              └──────┬──────┘
                     ↓
              ┌─────────────┐
              │ Pending I/O │
              └──────┬──────┘
                     ↓
                poll-related
                   phases
                     ↓
              ┌─────────────┐
              │    Poll     │
              └──────┬──────┘
                     ↓
              ┌─────────────┐
              │    Check    │
              └──────┬──────┘
                     ↓
              ┌─────────────┐
              │    Close    │
              └──────┬──────┘
                     │
                     └──→ next iteration

This is why saying that Node has just one generic "callback queue" is useful for beginner explanations but not completely accurate.
Different kinds of callbacks are associated with different event-loop phases.
For example, setTimeout() is associated with timers, while setImmediate() is associated with the check phase.
8. The timer callback can now execute
Eventually the zero-delay timer becomes eligible to run.
Node invokes its callback, and V8 executes that JavaScript on the Call Stack:
() => {
  console.log("Timeout");
}

Now the output becomes:
Start
End
Promise
Timeout

So setTimeout(..., 0) still came after the Promise handler.
9. What happens if a callback creates another Promise?
Suppose a timer does this:
setTimeout(() => {
  console.log("Timer");

  Promise.resolve().then(() => {
    console.log("Promise inside timer");
  });
}, 0);

The timer callback executes:
Call Stack
     │
     ▼
"Timer"

Then it schedules a Promise reaction:
Microtask Queue
┌─────────────────────────┐
│ Promise inside timer    │
└─────────────────────────┘

After the callback finishes, Node reaches a microtask checkpoint and processes that microtask before moving on to other event-loop callbacks.
That's one reason microtasks can appear to "jump ahead."
10. Where libuv fits in
Another important distinction: libuv is not simply a place where every asynchronous operation runs on another thread.
libuv provides the event loop and asynchronous I/O infrastructure.
Depending on the operation and operating system, Node/libuv may use OS asynchronous facilities or the libuv worker thread pool.
For example, some filesystem, DNS, crypto, and compression operations can involve the worker pool.
A simplified model is:
JavaScript
    ↓
V8
    ↓
Call Stack
    │
    │ asynchronous request
    ▼
Node.js APIs
    ↓
libuv
 ┌──┴───────────────────────┐
 ↓                          ↓
OS async facilities     Worker pool
 ↓                          ↓
 └───────────┬──────────────┘
             ↓
       operation ready
             ↓
      Event-loop phase
             ↓
        JS callback
             ↓
         Call Stack
             ↓
            V8

So the callback itself normally executes as JavaScript on the main JavaScript thread; the asynchronous waiting/work is what happens outside that Call Stack.
Putting everything together
The more complete mental model is:
              JAVASCRIPT
                   │
                   ▼
          ┌─────────────────┐
          │       V8        │
          │ JS execution    │
          └────────┬────────┘
                   ▼
          ┌─────────────────┐
          │   CALL STACK    │
          └────────┬────────┘
                   │
          async operation?
            /           \
          no             yes
          │               │
       execute            ▼
                     Node.js APIs
                          │
                          ▼
                       libuv
                    /          \
                   ▼            ▼
              OS async      Worker Pool
                   \            /
                    └─────┬────┘
                          ▼
                    work becomes
                        ready
                          │
                          ▼
                   EVENT LOOP
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
            Timers       Poll        Check
              │           │           │
              └───────────┼───────────┘
                          ▼
                    JS callback
                          │
                          ▼
                     CALL STACK
                          │
                          ▼
                         V8

Meanwhile, microtasks are checked at microtask checkpoints:
JavaScript callback finishes
          │
          ▼
 process.nextTick queue
          │
          ▼
 Promise / queueMicrotask
          │
          ▼
 Continue event-loop work

So the most important execution priority to remember is approximately:
current synchronous JavaScript → process.nextTick() → Promise/queueMicrotask() microtasks → continue event-loop phases/callbacks.
For the original example, that's exactly why:
console.log("Start");

setTimeout(() => console.log("Timeout"), 0);

Promise.resolve()
  .then(() => console.log("Promise"));

console.log("End");

produces:
Start
End
Promise
Timeout

The key interview takeaway is: V8 executes JavaScript; the Call Stack tracks current execution; Node/libuv coordinate asynchronous work; the Event Loop decides when ready callbacks can run; and microtasks receive special priority between pieces of event-loop work.

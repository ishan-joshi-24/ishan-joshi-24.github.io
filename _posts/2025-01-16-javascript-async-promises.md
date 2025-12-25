---
layout: post
title: "Understanding Asynchronous JavaScript: Promises, Microtasks, and the Event Loop"
date: 2024-10-15
---

If you think promises are "asynchronous" or that `async/await` magically defers code execution, I have news for you. Most of what you think about async JavaScript is wrong. Let me break down how it actually works.

## The Fundamental Misconception

Here's what most people believe:

```javascript
const promise = new Promise((resolve) => {
  console.log('inside promise');
  resolve('done');
});

console.log('after promise');
```

Most people think the code inside the promise constructor runs "later" or "asynchronously." They're wrong. It runs immediately, synchronously, right now.

Output:
```
inside promise
after promise
```

Not "after promise" then "inside promise." The promise executor runs immediately.

## Promises Execute Synchronously (Until They Don't)

Let me be clear: promises are not asynchronous. The promise executor function is synchronous. It runs right away, blocking everything else.

```javascript
console.log('1');

const promise = new Promise((resolve) => {
  console.log('2');
  resolve('3');
});

console.log('4');
```

Output:
```
1
2
4
```

The promise executor (the function you pass to `new Promise`) is NOT deferred. It runs immediately. You pass it, it runs. Period.

What IS asynchronous is what happens when the promise resolves. The `.then()` callbacks, the `.catch()` callbacks - those are deferred. But the promise creation itself? Synchronous.

## Where The Confusion Comes From

Here's a more realistic example:

```javascript
console.log('1');

const promise = fetch('https://api.example.com/data');

promise.then((response) => {
  console.log('2');
});

console.log('3');
```

Output:
```
1
3
2
```

People see this and think "ah, the promise is async, so it deferred the execution." But that's not what happened. The `fetch()` call executed immediately (it made an HTTP request). But the `.then()` callback was deferred because the promise hasn't resolved yet.

The HTTP request is what's truly asynchronous. The promise is just tracking when it completes.

## The Event Loop: Where Asynchronicity Happens

JavaScript has one call stack. One. It executes one thing at a time, synchronously. There is no parallelism in JavaScript.

But wait, if JavaScript is single-threaded and synchronous, how does anything truly asynchronous happen?

The answer: the browser does it. The event loop manages when code runs.

Here's the model:

1. **Call Stack**: Where your code executes
2. **Microtask Queue**: Where `.then()`, `.catch()`, `queueMicrotask()` callbacks wait
3. **Macrotask Queue**: Where `setTimeout()`, `setInterval()`, I/O callbacks wait
4. **Event Loop**: The orchestrator that decides what runs next

The event loop works like this:

```
while (eventLoop.waitForTask()) {
  // 1. Execute all synchronous code on the call stack
  if (callStack.hasCode()) {
    callStack.execute();
  }

  // 2. If call stack is empty, drain the microtask queue completely
  if (callStack.isEmpty() && microtaskQueue.hasItems()) {
    while (microtaskQueue.hasItems()) {
      microtaskQueue.execute();
    }
  }

  // 3. If everything is empty, check for macrotasks
  if (callStack.isEmpty() && microtaskQueue.isEmpty()) {
    if (macrotaskQueue.hasItems()) {
      macrotaskQueue.execute();
    }
  }
}
```

The key rule: **microtasks always run before macrotasks**, and the microtask queue is completely drained before any macrotask runs.

## Seeing It In Action

Let's trace through a real example:

```javascript
console.log('Script start');

setTimeout(() => {
  console.log('setTimeout');
}, 0);

Promise.resolve('promise')
  .then((message) => {
    console.log(message);
  });

console.log('Script end');
```

Let me trace the execution:

1. `console.log('Script start')` - Call stack executes → Output: "Script start"
2. `setTimeout(...)` - Registers a macrotask, moves to macrotask queue
3. `Promise.resolve('promise').then(...)` - Registers a microtask, moves to microtask queue
4. `console.log('Script end')` - Call stack executes → Output: "Script end"
5. Call stack is empty. Event loop checks microtask queue.
6. Microtask queue has a promise callback → Execute it → Output: "promise"
7. Microtask queue is empty. Event loop checks macrotask queue.
8. Macrotask queue has a setTimeout → Execute it → Output: "setTimeout"

Final output:
```
Script start
Script end
promise
setTimeout
```

This is the most important rule: **Promise callbacks (microtasks) always run before setTimeout (macrotask), even if setTimeout is 0.**

## Why Does This Matter?

Imagine you have this:

```javascript
let data = null;

fetch('/api/data')
  .then((response) => response.json())
  .then((json) => {
    data = json;
  });

setTimeout(() => {
  console.log(data); // Not null, guaranteed
}, 0);
```

The `data` variable will be populated before the setTimeout runs, even though setTimeout is `0`. Why? Because the promise `.then()` callbacks (microtasks) run before setTimeout (macrotask).

This is guaranteed by the JavaScript spec. Microtasks always win.

## Async/Await Doesn't Change This

Here's what `async/await` actually does:

```javascript
async function foo() {
  console.log('1');
  await Promise.resolve();
  console.log('2');
}

foo();
console.log('3');
```

Output:
```
1
3
2
```

Wait, what? Why is "2" after "3"?

`await` is syntactic sugar. The code after `await` is wrapped in a `.then()` callback. So the above is equivalent to:

```javascript
function foo() {
  console.log('1');
  return Promise.resolve()
    .then(() => {
      console.log('2');
    });
}

foo();
console.log('3');
```

When `foo()` executes:
1. `console.log('1')` runs immediately → "1"
2. `Promise.resolve().then(...)` registers a microtask and returns
3. Next line: `console.log('3')` runs → "3"
4. Microtask queue has a callback → "2"

`async/await` is promises under the hood. It doesn't change the fundamental execution model.

## Common Misconceptions Debunked

### Misconception 1: "Async functions run asynchronously"

False. They run synchronously until they hit an `await`.

```javascript
async function foo() {
  console.log('1'); // Runs immediately
  await somePromise();
  console.log('2'); // Deferred to microtask queue
}

foo();
console.log('3'); // Runs immediately
```

Output: 1, 3, (later) 2

The function runs until it hits `await`, then the rest is deferred.

### Misconception 2: "Promises are asynchronous"

False. The executor is synchronous. The resolution is synchronous. Only the callbacks (`.then()`, `.catch()`) are asynchronous.

```javascript
const promise = new Promise((resolve) => {
  console.log('runs now');
  resolve('also runs now');
});

// This runs later (microtask)
promise.then((value) => {
  console.log(value);
});
```

### Misconception 3: "await pauses execution"

No. `await` doesn't pause anything. It's just syntactic sugar for `.then()`. The function returns a promise immediately, and the rest of the function is scheduled as a microtask.

```javascript
async function foo() {
  const result = await somePromise();
  console.log(result); // This is a microtask
}

console.log('before');
foo(); // Returns a promise immediately
console.log('after');
```

Output: before, after, (later) the result

`foo()` returns a promise on the first line. The caller doesn't wait. They continue.

### Misconception 4: "Multiple awaits run in parallel"

False. They run sequentially.

```javascript
async function foo() {
  const a = await Promise.resolve(1);
  const b = await Promise.resolve(2); // Waits for a first
  console.log(a, b);
}
```

This is equivalent to:

```javascript
Promise.resolve(1)
  .then((a) => {
    return Promise.resolve(2)
      .then((b) => {
        console.log(a, b);
      });
  });
```

Each `await` waits for the previous one. If you want parallel execution, use `Promise.all()`:

```javascript
async function foo() {
  const [a, b] = await Promise.all([
    Promise.resolve(1),
    Promise.resolve(2)
  ]);
  console.log(a, b); // Both resolve at the same time
}
```

## The Microtask Queue Is Special

Here's something that trips people up:

```javascript
Promise.resolve()
  .then(() => {
    console.log('1');
    Promise.resolve()
      .then(() => {
        console.log('3');
      });
  })
  .then(() => {
    console.log('2');
  });
```

Output:
```
1
2
3
```

Why is "2" before "3"?

The microtask queue is completely drained in order. Here's the trace:

1. First `.then()` is in the queue
2. Event loop executes it → "1"
3. Inside, another promise chain is created and queued
4. But we're still in the microtask queue drain phase
5. Continue draining: second `.then()` runs → "2"
6. Continue draining: the inner promise's `.then()` runs → "3"

The microtask queue is drained completely, in order, before any macrotask runs.

## Macrotasks vs Microtasks

**Microtasks** (drain completely before macrotask):
- Promise `.then()`, `.catch()`, `.finally()`
- `queueMicrotask()`
- `MutationObserver`

**Macrotasks** (one at a time):
- `setTimeout()`
- `setInterval()`
- `setImmediate()`
- I/O operations
- UI rendering

The event loop always drains all microtasks before running the next macrotask.

```javascript
setTimeout(() => console.log('macrotask 1'), 0);
Promise.resolve().then(() => console.log('microtask 1'));
setTimeout(() => console.log('macrotask 2'), 0);
Promise.resolve().then(() => console.log('microtask 2'));
```

Output:
```
microtask 1
microtask 2
macrotask 1
macrotask 2
```

All microtasks run first, then macrotasks run one at a time.

## Why This Matters In Practice

Knowing this matters because:

1. **Performance**: Understanding microtask vs macrotask helps you avoid janky UI. Microtasks run before rendering, macrotasks might not.

2. **Debugging**: You'll understand why your code runs in an unexpected order.

3. **Race conditions**: You can rely on microtasks running before certain operations (like setTimeout).

4. **Batching**: Promises naturally batch operations because microtasks are drained together.

## The Key Insight

JavaScript is synchronous and single-threaded. Promises don't change that. They just give you a way to schedule work to run later, in a specific order, through the event loop's microtask queue.

What's truly asynchronous is the browser's handling of I/O (network requests, timers, user input). JavaScript itself is orchestrating when callbacks run in response to those I/O operations.

The promise executor runs immediately. The `.then()` callbacks run in the microtask queue. `setTimeout` callbacks run in the macrotask queue. The event loop decides the order.

Master this model, and async JavaScript stops being magic and becomes completely predictable.

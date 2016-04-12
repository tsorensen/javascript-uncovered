# JavaScript Uncovered

Almost all of this content comes from Philip Roberts' JSConf EU talk entitled:
[What the heck is the event loop anyway?][event-loop]. He explained it very well
and helped me understand how JavaScript works and why Single-Threaded
and Asynchronous works in JavaScript.

# Intro + Prerequisites

Aside from being a pretty simple explanation, it does help to understand some
basic programming concepts. Here's what should bring you up to speed:

## Stack

A `stack` is a list of items where you can add items onto the top, and take
items off of the top. It's a First-In-Last-Out construct.

A "stack" of plates is a stack, where if you put a plate on top of the other
plates, you have to take off that top plate in order to get to the other plages.

If you park cars one behind another, in order to back them all out, the last one
in the row needs to back out, followed by the next to last one, and so on.

The ["Tower of Hanoi"][hanoi] is also an example of 3 stacks, where you take of
off the top of a stack, and put it onto the top of another stack.

To take off of the top of a stack is to "pop" it from the stack.

To put onto the top of a stack is to "push" it to the stack.

Both of these methods are implemented in JavaScript's array prototype:

```js
var arr = []
arr.push(5)

assert.deepEqual(arr, [ 5 ])

arr.push(1, 2, 3)

assert.deepEqual(arr, [ 5, 1, 2, 3 ])

arr.pop()

assert.deepEqual(arr, [ 5, 1, 2 ])
```

So the end of the array acts as the top of the array. "Push"ing adds to the end
of the array, and "Pop"ing takes off the last element.

## Queue

A `queue` is a list of items where you can add items onto the top, and take
items off of the bottom. It's a First-In-First-Out construct.

A "queue" is like a line at the bank or at a retail store. The first person in
line gets taken care of first, then leaves the queue, while new people get
added onto the end of the line.

To take off of the beginning of a queue, it's called "dequeue".

To put onto the end of a queue, it's called "enqueue".

In Javascript, the methods are a bit different:

```js
var arr = []
arr.push(5, 6, 3) // Enqueue

assert.deepEqual(arr, [ 5, 6, 3 ])

arr.shift() // Dequeue

assert.deepEqual(arr, [ 6, 3 ])
```

I only bring these constructs up because it's used when explaining what
JavaScript and its event loop does.

# JavaScript

If you've heard a senior developer or a website try to explain JavaScript to
you, I'm sure you've heard something to the effect of:

"It's a single-threaded, non-blocking, asynchronous, concurrent language".

...

"It has a call stack, an event loop, a callback queue, and some other apis".

...

When I first started with JavaScript, these made no sense to me. Some of these
didn't make sense to me until I saw this [aforementioned talk][event-loop]. So
we're going to break it down for you one step at a time.

## Single-Threaded Call Stack

This means that the processor can execute a single command at a time. A thread
is basically a set of commands to execute on a single processor core, so being
single-threaded means that it runs on a single core.

One Thread means that there can only be one call stack. The call stack is the
list of instructions currently being executed. The call stack is that same stack
referred to in a stack trace.

one thread == one call stack == one thing at a time

When a function gets called, that function gets pushed onto the call stack.
Every function that gets called from within that function also gets pushed onto
the call stack. Whenever a function finishes execution, it gets popped off of
the call stack.

Whenever an error is thrown, the error contains the state of the call stack at
the moment of the error, so the top of the stack trace will be where the error
was actually thrown, and the next line will be the function that called that
function, and so on.

Whenever you have a recursive function (a function that calls itself) but it
doesn't have stop condition, it will eventually reach an error, `max call stack
exceeded`. This means that the call stack has a limit, and we reached it. You
should only normally reach this condition if you've forgotten an end case in a
recursive function call.

## Blocking

Blocking just means that on this single thread, it's chugging through some
process(es) that are keeping the call stack from emptying quickly. You don't run
into this in the browser often, unless you use the `alert()` function or you
like to run lots of really long running loops.

Notice that in the browser when you call `alert('foo')`, it will stop everything
on the page. If you have a `console.log()` immediately following that alert, it
will block execution on the call stack until the user has interacted with the
alert.

There is an old jQuery method that you could use to make an HTTP request called
`$.getSync()`. It makes an HTTP request, but blocks the entire thread until it
gets a response back, which could be seconds. When things get blocked in the
browser, buttons become unclickable, things stop moving/interacting, and all
sorts of havoc.

What's the solution? Asynchronous callbacks.

## Asynchronous/Concurrent

Asynchronous callbacks are implemented by browser/node/platform APIs. Examples
of these are `setTimeout()`, `setInterval`, AJAX request, and so forth. All of
these implement a callback method of sorts, where you give the method a function
to call at a later time, when the event is complete or when triggered by a user
action.

Concurrency means things running at the same time, but this seems to go against
what we said earlier when we said JavaScript was single-threaded. It's true,
that only a single thing can be processed through the call stack at a time, but
our web apis can still run while the call stack is being processed.

When those apis finish processing whatever they were processing, they put the
callback function into a callback queue, which is a queue waiting in line to be
processed by the call stack. If the call stack is empty, and there are items in
the callback queue, the Event Loop will dequeue the first item in the callback
queue and put it into the call stack. See the following code as an example:

```js
console.log('Hi')

setTimeout(function() {
  console.log('There')
}, 5000)

console.log('DGM 4790')
```

We'll show a diagram for what happens to all these different components as it
gets executed.

First, we call `console.log('Hi')`

```
  Call Stack         Web Apis and stuff         Callback Queue
|            |     |                    |     |                |
|            | --> |                    | --> |                |
| console.log|     |                    |     |                |
| ---------- |     | ------------------ |     | -------------- |
       ^                                               |
       |----------------- Event Loop <-----------------|
```

Then it's done:

```
  Call Stack         Web Apis and stuff         Callback Queue
|            |     |                    |     |                |
|            | --> |                    | --> |                |
|            |     |                    |     |                |
| ---------- |     | ------------------ |     | -------------- |
       ^                                               |
       |----------------- Event Loop <-----------------|
```

Now we call `setTimeout()` with that callback function:

```
  Call Stack         Web Apis and stuff         Callback Queue
|            |     |                    |     |                |
|            | --> |                    | --> |                |
| setTimeout |     |                    |     |                |
| ---------- |     | ------------------ |     | -------------- |
       ^                                               |
       |----------------- Event Loop <-----------------|
```

That setTimeout gets executed, and hands execution over to the web apis for the
5000 millisecond timer:

```
  Call Stack         Web Apis and stuff         Callback Queue
|            |     |                    |     |                |
|            | --> |                    | --> |                |
|            |     | setTimeout(cb)     |     |                |
| ---------- |     | ------------------ |     | -------------- |
       ^                                               |
       |----------------- Event Loop <-----------------|
```

Then it calls `console.log('DGM 4790')`:

```
  Call Stack         Web Apis and stuff         Callback Queue
|            |     |                    |     |                |
|            | --> |                    | --> |                |
| console.log|     | setTimeout(cb)     |     |                |
| ---------- |     | ------------------ |     | -------------- |
       ^                                               |
       |----------------- Event Loop <-----------------|
```

And that completes:

```
  Call Stack         Web Apis and stuff         Callback Queue
|            |     |                    |     |                |
|            | --> |                    | --> |                |
|            |     | setTimeout(cb)     |     |                |
| ---------- |     | ------------------ |     | -------------- |
       ^                                               |
       |----------------- Event Loop <-----------------|
```

Now after about 5000 milliseconds have passed, the setTimeout callback gets
enqueued onto the callback queue:

```
  Call Stack         Web Apis and stuff         Callback Queue
|            |     |                    |     |                |
|            | --> |                    | --> |                |
|            |     |                    |     | cb             |
| ---------- |     | ------------------ |     | -------------- |
       ^                                               |
       |----------------- Event Loop <-----------------|
```

The event loop detects that the call stack is empty, and moves the callback from
our setTimout over to the call stack:

```
  Call Stack         Web Apis and stuff         Callback Queue
|            |     |                    |     |                |
|            | --> |                    | --> |                |
| cb         |     |                    |     |                |
| ---------- |     | ------------------ |     | -------------- |
       ^                                               |
       |----------------- Event Loop <-----------------|
```

Our callback calls `console.log('There')`

```
  Call Stack         Web Apis and stuff         Callback Queue
|            |     |                    |     |                |
| console.log| --> |                    | --> |                |
| cb         |     |                    |     |                |
| ---------- |     | ------------------ |     | -------------- |
       ^                                               |
       |----------------- Event Loop <-----------------|
```

It finishes calling console.log

```
  Call Stack         Web Apis and stuff         Callback Queue
|            |     |                    |     |                |
|            | --> |                    | --> |                |
| cb         |     |                    |     |                |
| ---------- |     | ------------------ |     | -------------- |
       ^                                               |
       |----------------- Event Loop <-----------------|
```

and finishes execution of the callback

```
  Call Stack         Web Apis and stuff         Callback Queue
|            |     |                    |     |                |
|            | --> |                    | --> |                |
|            |     |                    |     |                |
| ---------- |     | ------------------ |     | -------------- |
       ^                                               |
       |----------------- Event Loop <-----------------|
```

This was a very simple example, but it goes through all of the steps that an
asynchronous call would go through. To see a more visual example of code slowed
down, see this little [tool][callstack-visual] that slows down code to a pace
where you can see it execute.

[hanoi]: https://en.wikipedia.org/wiki/Tower_of_Hanoi
[event-loop]: https://www.youtube.com/watch?v=8aGhZQkoFbQ
[callstack-visual]: http://latentflip.com/loupe

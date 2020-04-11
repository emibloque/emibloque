---
layout: post
title: "The main threat of a single thread"
date: 2020-04-10
categories: blog
tags: javascript nodejs

image: "/assets/images/posts/single-thread.jpg"
---

When doing interviews with Node.js candidates at [The Cocktail](https://the-cocktail.com){:target="\_blank"}, one of the questions I usually ask is the following:

> Imagine you have a Node server started with `node index.js`. In this file, we have a route that runs some code which isn't I/O and takes 1 minute to complete. Something like the following:

{% highlight javascript %}
function longTask() {
  // Imaging a task like processing each pixel of a very large image
  // or sorting a large array
  for (let i = 0; i <= 10_000_000_000; i++) {}
}

app.get('/foo', () => {
  longTask()
})
{% endhighlight %}

> What is the problem we face when 3 concurrent users try to access this route? How can we solve it?

This question helps me to understand how well they know how JavaScript and Node.js work, in addition to how much knowledge they have working with distributed systems, message queues and background tasks.

I found that some candidates have a common answer:

> JavaScript is asynchronous, so there is nothing to worry about.

This is kind of true for **I/O operations** (calling a REST API, a database, reading a file...), but not for **CPU intensive tasks**, and before getting into this I'd like to review some concepts.

## Some background

### Language specification != implementation

First of all, I think it's important to note that a language specification is not the same as the language implementation.

When you create a programming language, you use a language specification to know **what** to implement, but **how** you implement the compiling or the runtime of that language _is up to you_.

The language specification, in the case of JavaScript, is the [ECMAScript® Language Specification](https://tc39.es/ecma262/){:target="\_blank"}.

Then, there are several engines that implement JavaScript, some of them are [Rhino](https://developer.mozilla.org/en-US/docs/Mozilla/Projects/Rhino){:target="\_blank"}, [JavaScriptCore](https://developer.apple.com/documentation/javascriptcore){:target="\_blank"}, [SpiderMonkey](https://developer.mozilla.org/en-US/docs/Mozilla/Projects/SpiderMonkey){:target="\_blank"} and [V8](https://v8.dev/){:target="\_blank"} (the engine Node.js uses).

You may have heard something about the [_**Event Loop**_](https://developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop){:target="\_blank"} – if not, I really recommend you to watch [this talk](https://www.youtube.com/watch?v=cCOL7MC4Pl0){:target="\_blank"} or [this one](https://www.youtube.com/watch?v=8aGhZQkoFbQ){:target="\_blank"} –, but maybe, what you don't know is that the Event Loop is not part of the JavaScript specification. In fact, the Event Loop belongs to the [Web application APIs of the HTML specification](https://html.spec.whatwg.org/multipage/webappapis.html#event-loops){:target="\_blank"}.

The ECMAScript specification talks about [execution contexts](https://tc39.es/ecma262/#sec-execution-contexts){:target="\_blank"}, [agents](https://tc39.es/ecma262/#sec-agents){:target="\_blank"} and the [executing thread](https://tc39.es/ecma262/#executing-thread){:target="\_blank"} of those agents, but nothing about an Event Loop.

Finally, Node.js is a JavaScript runtime (using V8 underneath) which wasn't meant to run in within a browser, so it can avoid having an Event Loop. However, using an Event Loop was the [primary motivation to create Node.js](https://www.youtube.com/watch?v=F6k8lTrAE2g){:target="\_blank"}, in that talk you can see that **each instance** of Node.js has **only one thread** – usually in production applications you'd use something like [PM2](https://pm2.keymetrics.io/){:target="\_blank"} in order to have more than one process of Node.js in the same machine.

With that said, if we were to create a JavaScript runtime that doesn't have an Event Loop and runs in multiple threads, it would be a totally valid JavaScript runtime.

### Concurrent vs parallel execution

Now that we know that Node.js has only one thread and an Event Loop, we need to know the difference between a concurrent and parallel execution of tasks. The easiest way to understand this is with an example:

Imagine that we are in a kitchen and we need to chop some onions and some potatoes. So:

- In a **concurrent** execution, I chop some onions, then some potatoes, then again some onions, then some potatoes, and so forth. In no particular order.
- In a **parallel** execution, I chop all the onions and **another person** chops all the potatoes, at the same time.

In summary, for a concurrent execution to be parallel, it needs more than one unit of _processing_ – or _chopping_ in this example 😂. So, if our runtime only uses **one thread** to run our code, it can never be parallel.

### Run-to-completion scheduling

In a concurrent execution of tasks, we can have two different schemes of how to schedule tasks: non-preemptive (or _run-to-completion_) scheduling or preemetive scheduling.

Using the example above, the difference will be the following:

- With a **run-to-completion** scheduling, we chop all the onions and then we can chop all the potatoes.
- With a **preemptive** scheduling, we start chopping onions, after 5 minutes, we start chopping potatoes, then after 5 minutes we go back to chop onions, and so forth.

Both have benefits and downsides:

- If I have too many onions, with **run-to-completion** my colleagues will have to wait too long to have potatoes ready to cook.
- But with **preemptive** scheduling, I will have to worry about using different tables and knives to chop each item (no shared memory between processes) or cleaning them each time and be extremely careful.

Erlang, for instance, uses _preemptive_ scheduling, and as you may guess from the question, Node.js uses _run-to-completion_ scheduling.

We could achieve some kind of _preemptive_ scheduling in Node.js using [_task partitioning_](https://nodejs.org/en/docs/guides/dont-block-the-event-loop/#partitioning){:target="\_blank"}, but that would make your code a lot harder to read.

### Synchronous vs asynchronous

One of the features of JavaScript is that functions are first-class citizens of the language, meaning that we can assign functions to variables or pass them as arguments to another function, for example:

{% highlight javascript %}
function hello() {
  return 'Hello'
}

function print(callback) {
  console.log(callback())
}

print(hello)
print(() => 'world')

const anotherFunction = hello
{% endhighlight %}

The previous code, although it is using callbacks, everything happens within a synchronous execution, each line is executed in order. To make this code asynchronous, we need to _delay_ the execution of some lines.

In Node.js (and in browsers), we can use the `setTimeout` API to _enqueue_ a task to be executed after at least some amount of time. When we call this function, the runtime will enqueue this task to the Event Loop, and once **the current execution finishes** and the time since it was queued **is equal or greater** than the time specified, the queued task will be run.

For example, this code is asynchronous:

{% highlight javascript %}
function print() {
  console.log('world')
}

setTimeout(print, 1000);

console.log('Hello')

// Hello # This is printed immediately
// world # This is printed after the last line
//         is ran and at least 1 second
//         has passed since print was called.
{% endhighlight %}

However, the main execution and the enqueued task are run in the same single thread.

#### I/O tasks

Bear in mind that, for input/output operations, asynchrony is achieved by offloading that operating system call (such as reading a file, perform TCP request, handling a socket connection and so on) to another process.

For example, in the following code:

{% highlight javascript %}
function myCallback(err, data) {
  console.log(data)
}

fs.readFile('/my-file.txt', myCallback)
{% endhighlight %}

Reading the file can take hours to be performed, but Node.js will run this task in an OS process, outside the main JavaScript runtime thread and will continue the execution of the remaining code. Once the reading of the file is finished `myCallback` will be executed in the main thread.

This is similar in the browsers, when we perform an HTTP request, it is performed by the browser outside the main thread.

### Recap

> **Node.js** runs JavaScript **concurrently** in a **single thread** using an **Event Loop** and **run-to-completion** scheduling of tasks.

## The question

After reviewing all of this, we can go back to the question:

> Imagine you have a Node server started with `node index.js`. In this file, we have a route that runs some code which isn't I/O and takes 1 minute to complete. Something like the following:

{% highlight javascript %}
function longTask() {
  for (let i = 0; i <= 10_000_000_000; i++) {}
}

app.get('/foo', () => {
  longTask()
})
{% endhighlight %}

> What is the problem we face when 3 concurrent users try to access this route?

The first user will block for 1 minute the start of the request of the next user, and so on.

We now know that `app.get('/foo')` is part of an I/O task, so it will be processed outside of the _main thread_. But, once a request arrives, the main thread will run `() => longTask()` and, as that function is a CPU expensive computation and there is no I/O involved, this function will block the main thread for 1 minute while it is being executed.

> How can we solve it?

There are several answers here, some wrong and some valid. It's worth reviewing them all.

### Common pitfalls

> As it is a callback it doesn't block the main thread

We saw before this isn't true.

> We wrap it within a `setTimeout`

This doesn't work, as we mention before, Node.js execution isn't parallel, so this only delays the blocking for the future.

If we do something like this:

{% highlight javascript %}
function longTask() {
  for (let i = 0; i <= 10_000_000_000; i++) {}
}

app.get('/foo', () => {
  setTimeout(longTask, 1000)
})
{% endhighlight %}

The users that arrive within one second of the first user, will have their request accepted. However, once the second has passed, the server will be blocked.

> We make the route an `async` function

This doesn't work either. Async functions are like _Promises_, when the function is called, it is enqueued in the Event Loop, but once it pops out of the Event Loop and it is executed, it runs in the main thread and it will block it anyway.

So, something like this won't work:

{% highlight javascript %}
function longTask() {
  for (let i = 0; i <= 10_000_000_000; i++) {}
}

app.get('/foo', async () => {
  longTask()
})
{% endhighlight %}

### Possible solutions to our problem

To solve this problem, we need to offload the computation of `longTask` to the background. We can find different approaches here:

> Transform the operation to an external I/O call

If we implement this function as a background job or another _service_, we can transform a CPU intensive call to an I/O call.

For example, the main server will have the following code:

{% highlight javascript %}
app.get('/foo', () => {
  console.log('Request received')
  // We don't await this call in order to avoid
  // keeping open user connections waiting for a result
  https.get('https://another-service/longTask')
  console.log('Request finished but task still running')
})
{% endhighlight %}

The server that will be blocked will be `another-service` (which we can scale as we want), and our main server can take as users _as we want_ because `https.get` is an I/O task and it won't block any incoming request.

Note that we can use a message queue to communicate between services or background jobs, it is up to you.

> Use Node.js Child Process

This approach, with its benefits and downsides, is really well explained [here](https://nodejs.org/en/docs/guides/dont-block-the-event-loop/#offloading){:target="\_blank"}.

> Using Node.js Worker threads

Ok, fine, I've hidden some information about Node.js threads 🙈. Since `v10.5.0` of Node.js we can use what is called [Worker threads](https://nodejs.org/api/worker_threads.html){:target="\_blank"}, in browsers we have something similar called [WebWorkers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers){:target="\_blank"}.

Using a worker thread, we can offload the execution of this task to another thread (sharing memory space) within the same machine.

## Final thoughts

Usually, in the majority of programs that we write, it's hard to reach a situation where we accidentally block the main thread with a CPU intensive task.

However, I hope this article helps you in the future finding possible bottlenecks or slowdowns in your web server or frontend application.

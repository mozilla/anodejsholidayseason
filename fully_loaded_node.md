## Fully Loaded Node

> Episode 2 in the *A Node.JS Holiday Season* series from Mozilla's Identity team searches for an optimal server application architecture for computation heavy workloads.
>
> This is a prose version of a short talk given by [Lloyd Hilaiel at Node Philly 2012][] with the same title.

  [Lloyd Hilaiel at Node Philly 2012]: http://www.youtube.com/watch?v=U0hNgO5hrtc

A Node.JS process runs almost completely on a single processing core, because of this building scalable servers requires special care.
With the ability to write native extensions and a robust set of APIs for managing processes, there are many different ways to design a Node.JS application that executes code in parallel: in this post we'll evaluate these possible designs.

This post also introduces the [compute-cluster][] module: a small Node.JS library that makes it easy to manage a collection of processes to distribute computation.

## The Problem

We chose Node.JS for [Mozilla Persona][], where we built a server that could handle a large number of requests with mixed characteristics.
Our "Interactive" requests have low computational cost to execute and need to get done fast to keep the UI feeling responsive, while "Batch" operations require about a half second of processor time and can be delayed a bit longer without detriment to user experience.

  [Mozilla Persona]: https://persona.org

To find a great application design, we looked at the type of requests our application had to process, thought long and hard about usability and scaling cost, and came up with four key requirements:

  * **saturation**: The solution will be able to use every available processor.
  * **responsiveness**: Our application's UI should remain responsive. Always.
  * **grace**: When overwhelmed with more traffic than we can handle, we should serve as many users as we can, and promptly display a clear error to the remainder.
  * **simplicity**: The solution should be easy to incrementally integrate into an existing server.

Armed with these requirements, we can meaningfully contrast the approaches:

### Approach 1: Just do it on the main thread.

When computation is performed on the main thread. the results are terrible:
You cannot **saturate** multiple computation cores, and with repeated half second starvation of interactive requests you cannot be **responsive** nor **graceful**.
The only thing this approach has going for it is **simplicity**:

    function myRequestHandler(request, response) [
      // Let's bring everything to a grinding halt for half a second.
      var results = doComputationWorkSync(request.somesuch);
    }

Synchronous computation in a Node.JS program that is expected to serve more than one request at a time is a bad idea.

### Approach 2: Do it Asynchronously.

Using asynchronous functions that run in the *background* will improve things, right?
Well, that depends on what precisely the *background* means:
If the computation function is implemented in such a way that it actually performs computation in javascript or Native code on the main thread, then performance is no better than with a synchronous approach.
Have a look:

    function doComputationWork(input, callback) {
      // Because the internal implementation of this asynchronous
      // function is itself synchronously run on the main thread,
      // you still starve the entire process.
      var output = doComputationWorkSync(input);
      process.nextTick(function() {
        callback(null, output);
      });
    }

    function myRequestHandler(request, response) [
      // Even though this *looks* better, we're still bringing everything 
      // to a grinding halt.
      doComputationWork(request.somesuch, function(err, results) {
        // ... do something with results ...
      });
    }

The key point is usage of an asynchronous API in NodeJS does not necessarily yield an application that can use multiple processors.

### Approach 3: Do it Asynchronously with Threaded Libraries!

Given a library that is written in native code and cleverly implemented, it *is* possible to execute work in different threads from within NodeJS.
Many examples exist, one being the excellent [bcrypt library][] from [Nick Campbell][].

  [bcrypt library]: http://github.com/ncb000gt/node.bcrypt.js
  [Nick Campbell]: http://github.com/ncb000gt

If you test this out on a four core machine, what you will see will look fantastic!  Four times the throughput, leveraging all computation resources!  If you perform the same test on a 24 core processor, you won't be as happy: you will see four cores fully utilized while the rest sit idle.

The problem here is that the library is using NodeJS's internal threadpool for a problem that it was not designed for, and this threadpool has a [hardcoded upper bound of 4][].

  [hardcoded upper bound of 4]: https://github.com/joyent/node/blob/e2bcff9aa75e51b9ba071330fe712180abed03e0/deps/uv/src/unix/threadpool.c#L31

Deeper problems exist with this approach, beyond from these hardcoded limits:

  * Flooding NodeJS's internal threadpool with computation work can starve network or file operations, which hurts **responsiveness**.
  * There's no good way to control the backlog - If you have 5 minutes of computation work already sitting in your queue, do you really want to pile more on?

Libraries that are "internally threaded" in this manner fail to **saturate** multiple cores, adversely affect **responsiveness**, and limit the application's ability to degrade **gracefully** under load.

### Approach 4: Use node's cluster module!

NodeJS 0.6.x and up offer a [cluster module][] which allows for the creation of processes which "share a listening socket" to balance load across some number of child processes.
What if you were to combine cluster with one of the approaches described above?

  [cluster module]: http://nodejs.org/docs/v0.8.14/api/all.html#all_how_it_works

The resultant design would inherit the shortcomings of synchronous or internally threaded solutions: which are not **responsive** and lack **grace**.

Simply spinning new application instances is not always the right answer.

### Approach 5: Introducing compute-cluster

Our current solution to this problem in Persona is to manage a cluster of single-purpose processes for computation.
We've generalized this solution in the [compute-cluster][] library.

  [compute-cluster]: https://github.com/lloyd/node-compute-cluster

`compute-cluster` spawns and manages processes for you, giving you a programatic means of running work on a local cluster of child processes.
Usage is thus:

    const computecluster = require('compute-cluster');
    
    // allocate a compute cluster
    var cc = new computecluster({ module: './worker.js' });

    // run work in parallel
    cc.enqueue({ input: "foo" }, function (error, result) {
      console.log("foo done", result);
    });
    cc.enqueue({ input: "bar" }, function (error, result) {
      console.log("bar done", result);
    });

The file `worker.js` should respond to `message` events to handle incoming work:

    process.on('message', function(m) {
      var output;
      // do lots of work here, and we don't care that we're blocking the 
      // main thread because this process is intended to do one thing at a time.
      var output = doComputationWorkSync(m.input);
      process.send(output);
    });

It is possible to integrate `compute-cluster` behind an existing asynchronous API without modifying the caller, and to start really performing work in parallel across multiple processors with minimal code change.

So how does this approach achieve the four criteria?

**saturation**: Multiple worker processes use all available processing cores.

**responsiveness**: Because the managing process is doing nothing more than process spawning and message passing, it remains idle and can spend most of its time handling interactive requests.
Even if the machine is loaded, the operating system scheduler can help prioritize the management process.

**simplicity**: Integration into an existing project is easy: By hiding the details of `compute-cluster` behind a simple asynchronous API, calling code remains happily oblivious of the details.

Now what about **gracefully** degrading during overwhelming bursts of traffic?
Again, the goal is to run at maximum efficiency during bursts, and serve as many requests as possible.  

Compute cluster enables a graceful design by managing a bit more than just process spawning and message passing.
It keeps track of how much work is running, and how long work takes to complete on average.
With this state it becomes possible to reliably predict how long work will take to complete *before it is enqueued*.

Combining this knowledge with a client supplied parameter, `max_request_time`, makes it possible to preemptively fail on requests that are likely to take longer than allowable.

This feature lets you easily map a user experience requirement into your code: "The user should not have to wait more than 10s to login", translates into a `max_request_time` of about 7 seconds (with padding for network time).

In load testing the Persona service, the results so far are promising.
Under times of extreme load we are able allow authenticated users to continue to use the service, and block a portion of unauthenticated users right up front with a clear error message.

## Next Steps

Application level parallelization using processes works well for a single tier deployment architecture - An arrangement where you have only one type of node and simply add more to support scale.
As applications get more complex however, it is likely the deployment architecture will evolve to have different application tiers to support performance or security goals.

In addition to multiple deployment tiers, high availability and scale often require application deployment in multiple colocation facilities.  Finally, cost effective scaling of a computationally bound application can be achieved by leveraging on-demand cloud based computation resources.

Multiple tiers in multiple colos with demand spun cloud servers changes the parameters of the scaling problem considerably while the goals remain the same.

The future of [compute-cluster][] may involve the ability to distribute work over multiple different tiers to maximally saturate available computation resources in times of load.
This may work cross-colo to support geographically asymmetric bursts.
This may involve the ability to leverage new hardware that's demand spun...

Or we may solve the problem a different way!  If you have thoughts on an elegant way to enable [compute-cluster][] to distribute work over the network while preserving the properties it has thus far, I'd love to hear them!

Thanks for reading, and you join the discussion and learn more about current scaling challenges and approaches in Persona on [our email list][].

  [our email list]: https://groups.google.com/d/msg/mozilla.dev.identity/st_eL7kUpUw/hrOUUatl5i0J

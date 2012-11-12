## Fully Loaded Node

> Episode 2 in the *A Node.JS Holiday Season* series from Mozilla's Identity team searches for an optimal server application architecture for computation heavy workloads.
>
> This is a prose version of a short talk given by [Lloyd Hilaiel at Node Philly 2012][] with the same title.

  [Lloyd Hilaiel at Node Philly 2012]: http://www.youtube.com/watch?v=U0hNgO5hrtc

There was time, not too long ago, when having multiple processing cores within a single CPU was not an option: in order to build a parallel workstation you needed an expensive motherboard that supported multiple CPUs.  These boards were decidedly not for everyday consumers.  Much of the software available, including OS kernels and drivers, were not optimized for multiple processors.  The reward for a technolust to deploy multiple cores in data centers or as workstations was often rewarded with unstable systems and diminishing returns.  *Those days are gone.*  Today we reliably run at least two cores on our desktops, in our laptops, and on our smart phones.  Spinning up a 12 core machine in the cloud is commonplace.   We live in a multiprocessing world, and we're not turning back - our software must adapt.

Given this trend, the frenzy around Node.JS as an emergent server software platform may seem initially ironic.  With Node.JS, all of your application code runs on a single processor.  The word *thread* is not present in the current stable Node.JS API and there are no synchronization primitives.  It turns out that this was not an oversight in the design of Node.JS.  There are many ways to build a service that leverages all processing cores available.  Approaches range from internal threading in libraries you use, to spawning multiple processes either manually, or managed by software, or by your Node.JS applications themselves.

This post will explore the various high level approaches available to building Node.JS servers that run extra hot. We will present the [compute-cluster][] module, which is a small Node.JS library that makes it easy to manage a collection of processes which distribute computation across multiple processors.
  
## The Problem

We choose Node.JS for [Mozilla Persona][], where we built a server that could handle a large number of requests with mixed characteristics.  Our "Interactive" requests have low computational cost to execute and need to get done fast to keep the UI feeling responsive, while "Batch" operations require about 500ms of processor time and can be delayed a bit longer without making the user sad.

  [Mozilla Persona]: https://persona.org

To find a great application design, we matched this mix of requests with our desires for how things should work, and came up with four key requirements:

  * **saturation**: The solution will be able to use every available processor.
  * **responsiveness**: Our application's UI should remain responsive.  Always.
  * **grace**: When overwhelmed with more traffic than we can handle, we should serve as many users as we can, and display a clear error to the remainder.
  * **simplicity**: The solution should easy to incrementally integrate into an existing server.

Armed with this, we can meaningfully contrast several different approaches:

### Approach 1: Just do it on the main thread.

If we simply perform computation work on the main thread, the results are terrible.  You cannot *saturate* multiple computation cores, and you cannot be *responsive* nor *graceful* with repeated half second starvation of interactive requests.  The only thing this approach has going for it is *simplicity*:

    function myRequestHandler(request, response) [
      // Let's bring everything to a grinding halt for half a second.
      var results = doComputationWorkSync(request.somesuch);
    }

**tl;dr;**: Synchronous computation is bad.

### Approach 2: Do it Asynchronously.

We can improve this implementation by using asynchronous functions that run in the *background*, right?  Well, kind of.  It depends on what precisely the *background* means.  If your computation function is implemented in such a way that it actually performs computation in javascript or Native code on the main thread, then you are doing no better than with a synchronous approach:  

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
        // … do something with results ...
      });
    }

**tl;dr;** An asynchronous API in NodeJS does not imply that work is run on a different processor.

### Approach 3: Do it Asynchronously with Threaded Libraries!

If you have a library that is written in native code and cleverly implemented, you can actually execute the costly work in different threads from within NodeJS.  Many examples exist, one being the excellent [bcrypt library][] from [Nick Campbell[].

  [bcrypt library]: http://github.com/ncb000gt/node.bcrypt.js
  [Nick Campbell]: http://github.com/ncb000gt

If you test this out on a four core machine, what you will see will look fantastic!  Four times the throughput, leveraging all computation resources!  If you perform the same test on a 24 core processor, you won't be as happy: you will see four cores fully utilized while the
rest sit idle.  The problem here is that the library is using NodeJS's internal threadpool for a problem that it was not designed for, and this threadpool has a [hardcoded upper bound of 4][].

  [hardcoded upper bound of 4]: https://github.com/joyent/node/blob/e2bcff9aa75e51b9ba071330fe712180abed03e0/deps/uv/src/unix/threadpool.c#L31

A more fundamental problem with this approach (which partially explains why upping this limit is a bad idea) is because we are now flooding NodeJS's internal threadpool, we will starve other work that runs on it.  This can include things like network or file IO.  

Libraries that are "internally threaded" in this manner both fail to **saturate** multiple cores and adversely affect **responsiveness** under load.

**tl;dr;** Don't use library features that claim to be "internally threaded" to parallelize compute work.

### Use node's cluster module!

NodeJS 0.6.x and up offer a [cluster module][] that allows you to create processes which "share a listening socket" to balance load across some number of spun child processes.  What if you were to combine cluster with one of the approaches described above?

  [cluster module]: http://nodejs.org/docs/v0.8.14/api/all.html#all_how_it_works

The problem with this approach that we inherit the shortcomings of synchronous or internally threaded solutions.  Mainly, this is not a road that leads to **responsiveness** and **grace**.

**tl;dr;**: Creating more application instances isn't always the answer.

## Introducing compute-cluster

Our current solution to this problem in Persona is to manage a cluster of single-purpose processes for computation.  We've generalized this solution in the [compute-cluster][] library.

  [compute-cluster]: https://github.com/lloyd/node-compute-cluster

`compute-cluster` spawns and manages processes for you, giving you a programatic means of running work on a local cluster of child processes.  Usage is thus:

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
      // do lots of work
      process.send(output);
    });

You can integrate compute-cluster behind an existing asynchronous API without modifying the caller, and suddenly you really are performing work in parallel across multiple processors.

So how can we achieve our four criteria armed with this new module?

**saturation**: because we can dial up the number of worker processes, we can fully leverage any number of cores on a machine.

**responsiveness**: Because the managing process is doing nothing more than process spawning and message passing, it remains idle and can spend most of its time handling interactive requests.  Even if the machine is loaded, the operating system scheduler can help prioritize the management process.

**simplicity**: By hiding the details of `compute-cluster` behind a simple API with expected asynchronous calling semantics, you can keep code happily oblivious of the details, making integration into an existing project easy.

### The Question of *Grace*

Compute cluster manages a bit more than just process spawning and message passing.  It knows how much work is running, and knows empirically how long work takes work to finish on average.  These bits allow us to reliably predict how long a new bit of work will take to complete.

When we combine this knowledge with a client supplied parameter, `max_request_time`, it becomes possible to preemptively fail on requests that are likely to take longer than allowable.  

This feature lets you easily map a user experience requirement into your code: "The user should not have to wait more than 10s to login", results in a `max_request_time` of about 7 seconds (with padding for network time).

In load testing the persona service, the results so far are promising. Under times of extreme load we are able allow authenticated users to continue to use the service, and block a portion of unauthenticated users right up front.

## Next Steps

Application level parallelization using processes works well for a single tier deployment architecture - An arrangement where you have only one type of node and simply add more to support scale.  As applications get more complex however, it is likely the deployment architecture will evolve to have different application tiers to support performance
or security goals.

In addition to multiple deployment tiers, high availability and scale often require application deployment in multiple colocation facilities.

Finally, cost effective scaling of a computationally bound application can be achieved by leveraging on-demand cloud based computation resources.

Multiple tiers in multiple colos with demand spun cloud servers changes the parameters of the scaling problem considerably while the goals remain the same.

The future of `compute-cluster` may involve the ability to distribute work over multiple different tiers to maximally saturate available computation resources in times of load.  This may work cross-colo to support geographically asymmetric bursts.  This may involve the ability to leverage new hardware that's demand spun at some trusted cloud compute provider…
Or we may solve the problem a different way!

Whatever we do, we'll be sure to blog up our learnings!  Thanks for reading, and you can learn more about current scaling challenges and approaches in Persona on [our email list][].

  [our email list]: https://groups.google.com/d/msg/mozilla.dev.identity/st_eL7kUpUw/hrOUUatl5i0J

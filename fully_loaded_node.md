## Fully Loaded Node

> Episode 2 in the *A Node.JS Holiday Season* series from Mozilla's Identity
> team searches for an optimal server application architecture for computation
> heavy workloads.
>
> This is a prose version of a short talk given by
> [Lloyd Hilaiel at Node Philly 2012][] with the same title.

  [Lloyd Hilaiel at Node Philly 2012]: http://www.youtube.com/watch?v=U0hNgO5hrtc

A common moment of discomfort when you first meet NodeJS is when you
learn that you don't have threads.  A single NodeJS program runs
almost completely on one processor.  Crazy, right?  Some people, upon
discovering this limitation, conclude that NodeJS simply isn't a
serious server platform.

What if you have half a second of computation work that you need to
do?  If you do this work on the main javascript evaluation thread, it
means that for an excruciating half a second that is *all* you are doing:
No other application javascript can run, no request handling, no joy!

How can all of this be true and the author *still* posit that NodeJS
turns out to be a fantastic environment for computationally intensive
applications that need to leverage servers with tens of processing
cores?  Because we don't need no threads!  We've got processes which
give you full memory isolation, fail gracefully, start fast, sit in a
relatively piddling little pool of memory - they give you all the
benefits of threading without the angst.

If you spend ten more minutes with me, I'll explain an application
architecture that rocks for compute heavy workloads, and present
a teensy tinsy library that makes that architecture
easy to realize.  We shall let no processor go underutilized.

## The Problem

The specific problem that we faced in [Mozilla Persona][], the service
that motivated this post, was in building a server in NodeJS that
could handle a large number of requests with mixed characteristics.
"Interactive" requests have low computational cost to execute and need
to get done fast to keep the UI feeling responsive, while "Batch"
operations require between 100ms and 500ms of processor time and can
be delayed a bit longer without making the user sad.

We matched this mix of requests with our desires for how the application
should work, and came up with four key requirements of a solution:

  * **saturation**: The solution will use every available processor
     under load.
  * **responsiveness**: Even under maximal load, our applications UI
     should remain responsive.
  * **grace**: We expect the unexpected: at some point be we will be
     overwhelmed with more traffic than we can handle.  At this time
     we should serve as many users as we can, with an excellent user
     experience, while displaying a fast clear error to the remainder.
  * **simplicity**: When we deploy this solution, we should not have
     to buy new software, stand up new servers.  It should be simple
     to incrementally integrate into the service into areas of code
     that are CPU intensive.

Armed with these requirements, we can meaningfully contrast a couple
different approaches.

## There Is More Than One Way To Do It

While there are an unbounded number of ways to solve this problem,
let's distill out the key approaches and examine each briefly.

### Just do it on the main thread.

If we simply perform computation work on the main thread, the solution
falls down fast.  You cannot *saturate* multiple computation cores,
you cannot be *responsive* nor *graceful* with repeated half second
starvation of interactive requests, though this approach is certainly
simple:

    function myRequestHandler(request, response) [
      // here we'll bring everything to a grinding halt for
      // a half second.
      var results = doComputationWorkSync(request.somesuch);
    }

**tl;dr;**: Synchronous computation is bad.

### Do it Asynchronously.

We can improve this implementation by using asynchronous functions
that run in the *background*, right?  Well, kind of.  It depends on 
what precisely the *background* means.  If your computation function
is implemented in such a way that it actually performs computation
in javascript or Native code on the main thread, then you actually
are doing no better than a synchronous approach:  

    function doComputationWork(input, callback) {
      // Because here the internal implementation of an asynchronous
      // function is itself synchronously run on the main thread,
      // you still starve the entire process.
      var output = processInputForHalfASecond(input);
      process.nextTick(function() {
        callback(output);
      });
    }

    function myRequestHandler(request, response) [
      // here we'll bring everything to a grinding halt for
      // a half second.
      var results = doComputationWorkSync(request.somesuch);
    }

**tl;dr;** Asynchronous in NodeJS does not imply that work is run
in "the background" or is somehow innately parallelizable.

### So Do it Asynchronously with Carefully Written Libraries!

If you have a library that is written in native code and cleverly
implemented, you can actually execute the costly function in different
threads from within NodeJS [footnote 1].  Many examples exist, one
being the excellent [bcrypt library][] from [Nick Campbell[].

  [bcrypt library]: http://github.com/ncb000gt/node.bcrypt.js
  [Nick Campbell]: http://github.com/ncb000gt

If you test this out on a four core machine, what you will see will
look fantastic!  Four times the throughput, leveraging all computation
resources!  If you perform the same test on a 24 core processor, you
won't be as happy: you will see four cores fully utilized while the
rest sit idle.  The problem here is that the library is using NodeJS's
internal threadpool for a problem that it was not designed for, and this
threadpool has a hardcoded upper bound of 4 [footnote 2].

A more fundamental problem with this approach is that because you
are now flooding NodeJS's internal threadpool, you will starve
other work that runs on it.  This can include things like network or
file IO.  

Libraries that are "internally threaded" in this manner both fail to
**saturate** multiple cores and adversely affect **responsiveness** under
load.

**tl;dr;** Don't use library features that claim to be
"internally threaded" to parallelize compute work.

### Use node's cluster module!

NodeJS 0.6.x and up offers a very cute [cluster module][] that allows you
to create servers that "share a listening socket" to load balance
across some number of spun child processes.  What if you were to
combine cluster with one of the approaches described above?

  [cluster module]: XXX

The problem with this approach is you must combine it with one of the
approaches we've already explored, and you inherit the shortcomings of
them.  Mainly, this is not a road that leads to **responsiveness** and
**grace**.

**tl;dr;**: Creating more application instances isn't always the
solution.

## Introducing compute-cluster

Our current solution to this problem in persona is our very own node
module called [compute-cluster][].

  [compute-cluster]: https://github.com/lloyd/node-compute-cluster

`compute-cluster` is small library that spawns and manages processes
for you, giving you a programatic means of running work on a local
cluster of child processes.  Usage is simple:

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

The file `worker.js` should respond to `message` events to handle incoming
work:

    process.on('message', function(m) {
      var output;
      // do lots of work
      process.send(output);
    });

You can integrate compute-cluster into a javascript library and expose
an asynchronous API the just works, and really does perform work in
the background and in parallel.

So how can we achieve our four criteria armed with this new module?

**saturation**: because we can dial up the number of worker processes,
we can fully leverage any number of cores on a machine.

**responsiveness**: Because the managing process is doing nothing more 
than process spawning and message passing, it remains idle and can spend
most of it's time handling other, interactive requests.  Even if the
machine is slammed, the operating system scheduler can help prioritize
the management process.

**simplicity**: By hiding the details of `compute-cluster` behind a
simple API with expected asynchronous calling semantics, you can
calling code happily oblivious of the details, making integration
into an existing project easy.

### The Question of *Grace*

Compute cluster manages a bit more than just process spawning and message
passing.  It knows how much work is running, and knows empirically how
long work takes to finish on average.  These bits allow us to reliably
predict how long a new bit of work will take to finish.

When we combine this knowledge with a client supplied parameter,
`max_request_time`, it becomes possible to preemptively fail on requests
that are likely to take longer than allowable.  

This feature let's you easily map a user experience requirement into
your code: "The user should not have to wait more than 10s to login", results
in a `max_request_time` of about 7 seconds (with padding for network time).

In load testing the persona service, the results so far are promising.
Under times of extreme load we are able allow authenticated users to
continue to use the service, and block a portion of unauthenticated users right
up front.

## Next Steps

Application level parallelization using processes works well for a
single tier deployment architecture - An arrangement where you have
only one type of node and simple add more to support scale.  As applications
get more complex however, it is likely the deployment architecture
will evolve to have different application tiers to support performance
or security goals.

In addition to multiple deployment tiers, high availability and scale often
require application deployment in multiple colocation facilities.

Finally, cost effective scaling of a computationally bound application
can be achieved by leveraging on-demand cloud based computation resources.

Multiple tiers in multiple colos changes the parameters of the scaling problem
considerably while the goals remain the same.

The future of `compute-cluster` may involve the ability to distribute work
over multiple different tiers to maximally saturate available computation
resources in times of load.  This may work cross-colo to support
geographically asymmetric bursts.  This may involve the ability to leverage
new hardware that's demand spun at some trusted cloud compute provider.
Or we may solve the problem a different way.

Whatever we do, we'll be sure to blog up our learnings!  Thanks for reading, and
you can learn more about current scaling challenges and approaches in Persona on
[our email list][].

  [our email list]: XXX link to scaling approach email


[footnote 1] bcrypt does this by leveraging the same threadpool that
NodeJS itself uses to internally parallelize requests.  This calls
out that when I said "NodeJS runs almost completely on one processor"
I was over-simplifying.  As of 0.8.x node's internal parallelization allows
single threaded application code to run at 150% to 200% processor usage
by implementing concurrency at a low level that the application remains
blissfully ignorant of.
[footnote 2] XXX say more about this upper bound
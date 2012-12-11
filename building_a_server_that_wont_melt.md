## Building A Node.JS Server That Never Melts

This post presents a tiny Node.JS library that allows you build a server that, faced with *impossible* load, keeps running and stays responsive.

The remainder of the post will explain these five lines of code:

    var toobusy = require('toobusy');

    app.use(function(req, res, next) {
      if (toobusy()) res.send(503, "I'm busy right now, sorry.");
      else next();
    }); 

## Why Bother?

If your application is important to people, then it's worth spending a moment thinking about "disaster" scenarios.  These are the good kind of disasters where your project becomes the apple of social media's eye and you go from ten thousand users a day to a million.  If you think about instantaneous two-order-of-magnitude growth for a moment, you can build something that stays up and serves as many users as possible during this event while you bring on the hardware.  If you don't, then it's highly likely your service will become completely unusable at precisely the Wrong Time.

Another great reason to think about legitimate bursts traffic, is malicious bursts of traffic.  The first step to mitigating DoS attacks is building servers that don't melt.

## Your Server Under Load

To illustrate how applications with no considerations for burst behave, I built an application server with an HTTP API that consumes 10ms of processor time spread over five asynchronous function calls (which is a rough average of the cost of request handling in the [Persona][] service).  Here is a depiction of call latency as the number of requests doubles  every ten seconds:

[INSERT GRAPH HERE]

The graph tells us that if we considered 300ms to be the maximum allowable latency for an API call, the maximum capacity of the application is around X API calls / second.  The application breaking point is around Y API calls / second - where users wait time gets up in the 30s range for a single request.  

## Failing Gracefully

Instrumenting the same application with the five lines of code from the beginning of this post, the graph looks like this:

[INSERT GRAPH HERE]

Notice the presence of a new line, which is "refused requests".  Sending pre-emptive 503 (server too busy) HTTP responses is an approach you can use to throttle requests before they consume your servers resources.  In this case when we hit 300ms of latency, the server begins denying requests which exceed its maximum capacity.

## How To Use It

[node-toobusy][] is available on [npm][] and [github][].  Simply include "toobusy" in your project, and in your code require it:

    var toobusy = require('toobusy');

At the moment the library is included, it will begin actively monitoring the process, and will determine when the process is "too busy".  You can then check if the process is toobusy at key points in your application:

    // The absolute first piece of middleware we would register, to block requests
    // before we spend any time on them.
    app.use(function(req, res, next) {
      if (toobusy()) res.send(503, "I'm busy right now, sorry.");
      else next();
    });

This application of [node-toobusy][] gives you a basic level of robustness at load, which you can tune and customize to fit the design of your application.

## How It Works

[node-toobusy][] polls node.js's internal event loop multiple times a second and looks for lag.  That is, toobusy asks to be woken in exactly 500ms, and the number of milliseconds *late* it is, is the "lag" in the event loop.  When a server is well constructed (no computational blocks of more that about 10ms in the main application process), this lag occurs when there is more work to be done than the Node.JS process can computationally handle.

Using event loop lag as a means to determine when requests can be blocked is a good solution because it is:

  1. Extremely inexpensive to check
  2. Extremely simple
  3. Equally well detects an overloaded host process and an overloaded host *machine*.


## More Things to Think About




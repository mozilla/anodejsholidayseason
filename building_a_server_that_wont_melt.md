## Building A Node.JS Server That Won't Melt

How can you build a Node.JS application that keeps running, even under *impossible* load?

This post offers an answer, which is captured in the following five lines of code:

    var toobusy = require('toobusy');

    app.use(function(req, res, next) {
      if (toobusy()) res.send(503, "I'm busy right now, sorry.");
      else next();
    });

## Why Bother?

If your application is important to people, then it's worth spending a moment thinking about *disaster* scenarios.
These are the good kind of disasters where your project becomes the apple of social media's eye and you go from ten thousand users a day to a million.
With a bit of preparation, you can build something that serves as many users as possible during instantaneous two-order-of-magnitude growth, while you bring on the hardware.
If you forego this preparation, then your service will become completely unusable at precisely the Wrong Time - when everyone is watching.

Another great reason to think about legitimate bursts traffic, is malicious bursts of traffic.
The first step to mitigating DoS attacks is building servers that don't melt.

## Your Server Under Load

To illustrate how applications with no considerations for burst behave, I [built an application server][] with an HTTP API that consumes 5ms of processor time spread over five asynchronous function calls.
By design, a single instance of this server is capable of handling 200 requests per second.

  [built an application server]: https://gist.github.com/4532177

This roughly approximates a typical request handler that perhaps does some logging, interacts with the database, renders a template, and streams out the result.
What follows is a graph of server latency and TCP errors as we linearly increase connection attempts from 40 to 1500 attempts per second:

![Your server without limits](../../raw/master/building_a_server_that_wont_melt/without.png)

Analysis of the data from this run tells a clear story:

**This server is not responsive**:  At 2x capacity (400 requests/second) the average request time is 3 seconds, and at 4x capacity it's 9 seconds.  After a couple minutes of 5x maximum capacity, the server performs with *40 seconds of average request latency*.

**These failures suck**:  With over 80% TCP failures and high latency, users will see a confusing failure after *up to a minute* of waiting.

## Failing Gracefully

Next, I instrumented the [same application with the code from the beginning of this post][].
This code causes the server to detect when load exceeds capacity and pre-emptively refuse requests.
The following graph depicts the performance of this version of the server as we increase connections attempts from 40 to 3000 per second.

  [same application with the code from the beginning of this post]: https://gist.github.com/4532198#file-application_server_with_toobusy-js-L26-L29

![Your server with limits](../../raw/master/building_a_server_that_wont_melt/with.png)

What do we learn from this graph and the underlying data?

**Pre-emptive limiting adds robustness**: Under load that exceeds capacity by an order of magnitude the application continues to behave reasonably.

**Success and Failure is fast**: Average response time stays for the most part under 10 seconds.

**These failures don't suck**:  With pre-emptive limiting we effectivly convert slow clumsy failures (TCP timeouts), into fast deliberate failures (immediate 503 responses).

## How To Use It

node-toobusy is available on [npm][] and [github][].  After installation (`npm install toobusy`), simply require it:

  [github]: https://github.com/lloyd/node-toobusy
  [npm]: https://npmjs.org/package/toobusy

    var toobusy = require('toobusy');

At the moment the library is included, it will begin actively monitoring the process, and will determine when the process is "too busy".
You can then check if the process is toobusy at key points in your application:

    // The absolute first piece of middleware we would register, to block requests
    // before we spend any time on them.
    app.use(function(req, res, next) {
      if (toobusy()) res.send(503, "I'm busy right now, sorry.");
      else next();
    });

This application of `node-toobusy` gives you a basic level of robustness at load, which you can tune and customize to fit the design of your application.

## How It Works

`node-toobusy` polls node.js's internal event loop multiple times a second and looks for lag.
That is, toobusy asks to be woken in exactly 500ms, and the number of milliseconds *late* it is, is the "lag" in the event loop.
When a server is well constructed (no computational blocks of more that about 10ms in the main application process), this lag occurs when there is more work to be done than the Node.JS process can computationally handle.

Using event loop lag as a means to determine when requests can be blocked is a good solution because it is:

  1. Extremely inexpensive to check
  2. Extremely simple
  3. Equally well detects an overloaded host process and an overloaded host *machine*.


## More Things to Think About




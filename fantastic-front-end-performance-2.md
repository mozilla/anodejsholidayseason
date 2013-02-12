> This is episode 6, out of a total 12, in the [A Node.JS Holiday Season series](https://hacks.mozilla.org/category/a-node-js-holiday-season/) from Mozilla’s Identity team. It’s the second post about achieving better front-end performance.

You've implemented the [Three C's of client side performance](https://hacks.mozilla.org/2012/12/fantastic-front-end-performance-part-1-concatenate-compress-cache-a-node-js-holiday-season-part-4/), concatenate, compress, and cache, but how do you know they're working? This post introduces simple tools for measuring the performance impact of optimizations like HTTP caching.

## Measuring Performance: HARs, Heuristics, and Statistics

### HTTP Archives

While there are many open-source and commercial tools available to measure website performance, practically all record data in the same format: HTTP Archive (or "HAR") files. HAR files are standard JSON documents containing performance data like HTTP headers, network / DNS lookup times, data transfer times, and other details.

Most tools display the performance data in the same way, too: using a 'waterfall' visualization. Here's one example, click through for more:

    insert waterfall screenshot here
    linked to http://www.webpagetest.org/result/130212_K7_ARC/1/details/

The waterfall represents the sequence of events from initial request, through all page components being requested, downloaded, and processed. Scrolling downward through the list of files goes forward in time. Similarly, the individual files' entries flow from left to right through time, showing how much time passed between request and completed response.

Most browsers have tools built-in or available as extensions for generating and visualizing HAR files, such as the [NetExport Firebug extension](http://www.softwareishard.com/blog/netexport/) for Firefox.

### Heuristics

To help make sense of the information, tools like [YSlow](http://developer.yahoo.com/yslow/) use heuristics to suggest potential optimizations to a site.

    screenshot of YSlow on login.persona.org

While the number of optimizations can seem overwhelming, the golden rule is to minimize the number of HTTP requests made on a given page. Thus, one simple way to measure performance is to count up the number of HTTP requests and the overall page load time. Depending on the nature of your app, you might care about more nuanced measurements, but page load time is a fine starting point.

### Simple Statistics

Measurements of page load time will fluctuate with network congestion and server load. In the face of that uncertainty, how can we be sure that we've made meaningful changes? The answer: applying simple statistics over many measurements.

We'll take a series of measurements before and after our optimizations, calculate the mean and standard deviation of each data set, and compare the difference in means to quantify the improvement. Lastly, we'll use a t-test to determine if the improvement is statistically significant, or if it's potentially due to random chance.

# HTTP caching review

HTTP provides two ways for servers to control client-side caching of page components:
  * freshness may be based on a date or a token whose meaning is app-specific
  * whether or not to confirm the cached version is up-to-date with the server

This breaks down as follows:
  * Cache locally and don't check before using.
    * This avoids a network request completely.
    * Expires header - asks the browser to use the local copy until some date
    * Cache-Control:max-age header - asks the browser to use the local copy until a number of seconds after download
  * Cache locally, but check before using. 
    * This requires a network request to contact the server, but avoids bandwidth costs of re-downloading.
    * Last-Modified header - asks the browser to confirm the server hasn't updated the component since the Last-Modified date
    * ETag header - asks the browser to confirm the component on the server has the same ETag as the cached copy.

## Caching static assets is easy, because static URLs can often change transparently

The golden rule of front-end performance is to minimize HTTP requests, so we use far-future Expires or Cache-Control:max-age headers aggressively. If we are willing to break URLs, we can set the cache far in the future; this is exactly what we did with the connect-cachify tool we saw last week:

  1. when a static asset is downloaded, it's given a far-future caching header (Expires or Cache-control:max-age)
  2. if the asset changes, its URL is changed, breaking the cache & causing the browser to re-download the new file

## Caching dynamic content is harder, because the rate of change is unpredictable and URLs must be stable

For dynamic content with persistent URLs, like templated web pages, we can't use the same approach:
  * we can't set far-future caching headers, unless we know exactly when the page will be updated in the future
  * we can't break the URL without hurting page discoverability, breaking bookmarks, and generally breaking the web

Although the browser has to check with the server that its cached copy is fresh, we can at least save the bandwidth and time required to redownload the file.

Because dynamic content might change at any time, we can't use the Last-Modified header; we have to use ETags to make dynamic content cacheable at all.

## ETags allow dynamic content to be cached using an app-specific "opaque token"

An ETag, or entity tag, is an opaque token that identifies a version of the component served by a particular URL. The token can be anything enclosed in quotes; often it's an md5 hash of the content, or the content's VCS version number. If you're dealing with internationalized templates, the ETag should be different for each localized version. In general, ETag implementations should respect variations in content usually specified with Vary headers:
  * Vary:Accept-Language is used to signal to browsers that different representations exist, and should be cached separately, depending on the value of the Accept-Language request header.
  * Vary:Cookie is used to signal that the same page, though it might be seen by anonymous and logged-in users, should be cached separately--otherwise, logged-in users would see the anonymous version until they force-refreshed their browsers.

## ETags and If-None-Match in action

Suppose the browser requests a page, and the server response includes an ETag in the header:

    request:
    GET /foobar HTTP/1.1
    Host: example.com

    response:
    HTTP/1.1 200 OK
    ETag: "0bba161a7165a211c7435c950ee78438"

Later on, when the browser loads the page again, it checks if its version is fresh by sending the ETag's token as an If-None-Match header:

    2nd request:
    GET /foobar HTTP/1.1
    Host: example.com
    If-None-Match: "0bba161a7165a211c7435c950ee78438"

If the browser's version is up-to-date, the server will return an empty 304 response:

    2nd response if unchanged:
    HTTP/1.1 304 Not Modified

But, if the browser's version is out of date, the server returns the new version with a 200 and a new ETag:

    2nd response if changed:
    HTTP/1.1 200 OK
    ETag: "bb547b3e79259ccc01960bd5779439c5"

Because the response only sometimes returns content, it's called a conditional GET. Conditional GETs are also possible using date-based validation, via the Last-Modified header. The behavior is the same as ETags/If-None-Match headers, except the browser returns the Last-Modified date as an If-Modified-Since request header:

    request:
    GET /foobar HTTP/1.1
    Host: example.com

    response:
    HTTP/1.1 200 OK
    Last-Modified: Tue, 15 Jan 2013 10:33:26 GMT

    2nd request:
    GET /foobar HTTP/1.1
    Host: example.com
    If-Modified-Since: Tue, 15 Jan 2013 10:33:26 GMT

    2nd response if unchanged since If-Modified-Since date:
    HTTP/1.1 304 Not Modified

    2nd response if changed since If-Modified-Since date:
    HTTP/1.1 200 OK
    Last-Modified: Mon, 10 Feb 2013 10:14:08 GMT

You can use either ETag or Last-Modified headers, or both, or neither; the HTTP 1.1 RFC actually recommends using both, in which case the server would only return a 304 if both the If-None-Match token and the If-Modified-Since date were fresh.

## ETags require some configuration to be helpful; otherwise, they can cause caching problems.

The original YSlow rules, and the book High Performance Web Sites, suggest disabling ETags unless you take the time to properly configure them. This is because Apache and IIS both have terrible default values for ETags, using server-specific node info or server-specific timestamps, so that the ETag set on a component is different for each node in a server farm. Since ETags provide comparatively little performance benefit in general (conditional GETs still require an HTTP request), it's often an improvement just to disable them.

## ETags have cool applications we can't get into atm.

ETags are more than just a caching header; they identify a version of a representation served at a URL. This leads to some cool applications we'll just mention in passing:
  * optimistic concurrency: if 2 authors try to update a shared document, or 2 nodes try to update the same RESTful endpoint, they can avoid clobbering others' edits by passing the last-seen ETag as an If-Match header in a conditional PUT or conditional DELETE. If the version on the server differs from the version the client has edited, then the client's edits shouldn't be allowed.
  * sub-second updates: if a firehose API endpoint or auction webpage changes multiple times per second, and gets lots of traffic, ETags save clients and server a ton of bandwidth, and allow clients to sync with the server continuously. Without ETags, the server would have to use no caching, and let clients redownload stale content, or use HTTP date-based caching, forcing clients to a latency of at least 1 full second between updates.
  * 304s in xhr responses: returning 304s from CRUD-style model endpoints can make short-polling real-time apps more efficient: although the app needs to be built to handle empty 304 responses, doing so can avoid the network and CPU cost (and potential UI lag) of re-downloading, re-JSONifying, and re-processing stale server input.
  * weak ETags can be used to issue partial Range requests of specific byte ranges. This is weird, wild stuff; if you need 206 Partial requests or the like, have fun digging into the RFC :-)

## connect-etagify is a simple, efficient library to add ETag support to internationalized templates within Connect apps.

The basic theme of this article is to suggest that you always quantify the benefits of the latest optimization trick or tactic, to avoid adding complexity without substantial return.

Here we'll look at a tiny library that throws ETags on connect endpoints without touching the underlying filesystem, so that it's unobtrusive in terms of code integration and server resource usage. 

Our ETag requirements for Persona were:
  * Enable caching on dynamically-generated content
  * respect internationalization via the Vary header
  * avoid wasteful IO caused by stat-ing the filesystem

connect-etagify was our answer to those requirements. You can jump straight to the code on github if you want to play; I'll walk through installing it and measuring its impact.

### connect-etagify: so easy to use, you'll forget it's there

#### theory of operation: md5 outgoing stuff, save it in an object in memory

connect-etagify is a middleware that generates ETags by md5 hashing outgoing responses & saving the hash in an in-memory JS object, not by stat-ing the filesystem to check file attributes. Because it works passively, it needs to generate an ETag on the first pass, and can serve it on the second pass. This is important to point out as it can have weird effects on dev testing if it's ignored.

#### installation & operation are ridiculously easy

  1. ```npm install etagify``` 
  2. Add this one-liner to your app: ```app.use(require('etagify'))```

#### statistical rundown of improvements

Need to throw this on an instance, measure + do some more stats. Smaller improvement => going to need more measurements.

## What about the ETag implementation in Connect?

Connect adds ETags to static, not dynamic, content. Hence, this sweet little thang.

## todo: refs :-)

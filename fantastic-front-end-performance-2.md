# Fantastic front-end performance, part 2: generate ETags on the fly with etagify


## How do you cache your internationalized templates?

You might know that [Connect](https://github.com/senchalabs/connect) puts [ETags](https://gist.github.com/6a68/4971859) on static content, but not dynamic content. Internationalized templates, even if they're static between releases, don't get caching headers at all--unless you add a build step to pregenerate all templates in all languages. What a lame chore.

This article introduces [etagify](https://github.com/lloyd/connect-etagify), a Connect middleware that generates ETags on the fly by md5-ing outgoing response bodies, storing the hashes in memory. Etagify lets you skip the build step, improves performance more than you might think (we measured a 9% load time improvement in our tests), and it's super easy to use:

### 1. register etagify at startup
    myapp = require('express').createServer();
    myapp.use(require('etagify')()); // <--- like this.

### 2. call etagify on routes you want to cache
    app.get('/about', function(req, res) {
      res.etagify();  // <--- bam.
      var body = ejs.render(template, options);
      res.send(body);
    });

Read on to learn more about etagify: how it works, when to use it, when not to use it, and how to measure your results.

### How etagify works

By focusing on a single, concrete use case, etagify gets the job done in just a hundred lines of code (including documentation). Let's take a look at the fifteen lines that cover the basics, leaving out edge cases around Vary header handling.

There are two parts to consider: hashing outgoing responses & caching the hashes; checking the cache against incoming conditional GETs.

First, here's where we add to the cache. Comments inline.

    // simplified etagify.js internals
    
    // start with an empty cache
    // example entry: 
    //   '/about': { md5: 'fa88257b77...' }
    var etags = {};

    var _end = res.end;
    res.end = function(body) {
      var hash = crypto.createHash('md5');

      // if the response has a body, hash it
      if (body) { hash.update(body); }

      // then add the item to the cache
      etags[req.path] = { md5: hash.digest('hex') };

      // back to our regularly-scheduled programming
      _end.apply(res, arguments);
    }

Next, here's how we check against the cache. Again, comments inline.

    var cached = etags[req.path]['md5'];
    
    // always add the ETag if we have it
    if (cached) { res.setHeader('ETag', '"' + cached + '"' }

    // if the browser sent a conditional GET,
    if (connect.utils.conditionalGET(req)) {

      // check if the If-None-Match and ETags are equal
      if (!connect.utils.modified(req, res)) {

        // yay cache hit! browser's version matches cached version.
        // strip out that ETag and bail with a 304 Not Modified.
        res.removeHeader('ETag');
        return connect.utils.notModified(res);        
      }
    }

### When NOT to use etagify

Etagify's approach is super simple, so it's a great solution for internationalizing templated pages that don't change while the server is running. However, other common cases wouldn't be a good fit:

* if pages change after being first cached, users will always see the stale, cached version
* if pages are personalized for each user, two things could happen:
  * if a Vary:cookie header is used to cache users' individual pages separately, then etagify's cache will grow without bound
  * if no Vary:cookie header is present, then the first version to enter the cache will be shown to all users

Note also that the default Apache and IIS configurations for ETags are [badly broken](http://developer.yahoo.com/performance/rules.html#etags). If you use either of these servers, be sure to disable their ETag support.

## Making reliable measurements

We didn't foresee huge performance wins with etagify, because conditional GETs still require an HTTP roundtrip, and avoiding page redownloading only saves the user a few KB (see screenshot). However, etagify is a really simple optimization, so even a small gain would justify including it in our stack.

![firebug screen cap showing 2kb savings](http://i.imgur.com/MVSQYKo.jpg)

The way we tested this was to spin up a dev instance of [Persona](https://github.com/mozilla/browserid) on an [awsbox](https://github.com/mozilla/awsbox), and repeatedly measure load time, both with and without etagify enabled. (Page load times are an adequate metric for our use case; you might care more about time till above-the-fold content renders, or the first content hits the page, or the first ad is displayed. As always, do what makes sense for your use case.)

After gathering numbers, we did some quick statistics to check the measurements were reliable. Without getting too stats-nerdy, we verified that the odds were good (at least 95%) that the observed improvement wasn't caused by random measurement fluctuations. (You can read about [standard deviations](http://en.wikipedia.org/wiki/Std_dev) and the [t-test](http://en.wikipedia.org/wiki/Student's_t-test) if you want to go deeper.)

### Surprise! 9% improvement

We opened up firebug, primed the cache, and reloaded Persona's 'about' page 50 times, with and without etagify enabled.

Surprisingly, we found that **etagify reduced load time by 9%**, from 1.65 (SD = 0.19) to 1.50 (SD = 0.13) seconds. That's a serious gain for almost no work.

Here's some graphs of the data:

### Without etagify: 1.65 sec load time (SD = 0.19)

![performance before adding etagify](http://i.imgur.com/PTE5AfP.png)

### With etagify: 1.50 sec load time, 9% improvement (SD = 0.13)

![significant improvement with etagify](http://i.imgur.com/cMxQvC2.png)

### Comparing the averages

Here's the data, modeled as bell curves (normal distribution):

![normal distributions with and without etagify](http://i.imgur.com/Tc45vHg.png)

## Bringing it all back home

We think cachify is a tiny bundle of win. Even if it's not the right tool for your current project, hopefully our approach of (1) writing focused tools to solve just the problem at hand, and (2) measuring just rigorously enough to be sure you're getting somewhere, gives you inspiration or food for thought. While we have a big core team (and we're hiring), we also love volunteer hackers. Hit us up if you want to help build a better web.

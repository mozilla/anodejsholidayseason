# Fantastic front-end performance, part 2: caching dynamically-generated content with etagify

You might know that [Connect](https://github.com/senchalabs/connect) puts [ETags](http://en.wikipedia.org/wiki/HTTP_ETag) on static content, but not dynamic content. Unfortunately, if you dynamically generate i18n versions of static pages, those pages don't get caching headers at all--unless you add a build step to pregenerate all pages in all languages. What a lame chore.

This article introduces [etagify](https://github.com/lloyd/connect-etagify), a Connect middleware that generates ETags on the fly by md5-ing outgoing response bodies, storing the hashes in memory. Etagify lets you skip the build step, improves performance more than you might think (we measured a 9% load time improvement in our tests), and it's super easy to use:

### 1. register etagify at startup
    myapp = require('express').createServer();
    myapp.use(require('etagify')()); // <--- like this.

### 2. call etagify on routes you want to cache
    app.get('/about', function(req, res) {
      res.etagify();  // <--- like that.
      var body = ejs.render(template, options);
      res.send(body);
    });

Read on to learn more about etagify: how it works, when to use it, when not to use it, and how to measure your results.

(Need a refresher on ETags and HTTP caching? We've put together a [cheat sheet](https://gist.github.com/6a68/4971859) to get you back up to speed.)

## How etagify works

By focusing on a single, concrete use case, etagify gets the job done in just a hundred lines of code (including documentation). Let's take a look at the fifteen lines that cover the basics, leaving out Vary header handling edge cases.

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

    // the etagify middleware
    return function(req, res, next) {
      var cached = etags[req.path]['md5'];
      
      // always add the ETag if we have it
      if (cached) { res.setHeader('ETag', '"' + cached + '"' }

      // if the browser sent a conditional GET,
      if (connect.utils.conditionalGET(req)) {

        // check if the If-None-Match and ETags are equal
        if (!connect.utils.modified(req, res)) {

          // cache hit! browser's version matches cached version.
          // strip out that ETag & bail with a 304 Not Modified.
          res.removeHeader('ETag');
          return connect.utils.notModified(res);        
        }
      }
    }

### When (and when not) to use etagify

Etagify's approach is super simple, and it's a great solution for dynamically-generated pages that don't change while the server is running, like i18n static pages. However, etagify has some gotchas when dealing with other common user cases:

* if pages change after being first cached, users will always see the stale, cached version
* if pages are personalized for each user, two things could happen:
  * if a Vary:cookie header is used to cache users' individual pages separately, then etagify's cache will grow without bound
  * if no Vary:cookie header is present, then the first version to enter the cache will be shown to all users

## Measuring performance improvements

We didn't foresee huge performance wins with etagify, because conditional GETs still require an HTTP roundtrip, and avoiding page redownloading only saves the user a few KB (see screenshot). However, etagify is a really simple optimization, so even a small gain would justify including it in our stack.

![firebug screen cap showing 2kb savings](http://i.imgur.com/MVSQYKo.jpg)

We tested etagify's effects on performance by spinning up a dev instance of [Persona](https://github.com/mozilla/browserid) on an [awsbox](https://github.com/mozilla/awsbox), opening up firebug, and taking 50 load time measurements of our 'about' page--with and without etagify enabled. (Page load times are a good-enough metric for our use case; you might care more about time till above-the-fold content renders, or the first content hits the page, or the first ad is displayed.)

After gathering raw data, we did some quick statistics to see how much etagify improved performance. We found the mean and standard deviation for both data sets, assuming the measured values were spread out like a [bell curve](http://en.wikipedia.org/wiki/Normal_distribution) around the averages.

Surprisingly, we found that **etagify reduced load time by 9%**, from 1.65 (SD = 0.19) to 1.50 (SD = 0.13) seconds. That's a serious gain for almost no work.

Next, we used the [t-test](http://en.wikipedia.org/wiki/Student%27s_t-test) to check the odds that the improvement could be observed without adding etagify at all. Our [p-value](http://en.wikipedia.org/wiki/P-value) was less than 0.01, meaning less than 1% chance that randomness could have caused the apparent improvement. We can conclude that the measured improvement is statistically significant.

Here's a chart of the averaged before and after data:

![normal distributions with and without etagify](http://i.imgur.com/Tc45vHg.png)

## Bringing it all back home

We think [etagify](https://github.com/lloyd/connect-etagify) is a tiny bundle of win. Even if it's not the right tool for your current project, hopefully our approach of (1) writing focused tools to solve just the problem at hand, and (2) measuring just rigorously enough to be sure you're getting somewhere, gives you inspiration or food for thought.

# Fantastic front-end performance Part 1 - Concatenate, Compress & Cache

In this part of our "A Node.JS Holiday Season" series we'll talk about front-end performance and introduce you to tools we've built and use in Mozilla to make the Persona front-end be as fast as possible.

We'll talk about connect-cachify[https://github.com/mozilla/connect-cachify/], a tool to automate some of the most important parts of front-end performance.

Before we do that, though, let's recap quickly what we can do as developers to make our solutions run on the machines of our users as smooth as possible. If you already know all about performance optimisations, feel free to proceed to the end and see how Connect-cachify helps automate some of the things you might do by hand right now.

## Three Cs of client side performance

The web is full of information related to performance best practices. While many advanced techniques exist to tweek every last millisecond from your site, three basic tools should form the foundation - concatenate, compress and cache.

## Concatenation

The goal of concatenation is to minimize the number of requests made to the server. Server requests are costly. The amount of time needed to establish an HTTP connection is sometimes more expensive than the amount of time necessary to transfer the data itself. Every request adds to the overhead that it takes to view your site and can be especially problematic on mobile devices where there is significant connection latency. Have you ever browsed to a shopping site on your mobile phone while connected to the Edge network and grimaced as each image loaded one by one? That is connection latency rearing its head.

SPDY[http://en.wikipedia.org/wiki/SPDY] is a new protocol built on top of HTTP that aims to reduce page load time by combining resource requests into a single HTTP connection. Unfortunately not all browsers support this new protocol.

The old fashioned approach of combining external resources wherever possible works everywhere and does not degrade with the advent of SPDY. Aside from HTML, most sites are composed of three types of external resources - Javascript, CSS and images. Tools exist to combine each of these.


### Javascript & CSS

A site with more than one script included in the HTML should consider combining their Javascript into a single file for production. Browsers have traditionally blocked all other rendering while Javascript is downloaded. Since each requested Javascript resource adds an amount of latancy to the overall download time, the following is slower than it needs to be:

```
  <script src="jquery.min.js"></script>
  <script src="main.js"></script>
  <script src="image-carousel.js"></script>
  <script src="widget.js"></script>
```
The browser will still block while it downloads the Javascript, but by combining four requests into one the total delay due to latancy will be signifcantly reduced.

```
  <script src="main.production.js"></script>
```

Note that this is usually done only in production mode. Trying to work with combined Javascript while still in development can be very difficult.

Like Javascript, individual CSS files should be combined into a single file for production. The process is the same.

### Images

Data URIs and image sprites are the two primary methods that exist to reduce the number of requested images.

#### data: URI

A data URI is a special form of a URL used to embed images directly into HTML or CSS. Data URIs can be used in either the src attribute of an img tag or as the url value for a background-image in CSS. Because embedded images are base64 encoded, they require more bytes than the original binary image. The increased size is usually offset by the reduction in the number of HTTP requests. Neither IE6 nor IE7 support data URIs so know your target audience before using them.

#### Image sprites

Image sprites are a great alternative whenever a data URI cannot be used. An image sprite is a collection of images combined into a single larger image. For each individual image, CSS is used to show the only relevant portion of the sprite. Many tools exist to create a sprite out of a collection of images.

A drawback to sprites is they can be difficult to maintain. The addition, removal or modification of an image within the sprite frequently requires a congruent change to the CSS as well.

http://www.spritecow.com/

## Removing extra bytes - minification, optimization & compression

Combining resources to minimize the number of HTTP requests goes a long way to speeding up a site, but we can still do more. Once resources are combined we should minimize the number of bytes that are transferred to the user. Minimizing bytes usually takes the form of minification, optimization and compression.

### Javascript & CSS

Javascript and CSS are text resources that can be effectively minified. Minification is a process that transforms the original text by eliminating anything that is irrelevent to the browser. In both Javascript and CSS the transformations include removing comments and extra whitespace. Javscript minification is much more complex. Some minifiers perform complex transforms that replace multi-character variable names with short variable names, remove language constructs that are not strictly necessary and even go so far as to replace entire statements with shorter equivalent statements.

UglifyJS[https://github.com/mishoo/UglifyJS], YUICompressor[http://developer.yahoo.com/yui/compressor/] and Google Closure Compiler[https://developers.google.com/closure/compiler/] are three popular tools to minify Javascript.

Two CSS minifiers include YUICompressor and UglifyCSS[https://github.com/fmarcia/UglifyCSS].

### Images

Images frequently contain data that can be removed without affecting its visual quality. Removing these extra bytes is not difficult, but does require specialized image handling tools. Our own Francois Marier has written blog posts on working with PNGs[http://feeding.cloud.geek.nz/2011/12/optimising-png-files.html] and GIFs[http://feeding.cloud.geek.nz/2009/10/reducing-website-bandwidth-usage.html].

Smush.it[http://www.smushit.com/ysmush.it/] from Yahoo! is an online optimization tool. ImageOptim[http://imageoptim.com/] is a useful OSX offline tool - simply drag and drop your images into the tool and it will reduce their size automatically. You don't need to do anything - ImageOptim simply replaces the original files with the much smaller ones.

If a loss of visual quality is acceptable, re-compressing an image at a higher compression level is an option.


### The Server Can Help Too!

Even after combining and minifying resources, there is more. Almost all servers and browsers support HTTP compression[http://en.wikipedia.org/wiki/HTTP_compression]. The two most popular compression schemes are deflate and gzip. Both of these make use of efficient compression algorithms to reduce the number of bytes before they ever leave the server.

## Caching

Concatenation and compression help first time visitors to our sites, caching on the other hand helps visitors that return. A user should only have to download a given static resources once. HTTP provides two widely adopted caching mechanisms, cache headers and ETags.

Cache headers come in two forms and are suitable for static resources that change infrequently, if ever. The two header options are `Expires` and `Cache-Control: max-age`. The `Expires` header specifies the date after which the resource must be re-requested. The `max-age` specifies how many seconds the resource is valid for. If a resource has a cache header, the browser will only re-request that resource once the cache expiration date has passed.

An ETag is essentially a resource version tag and is used to validate whether the local version of a resource is the same as the server's version. An ETag is suitable for dynamic content or content can change at any time. When a resource has an ETag, it says to the browser "Check the server to see if the version is the same, if it is, use the version you already have." Because an ETag requires interaction with the server, it is not as efficient as a fully cached resource.

### Cache-busting

The advantage to using time/date based cache-control headers instead of ETags is that resources are only re-requested once the cache has expired. This is also its biggest drawback. What happens if a resource changes? The cache has to somehow be busted.

Cache-busting is usually done by adding a version number to the resource URL. Any change to a resources URL causes a cache-miss which in turn causes the resource to be re-downloaded.

For example if http://example.com/logo.png has a cache header set to expire in one year but the logo changes, users who have already downloaded the logo will only see the update a year from now. This can be fixed by adding some sort of version identifier to the URL.

```
  http://example.com/v8125/logo.png
```

or

```
  http://example.com/logo.png?v8125
```

When the logo is updated, a new version is used meaning the logo will be re-requested.

```
  http://example.com/v8126/logo.png
```

or

```
  http://example.com/logo.png?v8126
```

## Connect-cachify - A NodeJS library to serve concatinated and cached resources

Connect-cachify[https://github.com/mozilla/connect-cachify/] is a NodeJS library developed by Mozilla that makes it simple to serve concatinated and cached resources. Concatenated resources with far future cache headers are served in production mode while individual resources without cache headers are served in development mode.

Connect-cachify does not perform concatenation itself but instead relies on you to do this in your project's build script. Connect-cachify takes a map of production to development resources and generates the appropriate `script` or `link` tags for the environment.

Servers in development mode will send individual resources, allowing developers to easily debug. A server in production mode will send pre-generated production resources and set the proper cache headers.


Example initialization of connect-cachify:

```
...
const cachify = require('connect-cachify');

var assets = {
  "/js/main.min.js": [
    '/js/lib/jquery.js',
    '/js/magick.js',
    '/js/laughter.js'
  ],
  "/css/dashboard.min.css": [
    '/css/reset.css',
    '/css/common.css',
    '/css/dashboard.css'
  ]
};

app.use(cachify.setup(assets, {
  root: __dirname,
  production: your_config['use_minified_assets'],
}));
...
```

To keep code DRY, the asset map can be externalized into its own file and used as configuration to both connect-cachify and your build script. Finally, templates need to be updated to make use of cachify.

EJS templates that previously looked like this:
```
...
<head>
  <title>Dashboard: Hamsters of North America</title>
  <link rel="stylesheet" type="text/css" href="/css/reset.css" />
  <link rel="stylesheet" type="text/css" href="/css/common.css" />
  <link rel="stylesheet" type="text/css" href="/css/dashboard.css" />
</head>
<body>
  ...
  <script type="text/javascript" src="/js/lib/jquery.js"></script>
  <script type="text/javascript" src="/js/magick.js"></script>
  <script type="text/javascript" src="/js/laughter.js"></script>
</body>
...
```

Will now look like this:

```
...
<head>
  <title>Dashboard: Hamsters of North America</title>
  <%- cachify_css('/css/dashboard.min.css') %>
</head>
<body>
  ...
  <%- cachify_js('/js/main.min.js') %>
</body>
...
```

While in production mode, cachify will set far future cache headers of all cachified resources. Cache busting is built in by inserting the MD5 hash of a resource into its URL. Whenever the resource is updated, cachify updates the MD5 hash, propagating any changes out to the user.

The cachified template will be rendered as:

```
...
<head>
  <title>Dashboard: Hamsters of North America</title>
  <link rel="stylesheet" type="text/css" href="/v/2abdd020a6/css/dashboard.min.css" />
</head>
<body>
  ...
  <script type="text/javascript" src="/v/acdd2ab372/js/main.min.js"></script>
</body>
...
```

## Conclusion
There are a lot of easy wins when looking to speed up a site. By going back to basics and using the three Cs - concatenation, compression and caching - you will go a long way towards improving the load time of your site and the experience for your users.

In the next installment of A NodeJS Holiday Season, we will look at how to generate dynamic resources and make use of ETagify to still serve maximally cacheable resources.

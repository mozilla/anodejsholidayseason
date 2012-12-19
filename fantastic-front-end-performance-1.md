# Fantastic front-end performance Part 1 - Concatenate, Compress & Cache

In this part of our "A Node.JS Holiday Season" series we'll talk about front-end performance and introduce you to tools we've build and use in Mozilla to make the Persona front-end be as fast as possible. 

We'll talk about Connect-cachify[https://github.com/mozilla/connect-cachify/], a tool to automate some of the most important parts of front-end performance. 

Before we do that, though, let's recap quickly what we can do as developers to make our solutions running on the machines of our users as smoothly is possible. If you already know all about performance optimisations, feel free to proceed to the end and see how Connect-cachify helps automating some of the things you might do by hand right now.

##Three Cs of client side performance

The web is full of information related to performance best practices. There are many techniques to fine tune and crank up the performance of your site, today we are going to go back to basics and focus on the three building blocks to make your site fast - concatenate, compress and cache. Connect-cachify is a NodeJS library developed by Mozilla that makes two of these three easy.

## Concatenation

The goal of concatenation is to minimize the number of requests made to the server. Server requests are costly. The amount of time needed to establish an HTTP connection is sometimes more expensive than the amount of time necessary to transfer the data itself. Every request adds to the overhead that it takes to view your site and can be especially problematic on mobile devices where there is significant connection latency. Have you ever browsed to a shopping site on your mobile phone while connected to the Edge network and grimaced as each image loaded one by one? That is connection latency rearing its head.

SPDY[http://en.wikipedia.org/wiki/SPDY] is a new protocol built on top of HTTP that aims to reduce web page load time by combining resource requests into a single HTTP connection.

It is easy to mitigate connection latency by combining resources wherever possible. Aside from HTML, most sites are primarily composed of Javascript, CSS and images. Luckily, tools exist to combine each of these.

### Javascript & CSS

Sites with more than one script downloaded at the same time should consider concatenating their Javascript into a single file for production. Trying to combine Javascript resources while still in development makes debugging difficult, so this is usually done in a build stage before deployment.

Example:

  <script src="jquery.min.js"></script>
  <script src="main.js"></script>
  <script src="image-carousel.js"></script>
  <script src="widget.js"></script>

A build script would take these four scripts and generate a single output file with the resources combined in order. Assuming the build script creates an output file named main.production.js, four scripts can be replaced with one:

  <script src="main.production.js"></script>

Like Javascript, individual CSS files should be combined into a single file for production. The process is the same.

### Images

There are two primary methods to reduce the number of requested images - using a data URI to inline an image or combining images into an image sprite.

#### data: URI

A data URI is a special value URI where the image data is encoded and embedded directly in HTML or CSS. Data URIs can be used in either the src attribute of an img tag or as the url value for a background-image in CSS.

Data URIs are base64 encoded and the resulting image data will require more bytes than the original binary image, but will require one less HTTP request. Data URIs are not supported in IEs 6 and 7, so know your target audience before using them. 

#### Image sprites

Image sprites are a great alternative whenever a data URI cannot be used. An image sprite is a collection of images combined into a single larger image. For each individual image, CSS is used to show the only relevant portion of the sprite. Many tools exist to create a sprite out of a collection of images.

One major drawback to sprites is they can be difficult to maintain. The addition, removal or modification of an image within the sprite frequently requires a congruent change in the CSS as well.

http://css-sprit.es/
http://www.spritecow.com/

## Removing extra bytes - minification & compression

Combining resources to minimize the number of HTTP requests goes a long way to speeding up our site, but we can still do better. Once our resources are combined, we can go a step further and minimize the number of bytes that are transferred to the user.

This usually takes the form of minification and compression of text resources and optimizing images to remove extra bytes.

### Javascript & CSS

Javascript and CSS are text resources that can be effectively minified. Minification is a process that transforms the original text by eliminating anything that is irrelevent to the browser. In both Javascript and CSS the transformation includes removing comments and extra whitespace. In Javascript, minification often involves complex transforms that include replacing variable names with shorter variable names, and removing or replacing language constructs that are not strictly necessary.

UglifyCSS and YUI Compressor

UglifyJS, YUICompressor and Google Closure Compiler are three popular tools to minify Javascript

### Images

Images frequently contain data that can be removed without affecting its visual quality. Removing these extra bytes is not difficult, but does require specialized image handling tools. Our own Francois Marier has written several blog posts on working with PNGs.

http://feeding.cloud.geek.nz/2011/12/optimising-png-files.html

A very useful offline tool for this is ImageOptim[http://imageoptim.com/] - simply drag and drop your images into the tool and it will reduce their size automatically. You don't need to do anything - imageOptim simply replaces the original files with the much smaller ones.

If a loss of visual quality is acceptable, re-compressing an image at a higher compression level is an option.


### Server Level

Even after combining and minifying resources, there is more we can do to reduce the number of bytes served to the user. Almost all servers and browsers support HTTP compression. The two most popular compression schemes are deflate and gzip. In both of these, the server uses efficient compression algorithms to reduce the number of bytes sent to the user.

http://en.wikipedia.org/wiki/HTTP_compression

## Caching

Once we have concatenated and compressed our resources, there is still more work to do. A user who downloads a resource once should not have to download the same resource again. This is where HTTP caching comes into play.

There are two HTTP caching mechanisms, cache headers and ETags.

Cache headers are suitable for static resources that change infrequently, if ever. If a resource has a cache header, the browser will only re-request that resource once the cache expiration date has passed. Cache headers come in two forms, `Expires` and `Cache-Control: max-age`.

The `Expires` header specifies the date after which the resource must be re-requested. The `max-age` specifies how many seconds the resource is valid for.

An ETag is essentially a resource version tag and is used to validate whether the local version of a resource is the same as the version on the server. An ETag is suitable for dynamic content or content can change at any time. When a resource has an ETag, it says to the browser "Check the server to see if the version is the same, if it is, use the version you already have." Because an ETag requires interaction with the server, it is not as efficient as a fully cached resource.

### Cache-busting

The advantage to using cache-control headers is that resources are only re-requested once the cache has expired. This is also its biggest drawback. What happens if a resource expires? The cache has to somehow be busted.

This is usually done by adding a version number to the resource URL. Any change to a resources URL causes a cache-miss which in turn causes the resource to be re-downloaded.

For example if http://example.com/logo.png has a cache header set to expire in one year but the logo changes, users who have already downloaded the logo only see the update a year from now. This can be fixed by adding some sort of version identifier into the URL.

  http://example.com/v8125/logo.png

or

  http://example.com/logo.png?v8125

When the logo is updated, a new version is used meaning the logo will be re-requested.

  http://example.com/v8126/logo.png

or

  http://example.com/logo.png?v8126

## Connect-Cachify

Connect-cachify[https://github.com/mozilla/connect-cachify/] is a NodeJS middleware developed by the fine folks in Mozilla's Identity group that helps with two of the three Cs - concatenation and caching. Connect-cachify will serve up concatenated resources with far future cache headers in production mode and individual resources without cache headers in development mode.

Cachify does not perform concatenation directly but instead relies on the output of your project's build script. Cachify takes a map of production to development resources and generates the appropriate `script` or `link` tags for the environment.

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

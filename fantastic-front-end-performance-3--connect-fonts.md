# Fantastic front end performance, part 3 - Serving smaller fonts for big performance wins - A Node.js holiday season, part 8

We reduced Persona's font footprint 85%, from 300 KB to 45 KB, using font subsetting. This post outlines exactly how we implemented these performance improvements, and gives you tools to do the same.

## Introducing connect-fonts

```connect-fonts``` is a Connect font-management middleware that improves ```@font-face``` performance by serving locale-specific subsetted font files, significantly reducing downloaded font size. It also generates locale/browser-specific ```@font-face``` CSS, and manages the CORS header required by Firefox and IE 9+. Subsetted fonts are served from a *font pack*--a directory tree of font subsets, plus a simple JSON config file. Some common open-source fonts are available in pregenerated font packs [on npm](https://npmjs.org/browse/keyword/connect-fonts), and creating your own font packs is straightforward.

(Feeling lost? We've [put together](https://gist.github.com/6a68/5187976) a few references to good ```@font-face``` resources on the web.)

### Static vs dynamic font loading

When you are just serving one big font to all your users, there's not much involved in getting web fonts set up:
  * generate ```@font-face``` CSS and insert into your existing CSS
  * generate the full family of web fonts from your TTF or OTF file, then put them someplace the web server can access
  * add CORS headers to your web server if fonts are served from a separate domain, as Firefox and IE9+ enforce the same origin policy with fonts

These steps are pretty easy; the awesome [FontSquirrel generator](http://www.fontsquirrel.com/tools/webfont-generator) can generate all the missing font files and the ```@font-face``` CSS declaration for you. You've still got to sit down with Nginx or Apache docs to figure out how to add the CORS header, but that's not too tough.

If you want to take advantage of font subsetting to hugely improve performance, things become more complex. You'll have font files for each supported locale, and will need to dynamically modify the ```@font-face``` CSS declaration to point at the right URL. CORS management is still needed. This is the problem ```connect-fonts``` solves.

### Font subsetting: overview

By default, font files contain lots of characters: the Latin character set familiar to English speakers; accents and accented characters added to the Latin charset for languages like French and German; additional alphabets like Cyrillic or Greek. Some fonts also contain lots of funny symbols, particularly if they support Unicode ([☃](http://en.wikipedia.org/wiki/Snowman#Unicode) anyone?). Some fonts additionally support East Asian languages. Font files contain all of this so that they can ably serve as many audiences as possible. All this flexibility leads to large file sizes; Microsoft Arial Unicode, which has characters for every language and symbol in Unicode 2.1, weighs in at an unbelievable 22 megabytes.

In contrast, a typical web page only needs a font to do one specific job: display the page's content, usually in just one language, and usually without exotic symbols. By reducing the served font file to just the subset we need, we can shave off a ton of page weight.

### Performance gains from font subsetting

Let's compare the size of the localized font files vs the full file size for some common fonts and a few locales. Even if you just serve an English-language site, you can shave off a ton of bytes by serving an English subset.

Smaller fonts mean faster load time and a shorter wait for styled text to appear on screen. This is particularly important if you want to use ```@font-face``` on mobile; if your users happen to be on a 2G network, saving 50KB can speed up load time by 2-3 seconds. Another consideration: mobile caches are small, and subsetted fonts have a far better chance of staying in cache.

#### Open Sans regular, size of full font (default) and several subsets (KB):

![Chart comparing file sizes of Open Sans subsets. Full font, 104 KB. Cyrillic, 59 KB. Latin, 29 KB. German, 22 KB. English, 20 KB. French, 24 KB.](https://gist.github.com/6a68/5122048/raw/23d41ed71f079adb21656518f85e262a0fccaade/unzipped.png)

#### Same fonts, gzipped (KB):

![Chart comparing file sizes of Open Sans subsets when gzipped. Full font, 63 KB. Cyrillic, 36 KB. Latin, 19 KB. German, 14 KB. English, 13 KB. French, 15 KB.](https://gist.github.com/6a68/5122048/raw/3f5d6cce8fe1db2721583350dbd01224dd1feb30/gzipped.png)

Even after gzipping, you can reduce font size 80% by using the English subset of Open Sans (13 KB), instead of the full font (63 KB). Consider that this is the reduction for just one font file--most sites use several. The potential is huge!

**Using ```connect-fonts```, Mozilla Persona's font footprint went from 300 KB to 45 KB, an 85% reduction.** This equates to several seconds of download time on a typical 3G connection, and up to 10 seconds on a typical 2G connection.

### Going further with optimizations

If you're looking to tweak every last byte and HTTP request, ```connect-fonts``` can be configured to return generated CSS as a string instead of a separate file. Going even further, ```connect-fonts```, by default, serves up the smallest possible @font-face declaration, omitting declarations for filetypes not accepted by a given browser.

## Example: adding connect-fonts to an app

Suppose you've got a super simple express app that serves up the current time:

    // app.js
    const
    ejs = require('ejs'),
    express = require('express'),
    fs = require('fs');

    var app = express.createServer(),
      tpl = fs.readFileSync(__dirname, '/tpl.ejs', 'utf8');

    app.get('/time', function(req, res) {
      var output = ejs.render(tpl, {
        currentTime: new Date()
      });
      res.send(output);
    });

    app.listen(8765, '127.0.0.1');

with a super simple template:

    // tpl.ejs
    <!doctype html>
    <p>the time is <%= currentTime %>.

Let's walk through the process of adding ```connect-fonts``` to serve the Open Sans font, one of [several](https://npmjs.org/browse/keyword/connect-fonts) ready-made font packs.

### App changes

1. Install via npm:

        $ npm install connect-fonts
        $ npm install connect-fonts-opensans

2. Require the middleware:

        // app.js - updated to use connect-fonts
        const
        ejs = require('ejs'),
        express = require('express'),
        fs = require('fs'),
        // add requires:
        connect_fonts = require('connect-fonts'),
        opensans = require('connect-fonts-opensans');
    
        var app = express.createServer(),
          tpl = fs.readFileSync(__dirname, '/tpl.ejs', 'utf8');

3. Initialize the middleware:

         // app.js continued
         // add this app.use call:
         app.use(connect_fonts.setup({
           fonts: [opensans],
           allow_origin: 'http://localhost:8765'
         })
The arguments to ```connect_fonts.setup()``` include:
  * ```fonts```: an array of fonts to enable,
  * ```allow_origin```: the origin for which we serve fonts; ```connect-fonts``` uses this info to set the Access-Control-Allow-Origin header for browsers that need it (Firefox 3.5+, IE 9+)
  * ```ua``` (optional): a parameter listing the user-agents to which we'll serve fonts. By default, ```connect-fonts``` uses UA sniffing to only serve browsers font formats they can parse, reducing CSS size. ```ua: 'all'``` overrides this to serve all fonts to all browsers.

4. Inside your route, pass the user's locale to the template:

        // app.js continued
        app.get('/time', function(req, res) {
          var output = ejs.render(tpl, {
            // pass the user's locale to the template
            userLocale: detectLocale(req),
            currentTime: new Date()
          });
          res.send(output);
        });
5. Detect the user's preferred language. Mozilla Persona uses [i18n-abide](https://github.com/mozilla/i18n-abide), and [locale](https://github.com/jed/locale) is another swell option; both are available via npm. For the sake of keeping this example short, we'll just grab the first two chars from the [Accept-Language header](https://developer.mozilla.org/en-US/docs/HTTP/Content_negotiation#The_Accept-Language.3A_header):

        // oversimplified locale detection
        function detectLocale(req) {
          return req.headers['accept-language'].slice(0,2);
        }

        app.listen(8765, '127.0.0.1');
        // end of app.js

### Template changes

Now we need to update the template. ```connect-fonts``` assumes routes are of the form

        /:locale/:font-list/fonts.css
for example,

        /fr/opensans-regular,opensans-italics/fonts.css
In our case, we'll need to:

5. add a stylesheet ```<link>``` to the template matching the route expected by ```connect-fonts```:

        // tpl.ejs - updated to use connect-fonts
        <!doctype html>
        <link href="/<%= userLocale %>/opensans-regular/fonts.css" rel="stylesheet">

6. Update the page style to use the new font, and we're done!

        // tpl.ejs continued
        <style>
          body { font-family: "Open Sans", "sans-serif"; }
        </style>
        <p>the time is <%= currentTime %>.

The CSS generated by ```connect-fonts``` is based on the user's locale and browser. Here's an example for the 'en' localized subset of Open Sans:

    // this is the output with the middleware's ua param set to 'all'.
    @font-face {
      font-family: 'Open Sans';
      font-style: normal;
      font-weight: 400;
      src: url('/fonts/en/opensans-regular.eot');
      src: local('Open Sans'),
           local('OpenSans'),
           url('/fonts/en/opensans-regular.eot#') format('embedded-opentype'),
           url('/fonts/en/opensans-regular.woff') format('woff'),
           url('/fonts/en/opensans-regular.ttf') format('truetype'),
           url('/fonts/en/opensans-regular.svg#Open Sans') format('svg');
    }

If you don't want to incur the cost of an extra HTTP request, you can use the ```connect_fonts.generate_css()``` method to return this CSS as a string, then insert it into your existing CSS files as part of a build/minification process.

That does it! Our little app is serving up stylish timestamps. This example code is available on [github](https://github.com/6a68/connect-fonts-example) and [npm](https://npmjs.org/package/connect-fonts-example) if you want to play with it.

We've covered a pre-made font pack, but it's easy to create your own font packs for paid fonts. There are instructions on the ```connect-fonts``` [readme](https://github.com/shane-tomlinson/connect-fonts).

## Wrapping up

Font subsetting can bring huge performance gains to sites that use web fonts; ```connect-fonts``` handles a lot of the complexity if you self-host fonts in an internationalized Connect app.  If your site isn't yet internationalized, you can still use ```connect-fonts``` to serve up your native subset, and it'll still generate ```@font-face``` CSS and any necessary CORS headers for you, plus you'll have a smooth upgrade path to internationalize later.

### Future directions

Today, ```connect-fonts``` handles subsetting based on locale. What if it also stripped out font hinting for platforms that don't need it (everything other than Windows)? What if it also optionally gzipped fonts and added far-future caching headers? There's cool work yet to be done. If you'd like to contribute ideas or code, we'd love the help! Grab the [source](https://github.com/shane-tomlinson/connect-fonts) and dive in.

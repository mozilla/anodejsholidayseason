# Localization in action

## Using Our Strings

So [first we added the i18n-abide module to our code](https://hacks.mozilla.org/2013/04/localize-your-node-js-service-part-1-of-3-a-node-js-holiday-season-part-9/), then our [L10n team did some string wrangling](https://hacks.mozilla.org/2013/04/localization-community-tools-process-part-2-of-3-a-node-js-holiday-season-part-10/), now we've got several locales with translated strings...

Let's get these strings ready for Node.js and see this puppy in action!

The next step is that we'll need your PO files, typically in a file system like this:

    locale
      en
        LC_MESSAGES
          messages.po
      de
        LC_MESSAGES
          messages.po
      el
        LC_MESSAGES
          messages.po

We need a way to get strings from our PO files into our application at runtime. There are a few ways you can do this.

The first way, is to have **server side strings** and the `gettext` function provided by i18n-abide will work it's magic.

The second way, is to have **client side strings** and you'll include a gettext.js script in your code.
This is distributed with i18n-abide.

**Both of these methods require** the strings to be in a **JSON** file format.
The server side translation loads them on application startup, and the client side translation loads them via HTTP (or you can put them into your built and minified JavaScript).


Since this system is compatible with GNU Gettext, a third option for server side strings is to use [node-gettext](https://github.com/andris9/node-gettext). It's quite efficient for doing server side translation.

We'll use the first option in this blog post, as it is the most common way to use i18n-abide.

### compile-json

So, how do we get our strings out of the PO files and into JSON files?

Our build script is called `compile-json`.

Assuming out files are in a top level directory `locale` of our project, and we want the `.json` files to go into `static/i18n`, we'd do this:

Example:

    $ mkdir -p static/i18n
    $ ./node_modules/.bin/compile-json locale static/i18n

And we get a file structure like:

    static
      i18n
        en
          messages.json
          messages.js
        de
          messages.json
          messages.js
        el
          messages.json
          messages.js

`compile-json` loops over each of our `.po` files and calls `po2json.js` on it, producing `.json` and `.js` files. `po2json.js` is another program provided by i18n-abide.

If we take the messages.po we have so far from these blog posts, we'd see:

    # Spanish translations for PACKAGE package.
    # Copyright (C) 2013 THE PACKAGE'S COPYRIGHT HOLDER
    # This file is distributed under the same license as the PACKAGE package.
    # Austin King <ozten@localhost>, 2013.
    #
    msgid ""
    msgstr ""
    "Project-Id-Version: PACKAGE VERSION\n"
    "Report-Msgid-Bugs-To: \n"
    "POT-Creation-Date: 2012-06-24 09:50+0200\n"
    "PO-Revision-Date: 2013-04-24 16:42-0700\n"
    "Last-Translator: Austin King <ozten@nutria.localdomain>\n"
    "Language-Team: Spanish\n"
    "Language: es\n"
    "MIME-Version: 1.0\n"
    "Content-Type: text/plain; charset=UTF-8\n"
    "Content-Transfer-Encoding: 8bit\n"
    "Plural-Forms: nplurals=2; plural=(n != 1);\n"

    #: /home/ozten/abide-demo/views/homepage.ejs:3
    msgid "Mozilla Persona"
    msgstr "Mozilla Personidada"

which would be converted into

    {
       "messages": {
          "": {
             "Project-Id-Version": " PACKAGE VERSION\nReport-Msgid-Bugs-To: \nPOT-Creation-Date: 2012-06-24 09:50+0200\nPO-Revision-Date: 2013-04-24 16:42-0700\nLast-Translator: Austin King <ozten@nutria.localdomain>\nLanguage-Team: German\nLanguage: de\nMIME-Version: 1.0\nContent-Type: text/plain; charset=UTF-8\nContent-Transfer-Encoding: 8bit\nPlural-Forms: nplurals=2; plural=(n != 1);\n"
          },
          "Mozilla Persona": [
             null,
             ""
          ]
       }
    }

[EDITOR NOTE - There is a bug in i18n-abide here, which I will fix.]

So we can use these .json files server side form Node code, or client side by requesting them via AJAX.

The `static` directory is exposed to web traffic, so a request to `/i18n/el/messages.json` would get the Greek JSON file.

This `static` directory is an express convention, you can store these files where ever you wish. You can serve up static files this via Node.js or a web server such as `nginx`.

**Note:** You **don't need** the .PO files **to be deployed to production**, but it doesn't hurt to ship them.

## Configuration

`i18n-abide` requires some configuration to decide which languages are supported and to know where to find our JSON files.

As we saw [in the first installment](https://hacks.mozilla.org/2013/04/localize-your-node-js-service-part-1-of-3-a-node-js-holiday-season-part-9/#language-detection), here is the required configuration for our application

    app.use(i18n.abide({
      supported_languages: ['en-US', 'de', 'el', 'zh-TW'],
      default_lang: 'en-US',
      translation_directory: 'static/i18n'
    }));

`supported_languages` tells the app that it supports English, German, Greek, Chinese (Traditional).
The `translation_directory` config says that the translated JSON files are under static/i18n.
Note that `translation_directory` is needed for **server side** gettext only.

We explained in the first post that i18n-abide will do it's best to serve up an appropriate localized string.

It will look at `supported_languages` in the configuration to find the best language match.

You should only put languages in `supported_languages`, where you have a locale JSON file ready to go.

## Start you engines

Okay, now that configs are in place and we have at least one locale translated, let's fire it up!

    npm start

In your web browser, change your preferred language to one which you have localized.

![](lang_selection.png)

Now load a page for your application. You should see it translated now.

Here is Mozilla Persona in **Greek**. So, cool!

![](dialog-greek.png)

### gobbledygook

If you want to **test** your L10n setup, **before you have real translations** done, we're built a great test locale.
It is inspired by [David Bowie's Labyrinth](https://www.youtube.com/watch?v=rJLnGjhPT1Q).

To use it, just add `it-CH` or another locale you're not currently using to your config under **both `supported_languages`** as well as the **debug_lang** setting.

Example *partial* config:

    app.use(i18n.abide({
      supported_languages: ['en-US', 'de', 'el', 'zh-TW', 'it-CH'],
      debug_lang: 'it-CH',
      ...

Now if you set your browser's preferred language to Italian/Switzerland (it-CH), i18n-abide will use gobbledygook to localize the content.

![](it-CH-chooser.png)

This is a handy way to ensure your visual design and prose work for bi-directional languages like Hebrew.
Your web designer can test their RTL CSS, before you have the resources to create actual Hebrew strings.

![](db-LB.png)

## Going Deeper
We've just scratched the surface of i18n and l10n.
If you ship a Node.js based service in multiple locales, you'll find many gotchas and interesting nuances.

Here is a heads up on a few more topics.

### String interpolation

i18n-abide provides a `format` function which can be used in client or server side JavaScript code.

Format takes a formatted string and replaces parameters with actual values at runtime.
This function can be used in one of two flavors of parameter replacements.

Formats
* %s - `format` is called with a format string and then an array of strings. Each will be replaced in order.
* %(named)s - `format` is called with a format string and then an object where the keys match the named parameters.

You can use `format` to keep HTML in your strings to a minimum.

Consider these three examples

    <%= gettext('<p>Buy <a href="/buy?prod=blue&tyep=ticket">Blue Tickets</a> Now!</p>') %>
    <p><%= format(gettext('Buy <a href="%s">Blue Tickets</a> Now!', ['/buy?prod=blue&tyep=ticket'])) %></p>
    <p><%= format(gettext('Buy <a href="%(url)s">Blue Tickets</a> Now!', {url: '/buy?prod=blue&tyep=ticket'})) %></p>

In the PO file, they produce these strings:

    msgid "<p>Buy <a href=\"/buy?prod=blue&tyep=ticket\">Blue Tickets</a> Now!</p>"
    msgid "Buy <a href="%s">Blue Tickets</a> Now!"
    msgid "Buy <a href="%(url)s">Blue Tickets</a> Now!"

The first example has a paragraph tag that shows up in the PO file. Yuck.
If you ever change the markup... you may have to update it in every locale!

Also, look at that ugly URL.

Reasons to use `format`:
* Avoid confusing localizers who aren't familiar with HTML and may break your code accidentally
* Avoid maintenance issues

The named parameters are nice, in that they are self documenting.
The localizer knows that the variable is a URL.

String interpolation is quite common in localizing software.

Another example is runtime data injected into your strings.

    <p><%= format(gettext('Welcome back, %(user_name)s'), {user_name: user.name}) %></p>

## Avoid Inflexible Designs

We need to put our L10n hats on early.
As early as when we review the initial graphic design of the website.

Avoid putting text into images. Use CSS to keep words as plain text positioned over images.

Make sure [CSS is bulletproof](http://simplebits.com/publications/bulletproof/). An English word in German can be many times larger and destroy a
poorly planned design.

Try this bookmarklet: <a href="javascript:(function(){var%20prefixes=['','glocken','das','borfa','maushe','uber'],suffixes=['','hausen','%20die%20vander','gleuten','noshan','flagellan','mek','dak','en%20das','ga'],xPathResult=document.evaluate('.//text()[normalize-space(.)!=\'\']',document.body,null,XPathResult.ORDERED_NODE_SNAPSHOT_TYPE,null),i,textNode,cnt,out,j,pfx,sfx;for(i=0,l=xPathResult.snapshotLength;i<l;i++){textNode=xPathResult.snapshotItem(i);if(textNode.parentNode.nodeName.toLowerCase()=='script'||textNode.tagName=='style')continue;cnt=textNode.data.split(/\s/g);out=[];for(j=0;j<cnt.length;j++){if(cnt[j].replace(/[\s]/g,'')=='')continue;pfx=(!Math.floor(Math.random()*10))?'':prefixes[Math.floor(Math.random()*prefixes.length)];sfx=(!Math.floor(Math.random()*10))?'':suffixes[Math.floor(Math.random()*suffixes.length)];out.push(pfx+cnt[j]+sfx);}textNode.data='%20'+out.join('%20')+'%20';}})();)">Fauxgermanhausen das Pagen!</a>

Database backed websites have already taught us to think about systematic design way, but designers may not be used to allowing for variable length labels or buttons.

Overly "tight" or clever designs simply will not work in a localized context.


## String Freeze

Remember our build step to prepare files for localizers to translate?
And in this post we learned about `po2json` for using these strings in our app...
Well, this means we're going to need to coordinate our software releases with our L10n community.

Continuous deployment isn't a solved problem with L10n. Either you have to block on getting 100%
of your strings translated before deploying, or be comfortable with a partially translated app in some locales.

L10n teams may need 1, 2 or even 3 weeks to localize your app, depending on how many strings there are.
Schedule this to happen during the QA cycle.

Provide a live preview site, so that localizers can check their work.

You should build a string freeze into your project schedule.

## Wrapping up

In these three blog posts, we've seen how to develop a localized app with `i18n-abide`.
How to add a L10n phase to our release build.
And lastly, how to test our work.

Localizing your website or application will make your site valuable to an even larger global audience.
Node.js hackers, go make your services accessible to a larger audience!
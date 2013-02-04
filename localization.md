
Outline

1) i18n-abide, gettext
tutorial style walkthrough of i18n-abide ifying a site?

2) community and tools

jsxgettext
po2json
history

3) Deep

middleware - accept language matching
format
gobbdegook
skipping po files

4) Beyond Node

images
BIDI

------------------



# How to Localize Your Node.js service

## What is localization?

The products and services which Mozilla provides are localized into as many as 90 languages! Localizing a website or native app involves translating copy, adding/removing/tweaking components to fit with cultural norms, as well as matching the technical differences of region or language.

The following are just a few examples of localization:
* Providing copy translated into a specific regional variation of a language
* Rendering a screen right to left for a given language
* Bullet proofing designs to accomodate more or less text
* Naming things or providing graphics that resonate with a local audience

In this series of posts, I'm going to cover some technical aspects of how to localize a Node.js service, as well as some more general techniques that work regardless of the technology.

I'll be using the terms L10n (<b>L</b>ocalizatio<b>n</b>) and <b>I</b>nternationalizatio<b>n</b>, becuase we love our acronyms, right?

## i18n-abide

The Mozilla Persona service is a Node.js based service localized into X locales. To accomplish this, it uses the following modules

* i18n-abide
* jsxgettext
* gobbledygook

i18n-abide is the most important module from a developer's perspective. Let's walk through adding it to your service.

For the sake of examples, we'll assume an Express website and EJS templates.

## Installation

    npm install i18n-abide

In your code

    var i18n = require('i18n-abide');

    app.use(i18n.abide({
      supported_languages: ['en-US', 'de', 'es', 'zh-TW', 'it-CH'],
      default_lang: 'en-US',
      debug_lang: 'it-CH',
      locale_directory: 'locale'
    }));
[TODO: locale_directory is this needed with .json files?]

The i18n `abide` middleware sets up request processing and injects various functions we'll use for translation.

The key thing abide does, is it injects into the Node and express framework references to `Gettext` functions. This allows you to reference gettext strings in Node JavaScript code or in your HTML templates.

Here is an example in a template file:

    <html lang="{{LANG}}" dir="{{DIR}}">
      <head>
        <title>{{gettext('Mozilla Persona')}}</title>

Abide provides `LANG`, `DIR`, and `gettext`. `LANG` is the language code based on the user's browser and preferred language settings. `DIR` is for [bidirectional text](http://en.wikipedia.org/wiki/Bi-directional_text) support.

`gettext` is a JS function which will take an English string and return a localize string, again based on the user's preferred region and language.

Here is an example in a JavaScript file:

    app.get('/', function(req, res) {
        res.render('homepage', {
            title: req.gettext('Hello, World!')
        });
    });

We can see that these variables and functions are placed in the `req` object.

At runtime, the `i18n-abide` module will detect the user's prefered locale and output "Hello, World!" localized to them.

So to setup our site for localization, we must look through all of our code and templates and wrap strings in calls to gettext.

-----------------------

# Localization community, tools, and ...

## The toolchain

Mozilla is over a decade old. It's had one of the bigest (and coolest) l10n communities in Open Source. As a result, it has many tools and the lowest levels are traditional tools.

### Gettext
GNU Gettext is a toolchain that allows you to extract Copy from your Node.js code. These are called Strings (after the C name for ... strings). When you write your Node.js code and templates, you put English Strings<sup>[1]</sup> in like normal, but you wrap them in a functional call to gettext.

This does a few different things for you:
* As a build step, you can extract all the strings
* At runtime, you can get a localize string based on the English string

All strings end up in text files that end with the `.po` file suffix. I'll refer to these as PO files.

There are many other tools that Gettext provides, for managing strings, PO files, etc. We'll cover these in a bit.

### SVN
[Can we get away with not mentioning SVN?]
Mozilla manages it's PO files for various projects with SVN. Mozilla is no stranger to git and other modern version control systems, but SVN is required within the L10n Community.

Of course you don't have to use SVN with your Node.js service. It's mentioned here as we'll see it pop up later.

## Why this toolchain?
Before we get into the Node modules that make working with Gettext easy, we must ask ourselves... why this toolchain?

A year ago I did a deep survey of all the Node l10n and i18n modules. They all "reinvent the wheel", creating their own JSON based formats for storing strings.

In order to work with our community they we must maintain the contract that they give us PO files via SVN and our service is properly localized.

Coming from PHP and Python at Mozilla, I've found that Gettext works very well. As a web service gets large and has more copy, there are many nuances of localizing copy that require the well tested API of gettext.

Example:
* <div id="system-updates">You have three updates.</div>
* <div id="social-inbox">You have three udpates.</div>

Assuming you used the word updates in two different contexts, some regions might wish to use two different terms for updates. So you need a contextual tag, so that they appear as two different strings.

Also, those sentencaes are plural. In some languages there will be different copy depending on the number of things your refering too.

Proper localization is hard! You better have an API that supports proper translations.

That said, you can always go with a newer, more modern setup and add the missing features paper cut by paper cut. Just, please, look at Gettext and other older systems for API inspiration. They have millions of people hours of usage behind them.

## String wrangling

The good news is that the GNU Gettext tools only have to be installed on the machine where the L10n wrangler is going to do that work. This can be an automated process, or the work of an individual.

The following tasks are done in this phase

* First time extraction of strings from the software
* Extracting new, changed, or detecting deleted strings in later releases
* Preparing the PO files for hand off to localizers
* Resolving conflicts and marking strings which have changed or been deleted

Developers won't need these tools on their machines and the runtime Node.js system can be blissfully ignorant of them, but `msginit`, `xgettext`, `msgfmt` and other GNU Gettext tools are a powerful way to manage catalogs of strings.

## Node.js modules
Now that we understand the technical requirements of our Node.js app, let's look at how this was done for Mozilla Persona.

### po2json

Our strings are in PO files, typically in a file system like this:

    locale
      en
        LC_MESSAGES
          messages.po
      de
        LC_MESSAGES
          messages.po
      pt_BR
        LC_MESSAGES
          messages.po

We need a way to get strings into our app at runtime. There are a few ways you can do this.

The first way, for server side strings, is to use the fine node-gettext module. We launched with this module and it's quite efficient for doing server side translation.

The second way, is to have a build step which creates JSON language files out of our PO files. We've switched to this method for all our strings, because we had to support this for client side translation, which make up the majority of our string usage.

Our build script is called `po2json.js`.

Example:

    $ locale/po2json.js static/i18n locale
    ...

And we get a file structure like:

    static
      i18n
        en
          messages.json
        de
          messages.json
        pt_BR
          messages.json


### jsxgettext

So how do we create these PO files? You can use traditional GNU Gettext utilities, but we've also written a Node module `jsxgettext`, which is a nice cross platform way to go.

Another wrinkle is that GNU Gettext doesn't know how to parse JavaScript or various flavors of Node templating languages. With `jsxgettext`, we can do a better job here.

jsxgettext parses your source files looking for uses of Gettext functions, and then it extracts just the string part. It then formats a PO file, which is compatible with any other Gettext tool.

Here is an exceprt from a PO File.

    #: resources/views/about.ejs:46
    msgid "Persona preserves your privacy"
    msgstr ""

    #: resources/views/about.ejs:47
    msgid ""
    "Persona does not track your activity around the Web. It creates a wall "
    "between signing you in and what you do once you're there. The history of "
    "what sites you visit is stored only on your own computer."
    msgstr ""
    ""

    #: resources/views/about.ejs:51
    msgid "Persona for developers"
    msgstr ""

After your localizers edit it, it will look more like this:

    #: resources/views/about.ejs:46
    msgid "Persona preserves your privacy"
    msgstr "Persona 保護您的隱私"

    #: resources/views/about.ejs:47
    msgid ""
    "Persona does not track your activity around the Web. It creates a wall "
    "between signing you in and what you do once you're there. The history of "
    "what sites you visit is stored only on your own computer."
    msgstr ""
    "Persona 只是連結您登入過程的一座橋樑，不會追蹤您在網路上的行為。您的網頁瀏覽"
    "紀錄只會留在您自己的電腦當中。"

    #: resources/views/about.ejs:51
    msgid "Persona for developers"
    msgstr "Persona 的開發人員資訊"

You can [view the complete zh_TW PO file](http://svn.mozilla.org/projects/l10n-misc/trunk/browserid/locale/zh_TW/LC_MESSAGES/messages.po), to get a better idea.

I won't go into all the details, but it is always a good idea to talk to the localizers before you start. If they use the Gettext tool chains, you'll probably start with a PO file tempalte (a .POT file) and then generate the various locale PO files from that.

Wow, we've covered a lot of ground. Let's look at some deeper details...

## Format, gettext, ngettext

We haven't mentioned `format` yet. This is another JS function which `i18n-abide` injects. String interpolation is quite common in localizaing software.

Here are some examples:

    format(gettext('<a href="$s">Register with our partner $s</a>), 'https://example.com', 'Example');

We have regional partners and the name and url vary by locale.

    <p>(format(gettext('Welcome back, $s'), user.name);</p>

We need to inject app or user data into strings.

    format(ngettext('You have $s followers', $s), $s);

Copy has plurals. `ngettext` captures this in the PO files as well as returning the correct translation based on the number of items.

## User's Preffered Language

What... so how do we know what the user's preferred language is?

The i18n-abide module looks at the `Accept-Language`
...


## Bells and Whistles

How does one test that their site is ready for localizers? We've created a node module called `gobbledygook` which ...

[1] You could use variable names or something else, instead of the actual copy. Then you'd "localize" the English version, just like any other locale. This is not how it is done for Mozilla web services.

## Beyond Node.js

Many important aspects of Internationalization and Localization are things you should be aware of regardless of the programming language or framework.

It is important to work L10n into most phases of software development. When your designers have mockups, have a L10n guru review them.

A design should support Right to Left layout, instead of only a Left to Right composition. If you have a large call to action block on the left side of a page and other secondary blocks of content to support it... Then in Hebrew and other RTL languages, you'll want the layout mirrored, so that the call to action has the same impact. Some clever CSS can take advantage of the `dir` HTML attribute.

Images with text in them are expensive and problematic. An image or any container needs to be bulletproof. Idiomatic English might look great in that trendy faux sticker, but then adding the same content in German may not be possible as 4 letters have become 14.

Just like designers learned with data driven websites, where layouts and elements are filled with different content from the database... Designers often have to re-learn that even static elements like Banners and promotional links will vary in size.

Getting to clever with a design can add expense, especially if an asset is manually created for each locale. This doesn't scale and can slow down your deployment.

In addition to a code review, have developers or L10n team members review code regularly for proper use of Gettext. In addition to words, numbers and dates require special care. Everyone in the world doesn't format 5,000 like 5,000. Nor do they do that on Jan 5th, 2013.

Reviews will catch these early, getting them into your string catalogs like PO files in the correct format. Correcting this in the middle of localization can be a nightmare, as you have to try to update N number of PO files with N L10n teams with members who often aren't comfortable with version control.

Many teams have a code freeze to control quality and schedules. Similarly you need you developers and copy writers to coordinate with your L10n team. You should have a string freeze and plan on giving L10n enough time to do their work. Luckily, this can often overlap wtih your QA and Security testing.

Just like a code freeze, only exceptional situations should change copy in the app before pushing to production.

Continuous deployment for localized applications is not a solved problem. It  is much easier to do scheduled deployments with L10n time backed in. You may have to wait until the slowest L10n team is done before deploying to production.
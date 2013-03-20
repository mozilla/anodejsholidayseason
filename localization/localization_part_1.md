# How to Localize Your Node.js service

![](dialog_fanv2.png)

Mozilla provides products and services which are localized into as many as 90 languages!

The following are just a few examples of localization:
* Providing text translated into a specific regional variation of a language
* Rendering a screen right to left for a given language
* Bulletproofing designs to accommodate variable length prose
* Making labels, headings, and buttons have names that resonate with a local audience

In this series of posts, I'm going to cover some technical aspects of how to localize a Node.js service.

I'll be using the terms L10n (<b>L</b>ocalizatio<b>n</b>) and I18n <b>I</b>nternationalizatio<b>n</b>.

I18n is the technical plumbing needed, to make L10n possible.

## i18n-abide

Mozilla Persona is a Node.js based service localized into X locales. To accomplish this, it uses the following tools

* i18n-abide
* jsxgettext
* po2json.js
* gobbledygook

### Goals
We created these modules, to meet the following goals
* Work well with existing Mozilla L10n community
* Let developers work with a pure JS toolkit

i18n-abide is the main module you'll use to integrate translations into your own service.
Let's walk through how to add it.

In these examples, we'll assume your code uses [Express](http://expressjs.com/) and [EJS templates](https://github.com/visionmedia/ejs).

## Installation

    npm install i18n-abide

## Preparing your codebase

In your code

    var i18n = require('i18n-abide');

    app.use(i18n.abide({
      supported_languages: ['en-US', 'de', 'es', 'zh-TW'],
      default_lang: 'en-US',
      translation_directory: 'static/i18n'
    }));

We will look at the configuration values in detail during the third installment of this L10n series.

The i18n `abide` middleware sets up request processing and injects various functions we'll use for translation.

The next step is to work through all of your code where you have user visible prose.

Here is an example template file:

    <html lang="<%= lang %>" dir="<%= lang_dir %>">
      <head>
        <title><%= gettext('Mozilla Persona') %></title>

The key thing abide does, is it injects into the Node and express framework references to the `gettext` function.

Abide also provides other variables and functions, such as `lang`, `lang_dir`.

`lang` is the language code based on the user's browser and preferred language settings.

`lang_dir` is for [bidirectional text](http://en.wikipedia.org/wiki/Bi-directional_text) support.
It will be either `ltr` or `rtl`. The English language is rendered `ltr` or left to right.

`gettext` is a JS function which will take an English string and return a localize string, again based on the user's preferred region and language.

When doing localization, we refer to **strings** or Gettext strings.
These are pieces of prose, labels, button, etc.
Any prose that is visible to the end user is a string.

Technically, we don't mean JavaScript String, as you can have strings which are part of your program, but never shown to the user.
String is overloaded to mean, stuff that must get translated.

Here is an example JavaScript file:

    app.get('/', function(req, res) {
        res.render('homepage.ejs', {
            title: req.gettext('Hello, World!')
        });
    });

We can see that these variables and functions (like `gettext`) are placed in the `req` object.

So to setup our site for localization, we must look through all of our code and templates and wrap **strings** in calls to `gettext`.

## Language Detection

By setting up the i18n-abide module, we've actually installed a new piece of middleware.

At runtime, the middleware will detect the user's preferred language.

But, how do we know what the user's preferred language is?

The i18n-abide module looks at the `Accept-Language` HTTP header.
This is sent by the browser and includes all of the user's preferred languages with a preference order.

i18n-abide processes this value and compares it with your app's `supported_languages` configuration.
It will make the best match possible and serve up that language.

If it cannot find a good match, it will serve up the strings you've put into your code and templates, which is typically English strings.

## Wrapping Up

In our next post, we'll look at how strings like "Hello, World!" are extracted, translated, and prepared for use by our service.

In the third post, we'll look more deeply at the middleware and configuration options.

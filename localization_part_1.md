
# How to Localize Your Node.js service
[Start with Conclusion... explain goals]
[Show what we did]
[Explain in detail why]
## What is localization?

The products and services which Mozilla provides are localized into as many as 90 languages! Localizing a website or native app involves translating copy, adding/removing/tweaking components to fit with cultural norms, as well as matching the technical differences of region or language.

The following are just a few examples of localization:
* Providing copy translated into a specific regional variation of a language
* Rendering a screen right to left for a given language
* Bulletproofing designs to accomodate more or less text
* Naming things or providing graphics that resonate with a local audience

In this series of posts, I'm going to cover some technical aspects of how to localize a Node.js service, as well as some more general techniques that work regardless of the technology.

I'll be using the terms L10n (<b>L</b>ocalizatio<b>n</b>) and <b>I</b>nternationalizatio<b>n</b>, becuase we love our acronyms, right?
[This is basically an intro to the series with too much copy]

## i18n-abide

Mozilla Persona is a Node.js based service localized into X locales. To accomplish this, it uses the following modules

* i18n-abide
* jsxgettext
* gobbledygook

### Goals
We created these modules, to meet the following goals
* Work well with existing Mozilla L10n community
* Pure JS toolkit

i18n-abide is the main module you'll use to integrate translations into your own service.
Let's walk through how to add it.

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

Here is an example template file:

    <html lang="{{LANG}}" dir="{{DIR}}">
      <head>
        <title>{{gettext('Mozilla Persona')}}</title>

Abide provides various variables and funcitons, such as `LANG`, `DIR`, and `gettext`.

`LANG` is the language code based on the user's browser and preferred language settings.

`DIR` is for [bidirectional text](http://en.wikipedia.org/wiki/Bi-directional_text) support.

`gettext` is a JS function which will take an English string and return a localize string, again based on the user's preferred region and language.

Here is an example JavaScript file:

    app.get('/', function(req, res) {
        res.render('homepage', {
            title: req.gettext('Hello, World!')
        });
    });

We can see that these variables and functions are placed in the `req` object.

At runtime, the `i18n-abide` module will detect the user's prefered locale and output "Hello, World!" localized to them.

So to setup our site for localization, we must look through all of our code and templates and wrap strings in calls to gettext.

[Look through i18n-abide tutorial, any other steps?]

In our next post, we'll look at how strings like "Hello, World!" are extracted, translated, and prepared for use by our service.

Stay tuned...
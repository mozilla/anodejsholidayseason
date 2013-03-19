
# Localization community, tools, and process

In our previous post "How to Localize Your Node.js service", we learned how to add i18n-abide to our code.

We wrapped strings in templates files as well as JavaScript files with calls to a function `gettext`. As developers, our work ends there. But the work of getting our copy localized has just begun.

## The toolchain

A goal for Mozilla Persona's Node.js code is to be compatible with the larger Mozilla community, while being Node friendly and flexible.

The Mozilla project is over a decade old.
It's had one of the bigest (and coolest) l10n communities in Open Source.
As a result, it has many     existing tools and the lowest levels are old crotchety tools.

### Gettext
GNU Gettext is a toolchain that allows you to localize copy and other Strings from webapps or native apps. These are called Strings (after the C name for ... strings). When you write your Node.js code and templates, you put English Strings<sup>[1]</sup> in like normal, but you wrap them in a functional call to gettext.

Wrapping with `gettext` does a few different things for you:
* As a build step, you can extract all the Strings
* At runtime, the gettext function replaces the English String with a localize string

This build step builds a catalog of Strings from your code and template files.

All these Strings end up in text files that end with the `.po` file suffix. I'll refer to these as PO files.

### PO Files

These are plain text files with a specific format that the Gettext tools can read, write, and merge.

An example snippet of a PO file named zh_TW/LC_MESSAGES/messages.po:

    #: resources/views/about.ejs:46
    msgid "Persona preserves your privacy"
    msgstr "Persona 保護您的隱私"

We'll examine this in more detail below, but we can see that `msgid` is the English String and `msgstr` has the Chinese translation. There are comments in the file for where in the codebase the String is used.

There are many other tools that Gettext provides, for managing Strings, PO files, etc. We'll cover these in a bit.


## Why a new toolchain?
Before we get into the Node modules that make working with Gettext easy, we must ask ourselves... why this toolchain?

A year ago I did a deep survey of all the Node l10n and i18n modules. Most "reinvent the wheel", creating their own JSON based formats for storing Strings.

In order to work with our community, we must use PO files. They have many tools such as [POEdit](http://www.poedit.net/), [Verbatim](https://localize.mozilla.org/), [Translate Toolkit](https://github.com/translate/translate), and [Pootle](https://github.com/translate/pootle)

So our basic constraint is to create a solution that uses `PO` files, which is how we'll tell our localizers what all of our strings are and how they will give us the finished translations.

Coming from PHP and Python at Mozilla, I've found that Gettext works very well. As a web service gets large and has more copy, there are many nuances of localizing copy that require the well tested API of gettext.

## Providing PO Files to localizers

So our code is marked up with `gettext` calls. Now what?
Get thee a **String wrangler**.
This person or persons can be you, a localization expert, or a build system guru.

So what does a String wrangler do?

* First time extraction of Strings from the software
* Extracting new, changed, or detecting deleted Strings in later releases
* Preparing the PO files for each localizer team
* Resolving conflicts and marking Strings which have changed or been deleted

This may sound complicated, but the good news is that only the String wrangler has to worry about this problems that crop up.
These steps can be automated.

Developers won't need these tools on their machines and the runtime Node.js system can be blissfully ignorant of them, but `msginit`, `xgettext`, `msgfmt` and other [GNU Gettext tools](TODO) are a powerful way to manage catalogs of Strings.

### Setup locale filesystem

    $ mkdir -p locale/templates/LC_MESSAGES

This directory is where we'll store the PO template or POT files.
The POT files are used by the Gettext toolchain.

### Extract the Strings

    $ ./node_modules/.bin/extract-pot --locale locale .

This script will recursively look through your source code and extract Strings.

So how does `extract-pot` create these PO files?
You can use traditional GNU Gettext utilities, but we've also written a Node module `jsxgettext`, which is a nice cross platform way to go.
`extract-pot` uses `jsxgettext` behind the scenes.

jsxgettext parses your source files looking for uses of Gettext functions, and then it extracts just the String part.
It then formats a PO file, which is compatible with any other Gettext tool.

Here is an excerpt from a PO File.

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

### Creating a locale

The Gettext tool `msginit` is used to create an empty PO file for a target locale, based on our POT file.

    $ for l in en_US de es; do
        mkdir -p locale/${l}/LC_MESSAGES/
        msginit --input=./locale/templates/LC_MESSAGES/messages.pot \
                --output-file=./locale/${l}/LC_MESSAGES/messages.po \
                -l ${l}
      done

We can see that we've created English, German, and Spanish PO files.

## PO files

So we've extracted our Strings and created a bunch of locales.

Here is a sample file system layout:

    locale/
      el/
        LC_MESSAGES/
          messages.po
      en_US
        LC_MESSAGES/
          messages.po
      es
        LC_MESSAGES/
          messages.po
      templates
        LC_MESSAGES/
          messages.pot

You can give your localizers access to this part of your codebase.
The Spanish team will need access to `locale/es/LC_MESSAGES/messages.po` for example.
If you have a really big team, you might have `es-ES` for Spain's Spanish and `es-AR` for Argentian Spanish, instead of just a base `es` for all Spanish locales.

You can grow the number of locales over time.

### Merging String changes

Over time, you'll add new Strings and change or delete others. You'll need to update all of the PO files with these changes.

Gettext has powerful tools to make this easy.

We provide a wrapper shell script called `merge_po.sh` which uses `msgmerge` under the covers.

Let's put the i18n-abide tools in our path:

    $ export PATH=$PATH:node_modules/i18n-abide/bin

And run a String merge:

    $ ./node_modules/.bin/extract-pot --locale locale .
    $ merge_po.sh ./locale

Just like the first time... `extract-pot` grabs all the Strings and updates the POT file. Next `merge_po.sh` updates each locale's PO file to match our codebase. You can now ask your L10n teams to localize any new or changed Strings.

### Gettext versus Not Invented Here
It is easy enough to throw out Gettext and re-invent the wheel using an invented JSON format.
This is the strategy that most node modules take.
If you have a healthy application, as you add locales and develop new features, you will find yourself frustrated by a thousand paper cuts.
Without `merge_po.sh`, you'll have to write your own merge tools.
This is because if you have 30 locales, you'll need to update 30 JSON files without losing the work they've already done.

Gettext offers a powerful merge feature, which will save us many painful hours of coordination.

# Wrapping Up

Now that we have various catalogs of strings in a po file per locale, we can hand these off to our localization teams.

It is always a good idea to talk to the localizers before you start the extract / merge steps.
You can read Gettext tutorials, as they are all comptaible with our setup.

Okay, go get your Strings translated and in the next installment, we'll put them to work!
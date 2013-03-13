
# Localization community, tools, and ...

In our previous post localizing Node.js services, we learned how to add i18n-abide to our service.

We wrapped strings in templates files as well as JavaScript files with calls to a function `gettext`. As developers, our work ends there. But the work of getting our copy localized has just begun.

We've built out a Node.js toolchain for localizing all these strings.

## The toolchain

A goal for Mozilla Persona's Node code is to be compatible with the larger Mozilla community, while being Node friendly and flexible.

Mozilla is over a decade old. It's had one of the bigest (and coolest) l10n communities in Open Source. As a result, it has many tools and the lowest levels are old crotchety tools.

### Gettext
GNU Gettext is a toolchain that allows you to extract Copy and other Strings from webapps or native apps. These are called Strings (after the C name for ... strings). When you write your Node.js code and templates, you put English Strings<sup>[1]</sup> in like normal, but you wrap them in a functional call to gettext.

This does a few different things for you:
* As a build step, you can extract all the strings
* At runtime, you can get a localize string based on the English string

All strings end up in text files that end with the `.po` file suffix. I'll refer to these as PO files.

### PO Files

These are plain text files with a specific format that the Gettext tools can read, write, and merge.

An example snippet of a PO file named zh_TW/LC_MESSAGES/messages.po:

    #: resources/views/about.ejs:46
    msgid "Persona preserves your privacy"
    msgstr "Persona 保護您的隱私"

We'll examine this in more detail below, but we can see that `msgid` is the English string and `msgstr` has the Chinese translation. There are comments in the file for where in the codebase the string is used.

There are many other tools that Gettext provides, for managing strings, PO files, etc. We'll cover these in a bit.

### SVN
[Can we get away with not mentioning SVN?]
Mozilla manages it's PO files for various projects with SVN. Mozilla is no stranger to git and other modern version control systems, but SVN is required within the L10n Community.

**Of course you don't have to use SVN** with your Node.js service. It's mentioned here as we'll see it pop up later.

So our basic constraint is to create a solution that uses `PO` files, which is what our localizers will deliver via SVN.

## Why a new toolchain?
[Delete? or put in Going Deep?]
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

## Providing PO Files to localizers

So our code is marked up with `gettext` calls. Now what? Get thee a String wrangler. This can be you, an L10n person, or a build system guru.

This may sound complicated, but the good news is that only the String wrangler has to worry about this peice. This can be an automated process, or the work of an individual.

The following tasks are done in this phase

* First time extraction of strings from the software
* Extracting new, changed, or detecting deleted strings in later releases
* Preparing the PO files for hand off to localizers
* Resolving conflicts and marking strings which have changed or been deleted

Developers won't need these tools on their machines and the runtime Node.js system can be blissfully ignorant of them, but `msginit`, `xgettext`, `msgfmt` and other GNU Gettext tools are a powerful way to manage catalogs of strings.

[Explain the steps]

[Explain how msgmerge is superior to homegrown .json systems]

### jsxgettext

So how do we create these PO files? You can use traditional GNU Gettext utilities, but we've also written a Node module `jsxgettext`, which is a nice cross platform way to go.

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
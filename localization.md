
Outline

9. "Localizing your node.js service, part 1" -
* Ready to take your web-app worldwide?
* The first step is to translate it!  This post will present a basic workflow for you can translate your project.  We'll apply the
* gettext() toolchain to a web application and present
* new pure-javascript tools that facilitate the workflow.

10. "Localizing your node.js service, part 2" - Here we present
* i18n middleware that helps you with the task of speaking your user's language - taking the
* Language-Accept header and matching it to the set of languages you support.  We'll also discuss how
* substitution can be performed based on this in server and client side code.

11. "Localizing your node.js service, part 3" - Finally, we'll talk about
* debugging the i18n implementation of your system.  We present
* gobbledygook, a small library that helps you visually confirm the correctness of your i18n implementation.

-------
Outline AOK

1) How to localize your node.js service
(i18n-abide, gettext)
* We wrote some modules
* tutorial style walkthrough of i18n-abide ifying a site?
*

2) Localization community, tools, and ...
* Intro - dev work done, l10n work begins
* Explain Gettext
* Explain PO files
* Mention SVN
* String Wrangling (providing our strings to lcoalizers)
* jsxgettext
* Using our strings
* po2json

3)  L10n in action, going deeper
* intro
* gobbdegook
* how middleware works with Accept-Language
* missing features (language picker, url based locales)

history

middleware - accept language matching
format

skipping po files

4) Beyond Node

images
BIDI

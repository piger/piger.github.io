---
title: International Keyboard
date: 2013-04-22
tags:
- osx
- keyboard
- euro
summary: Typing accented letters (both acute and grave) with a U.S. keyboard layout.
categories:
- tips
---

The best keyboard layout for programming or dealing with UNIX-like
systems is the U.S. one, but it can be problematic if english is not
your primary language; for example with an italian layout you have to
press 3 keys to type a *{* and you have to use a [dead key](http://en.wikipedia.org/wiki/Keyboard_layout#Dead_keys)
to type a
*\~* (nearly 70% of my command lines involves some file in my home
directory so I type *\~* **a lot**).

Yet this can't be considered a complete solution because if you want to
write in italian language you **must** use accented letters for some
words, for example *perché*.

My current solution is to use a layout inspired by GNOME's
U.S. International
(AltGr) created by Tobias Müller (who wrote a nice
[post](http://www.twam.info/hardware/us-international-on-os-x) about it)
and which can be downloaded
[here](http://www.twam.info/wp-content/uploads/2010/08/U.S.%20International%20wo%20dead%20keys.keylayout);
I've also used
[Ukelele](http://scripts.sil.org/cms/scripts/page.php?site_id=nrsi&id=ukelele)
to modify that layout to make the key *§* (§/± left of 1/!) a [dead key](http://en.wikipedia.org/wiki/Keyboard_layout#Dead_keys) to be able
to type letters with a grave accent.

- ALT+key: é í ó ú á €
- § - key: è ì ò ù à

Bonus feature: ALT+5 to type the € symbol.

You can download my layout here: [U.S. Euro](us_euro.keylayout); copy it in
`~/Library/Keyboard Layouts` and don't forget to close and reopen System
Preferences, to make it reload all the files.

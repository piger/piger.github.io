---
title: Create fortune(6) files from Wikiquote
date: 2013-07-26
tags:
- tv-shows
- fortune
summary: I've create a very simple script that create a fortune(6) file from a Wikiquote page.
categories:
- Amusement
---

# Create fortune(6) files from Wikiquote

Yesterday I was searching for some *modern* fortune(6) files because
I'm really bored by the existing collections that are mostly from the
'90s. So after finding the [Splitbrain](http://www.splitbrain.org/projects/fortunes) collection of fortunes which
includes quotes from "The X-Files" I've decided to roll my own
solution: a simple Python script that fetch a Wikiquote_ page and
extracts the quotes to create a fortune(6) file.

You can find the script on [github](https://github.com/piger/wikiquote2fortune).

Example usage; we fetch quotes for [Lost](https://en.wikiquote.org/wiki/Lost_%28TV_series%29) and store them
in `~/.fortunes`:

```
$ wikiquote2fortune -u 'https://en.wikiquote.org/wiki/Lost_%28TV_series%29' \
-O lost -n Lost
$ strfile lost
$ mkdir ~/.fortunes
$ mv lost* ~/.fortunes
```

You can test the installation running:

```
$ fortune ~/.fortunes
Dr. Brooks: Dave isn't your friend, Hugo. Because Dave doesn't exist.

    "Lost: Dave [2.18], Season Two"
```

It will probably have bugs since I've written it in a hurry ;)

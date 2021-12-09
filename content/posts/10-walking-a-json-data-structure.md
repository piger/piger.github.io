---
title: Extracting values from an arbitrary JSON data structure
date: 2016-07-02
summary: How to walk a JSON data structure to extract the selected leaf nodes.
tags:
- python
- json
categories:
- Tips and Tricks
---

A friend of mine came to me for help with a Python programming problem: he needed a way to extract some values from a JSON data structure by specifying the full *path* of each value; for example, given the following data structure:

```json
{
  "name": "cavallo",
  "attributes": {
    "good": "goloso",
    "bad": "dispettoso"
  }
}
```

he wanted to be able to extract `name` and `attributes.good`. One of my friend's requirements was that the code should read the list of *paths* from an external file, so that he could decouple code from configuration.

I came up with the following Python function, which is an iterator that yields tuples of `(path, value)` while walking the input JSON data structure:
<script src="https://gist.github.com/piger/169330a3d34cea11e556dde98e403dd6.js?file=walk_json.py"></script>

The JSON data structure used as test case can be found on the [gist page](https://gist.github.com/piger/169330a3d34cea11e556dde98e403dd6).

Demo:
```
$ python walk_json.py sample.json
(u'stats.timestamp', u'2016-07-02T13:35:07.076830981Z')
(u'stats.memory.usage', 49815552)
(u'stats.cpu.usage.total', 1526446336302)
(u'stats.cpu.usage.user', 1161250000000)
(u'name', u'/init.scope/system.slice/docker-7f9ed89c7297f99669b6e79d9d8d404d19f160ca40b40f42896506fa7942786b.scope')
(u'spec.has_network', False)
```

---
title: "OSX: switch font size when an external display is attached"
date: 2016-07-31
summary: How to change font size in Emacs and iTerm2 when an external display is attached.
tags:
- osx
- hammerspoon
- emacs
- iterm2
categories:
- tips
---

When I need to work long hours outside of the office I usually set a smaller font in iTerm2 and Emacs (those are the applications
that I use most) because I don't have access to a large external display. Having to manually set the font in two different applications every time is a bit unhandy,
so I started looking for a solution.

## iTerm2

It turns out that iTerm2 recently introduced the [Dynamic Profiles](https://www.iterm2.com/documentation-dynamic-profiles.html) feature,
which allows you to store your configuration profile on the filesystem and have iTerm2 pick up all the changes immediately. That's a start.

I created a base profile in iTerm2 called "Default", then I created two profile files, one for each font size; a profile file is a JSON file containing the profile
configuration:

```json
{
  "Profiles": [
    {
      "Name": "Default+Dynamic",
      "Guid": "90B59958-1F30-419B-9E4A-B52C24700CB5",
      "Dynamic Profile Parent Name": "Default",
      "Normal Font" : "FiraMono-Regular 11",
      "Non Ascii Font" : "FiraMono-Regular 11"
    }
  ]
}
```

This profile inherits from a profile named "Default" and overrides the "Normal" and the "Non Ascii" font configuration options. The `Guid` field can be generated with
the `uuidgen` command, I don't think it does really matter. The next step is to symlink the chosen profile to `$HOME/Library/Application Support/iTerm2/DynamicProfiles/iTerm2_Dynamic.json`
and make it the default in iTerm2 (I know, I didn't spend much time thinking about those names), but it would be great if this task could be automated.

I recently discovered [Hammerspoon](http://www.hammerspoon.org/), an automation software for OS X with a Lua programming interface; Hammerspoon allows you to bind Lua callbacks to some operating
system events, to react for example when an external device is connected.

The following code set up a watcher to react when the display configuration changes (e.g. when you plug or unplug an external display) and it will overwrite the symlink to the
iTerm2 dynamic profile using the large font profile when there's an external display connected.

```lua
function hasExternalMonitor()
   for _, screen in pairs(hs.screen.allScreens()) do
      if screen:name() == "Thunderbolt Display" then
         return true
      end
   end
   return false
end

-- Switch dynamic profile in iTerm2
function setIterm2Profile(filename)
   hs.execute("ln -sf $HOME/Preferences/iTerm2/" .. filename .. " \"$HOME/Library/Application Support/iTerm2/DynamicProfiles/iTerm2_Dynamic.json\"")
end

function displayWatcherCallback()
   if hasExternalMonitor() then
      setIterm2Profile("iTerm2_Dynamic_12.json")
   else
      setIterm2Profile("iTerm2_Dynamic_11.json")
   end
end

local monitorWatcher = hs.screen.watcher.new(displayWatcherCallback)
monitorWatcher:start()
```

This code expect that your store your dynamic profile files in `$HOME/Preferences/iTerm2/` and call them `iTerm2_Dynamic_11.json` and `iTerm2_Dynamic_12.json`.

## Emacs

To achieve the same goal in Emacs I wrote some an​ elisp function to detect if there's an external display attached and change the font accordingly; then it's all a matter
of running `(server-start)` and calling this function from `displayWatcherCallback()` in Hammerspoon.

Hammerspoon code:

```lua
function tellEmacsToChangeFont()
   hs.execute("/usr/local/bin/emacsclient -e '(set-the-right-font)'")
end

function displayWatcherCallback()
   tellEmacsToChangeFont()

   if hasExternalMonitor() then
      setIterm2Profile("iTerm2_Dynamic_12.json")
   else
      setIterm2Profile("iTerm2_Dynamic_11.json")
   end
end
```

Emacs code:

```lisp
(defun set-the-right-font ()
  "Set the right font according to the connected displays"
  (interactive)
  (let ((monitors (shell-command-to-string "system_profiler  SPDisplaysDataType | egrep '^ {8}[^ ]' | sed -e 's/^ *//' -e 's/:$//'"))
        (hasExternal nil))
    (dolist (monitor (split-string monitors "\n"))
      (when (string= monitor "Thunderbolt Display")
        (setq hasExternal t)))
    (if hasExternal
        (set-default-font "Fira Mono-12")
      (set-default-font "Fira Mono-11"))))
```

I have to say that I'm surprised that everything seems to be working well.

## Notes

- I think dynamic profiles was introduced in iTerm2 3.x.
- By "Emacs" I mean the OS X port of Emacs running in GUI mode, installed with Homebrew.

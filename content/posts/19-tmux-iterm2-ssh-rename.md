---
title: Rename the tmux or iTerm2 window when connecting to a remote host
date: 2019-12-13
tags:
- zsh
- ssh
- tmux
- iterm2
summary: Rename your tmux or iTerm2 window when connecting to a remote host via OpenSSH.
categories:
- tips
---

This is an update of my [old post](/posts/01-tmux-ssh-rename/).

The following zsh code wraps the ssh command to grab the remote hostname, shorten it as needed and
use it to rename the current tmux or iTerm2 window or pane. It's really handy when you connect to
several different servers at once.

```bash
# ssh wrapper function to set tmux window (pane?) title

# bypass this function if stdout is not a terminal, to avoid messing up
# the output with our printf() calls.
[[ -t 1 ]] || exec command ssh $@

local prev_window_name=$(tmux display-message -p "#{window_name}")
local prev_pane_title=$(tmux display-message -p "#{pane_title}")

local win_escape_start="\033k"
local pane_escape_start="\033]2;"
local escape_end="\033\\"

set_pane_title() {
    printf "$pane_escape_start$1$escape_end"
}

set_window_title() {
    printf "$win_escape_start$1$escape_end"
}

local hostname=$(ssh -G "$@" | awk 'NR > 2 { exit } /^hostname/ { print $2 }')

if [[ -z $hostname ]]; then
    command ssh $@
else
    # try to shorten the hostname

    # 1) split the hostname on "."
    local parts=(${(s:.:)hostname})
    local short_name

    # 2) if the parts of the hostname are less than 3 (e.g. mail.example.com -> 3 parts)
    # use the first part only (e.g. "mail")
    if (( $#parts <= 3 )); then
        short_name=${parts[1]}
    else
        # othewrise show 2/3 of the hostname
        local fraction=$(( $#parts / 3 * 2 ))
        short_name=${(j:.:)parts[1,$fraction]}
    fi
    # Set window name and pane title when there's only 1 pane (i.e. the whole window),
    # otherwise set just the pane title.
    if [[ $(tmux display-message -p "#{window_panes}") == 1 ]]; then
        # Use a shortened version of the hostname for the window title
        tmux rename-window "$short_name"
        set_pane_title $hostname
    else
        set_pane_title $hostname
    fi
    command ssh $@

    # Restore the old window and pane titles after ssh exits.
    if [[ $(tmux display-message -p "#{window_panes}") == 1 ]]; then
        tmux rename-window "$prev_window_name"
        tmux setw automatic-rename on
    fi
    set_pane_title $prev_pane_title
fi
```

A really nice trick that I found somewhere and modified to my own needs is used to get the canonical
hostname of the remote server I'm connecting to using OpenSSH:

```bash
local hostname=$(ssh -G "$@" | awk 'NR > 2 { exit } /^hostname/ { print $2 }')
```

Running `ssh -G <hostname>` will print the full OpenSSH configuration for that hostname, which
includes a single `Hostname <address>` line containing the actual hostname of the remote server (in
casey you're using aliases); the `awk` incantation will match the line starting with `hostname`,
print the second element of that line and exit after the 2nd line of output (this is because the
`hostname` statement has been, so far, the second line of the output).

I put this file in one of my zsh `$fpath` directories and autoload it from `~/.zshrc`:

```bash
typeset -U fpath
fpath+=~/Preferences/zsh/functions
autoload -Uz ssh
```

To do the same thing when running iTerm2 without tmux, you can remove all the lines invoking the
`tmux` command and replace the set window title with something like:

```bash
set_window_title() {
    echo -ne "\033]0;"$1"\007"
}
```

And to restore the default window title when ssh exits, you can just invoke `set_window_title`
passing an empty string, like `set_window_title ""`.

[This code](https://github.com/piger/Preferences/blob/master/zsh/functions/ssh) can also be found on
Github in my [dotfiles repo](https://github.com/piger/Preferences).

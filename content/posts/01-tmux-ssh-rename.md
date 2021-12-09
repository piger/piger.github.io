---
title: Use the name of the remote host for tmux window when running ssh
date: 2013-03-26
tags:
- zsh
- ssh
- tmux
categories:
- Tips and Tricks
summary: Rename your tmux window when connecting to a remote host
---

I've wrote a simple zsh function that wraps the ssh command to be able to get
the name of the remote host and use it to name the current tmux window; as a
bonus feature the function will restore the previous window's name upon ssh
termination.

Update 20-Jun-2016: I've updated the `ssh` function.

```bash
ssh() {
    # ssh wrapper function to set tmux window (pane?) title

    # bypass this function if stdout is not a terminal, to avoid messing up
    # the output with our printf() calls.
    [[ -t 1 ]] || exec command ssh $@

    local prev_window_name=$(tmux display-message -p "#{window_name}")
    local prev_pane_title=$(tmux display-message -p "#{pane_title}")
    local hostname=$(perl -e 'if (length(@ARGV) == 1) { print $ARGV[0]; exit; } else { foreach $h (@ARGV) { if ($h =~ /[a-z\._-]+\.[a-z]{2,}/) { print $h; exit; } } }' -- $@)

    if [[ -z $hostname ]]; then
        command ssh $@
    else
        # Set window name and pane title when there's only 1 pane (i.e. the whole window),
        # otherwise set just the pane title.
        if [[ $(tmux display-message -p "#{window_panes}") == 1 ]]; then
            printf "\033k${hostname}\033\\"
            printf "\033]2;${hostname}\033\\"
        else
            printf "\033]2;${hostname}\033\\"
        fi
        command ssh $@
        if [[ $(tmux display-message -p "#{window_panes}") == 1 ]]; then
            printf "\033k${prev_window_name}\033\\"
        fi
        printf "\033]2;${prev_pane_title}\033\\"
    fi
}
```

This can be placed in `~/.zshrc` or in a function file.

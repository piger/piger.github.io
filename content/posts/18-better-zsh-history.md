---
title: Better command line history in zsh with peco
date: 2019-12-13T15:20:00Z
lastmod: 2023-05-07
tags:
- zsh
- command-line
summary: How one simple change improved my command line experience by a lot.
lead: the good command-line stuff
categories:
- Tips and Tricks
---

{{< admonition type=tip title="2021 Update" >}}
Nowadays I'm using [fzf](https://github.com/junegunn/fzf); to see how I've integrated it into my
zsh configuration check [this section](https://github.com/piger/Preferences/blob/e0002cec9eb0ff0a9b1fefc92c7bba16ae031ef3/zsh/zshrc#L262-L393)
of my
[zshrc](https://github.com/piger/Preferences/blob/e0002cec9eb0ff0a9b1fefc92c7bba16ae031ef3/zsh/zshrc).
{{< /admonition >}}

Some years ago I bumped into [peco](https://github.com/peco/peco), a *Simplistic interactive
filtering tool* written in Go; at that time there was a bit of hype around this kind of command line
tools, with several implementations published on Github. I settled on `peco` because it was fast
enough, compared for example to the Python implementation, to be tolerable in my daily shell usage.

A nice and often overlooked feature of bash and zsh (provided by
[readline](https://tiswww.case.edu/php/chet/readline/rltop.html)) is `reverse-i-search`, which
allows you to search and reuse previous commands from your shell history; unfortunately the
*presentation* of this command is quite basic and kinda hard to navigate, and this is where `peco`
brings value.

By adding this short snippet of code to your `~/.zshrc` you can use `peco` to navigate the history
of your shell:

```bash
function exists() { which $1 &> /dev/null }

function peco_select_history() {
    local tac
    exists gtac && tac="gtac" || { exists tac && tac="tac" || { tac="tail -r" } }
    BUFFER=$(fc -l -n 1 | eval $tac | peco --layout=bottom-up --query "$LBUFFER")
    CURSOR=$#BUFFER         # move cursor
    zle -R -c               # refresh
}

zle -N peco_select_history
bindkey '^R' peco_select_history
```

This is how it looks like when I press `CTRL-R` and type "man":

![screenshot](/images/zsh-peco.png)

This become really useful when you configure zsh to have a huge history size:

```bash
SAVEHIST=10000
HISTSIZE=12000
```

This is really handy for command that you type over and over and you can't really script, like
pushing Chef files to the chef server.

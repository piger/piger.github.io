---
title: "Writing a prompt for Zsh"
date: 2022-05-30T23:11:06+01:00
tags:
- zsh
categories:
- Tutorial
summary: "Short guide to write prompts for Zsh"
toc: true
draft: true
---

## Introduction

To personalise the shell prompt in Zsh the process is (as far as I remember) similar to Bash: set
the variable `PS1` or `PROMPT` to your favourite sequence of obscure symbols and you're good to go!

These obscure symbols are documented in the `zshmisc(1)` man page, in the _EXPANSION OF PROMPT
SEQUENCES_ section, by the way.

If you have time to procrastinate and you feel adventourous you might try to write a _Zsh prompt_
instead; prompts in Zsh are like themes in other software, like the color schemes in your text
editor. Prompts, like themes, can be referred by name, previewed and most importantly shared with
others.

## Scaffolding for a basic prompt

First you need a directory to store your prompt's code; while you could technically just open a
file, start writing your code and then loading it, it's a bit more clean if you put your code in a
dedicated directory. For the sake of this tutorial let's defined this directory as: `~/code/zsh`.

Now, following the instructions found in `zshcontrib(1)` at the _PROMPT THEMES_ section, add the
following lines to your `~/.zshrc`:

```zsh
fpath=(~/code/zsh $fpath)
autoload -U promptinit
promptinit
```

These three lines accomplish the following:

- add your newly created code directory (`~/code/zsh`) to `fpath`, which is necessary for `autoload`
  to find the prompt file you're about to create
- load the _prompt framework_
- initialise the _prompt framework_

Note that you'll need to start a new shell session to activate these changes.

In your new shell session you can now experiment with the `prompt` command; for example, to list all
the available prompts run:

```
$ prompt -l
Currently available prompt themes:
adam1 adam2 bart bigfade clint default elite2 elite fade fire off oliver pws redhat restore suse walters zefram
```

To try a prompt, for example "fire", run the command `prompt fire`; to go back to a barebone basic
prompt, run `prompt restore`. The `prompt` command is documented in the man page mentioned a few
lines above.

Having taken care of this preliminary steps, we can now start to write our personalised prompt. For
the sake of this tutorial let's decide that this new prompt will be called `oompa`. Create and open
the file `~/code/zsh/prompt_oompa_setup` (no extension!).

Let's write a bit of code to define the basic structure of our prompt's code:

```zsh
# zsh prompt: oompa
# the previous line is just a comment, you can write whatever you want, or nothing at all.

prompt_oompa_help() {
    cat <<EOH
This is the documentation of the oompa prompt.

Usage:

$ prompt oompa
EOH
}

prompt_oompa_setup() {
    PROMPT='oompa loompa> '
}

prompt_oompa_preview() {
    prompt_preview_theme oompa "$@"
}

prompt_oompa_setup "$@"
```

Now edit your `~/.zshrc` again and insert a new `autoload` call to load your theme, before the call
to `promptinit`; the snippet of code in `~/.zshrc` should look like this:

```zsh
fpath=(~/code/zsh $fpath)
autoload -U prompt_oompa_setup
autoload -U promptinit
promptinit
```

If you start a new shell session you can already experiment with your new prompt; to see the help
message, run:

```
$ prompt -h oompa
Help for oompa theme:

This is the documentation of the oompa prompt.

Usage:

$ prompt oompa

Type `prompt -p oompa' to preview the theme, `prompt oompa'
to try it out, and `prompt -s oompa' to use it in future sessions.
```

and to preview the prompt:

```
$ prompt -p oompa

oompa theme:
oompa loompa> command arg1 arg2 ... argn
```

## The rest of the damn owl

The most important variables to set in a prompt, and documented in `zshparam(1)`, are:

- `PROMPT`: equivalent to `PS1` and corresponding to the main shell prompt, on the left side of the terminal
- `RPROMPT`: set text that will appear on the right side of the terminal
- `SPROMPT`: the prompt used for spelling corrections (another Zsh feature)
- `prompt_opts`: set prompt-related options

A custom prompt in Zsh is essentially a shell script that customise the value of these variables to
your liking.

The rest of this guide will focus on writing a prompt that follow a popular style found in other
prompts and that looks like this:

![prompt_piger](/images/zsh_prompt.png)

I recommend to select one or more palettes in advance rather than just trying random colors, as it
will make easier to find a good combination; there are [several](https://colorhunt.co/)
[websites](https://coolors.co/palettes/trending) that [can help](https://color.adobe.com/explore) in
this regard. The screenshot above is from a prompt created starting from [these
colors](https://color.adobe.com/October-color-theme-7856284) for example.

**NOTE**: the range of colors available for your prompt depends on the support of your terminal;
iTerm2 on macOS for example supports "true colors", while other terminals may only support 256
colours or less. This guide assume you can use true colors, which can be specified in Zsh with their
hexadecimal representation (for example `#520120`); if you have to use 256 colors or less you might
find [this](https://github.com/eikenb/terminal-colors) program useful.

**NOTE**: depending on how fancy you want to get, you might also need a font that supports the so
called "powerline glyphs" or "nerd fonts".

A prompt like the one shown in the screenshot is composed by several sections (or _segments_), each
having it's own foreground and background colour; I recommend creating variables in your prompt's
code to refer to the colours you chose as it will make the code more readable (and beliebe me, you
want to try to keep this code as readable and commented as possible).

All the code that follows must be placed inside the function `prompt_oompa_setup` shown earlier.

Let's define the variables for the colors we selected:

```zsh
local time_bg="#520120" \
      time_fg="#e5989b" \
      dir_bg="#08403E" \
      dir_fg="#83c5be" \
      symbol_fg="30" \
      vcs_bg="#706513" \
      vcs_fg="#ffb703" \
      vcs_logo="20" \
      vcs_staged_fg="2" \
      vcs_unstaged_fg="3"
```

Let's also define two variables containing the special characters from the Powerline font that will
be used to separate each segment and to show errors:

```zsh
local SEGMENT_SEPARATOR=$'\ue0b0'
local ERROR_SYMBOL=$'\u2718'
```

Now let's define the segments, but first a very important warning: while some programming languages
consider `'` and `"` as equivalent when quoting a string of text, in this context is very important
to use the right quotes depending on the content of the quoted text. Double quotes will cause values
to be expanded immediately, while single quoted strings will be expanded at runtime.

First we set `PROMPT` to an empty string to start from scratch, then we define our first segment:
the current time.

```zsh
PROMPT=""
PROMPT+="%K{$time_bg}%F{$time_fg} %T "
```

The tokens `%K`, `%k`, `%F` and `%f` sets and clear the background and the foreground colors and are
documented in the `zshmisc(1)` man page; here we're setting the background color to `$time_bg` and
the foreground color to `$time_fg`, after which we use our first prompt variable: `%T` (also
documented in `zshmisc(1)`).

Now we need to create the effect that blends the first segment into the second, which contains the
current directory. This effect is created by printing the `SEGMENT_SEPARATOR` character with the
background color of the segment that follows.

```zsh
PROMPT+="%K{$dir_bg}%F{$time_bg}$SEGMENT_SEPARATOR "
```

Here we're setting the background color to the color of the directory segment and the foreground
color to the color of the time segment.

Now we can add the segment containing the current directory:

```zsh
PROMPT+="%K{$dir_bg}%F{$dir_fg}%~ "
```

Finally we add the last segment, which shows the current `git` branch and other useful informations
(like the presence of staged files):

```zsh
PROMPT+='${vcs_info_msg_0_}'
```

Don't worry for now about this segment, it will be described further down in the guide.

This segment closes the prompt's first line, so we can reset both the background and foreground
colors and print a newline:

```zsh
PROMPT+="%k%f${prompt_newline}"
```

**NOTE**: `$prompt_newline` is defined by `promptinit`.

As I like to keep the part of the shell prompt where I type most clean, the second line of the
prompt will only contain the usual `$` character (or `#` if you're root):

```zsh
PROMPT+="%F{$symbol_fg}%(!.#.$)%f "
```

This concludes the definition of the main shell prompt; to further our customisation effort we'll
also define the formatting for the right side of the prompt and the one for the suggestion's prompt.

The following code define a right side prompt that will show the "error symbol" and the exit code of
commands that exit with a non-zero status:

```zsh
RPROMPT="%(?..%B%F{$err_fg}${ERROR_SYMBOL}%f %?%b)"
```

And to personalise the suggestion's prompt:

```zsh
SPROMPT="%B%R%b does not exists; did you mean %B%r%b (%BY%bes/%BN%bo/%BE%bdit/%BA%bbort)? "
```

Finally we need to set `prompt_opts` to enable some (XXX) options required to render this prompt
correctly:

```zsh
prompt_opts=(cr subst percent sp)
```

At this point most of our prompt is defined, with the notable exception of the segment that show
informations from `git`; the configuration of this segment looks different enough from the rest of
the code that requires a dedicated section.

## Git informations on the prompt

Zsh can show informations from several Version Control System via the `vcs_info` framework,
described in `zshcontrib(1)` at the _GATHERING INFORMATION FROM VERSION CONTROL SYSTEMS_ section.

The most important _styles_ of `vcs_info` to configure are the ones showing the segment in three
different states:

- when outside of a version control repository
- when inside a directory containing a version control repository
- when inside a directory containing a version control repository _and_ running special operations
  (like a _rebase_ in git)

We define the first style as follow, to just close the directory segment without showing any version
control information:

```zsh
zstyle ':vcs_info:*' nvcsformats "%k%F{$dir_bg}$SEGMENT_SEPARATOR"
```

The we define the two remaining states:

```zsh
zstyle ':vcs_info:*' formats \
       "%K{$vcs_bg}%F{$dir_bg}$SEGMENT_SEPARATOR %F{$vcs_logo} %F{$vcs_fg}%b%f %m%u%c %k%F{$vcs_bg}$SEGMENT_SEPARATOR"
zstyle ':vcs_info:*' actionformats \
       "%K{$vcs_bg}%F{$dir_bg}$SEGMENT_SEPARATOR %F{$vcs_logo} %F{$vcs_fg}%b%F{3}|%F{1}%a%f %m%u%c %k%F{$vcs_bg}$SEGMENT_SEPARATOR"
```

This is probably the most unreadable part of this prompt's code so I recommend keeping the
`zshcontrib(1)` man page open while reading it.

The first style, setting the format strin for `formats`, is basically using the following tokens:
`%b`, `%m`, `%u` and `%c`; everything else is just colors and special characters.

- `%b` is expanded to the current branch name
- `%m` is a miscellaneous replacement (see the man page!)
- `%c` and `%u` are expanded to the `stagedstr` and `unstagedstr` format strings that we'll define
  later, and which basically mark the presence of staged or unstaged files.

For the tokens in the second format, `actionformats`, refer to the `zshcontrib(1)` man page.

Next we'll define a couple more formats and sets some of `vcs_info`'s options:

```zsh
zstyle ':vcs_info:*' enable git svn
zstyle ':vcs_info:*' check-for-changes true
zstyle ':vcs_info:*' unstagedstr "%F{$vcs_unstaged_fg}●%f"
zstyle ':vcs_info:*' stagedstr "%F{$vcs_staged_fg}✚%f"
```

This last bit of code is required to make `vcs_info` works: we need to hook it into the `precmd`
phase of the shell rendering; by doing so `vcs_info` will refresh its view of the current version
control repository and sets the format strings to the appropriate value.

```zsh
autoload -Uz add-zsh-hook
autoload -Uz vcs_info
add-zsh-hook precmd vcs_info
```

## Final words

The prompt is now ready to be used. Remember to start a new shell, then run:

```
$ prompt oompa
```

If everything went well we should now see our new prompt in action.

A recap of the prompt's code is provided below, as taken from my current prompt:

```zsh
# piger's prompt
# Daniel Kertesz <daniel@spatof.org>

prompt_piger_help() {
    cat <<EOH
This prompt takes advantage of 'vcs_info' to display useful VCS informations in the prompt.
You can chose between the short and the extended format, where the latter will also display
the typical "username@hostname" part of the prompt.

This theme offers some variants: base and autumn.

Usage:

$ prompt piger [<variant> [extended]]

EOH
}

# The "right arrow" symbol (Powerline special character):
SEGMENT_SEPARATOR=$'\ue0b0'
# U+2718 : HEAVY BALLOT X
ERROR_SYMBOL=$'\u2718'
PROMPT_SYMBOL='$'

prompt_piger_precmd() {
    # Use "psvar" to store the value of the $VIRTUAL_ENV variable; this is refreshed
    # every prompt since this function is a precmd.
    if [[ ! -z $VIRTUAL_ENV ]]; then
        psvar[1]="${VIRTUAL_ENV:t}"
    else
        psvar[1]=""
    fi
}

prompt_piger_setup() {
    # this is usually defined by 'promptinit'
    [[ -z $prompt_newline ]] && prompt_newline=$'\n%{\r%}'

    # disallow python virtualenvs from updating the prompt
    export VIRTUAL_ENV_DISABLE_PROMPT=1

    autoload -Uz add-zsh-hook
    autoload -Uz vcs_info

    # variables to make easier to change colors and make variants
    local time_bg time_fg dir_bg dir_fg symbol_fg vcs_bg vcs_fg vcs_logo \
          vcs_staged_fg vcs_unstaged_fg err_fg venv_fg

    time_bg="#520120"
    time_fg="#e5989b"
    dir_bg="#08403E"
    dir_fg="#83c5be"
    symbol_fg="30"
    vcs_bg="#706513"
    vcs_fg="#ffb703"
    vcs_logo="20"
    vcs_staged_fg="2"
    vcs_unstaged_fg="3"
    err_fg="124"
    venv_fg="205"

    # the following 3 styles control the rendering of the "git" prompt segment; in particular the first and last chunks
    # renders the closing of the previous segment and the final prompt segment (the one that reset the background, %k).
    # The colors used here must match the colors of the previous segment, which in this case is the current directory segment.
    zstyle ':vcs_info:*' nvcsformats "%k%F{$dir_bg}$SEGMENT_SEPARATOR"
    zstyle ':vcs_info:*' formats \
           "%K{$vcs_bg}%F{$dir_bg}$SEGMENT_SEPARATOR %F{$vcs_logo} %F{$vcs_fg}%b%f %m%u%c %k%F{$vcs_bg}$SEGMENT_SEPARATOR"
    zstyle ':vcs_info:*' actionformats \
           "%K{$vcs_bg}%F{$dir_bg}$SEGMENT_SEPARATOR %F{$vcs_logo} %F{$vcs_fg}%b%F{3}|%F{1}%a%f %m%u%c %k%F{$vcs_bg}$SEGMENT_SEPARATOR"
    zstyle ':vcs_info:(sv[nk]|bzr):*' branchformat '%b%F{1}:%F{3}%r'
    zstyle ':vcs_info:*' enable git svn
    zstyle ':vcs_info:*' check-for-changes true
    zstyle ':vcs_info:*' unstagedstr "%F{$vcs_unstaged_fg}●%f"
    zstyle ':vcs_info:*' stagedstr "%F{$vcs_staged_fg}✚%f"

    add-zsh-hook precmd vcs_info
    add-zsh-hook precmd prompt_piger_precmd

    # First line of the prompt:
    PROMPT=""

    # time:
    PROMPT+="%K{$time_bg}%F{$time_fg} %T "
    PROMPT+="%K{$dir_bg}%F{$time_bg}$SEGMENT_SEPARATOR "

    # current directory
    PROMPT+="%K{$dir_bg}%F{$dir_fg}%~ "

    # git status
    # NOTE: ${vcs_info_msg_0_} must be passed literally (i.e. not expanded).
    PROMPT+='${vcs_info_msg_0_}'

    # reset colors and print newline
    PROMPT+="%k%f${prompt_newline}"

    # Second line of the prompt:

    # Show the current virtualenv as set in psvar[1]
    PROMPT+='%(1V.%F{$venv_fg}(%1v)%f .)'

    # Show user@hostname if this is a SSH connection
    if [[ $SSH_CONNECTION != "" || ! -z $extended ]]; then
        PROMPT+="%F{$(prompt_piger_hostname_color)}%n%f@%F{$(prompt_piger_hostname_color)}%m%f "
    fi

    # Show background jobs:
    PROMPT+="%(1j.|%j.)"

    # Add '#' for root shells or $PROMPT_SYMBOL for regular users:
    PROMPT+="%F{$symbol_fg}%(!.#.${PROMPT_SYMBOL})%f "

    # Use RPROMPT to display the exit status of the last executed command when non-zero.
    # NOTE: "CONDITIONAL SUBSTRINGS IN PROMPTS" in zshmisc.
    # read: if the exit code of the last command was 0, show an empty string, otherwise show it (%?) with colors and bold
    RPROMPT="%(?..%B%F{$err_fg}${ERROR_SYMBOL}%f %?%b)"

    # Spell checker prompt: shown when the command does not exists but look similar to an existing one.
    SPROMPT="%B%R%b does not exists; did you mean %B%r%b (%BY%bes/%BN%bo/%BE%bdit/%BA%bbort)? "

    # set prompt options through promptinit
    prompt_opts=(cr subst percent sp)
}

prompt_piger_preview() {
    prompt_preview_theme piger "$@"
}

prompt_piger_setup "$@"
```

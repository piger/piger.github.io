---
title: "Kqueue, iTerm2 and OpenSSH"
date: 2023-05-06T20:00:00+01:00
tags:
- macOS
- ssh
- Go
categories:
- Tips and Tricks
summary: "Explore a couple of solutions to write the hostname of an ssh connection in an iTerm2 badge."
toc: true
---

## Introduction

One of iTerm2's under appreciated features is the ability to display [badges](https://iterm2.com/documentation-badges.html) in the terminal window;
these badges are basically an overlay text that is rendered behind the actual terminal text, and can be really useful to give _context_ to a terminal
window.

In this post I'll describe how to integrate this feature with OpenSSH to display the remote hostname during an `ssh` session; the picture below shows
how it looks like on my computer:

{{< image src="/images/iterm2-badge.png" caption="A screenshot of iTerm2 with a badge" >}}

## A word of warning

Adding something that you don't fully understand to your everyday configuration might lead to unpleasant surprises, which is something that tend to
happen at the worst possible time. One of the _tricks_ that I'll mention later in this post, for example, had a subtle bug that went unnoticed for
years, until one day it manifested itself while I was working (of course!).

On the other hand having the hostname of the remote machine clearly displayed in my terminal's window is exactly the kind of things that I find very
useful at work, and I think it was worth the effort of researching, testing and implementing.

## Implementation

To display a badge in iTerm2 you need to _echo_ a special escape sequence; with a shell like bash or zsh, for example, you can try this:

```bash
printf "\e]1337;SetBadgeFormat=%s\a" "$(echo mymachine.net | base64)"
```

and to clear the badge you can simply print the same escape sequence without any text:

```bash
printf "\e]1337;SetBadgeFormat=\a"
```

Knowing this, the two remaining piece of the puzzle are:

- detecting the hostname of the machine we're connecting to with ssh
- print the escape sequence when ssh connects to the remote machine

Also, since we like nice things that works properly, it would be nice to find a way to clear the badge when the remote connection is closed.

### Doing it with a (zsh) shell script

The first option I'll describe uses a shell script, since after all this seems exactly the kind of problem shells are meant to solve. I've already
written about this in the past (in [two]({{< ref "/posts/01-tmux-ssh-rename.md" >}}) [different]({{< ref "/posts/19-tmux-iterm2-ssh-rename.md" >}})
posts), so here I'll just describe the strategy.

First we need to intercept the execution of the `ssh` command, and to do so we can define a _shell function_ with the same name that then calls the
actual command. With the proper separation of _general_ shell configuration and _interactive_ shell configuration, we can ensure that this function
will only be used in **interactive** shells and not in **shell scripts**; in Zsh this means that we won't load this shell function in `zshenv` and
instead we will load it in `zshrc`.

The actual complicate part of this approach is finding a way to extract the remote hostname from the command-line, taking into account that we could
be simply typing `ssh example.com` or we would be passing options to `ssh` itself; the initial version of my script used Perl's regular expressions to
do that:

```bash
local hostname=$(perl -e 'if ($#ARGV == 0) { print $ARGV[0]; } else {foreach $h (@ARGV) { if ($h =~ /^(?<!-)(?:\w+@)?([^.]+\.[^.]+\..*)/) { print $+; exit; } } }' -- $@)
```

Here we're assigning to `hostname` the output of a Perl script, to which we're passing `$@` as the input; `$@` is a shell positional parameter
containing all command-line arguments.

Breaking down this Perl oneliner we have:

```perl
if ($#ARGV == 0) {
    print $ARGV[0];
} else {
    foreach $h (@ARGV) {
        if ($h =~ /^(?<!-)(?:\w+@)?([^.]+\.[^.]+\..*)/) {
            print $+;
            exit;
        }
    }
}
```

This script iterates over `@ARGV` (again, the command-line arguments) when its length is major than 0; for each string in `@ARGV` we check with a
regular expression if it match what an hostname usually looks like.

{{< admonition type=tip >}}
To learn more about Perl's regular expression check the `perlre` man page.
{{< /admonition >}}

Let's ask ChatGPT to explain this regular expression:

- `/^` matches the start of the string.
- `(?<!-)` is a negative lookbehind assertion that matches the position where the previous character is not a dash (`-`).
- `(?:\w+@)?` is a non-capturing group that matches an optional word (alphanumeric characters and underscore) followed by an at symbol `@`. The `?` at the end of the group makes it optional.
- `([^.]+\.[^.]+\..*)` is a capturing group that matches a domain name. It matches the following:
  - `[^.]+` matches one or more characters that are not a period `.`.
  - `\.` matches a period `.`.
  - `[^.]+` matches one or more characters that are not a period `.`.
  - `\.` matches a period `.`.
  - `.*` matches any remaining characters in the string.

> So, the regular expression matches a string that starts with a domain name, optionally preceded by a word and an at symbol, and the domain name must
> not begin with a dash "-". For example, it would match "example.com" and "subdomain.example.co.uk", but not "-example.com" or "example.".

Unfortunately we can already see a flaw in this approach: the regexp won't match a single word hostname, like `localhost`; it's not uncommon to
configure OpenSSH to alias an hostname to a friendly single word name, like `db` for `db.example.com`. Nevertheless this approach was good enough and
I used it for a while.

For the next version of this script I copied an approach that I saw in someone else's dotfiles, which I found quite clever: use `getops` to simulate
the command-line arguments of OpenSSH. This way we can be sure that by the time we finish parsing all the _option_ arguments, the next one will be the
actual remote hostname:

```bash
while getopts ":1246AaCfGgKkMNnqsTtVvXxYyb:c:D:E:e:F:I:i:L:l:m:O:o:p:Q:R:S:W:w:" option; do
done

local hostname=""
eval hostname="\${$OPTIND}"
OPTIND=1
```

The last iteration of this script offloads the whole problem to OpenSSH by leveraging the `-G` option:

> `-G` Causes ssh to print its configuration after evaluating Host and Match blocks and exit.

If we prepend `-G` to the command-line arguments for `ssh`, the output will look like this:

```
user myuser
hostname meriadoc.lan
port 22
addressfamily any
batchmode no
[...]
```

And so we can easily get what `ssh` considers to be the remote hostname using `awk`:

```shell
ssh -G "$@" | awk 'NR > 2 { exit } /^hostname/ { print $2 }'
```

The output of `ssh -G` follows a fixed order, so we know that we can exit `awk` after the 2nd line (`NR > 2 { exit }`); we match any line starting
with `hostname` and we print the second token of that line, the remote hostname.

Technically we're done with the problem of detecting the remote hostname, but I had another requirement: the infrastructure that I managed at that
time had machines with long hostnames that described some of their metadata, and I didn't want to display all of that.

A typical hostname had the following components:

- the machine's "cattle" name (e.g. `proxy1`)
- a partition ID (e.g. `group1`)
- an geographical region, similar to what AWS uses (e.g. `use1` for `us-east-1`)
- a domain indicating whether the machine was in a testing or a production environment
- the top-level domain

Rather than having `proxy1.group1.use1.testing-example.com`, I wanted to have a shorter version: `proxy1.group1.use1`; do achieve that I wrote a
simple algorithm in Zsh:

```shell
local parts=(${(s:.:)hostname})
local short_name

# if it's an IP don't shorten it and use it as it is
if echo $hostname | grep -qE "^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$"; then
    short_name=$hostname
# if the parts of the hostname are less than 3 (e.g. mail.example.com -> 3 parts) use the first part
# only (e.g. "mail").
elif (( $#parts <= 3 )); then
    short_name=${parts[1]}
else
    # othewrise show 2/3 of the hostname
    local fraction=$(( $#parts / 3 * 2 ))
    short_name=${(j:.:)parts[1,$fraction]}
fi
```

Now we can _finally_ print the escape sequence for iTerm2 and display the short remote hostname in a badge. You can find this version of this script
in my dotfiles repository on GitHub: [ssh function](https://github.com/piger/Preferences/blob/e0002cec9eb0ff0a9b1fefc92c7bba16ae031ef3/zsh/functions/ssh).

So far this worked quite well for me, except for a small bug that I failed to notice for a long time: executing `ssh` without any option or parameter
would cause the whole shell session to terminate. This happens when the script fails to detect the remote hostname and executes `exec command ssh $@`;
`exec` replaces the current process with another one, and since that process terminates immediately, so does your terminal window.

One fix is to avoid using `exec` and instead calling `return` after invoking `ssh`, to exit the script.

### Doing it with kqueue

If you read the man page `ssh_config` you will notice that OpenSSH has a `LocalCommand` option:

> Specifies a command to execute on the local machine after successfully connecting to the server. The command string extends to the end of the line,
> and is executed with the user's shell. Arguments to LocalCommand accept the tokens described in the TOKENS section.
>
> The command is run synchronously and does not have access to the session of the ssh(1) that spawned it. It should not be used for interactive
> commands.

Among the available _TOKENS_ we have `%h` which contains the remote hostname. It would seem that by using `LocalCommand` we could really simplify the
solution to our problem... except for one small detail: the command is run **synchronously**, so we can set the badge after the connections is
established, but we can't unset it when the connection ends, as our `LocalCommand` must terminate to let `ssh` continue its execution.

As an aside, knowing how OpenSSH executes the command in `LocalCommand` might help our investigation, so let's [take a look](https://github.com/openssh/openssh-portable/blob/0e9e2663eb2c6e9c3e10d15d70418312ae67e542/sshconnect.c#L1654):

```C
int
ssh_local_cmd(const char *args)
{
	char *shell;
	pid_t pid;
	int status;
	void (*osighand)(int);

	if (!options.permit_local_command ||
	    args == NULL || !*args)
		return (1);

	if ((shell = getenv("SHELL")) == NULL || *shell == '\0')
		shell = _PATH_BSHELL;

	osighand = ssh_signal(SIGCHLD, SIG_DFL);
	pid = fork();
	if (pid == 0) {
		ssh_signal(SIGPIPE, SIG_DFL);
		debug3("Executing %s -c \"%s\"", shell, args);
		execl(shell, shell, "-c", args, (char *)NULL);
		error("Couldn't execute %s -c \"%s\": %s",
		    shell, args, strerror(errno));
		_exit(1);
	} else if (pid == -1)
		fatal("fork failed: %.100s", strerror(errno));
	while (waitpid(pid, &status, 0) == -1)
		if (errno != EINTR)
			fatal("Couldn't wait for child: %s", strerror(errno));
	ssh_signal(SIGCHLD, osighand);

	if (!WIFEXITED(status))
		return (1);

	return (WEXITSTATUS(status));
}
```

Essentially what happens here is the typical fork+exec, with the _local command_ being executed by the user's shell with the `-c` flag.

{{< admonition type=note >}}
The first part of this post wasn't really macOS specific, since many terminals and terminal multiplexers allow to alter the title of a window or a
panel by echoing escape sequences; the following part though only applies to BSDs and derivates, like Darwin. There might be a way to achieve the same
thing in Linux, but that will be left as an exercise for the reader :)
{{< /admonition >}}

In a StackOverflow [answer](https://serverfault.com/a/641004) by the user [wfaulk](https://serverfault.com/users/14858/wfaulk) I learned about a nice
trick: you can use `kqueue` to have a program wait for another program to terminate and receive an _event_ when that happens; specifically, we want
to set up a kqueue filter for `EVFILT_PROC`:

> Takes the process ID to monitor as the identifier and the events to watch for in fflags, and returns when the process performs one or more of the
> requested events. If a process can normally see another process, it can attach an event to it.

and we want to watch for the event `NOTE_EXIT`, which is emitted when the monitored process exits.

The StackOverflow [answer](https://serverfault.com/a/641004) shows how to do it in C, so if that's ok for you, the only other missing piece is to find
a way to encode the string that will be sent to iTerm2 in base64; for that you could use something like [b64.c](https://github.com/jwerle/b64.c).

In Go we need to venture into the [syscall](https://pkg.go.dev/syscall) package, from which we need:

```Go
func Kqueue() (fd int, err error)

func Kevent(kq int, changes, events []Kevent_t, timeout *Timespec) (n int, err error)

type Kevent_t struct {
	Ident  uint64
	Filter int16
	Flags  uint16
	Fflags uint32
	Data   int64
	Udata  *byte
}
```

{{< admonition type=tip >}}
The easiest way to read the Go documentation for these functions and structs is to run `go doc` on a macOS machine;
you'll also need to keep the man page for `kqueue` handy.
{{< /admonition >}}

First we need a _kqueue file descriptor_:

```Go
kq, err := syscall.Kqueue()
```

Then we need to create a `syscall.Kevent_t` describing the kind of event we want to monitor:

```Go
var ppid uint64 = 12345 // in the real program this will be our parent's pid.

event := syscall.Kevent_t{
	Ident:  ppid,
	Filter: syscall.EVFILT_PROC,
	Flags:  syscall.EV_ADD | syscall.EV_ONESHOT,
	Fflags: syscall.NOTE_EXIT,
	Data:   0,
	Udata:  nil,
}
```

With this struct we're telling the kernel that we want to add a new process filter, have it only trigger once (`EV_ONESHOT`) and only monitor for the
`NOTE_EXIT` event. Once we have this we need to set up a container for the event that we will receive from kqueue and then finally instantiate the
filter:

```Go
events := make([]syscall.Kevent_t, 1)

nev, err := syscall.Kevent(kq, []syscall.Kevent_t{event}, events, nil)
if err != nil {
	return fmt.Errorf("failed kevent(): %w", err)
}

if nev < 1 {
	return errors.New("no events returned")
}
```

Note that we're using `nil` for the last parameter, which expects a pointer to a `syscall.Timespec`; this is necessary if we want our kqueue filter to
expire after a set amount of time. In this case I'm fine with having no timeout at all, so we can pass `nil`.

The first return value of `syscall.Kevent`, `nev`, contains the number of events read, so we have to check if it reports at least one.

So now we have a way to detect when a process exits, and if we expect our program to be executed as sub-process of ssh, we can then use `os.Getppid()`
to get our parent's pid.

The escape sequence for iTerm2 is easy to implement, we only need to convert `\e` from the escape sequence in something that a non-shell can
understand; in this case `\e` corresponds to the escape sequence for the _escape_ character, which we can represent with `\033`.

```Go
func setItermBadge(msg string) {
	fmt.Printf("\033]1337;SetBadgeFormat=%s\a", base64.StdEncoding.EncodeToString([]byte(msg)))
}

func clearItermBadge() {
	fmt.Printf("\033]1337;SetBadgeFormat=\a")
}
```

The last piece of our puzzle is also the most complicate: ssh executes `LocalCommand` synchronously, but we really want our program to be asynchronous
and in Go we just can't use `fork()`. Well, technically we can, but doing so will break the Go runtime; what happens in practice is that if we run
something like `syscall.Syscall(syscall.SYS_FORK, 0, 0, 0)` and then in the child process we try to set up a Kqueue filter, we'll get an "interrupted
syscall" error message. So yeah, we can't really fork in the same way the [C program](https://serverfault.com/a/641004) in StackOverflow is doing.

Now wait... when we looked at the OpenSSH code that executes the local command, we saw that is using the equivalent of `bash -c`, so could we just
specify a `LocalCommand` that terminates with `&` and let the shell take care of running our command in background? Let's see.

We'll write a simple Go program that prints its parent PID, and run it as a `LocalCommand` in ssh, first in the regular way and then with `&` and see
what happens.

```Go
package main

import (
	"fmt"
	"os"
)

func main() {
	fmt.Printf("ppid: %d\n", os.Getppid())
}
```

In `$HOME/.ssh/config` we put this configuration, making sure that other `Host` lines won't interfere with our experiment:

```
Host *
  PermitLocalCommand yes
  LocalCommand /path/to/goppid/goppid
```

When we ssh to a remote machine we'll get something like this:

```
$ ssh meriadoc
ppid: 53169
Linux meriadoc 6.1.21-v8+ #1642 SMP PREEMPT Mon Apr  3 17:24:16 BST 2023 aarch64
```

Where 53169 is the PID of that `ssh` command. Now let's see what happens if we run the same `LocalCommand` with an `&` at the end: `LocalCommand
/path/to/goppid/goppid &`:

```
$ ssh meriadoc
ppid: 1
       Linux meriadoc 6.1.21-v8+ #1642 SMP PREEMPT Mon Apr  3 17:24:16 BST 2023 aarch64
```

The first difference is that the output from ssh is _confused_ and now the length of our Go program's output is prepended as whitespace to the actual
output from ssh; most importantly, the reported parent PID is now 1. In Linux pid 1 is `init` while in macOS is `launchd` and either way that's
definitely not the `ssh` command.

This generally happens when a parent process exits without properly waiting for its child processes to exit as well;
in this case the kernel will assign pid 1 as the parent process of the orphaned process.

So we can't use `fork()` like in C and we can't create a background process with the shell. What other options do we have? We know that `fork()` will
interfere with Go's runtime and the child process will misbehave, but nothing prevents us from using `fork` _and_ `exec` to spawn another process from
our Go program, which means that we can write a program that:

- gets its parent's pid (the `ssh` command)
- re-executes itself passing this pid as a command-line parameter
- exit without waiting for its _clone_ to terminate, to avoid blocking the execution of `ssh`

The last important detail is about `exec.Command`'s behavior in Go: an `exec.Cmd` struct will connect its Stdout and Stderr to `/dev/null`, unless we
attach them to something. We need iTerm2 to _see_ our escape sequence, so the parent Go process must pass its Stdout to its child.

There's just one thing which I'm not sure about: should the child process be placed in its own process group? From my testing so far it doesn't really
seems to make any difference.

I actually lied earlier as there's yet another important bit to consider, which also should make you realize why all of this is a fun learning
exercise but not a good idea: what happens when `ssh` is not executed by a human, but is instead executed by another program, like for example the
Emacs package [magit](https://magit.vc/) or [Ansible](https://www.ansible.com/)? Bad things, because the escape sequence for iTerm2 will be mixed in
ssh own output! Can we avoid this issue by linking the parent's Stderr to the child's Stdout? iTerm2 doesn't seems to mind, but a better solution is
not print anything at all when the destination stream is not a Terminal.

In Go we can check if a stream is a Terminal using [term.IsTerminal](https://pkg.go.dev/golang.org/x/term#IsTerminal), so our main process can just
check if its `os.Stdout` is a Terminal and exit early when it's not.

You can find the source code of this program on my GitHub: [https://github.com/piger/ssh-iterm2-badge](https://github.com/piger/ssh-iterm2-badge); to
use it you need to configure OpenSSH accordingly:

```
Host *
  PermitLocalCommand yes
  LocalCommand ssh-iterm2-badge %h
```

You just need to ensure that no other `Host` directive is setting another `LocalCommand` or disabling `PermitLocalCommand` before this configuration
block is reached.

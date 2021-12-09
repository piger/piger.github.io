---
title: Weird perl errors on OS X 10.10 (Yosemite)
date: 2014-10-20
tags:
- osx
- perl
- yosemite
summary: Somehow during the Yosemite installation some permissions on Perl directories got messed up.
categories:
- Tips and Tricks
---

```
Can't locate strict.pm: Permission denied at ...
```

You run some command involving Perl and...:

    $ ack foo
    Can't locate strict.pm:   Permission denied at /usr/local/bin/ack line 14.
    BEGIN failed--compilation aborted at /usr/local/bin/ack line 14.

or you try to install something funky with homebrew and you got some weird `Permission denied` errors on *automake-something.pm*. Apparently nobody else on the internet is experiencing the same problem.

Given that I sadly remember almost nothing about Perl I had to look up how to print Perl's *include* directories:

```
$ perl -wle'print for @INC'
/Library/Perl/5.18/darwin-thread-multi-2level
/Library/Perl/5.18
/Network/Library/Perl/5.18/darwin-thread-multi-2level
/Network/Library/Perl/5.18
/Library/Perl/Updates/5.18.2
/System/Library/Perl/5.18/darwin-thread-multi-2level
/System/Library/Perl/5.18
/System/Library/Perl/Extras/5.18/darwin-thread-multi-2level
/System/Library/Perl/Extras/5.18
.
```

to find out the directory with the wrong permissions:

    $ ls /Library/Perl/Updates/5.18.2
    ls: /Library/Perl/Updates/5.18.2: Permission denied
    $ sudo ls -ld /Library/Perl/Updates/
    drwxr-x---  3 root  wheel  102 19 Ott 12:38 /Library/Perl/Updates/

On a Linux system I would have used **strace**, but on OS X things are a little different:

    # dtruss -f 'su MYUSER -c perldoc perlre' 2>&1 | grep -C 3 Permission
    ...
    66887/0x34fda:  stat64("/Network/Library/Perl/5.18/Pod/Perldoc.pm\0", 0x7FFF51E0C960, 0x2000)            = -1 Err#2
    66887/0x34fda:  stat64("/Library/Perl/Updates/5.18.2/Pod/Perldoc.pmc\0", 0x7FFF51E0CA10, 0x2000)                 = -1 Err#13
    66887/0x34fda:  stat64("/Library/Perl/Updates/5.18.2/Pod/Perldoc.pm\0", 0x7FFF51E0C960, 0x2000)          = -1 Err#13
    66887/0x34fda:  write(0x2, "Can't locate Pod/Perldoc.pm:   Permission denied at /usr/bin/perldoc5.18 line 10.\nBEGIN failed--compilation aborted at /usr/bin/perldoc5.18 line 10.\n\004\b\0", 0x95)                 = 149 0
    ...

As you can see from the result of the `stat64(2)` calls there are two different type of errors (`Err#2` and `Err#13`), one for a non-existent file and one for two files which our user can't access. The first `stat64` call checks for a compiled version (`.pmc`) of the Perldoc library for which a **permission denied** error is negligible, but the same error on the `.pm` file is an hard error, the one which I was getting at the beginning.

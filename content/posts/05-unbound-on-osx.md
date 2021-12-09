---
title: Install unbound DNS(SEC) resolver on OS X
date: 2013-08-05
tags:
- dnssec
- dns
- macOS
summary: Describe the installation of unbound DNS(SEC) resolver on OS X.
categories:
- System Administration
---

Here's a little guide that explain how to install [unbound](https://www.unbound.net) 1.4.20 on the latest version of Mac OS X (10.8.4 at this time); as a bonus you will learn how to create system users on OS X :)

Please note that the setup here described is tailored for a workstation use; we will also enable the resolution of local DNS zones through a local DNS server (usually the wi-fi router).

In the following instructions we assume a LAN address space of 10.0.1.0/24 with our local DNS server on 10.0.1.1.

To install [unbound](https://www.unbound.net) you can use [homebrew](http://brew.sh/) or [macports](https://www.macports.org/):

    $ brew install unbound

Unbound will run as a system daemon so we need to create a new user and group. First we need to find a free unique id for our user and group in the range 1-500 (this range is reserved for system accounts); pick a random number between 1 and 500 and check if it's taken, for example \`451\`: :

    $ dscl . -list /Groups PrimaryGroupID | grep 451
    $ dscl . -list /Users PrimaryGroupID | grep 451

If both commands doesn't show any output we are safe to use 451 for our user and group id.

To create the group and the user for [unbound](https://www.unbound.net) we run the following commands:

    $ sudo dscl . -create /Groups/_unbound
    $ sudo dscl . -create /Groups/_unbound PrimaryGroupID 451
    $ sudo dscl . -create /Users/_unbound
    $ sudo dscl . -create /Users/_unbound RecordName _unbound unbound
    $ sudo dscl . -create /Users/_unbound RealName "Unbound DNS server"
    $ sudo dscl . -create /Users/_unbound UniqueID 451
    $ sudo dscl . -create /Users/_unbound PrimaryGroupID 451
    $ sudo dscl . -create /Users/_unbound UserShell /usr/bin/false
    $ sudo dscl . -create /Users/_unbound Password '*'
    $ sudo dscl . -create /Groups/_unbound GroupMembership _unbound

Now we can edit the configuration file of [unbound](https://www.unbound.net) which by default is located in \`/usr/local/etc/unbound/unbound.conf\`:

    server:
      verbosity: 1
      interface: 127.0.0.1
      access-control: 127.0.0.1/8 allow
      chroot: ""
      private-address: 10.0.1.0/24
      private-domain: "my.lan"
      domain-insecure: "my.lan"
      username: "_unbound"
      auto-trust-anchor-file: "/usr/local/etc/unbound/root.key"

    python:

    remote-control:
      control-enable: yes
      control-interface: 127.0.0.1
      server-key-file: "/usr/local/etc/unbound/unbound_server.key"
      server-cert-file: "/usr/local/etc/unbound/unbound_server.pem"
      control-key-file: "/usr/local/etc/unbound/unbound_control.key"
      control-cert-file: "/usr/local/etc/unbound/unbound_control.pem"

    stub-zone:
      name: "my.lan"
      stub-addr: 10.0.1.1

You can tell unbound about local domains with the private-domain parameter; in this configuration we are specifying my.lan as a private domain.

If you wish to enable DNS forwarding to an external DNS server you can specify one with a catch-all forward zone; for example to use Google Public DNS as a forwarder add this to the bottom of \`unbound.conf\`: :

    forward-zone:
      name: "."
      forward-addr: 8.8.8.8
      forward-addr: 8.8.4.4

In the next step we will fetch the root key needed for DNSSEC validation: :

    $ sudo unbound-anchor -a /usr/local/etc/unbound/root.key

Now we must create the certificate files needed by the unbound-control utility: :

    $ sudo unbound-control-setup -d /usr/local/etc/unbound

The unbound process needs read and write permission for the configuration directory; we use staff as group so our user will be able to run \`unbound-control\`: :

    $ sudo chown -R _unbound:staff /usr/local/etc/unbound
    $ sudo chmod 640 /usr/local/etc/unbound/*

To start [unbound](https://www.unbound.net) at boot time we need to create a plist file in \`/Library/LaunchDaemons/net.unbound.plist\`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>net.unbound</string>
    <key>ProgramArguments</key>
    <array>
  <string>/usr/local/sbin/unbound</string>
  <string>-d</string>
  <string>-c</string>
  <string>/usr/local/etc/unbound/unbound.conf</string>
    </array>
    <key>KeepAlive</key>
    <true/>
    <key>RunAtLoad</key>
    <true/>
  </dict>
</plist>
```

To start [unbound](https://www.unbound.net) now we must load the plist with launchctl (be aware that you must execute launchctl outside of tmux or *proxied* by [reattach-to-user-namespace](https://github.com/ChrisJohnsen/tmux-MacOSX-pasteboard)): :

    $ sudo launchctl load /Library/LaunchDaemons/net.unbound.plist

To test our setup we can follow the examples on the [DNSSEC](https://wiki.debian.org/DNSSEC#Test_DNSSEC) page on the Debian wiki: :

    $ dig org. SOA +dnssec @127.0.0.1
    ; <<>> DiG 9.8.3-P1 <<>> org. SOA +dnssec @127.0.0.1
    ;; global options: +cmd
    ;; Got answer:
    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 49736
    ;; flags: qr rd ra ad; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1
    ...

If you see **ad** in the *flags* field then DNSSEC is working.

You can also do a *TXT* query for test.dnssec-or-not.net to get a verbose confirmation that you are using DNSSEC; be aware that this test will fail if you are using an external DNS forwarder: :

    $ dig test.dnssec-or-not.net TXT @127.0.0.1

Now you should be all ready to use 127.0.0.1 as your DNS resolver and benefit from DNSSEC; to see if everything is going right you can check the system Console for errors.

There are still some things that could be done better, for example we could specify a chroot to isolate the [unbound](https://www.unbound.net) process; I've also encountered problems running unbound-control reload and my Console always shows this line:

> unbound\[35549\]: \[35549:0\] warning: did not exit gracefully last time (34581)

Hoepfully this post will be updated as I figure out how to resolve this problems :)

Some useful resources:

- [Installer unbound sous MacOSX](http://docs.chezwam.org/docs/2012/08/10_installer-unbound-sous-macosx.html) (where dscl commands come from :)
- [DNSSEC Validator add-on for browsers](http://www.dnssec-validator.cz/)
- [dnssec-trigger](https://www.nlnetlabs.nl/projects/dnssec-trigger/)

---
title: "Notes on Tailscale"
date: 2021-12-13T16:05:21Z
tags:
- networking
- linux
categories:
- Tutorial
summary: "The pleasure of a private network with Tailscale and how to configure ACLs."
toc: false
---

This is a post about [Tailscale](https://tailscale.com/), a _secure network that just works_ built
on top of [Wireguard](https://www.wireguard.com/).

The [Remembering the LAN](https://tailscale.com/blog/remembering-the-lan/) blog post by David
Crawshaw resonate with me, in particular this quote:

> I need to help new programmers who never got to experience simple, pleasurable programming in a
> safe environment understand that programming can be fun. You can set up your environment so you
> can focus on being creative.

With the Internet being a wild wild west it's refreshing to be able to enjoy tinkering with
computers in a _private_ environment, in particular when you can't keep your servers at home.

You can for example purchase a relatively cheap virtual server from
[Hetzner Cloud](https://www.hetzner.com/cloud), install Tailscale on it, then set up a simple
firewall rule to only allow inbound UDP traffic on port `41641`; now your new server is
only accessible from within your own private Tailscale network.

**NOTE**: you might want to disable _key expiry_ for this virtual server in your Tailscale control
panel.

Things of course can sometimes go wrong and it's good to have a secondary way to access your server,
which you have with the virtual console on Hetzner's control panel.

The benefit of setting up this firewall rule on Hetzner itself and not on your server is that you
don't have to deal with nasty surprises, like Docker exposing your containers on the public internet
despite your iptables rules were supposed to prevent that from happening.

For a good security posture you should also limit _outbound_ network traffic, which is a completely
different story. One thing you can do though is to leverage Tailscale's ACLs to limit what other
devices your new virtual server can access in your private network.

## ACLs in Tailscale

Tailscale's ACLs are documented in a [knowledge base article](https://tailscale.com/kb/1018/acls/),
so here I'll just provide a simple example.

The first thing to do is to assign a _tag_ to your virtual server, as documented in the
[Server role accounts with ACL tags](https://tailscale.com/kb/1068/acl-tags/) document; it's
important to know that when you provision a new device in Tailscale it is initially tied to your
user's permissions. If your ACL rules say "user me@github can access any device on port 22" it means
that any device provisioned by "me@github" is inheriting those permissions.

Assigning tags to a device allow you to have more fine grained permissions, because it won't inherit
your user's permissions anymore.

We can for example assign the tag `internet` to our new virtual server by running the following
command on your new virtual server:

```
sudo tailscale up --advertise-tags=tag:internet
```

If you now remove the "allow-all" ACL rule from your Tailscale configuration your new virtual server
won't have access to any device on your network; for reference this is the "allow-all" rule:

```json
{ "action": "accept", "users": ["*"], "ports": ["*:*"] },
```

Side note: in Emacs you can use `jsonc-mode` to edit Tailscale's ACL rules with syntax highlighting.

### A more realistic example

For a better example I'll describe the ACL rules for a small Tailscale network comprising three
devices: a laptop, a Raspberry Pi server and a virtual server on the internet.

First we define a group called `admin` that initially will only contain our user (`me@github`):

```json
"groups": {
  "group:admin": [
    "me@github"
  ],
}
```

Then we need to tell Tailscale that our user is the owner of the `internet` tag:

```json
"tagOwners": {
  "tag:internet": ["me@github"],
},
```

Then, since the Raspberry Pi doesn't have a tag, we need to add a `hosts` entry to be able to
reference it later on (**NOTE**: you must replace `100.100.100.100` with the Tailscale IP of your
Raspberry Pi):

```json
"hosts": {
  "raspberry-pi": "100.100.100.100",
},
```

Finally we can define the actual ACLs:

```json
"acls": [
  // Allow ALL:
  // { "action": "accept", "users": ["*"], "ports": ["*:*"] },
  { "action": "accept", "users": ["group:admin"], "ports": ["*:*"] },
  { "action": "accept", "users": ["tag:internet"], "ports": ["raspberry-pi:9100"] },
],
```

The first rule is the commented "allow-all" rule that I'll keep around in case I need it while in a
hurry; then we have two rules:

- the first rule allow any device _identified_ as a user in the `admin` group to access everything;
  this means any untagged device, so in this example the "laptop" and "raspberry-pi".
- the second rule allow any device tagged with `internet` to access port `9100` on the Raspberry Pi.

To summarise the rules we have defined:

- the "laptop" and "raspberry pi" devices can access all ports of any device in our Tailscale
  network
- the virtual server (by the virtue of being tagged with the tag `internet`) can only access the
  "raspberry pi" device on port `9100`

## Logging

If you run Tailscale on a Raspberry Pi you might want to consider limiting the amount of logging
produced by the `tailscaled` daemon which is written on the SD card; if you're running the Raspberry
Pi OS (i.e. Debian) and you want to just drop all `tailscaled` logs then write the following
configuration in `/etc/rsyslog.d/10-tailscaled.conf`:

```
:programname, isequal, "tailscaled" stop
```

Restart `rsyslog` to apply the configuration:

```
sudo systemctl restart rsyslog
```

## Closing words

As a person who played with other VPN softwares in the past, like IPSEC, OpenVPN and tinc, the
simplicity of [Wireguard](https://www.wireguard.com/) feels incredible. What Tailscale adds to that
is ease of use and nice features that really bring back the joy of the LAN.

[Donate to Wireguard](https://www.wireguard.com/donations/) if you can.

[Get Tailscale for free](https://tailscale.com/pricing/) or check their pricing page.

---
title: Wireguard using a secondary egress IP
date: 2019-12-13
tags:
- vpn
- wireguard
- linux
- iptables
summary: How to use a secondary IP for Wireguard outgoing traffic
categories:
- Tutorial
---

I installed [Wireguard](https://www.wireguard.com/) on a machine that has two public IPs and I
wanted to use the secondary IP address for all the VPN's outgoing traffic; it turned out to be way
more difficult than I expected, given also my scarce expertise in Linux advanced
networking. Nevertheless I found a simple solution: SNAT.

If you're using `ferm` you can write something like this:

```
table nat {
  chain POSTROUTING {
    saddr $WIREGUARD_NETWORK_V4 outerface $DEV_INTERNET SNAT to $OTHER_PUBLIC_IP4;
    saddr $WIREGUARD_NETWORK_V4 MASQUERADE;
  }
}
```

Where `$WIREGUARD_NETWORK_V4` is the CIDR for your Wireguard local network (e.g. `192.168.0.0/24`),
`$DEV_INTERNET` is your public network interface (e.g. `eth0`) and `$OTHER_PUBLIC_IP4` is your
secondary public IP (e.g. `1.2.3.4`).

The above `ferm` configuration translates to the following iptables rules:

```
iptables -A POSTROUTING -s 192.168.0.0/24 -o eth0 -j SNAT --to-source 1.2.3.4
iptables -A POSTROUTING -s 192.168.0.0/24 -j MASQUERADE
```

I'm sure that there's a better or more elegant way to achieve the same result, but so far this has
been working just fine.

---
title: ferm and docker playing together
date: 2018-12-02
summary: how to make ferm compatible with the dynamic iptables rules created by Docker
tags:
- docker
- linux
- iptables
- ferm
categories:
- Tutorial
---

When it comes to firewalling I always had a strong preference on [PF](https://www.openbsd.org/faq/pf/) over iptables for a very simple but fundamental reason: PF uses a configuration file while iptables is just _executed_.

While the kernel developers [try to figure out](https://lwn.net/Articles/747551/) which firewall implementation is the best for the future and while my [Linux distribution of choice](https://www.debian.org) still defaults to iptables, the closest thing to configure your firewall rules with a configuration files is [ferm](http://ferm.foo-projects.org/).

As a result of still not having a configuration file for iptables in the year 2018, any software that requires _dynamic_ firewalling rules will resort to brutally execute the iptables command to create the rules it requires, which doesn't really play nice with ferm, since there is no easy way to translate an iptables rule to the ferm syntax.

My problem is that refreshing (e.g. `systemctl reload ferm`) the firewall rules would clear all the dynamic rules created by Docker, breaking the networking for all the running containers; this is made even more difficult because the firewall rules created by Docker refer to network interfaces created with unpredictable names (although I could be wrong on this last bit).

[Several](https://github.com/diefans/ferment) [people](https://blog.urth.org/2018/06/01/making-docker-play-nice-with-ferm-firewalls-on-linux/) [tried](https://unrouted.io/2017/08/15/docker-firewall/) to [solve](https://github.com/wikimedia/puppet/commit/74050c6233c8b5ae291d3d7f5131a587941c50ac) this [problem](https://github.com/moby/moby/issues/12294#issuecomment-432921518), but none of the solutions I found was compatible with my setup, or up to date with the current Docker version, so today I took some time to implement my own solution.

The solution that follows is not particularly elegant and was tested on:

- Debian Stretch (9.6)
- Docker CE 18.09.0
- Docker Compose 1.8.0
- Ferm 2.4

Debian 9 ships with ferm 2.3 so for this to work you have to cheat and install version 2.4 with a workaround (sadly it looks like there's no such thing as Buster backports for Stretch), like:

```
curl -O http://ftp.de.debian.org/debian/pool/main/f/ferm/ferm_2.4-1_all.deb
dpkg -i ferm_2.4-1_all.deb
```

~~Two~~ Three very simple shell scripts are used to replicate the iptables rules created by Docker once your containers are running:

`/usr/local/bin/ferm_docker_filter.sh`

``` bash
#!/bin/bash

set -e

# table filter
# chain FORWARD
for address in $(docker network ls -f driver=bridge --format '{{ .ID }}' | xargs -n1 docker network inspect -f '{{range .IPAM.Config }}{{ .Gateway }}{{ end }}'); do
    ifname=$(ip -o a l | grep $address | awk '{ print $2 }')
    cat <<EOF
outerface $ifname {
  mod conntrack ctstate (RELATED ESTABLISHED) ACCEPT;
  jump DOCKER;
}
interface $ifname {
  outerface ! $ifname ACCEPT;
  outerface $ifname ACCEPT;
}
EOF
done
```

`/usr/local/bin/ferm_docker_nat.sh`

``` bash
#!/bin/bash

set -e

# table nat
# chain POSTROUTING
for address in $(docker network ls -f driver=bridge --format '{{ .ID }}' | xargs -n1 docker network inspect -f '{{range .IPAM.Config }}{{ .Gateway }}{{ end }}'); do
    ifname=$(ip -o a l | grep $address | awk '{ print $2 }')
    cat <<EOF
saddr ${address}/16 outerface ! $ifname MASQUERADE;
EOF
done
```

`/usr/local/bin/ferm_docker_ports.sh`

``` bash
#!/bin/bash

# can't use "set -e", some containers might not have ports forwarded and I'm too lazy to filter them out.

# Example rule to create:
# -A POSTROUTING -s 172.22.0.2/32 -d 172.22.0.2/32 -p tcp -m tcp --dport 32469 -j MASQUERADE

docker ps | awk '$0 !~ /^CONTAINER/ { print $1 " " $2}' | while read -r id name; do
    ip=$(docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $id)
    docker inspect --format='{{json .NetworkSettings.Ports}}' $id | jq -r 'to_entries[] | (.key | split("/")[0]) as $sport | (.key | split("/")[1]) as $proto | $sport + " " + $proto + " " + .value[].HostPort' | while read -r sport proto dport; do
        echo "saddr ${ip}/32 daddr ${ip}/32 proto $proto dport $dport mod comment comment \"${name}\" MASQUERADE;"
    done
done
```

My `ferm.conf` contains the configuration to create the basic set of rules required by Docker and then uses those two scripts to create the dynamic rules needed by my containers; please keep in mind that Docker is very flexible and your setup could be different than mine.

```

domain ip {
  table nat {
    chain DOCKER @preserve;

    chain PREROUTING {
      policy ACCEPT;
      mod addrtype dst-type LOCAL jump DOCKER;
    }

    chain OUTPUT {
      policy ACCEPT;
      daddr ! 127.0.0.0/8 mod addrtype dst-type LOCAL jump DOCKER;
    }

    chain POSTROUTING {
      policy ACCEPT;
      @include "/usr/local/bin/ferm_docker_nat.sh|";
      @include "/usr/local/bin/ferm_docker_ports.sh|";
    }

    chain INPUT policy ACCEPT;
  }

  table filter {
    chain (DOCKER DOCKER-ISOLATION-STAGE-1 DOCKER-ISOLATION-STAGE-2 DOCKER-USER) @preserve;

    chain INPUT {
      policy DROP;

      mod state {
        state INVALID DROP;
        state (ESTABLISHED RELATED) ACCEPT;
      }

      interface lo ACCEPT;
    }

    chain FORWARD {
      policy DROP;
      jump DOCKER-USER;
      jump DOCKER-ISOLATION-STAGE-1;

      @include "/usr/local/bin/ferm_docker_filter.sh|";

      mod state {
        state INVALID DROP;
        state (RELATED ESTABLISHED) ACCEPT;
      }
    }

    chain OUTPUT {
      policy ACCEPT;
      mod state state (ESTABLISHED RELATED) ACCEPT;
    }
  }
}
```

The trick is to use `@preserve` on any chain created by Docker so that ferm will just preserve it across restarts, while the two shell scripts take care of the remaining rules for the bridge interfaces. As a final note keep in mind that`@preserve` was introduced in ferm 2.4, which will appear in Debian in the next release.

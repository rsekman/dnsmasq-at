# dnsmasq-at
A dnsmasq@.service systemd unit for easy per-interface configuration of dnsmasq

## Usage

### Installation

* Copy `dnsmasq@.service` to a directory where systemd can see it (e.g., `/usr/lib/systemd/system`).
* Copy `dnsmasq@.conf` to where dbus will read it, (on Arch this is `/usr/share/dbus-1/system.d`).
 This file tells dbus to allow access to bus names prefixed with `org.uk.thekelleys.dnsmasq.` -- each instance of dnsmasq needs its own bus name.

If you are on Arch you can just use the `PKGBUILD` in the repo.

### Configuring and enabling

Write the configuration for the interface `$IF` to `/etc/dnsmasq.d/$IF.dnsmasq`.
See [dnsmasq(8)](https://man.archlinux.org/man/dnsmasq.8) for details.
```sh
systemctl enable --now dnsmasq@$IF.service
```

Note that the unit requires the interface `$IF` to become available (via `Requires=systemd-networkd-wait-online@%i.service`).
Thus `$IF` *must* be exactly the interface name you intend to listen on.

## Use case

I run a [pi-hole](pi-hole.net) on my LAN.
My router supports a guest WiFi network and I'd like guests to benefit from the pi-hole while still being cordoned off from my devices on the non-guest (private, wired) network.
This seems easy enough, since the router can allow devices on the guest network to communicate with each other.
We simply connect the pi-hole to both the
private (interface `eth0`, address `192.168.0.2`) and
guest (interface `wlan0`, address `192.168.0.130`)
networks.
Then we configure the router's DHCP server to provide both addresses as DNS servers.

The issue with this is that the router assigns addresses on both the private and guest networks to the same `/24` subnet.
The pi-hole then receives a DNS request from a guest with address, say, `192.168.0.76`, and tries to reply on `eth0`.
The router prevents the guest from ever receiving that response.

What we need to do is
* set the pi-hole to have `/25` addresses
* use the pi-hole as the DHCP server
* assign non-guest devices addresses on the `192.168.0.0/25` subnet, with DNS server `192.168.0.2`
* assign guest devices addresses on the `192.168.0.128/25` subnet with DNS server `192.168.0.130`

The pi-hole will then know which interface to reply on from the address the DNS request was received on.

Searching the web for how to split DHCP ranges by interface with dnsmasq turns up various resources that suggest using the `interface:` prefix to the `dhcp-range` option.
This feature of dnsmasq was, however, [removed in 2017](https://github.com/imp/dnsmasq/blame/770bce967cfc9967273d0acfb3ea018fb7b17522/CHANGELOG#L1587).
The modern way to achieve this is to run several instances of dnsmasq, each with its own, per-interface, configuration file.
They could minimally look something like this
```ini
/etc/dnsmasq.d/eth0.dnsmasq
---------------------------
# only bind to addresses on the interfaces we're listening on
bind-interfaces
# listen on eth0
interface eth0

# assign addresses in the 192.168.0.0/25 subnet
dhcp-authorative
dhcp-option=option:router,192.168.0.1
dhcp-range=192.168.0.3,192.168.0.126,24h
dhcp-option=option:dns-server,192.168.0.2
```

```ini
/etc/dnsmasq.d/wlan0.dnsmasq
----------------------------
bind-interfaces
interface wlan0
# dnsmasq binds to the loopback interface by default
# but the eth0 instance has already done that
except-interfaces lo

dhcp-authorative
dhcp-option=option:router,192.168.0.1
dhcp-range=192.168.0.131,192.168.0.254,24h
dhcp-option=option:dns-server,192.168.0.2
```

We then run two instances side-by-side,
```sh
dnsmasq -k --conf-file=/etc/dnsmasq.d/eth0.dnsmasq
dnsmasq -k --conf-file=/etc/dnsmasq.d/wlan0.dnsmasq
```

This repository basically just provides a unit to do this last part via systemd.

### Use case epilogue

If you paid close attention you'll've noticed that my goal was to run pi-hole, not trunk dnsmasq, because I want the filtering of pi-hole's forked dnsmasq.
So my *actual* setup is that
* `pihole-FTL.service` manages `eth0`
* `dnsmasq@wlan0.service` manages `wlan0`
*  `wlan0.dnsmasq` contains `server=192.168.0.2`

So a guest's DNS request is routed like
```
guest -> 192.168.0.130 -> 192.168.0.2 (-> 8.8.8.8)?
```
the last step occurring only if the domain is not in the pi-holes blocklist.

(Why not run two instances of pihole-FTL?
Because that's not possible. Unless you want to use some sort of virtualisation I guess, but I *don't*.)

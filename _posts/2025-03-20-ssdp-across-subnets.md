---
layout: post
title: "SSDP Across Subnets (on OpenWRT) the Easy Way"
date: "2025-03-20 09:37:04 +0000"
---

I have two VLANs in my home network; call them LAN and IoT.
My TV is connected to the IoT subnet, while my phone is connected to the LAN subnet.
My TV supports DLNA screen casting, and I want to cast to it from my phone.
Naturally, it doesn't work.
How can I make it work?

## SSDP

DLNA, and UPnP in general, uses the [Simple Service Discovery Protocol](https://en.m.wikipedia.org/wiki/Simple_Service_Discovery_Protocol).

(Note that it does not use mDNS, which powers AirPlay and other Apple services.
After installing `avahi` to my OpenWRT router and configuring the reflector,
I can cast to the TV via AirPlay just fine.
DLNA requires a different solution.)

There are existing SSDP reflector / relay solutions,
such as [alsmith/multicast-relay](https://github.com/alsmith/multicast-relay)
and [nberlee/bonjour-reflector](https://github.com/nberlee/bonjour-reflector).
They each have their own quirks, but more importantly, neither comes packaged with OpenWRT.
I want to do it with tools that comes with OpenWRT, so I have to go deeper.

### How does SSDP work

The specifications for SSDP can be found [here](https://openconnectivity.org/developer/specifications/upnp-resources/upnp/),
in the document [UPnP Device Architecture version 2.0](https://openconnectivity.org/upnp-specs/UPnP-arch-DeviceArchitecture-v2.0-20200417.pdf).
There was once a [draft RFC](https://datatracker.ietf.org/doc/html/draft-cai-ssdp-v1-03) for it,
but it never became an official RFC.

To achieve the barest of bare minimum SSDP support,
it must be possible to complete a multicast Search transaction over IPv4.

The transaction goes as follows:

- The control point (phone) sends a multicast UDP packet to 239.255.255.250:1900.
The source port is usually an ephemerous port.
- Upon receiving the packet,
the device (TV) responds with a unicast UDP packet to the source address:port of the multicast request.
The source port of the response is usually 1900,
but the specification does not require it explicitly.

So, to make this work across the two VLANs:

### Routing multicast search requests

I used [`smcroute`](https://github.com/troglobit/smcroute),
which has an [OpenWRT package](https://openwrt.org/packages/pkgdata/smcroute).

Just tell it to route multicast packets for that multicast group from LAN to IoT:

```
# /etc/smcroute.conf
# replace br-LAN and br-IoT with the actual interface names
phyint br-LAN enable
phyint br-IoT enable
mgroup from br-LAN group 239.255.255.250
mroute from br-LAN group 239.255.255.250 to br-IoT
```

[`igmpproxy`](https://openwrt.org/packages/pkgdata/smcroute) would probably also work for this.

### Allowing search responses through firewall

Add a firewall rule like this:

```
# /etc/config/firewall
config rule
        option name 'Allow SSDP responses from IoT to LAN'
        list proto 'udp'
        option src 'iot'
        option src_port '1900'
        option dest 'lan'
        option target 'ACCEPT'
```

And that's it for the basics.

There are more aspects of SSDP which is not supported by this configuration,
such as unicast Search transactions,
and device Advertisements that allow control points to passively discover devices.

Also, if there are SSDP devices in LAN,
their Advertisements will be routed into IoT.
Some people may be uncomfortable with such leakage.

Finally, do not forget that once control points have discovered devices,
it probably needs to connect to them in order to do stuff.
The firewall should be configured to allow such connections,
otherwise all is in vain.

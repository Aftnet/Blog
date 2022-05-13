---
title: "Flet's Hikari configuration"
date: 2022-05-12T22:44:30+09:00
draft: false
---

Japanese ISPs are, by and large, assholes, doing whatever they can to force their shit routers on customers.

Even NTT, the traditional monopolist and the most regulated one, which by law cannot refuse to let customers use their own hardware, hides configuration parameters to make it as difficult as possible.

Thankfully, some enterprising Japanese users have figured out configuration parameters for NTT's Flet's Hikari (フレッツ光) service and posted them online. I have aggregated this knowledge, in English and in one place for easy reference, here.

## NTT's Next Generation Network (NGN)

A term that comes up a lot when looking into this topic is NGN: this denotes NTT's fiber optic network (including VDSL for unlucky users in "mansion type" contracts) as opposed to their older ADSL and PSTN one, and it has one big difference compared to most other ISP networks: it is IPv6 *only*.

IPv4 connectivity, what 90% of the internet still uses, is provided in one of two ways, depending on one's choice of ISP:

- PPPoE tunneling, which usually gives a public IPv4 but which tends to be slow due to congestion and which is is on the way out
- IPoE, in one of three main flavors (MAP-E, MAP-T or DS-Lite): this is usually a lot faster and where investment is being made. Unless a public IPv4 is essential, this is the recommended way.

Which flavor of IPoE is used depends on ISP, and while all three are actual standards, only DS-Lite has any actual support in non Japan-only devices - so pick accordingly (I use IIJ myself). Most ISPs will allow to use PPPoE even when IPoE has been requested, and one can even contract with multiple ISPs simultaneously and establish up to three PPPoE tunnels concurrently.

Given that PPPoE is very well supported in routers, easy to setup and less performant than IPoE, I will not address it further here. One can setup basic PPPoE IPv4 only connectivity much like most ADSL ISPs in the rest of the world.

## To Hikari Denwa (光電話) or not?

With Flet's or any of the NTT collaboration providers one has the option of getting "value add" services like a VoIP home phone line, TV etc. VoIP in particular, marketed as 光電話, has a massive impact on configuration: by default, NTT provides IPv6 in a pretty fucked up way (no prefix delegation) that

- makes it difficult to impossible to have separate subnets and VLANs
- allows NTT to know exactly how many and what devices a user has connected to the internet at home
- makes it harder to set up a firewall

It's bad enough to make spending the extra 500 yen a month for VoIP a painful yet reasonable proposition even if one has zero need for the service.

The 10Gbit service, フレッツ光 X, is the exception, as it uses prefix delegation by default (VoIP is not even available with that service)

## IPv6 connectivity

IPv4 relies on this working, so it needs to be setup first.

To check things are working, one can go to [http://flets-v6.jp/](http://flets-v6.jp/): a website internal to NTT's NGN and only available via IPv6.

As a bonus, the speedtest there is the most accurate indicator of what speed one's line is capable of: if the speeds there are high but other speedtests and general performance is slow, it is a good indicator the ISP is at fault and moving can improve things.

### Without Hikari Denwa

Getting devices directly connected to the ONU to work is as easy as enabling IPV6 autoconfigration/SLAAC. One could even jack a PC/Mac directly and get a working IPv6 setup straight away (useful for testing)

Problems arise when one wants to have a firewall/router between their devices and the rest of the internet: NTT will only assign IPv6 addresses to directly connected devices, not routable prefixes, so even assigning other IPv6 addresses in the same /64 as the router's WAN interface to devices in the LAN will not work - NTT will not route incoming or response traffic to them.

The only workaround is to proxy NDP traffic between WAN and LAN interfaces (domestic routers do this behind the scenes), which is unfortunately not widely supported. The most accessible way I found is to run OpenWRT to take advantage of its [IPv6 Relay functionality](https://openwrt.org/docs/guide-user/network/ipv6/configuration#ipv6_relay)

### With Hikari Denwa

This is where the extra fees pay off: IPv6 is provided via standard prefix delegation, which is widely supported (I got the setup to work with EdgeOS, VyOS and OPNSense).

- Prefix length is /56. While I read of NTT delegating a /48 in places, requesting one has never worked for me
- NTT **requires** the DHCPv6 client's DUID to be in DUID-LL format (00:03:00:01:MAC_ADDRESS) where MAC_ADDRESS is a hex string in XX:XX:XX:XX:XX:XX format. If this is not the case NTT will not delegate a prefix
- NTT will **not** assign an IPv6 address to the interface requesting the delegation. I recommend assigning an address from the delegated prefix to the router's loopback interface as a fallback (useful for IPv4 connectivity)
- Gateway will be assigned via DHCPv6 as well

There is no plug and play with this setup: NTT does not send router advertisements, so directly attached devices will not autoconfigure IPv6 connectivity.

NTT also runs IPv4 DHCP, but it is only for VoIP: the device may get an address but it will not be able to use it to access the internet. One should be able to have something like Asterisk configure a SIP trunk using that DHCP configuration but it's not something I looked into.

## IPv4 connectivity

Once IPv6 connectivity is working, IPv4 can be set up. Fortunately, even in instances where router OSes do not directly mention DS-Lite, it can be setup via appropriately configured tunnels.

- [IPIP6 tunnels](https://docs.vyos.io/en/latest/configuration/interfaces/tunnel.html#ipip6) in VyOS and derivatives like EdgeOS 
- [GIF tunnels](https://docs.opnsense.org/manual/other-interfaces.html#gif) in OPNSense/PFSense
- There is probably an equivalent in "naked" Linux, if one wants to just setup something like a bog standard Debian for routing

In all cases, the configuration boils down to what internal and external IP addresses to use for the tunnel's local (the router) and remote (the AFTR servers in NTT's network) endpoints.

### Inner tunnel addresses

Inner tunnel (IPv4) addressing is constant and part of the DL-Lite standard itself ([rfc6333, section 5.7](https://www.rfc-editor.org/rfc/rfc6333)). On some OSes it may not be necessary to configure these.

Local address | Remote address | Subnet mask
--------------|----------------|-------------
192.0.0.2     | 192.0.0.1      | /29

### Outer tunnel addresses

Outer tunnel local endpoint is any IPv6 address assigned to the router: here is where assigning an IPv6 from the allocated prefix to the loopback interface comes in handy, since

- It's hard to impossible to configure most router OSes to do prefix delegation and static assignment on the same interface
- Using another (e.g. the LAN) physical interface's IPv6 address works only if that interface is actually connected to a switch/device

Outer tunnel remote endpoints (NTT's AFTR servers) depend on whether one is an NTT East or West customer. Choose either of the ones for your area.

NTT East (NTT東日本) | NTT West (NTT西日本)
---------------------|--------------------
2404:8e00::feed:100  | 2404:8e01::feed:100
2404:8e00::feed:101  | 2404:8e01::feed:101

## The rest

Depending on your router/OS you may still need to set up appropriate gateways and/or static routes to get connectivity working: without DHCP assigned gateways the router needs to be told where to forward packets to networks not directly connected, but that is standard configuration that applies everywhere.
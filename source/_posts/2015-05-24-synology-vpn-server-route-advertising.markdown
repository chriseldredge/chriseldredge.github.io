---
layout: post
title: "Synology VPN Server Route Advertising"
date: 2015-05-24 16:43
comments: true
categories: Synology sysadmin
---

I have a Synology at home and it's a great product. I recently installed
the VPN Server package and enabled L2TP/IPSec so I can get on my home
network when I'm roaming about.

The default options work fine if all you want to access is the Synology
system itself, but I noticed that after connecting my OS X client,
I couldn't reach any other computers on my home network.

After much googling and head scratching, I figured out that L2TP does
not have a built in mechanism for configuring classless static routes and that
VPN Server does not go out of its way to make this work out of the box.

After establishing a connection over L2TP, a client will typically issue
a DHCP Inform broadcast message over the connection requesting such routes.
I let my WiFi router do DHCP so my Synology DHCP service is disabled. I don't
know if it would reply to these messages over VPN or not.

Anyway, since the DHCP Inform message is sent on the L2TP virtual network,
it isn't rebroadcast on the local LAN, so my WiFi router DHCP server never
sees it and can't respond.

I was able to solve this problem by installing the 3rd party (unsigned)
[SynDnsmasq](http://syndnsmasq.the-ninth.com) package on my Synology box.

Once I had the package installed, I set the contents of
`/volume1/@appstore/dnsmasq/etc/dnsmasq.conf` to:

	except-interface=eth0
	except-interface=eth1
	except-interface=eth2
	except-interface=eth3
	
	domain=localnet
	
	dhcp-range=192.168.2.0,static
	dhcp-option=option:router
	dhcp-option=121,10.0.0.0/24,192.168.2.0
	dhcp-option=249,10.0.0.0/24,192.168.2.0
	dhcp-option=vendor:MSFT,2,1i

You'll need to adjust the subnets/masks and router IPs to match
your home network and L2TP address range.

What this does is respond to DHCP Inform requests with a classless
static route for the home network (10.0.0.0/24) via the L2TP server
IP address (192.168.2.0).

I restarted SynDnsmasq, connected my OS X VPN client, and was off
to the races.
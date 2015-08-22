---
layout: post
title:  "Fedora 21 VPN Problems"
date:   2015-02-03 08:41
categories: sysadmin
---

At the office, a few us use Fedora. The main reasons are that it's a fairly
good desktop distribution, and the software is nice and bleeding edge; perfect
for developers. But sometimes, it breaks.

I'm still on Fedora 20, but a handful of coworkers have either upgraded to
Fedora 21 or installed the latest version because they recently got new
computers (and crunchbang doesn't play well with newer Haswell setups).  All
the new setups had one problem: they couldn't connect to one of our VPNs.
We're in the process of migrating to [SoftEther][softether], but this
particular VPN still uses PPTP while we validate that SoftEther can handle our
load.

So what happened? The short answer is that the VPN wouldn't connect. It would
take ages for the connection attempt to fail, and [in the logs][vpn-fail-logs],
we'd see something along the lines of:

    LCP: timeout sending Config-Requests

After [a bit of digging][askubuntu], it turns out that there are a few packets
that [are being blocked by the firewall since the 3.18 kernel][marc-info]:

> As expected, with the 3.2.x kernel :
> 
> if neither nf_conntrack_proto_gre nor nf_conntrack_pptp are loaded,
> the first GRE packet of a plain GRE tunnel (as set by ip tunnel) or a
> PPTP connection is NEW ;
> 
> [...]
> 
> What has changed with the 3.18.x kernel : if neither nf_conntrack_pptp
> nor nf_conntrack_proto_gre are loaded, any GRE packet is INVALID.

The solution, thus, both on Ubuntu and Fedora (and I expect others as well) is
to always load the `nf_conntrack_pptp` module.

    # Fedora
    echo nf_conntrack_pptp | sudo tee /etc/modules-load.d/pptp-firewall.conf
    sudo modprobe nf_conntrack_pptp
    
    # Ubuntu and friends
    echo nf_conntrack_pptp | sudo tee /etc/modules/pptp-firewall.conf
    sudo modprobe nf_conntrack_pptp

Happy connecting!

[askubuntu]: http://askubuntu.com/questions/572497/cant-connect-to-pptp-vpn-with-ufw-enabled-on-ubuntu-14-04-with-kernel-3-18
[marc-info]: http://marc.info/?l=netfilter&m=142195881513706
[vpn-fail-logs]: https://gist.github.com/slauwers/ac2cd840aef6f3bbea84
[softether]: https://www.softether.org/

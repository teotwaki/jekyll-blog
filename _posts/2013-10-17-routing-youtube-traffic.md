---
layout: post
title:  "Routing YouTube traffic through a server"
date:   2013-10-17 20:03
categories: blog
tags:
---
My ISP, Free, is in a bit of a conflict with Google, on a number of subjects.
Advertising, bandwidth prices, the list is fairly long.  If this were a purely
commercial war, end-users wouldn't be hit. But, neither Google nor Free wanting
to back down, it stopped being a purely commercial war, and for the past year
or so, the pipe connecting Free users to YouTube's servers has shrunk to become
tiny.

I don't mind, I think [Free][1] is an excellent ISP, and even though I could
switch to another ISP and bypass these issues, their stance on protecting my
rights and privacy is unequalled. I value my privacy and their company more
than I value my YouTube bandwidth.

I own a couple of dedicated servers with unlimited bandwidth (courtesy
of [OVH][2]), so I decided to try and use one of those as a proxy to
YouTube. Here are the things you'll need:

* A DNS server on your local network (I'll be using `dnsmasq` on my
raspberrypi).
* A server with squid3 on it.

My router is a Freebox Revolution, and by default the DHCP will give
the router's IP as DNS server. I changed the primary DNS to point to
my raspberrypi, and kept the freebox's IP as a secondary DNS, as 
backup, in case the raspberry is down.

On the raspberrypi, install dnsmasq:

    sudo apt-get install dnsmasq

Next, configure dnsmasq to hijack all YouTube related traffic to your
server:

	server="123.123.123.123"
    echo "address=/youtube.com/$server" | sudo tee -a /etc/dnsmasq.conf

On my server, I already host quite a few websites, so port 80 is already
assigned to `lighttpd`. I could use `iptables` to send the traffic from my
home network to `squid` (on port 3128), but then I couldn't access sites
hosted by lighty.

Instead, I opted to use lighty as a reverse proxy onto squid, and added
the following rule to `/etc/lighttpd/lighttpd.conf`:

    else $HTTP["host"] =~ "youtube.com$" {
    	$HTTP["remoteip"] == "88.88.88.88" {
    		proxy.server = ("" =>
    			(( "host" => "127.0.0.1", "port" => 3128 ))
    		)
    	}
    }

Also, don't forget to add `mod_proxy` to the `server.modules` directive.

Now, install squid:

    sudo apt-get install squid3

Lastly, `/etc/squid3/squid.conf` needs a couple of lines for everything
to work properly:

    http_access allow localhost
    http_port 127.0.0.1:3128 transparent vhost allow-direct

And that's it! Now, obviously, this completely and utterly breaks every
SSL aspect on YouTube, but I don't care about those. Another
side-effect is that when logging-in to another Google service such as
GMail, one of the redirects that happens during the login process will
fail. I imagine this is Google ensuring all the cookies are set properly.
The workaround to this is to simply refresh the page you were trying to
get at initially; the cookies will have been set, and you will be
logged in.

[1]: http://www.free.fr
[2]: http://www.ovh.fr

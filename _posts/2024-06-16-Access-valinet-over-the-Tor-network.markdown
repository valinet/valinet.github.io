---
layout: post
title:  "Access valinet.ro over the Tor network"
date:   2024-06-16 00:00:00 +0000
categories: 
excerpt: "Lately I have decided to migrate away from Proxmox containers to full blown virtual machines, due to reasons like the inability of containers to host Docker inside the root volume when using zfs, so having to resort to all kinds of shenanigans, like storing `/var/lib/docker` in a separate xfs volume which nukes the built-in high availability, failover and hyperconvergence (using Ceph) features for that container, plus wanting to make the setup more portable and less dependant on the underlaying OS."
---

It is my pleasure to announce that, for almost 2 weeks now, valinet.ro is available over the Tor network as well, at [https://valinet6l6tq6d5yohaa6gdsf2ho4qcqcxyj2nahp5kd4z7nsa6ycdqd.onion/](https://valinet6l6tq6d5yohaa6gdsf2ho4qcqcxyj2nahp5kd4z7nsa6ycdqd.onion/). I have presented this setup in an assignment for one of the classes I take in uni, so here's the write-up that I have made for it.

Setup consists of an Alpine Linux-based VM which runs the tor server service and an Nginx Proxy Manager docker container which proxies the actual web site which, as you know, is publicly available on [GitHub](https://github.com/valinet/valinet.github.io). This way, I do not have to "build" the web site in 2 places (I use a static site generator), it is still hosted on GitHub Pages and relayed through that proxy to the clients accessing it via Tor.

```
Client => Tor network => Tor server on VM => nginx reverse proxy => github.io
```

I chose Alpine because it is lightweight, easy to setup and manage and I am familiar with it, and Nginx Proxy Manager because it is just easier to use a GUI to configure everything rather than nginx config files.

The config for `/etc/tor/torrc` looks like this (based on [this](https://medium.com/axon-technologies/hosting-anonymous-website-on-tor-network-3a82394d7a01)):

```
HiddenServiceDir /var/lib/tor/hidden_service
HiddenServiceVersion 3
HiddenServicePort 80 127.0.0.1:80
HiddenServicePort 443 127.0.0.1:443
```

The nginx docker container is configured to only listen on localhost; anyway, the VM runs in an isolated network anyway (I control the hypervisor and the infrastructure there - it's running on a Proxmox cluster at my workplace), and the web site is also available publicly via the clearnet anyway, so I do not really care if some may hypothetically access it bypassing the Tor network; that's why I decided running it over unix sockets was not worth the hassle, but if you want that, [check this out](https://kushaldas.in/posts/using-onion-services-over-unix-sockets-and-nginx.html).

For the Nginx Proxy Manager config for the web site, besides what is offered through the GUI (forwarding valinet...onion to https valinet.ro 443), I have also set a custom location for "/", again that directs to https valinet.ro 443, which allows me to set custom nginx directives. This is what I used:

```
proxy_set_header Host valinet.ro;
proxy_redirect https://valinet.ro/ https://valinet6l6tq6d5yohaa6gdsf2ho4qcqcxyj2nahp5kd4z7nsa6ycdqd.onion/;
```

`proxy_set_header` is necessary so that GitHub knows which web site it hosts we want fetched (a lot of web sites are "stored" at the same IP). Without this, of course GitHub returns a 404 Not Found error.

`proxy_redirect` sets the text that is changed in the "Location" and "Refresh" headers. Basically, this changes the address from "valinet.ro" to "valinet...onion".

To make the address look nice, following [this](https://opensource.com/article/19/8/how-create-vanity-tor-onion-address), I have used [mkp224o](https://github.com/cathugger/mkp224o.git) and generated a few addresses and picked up the one that I liked the most. On a powerful AMD Ryzen 5900x-based machine it only took around 15-20 minutes to generate 3 options to choose from. It was not initially in plan, but I saw this on Kushal's web site and I have decided I must also have that.

The Tor Project has a [web page](https://community.torproject.org/onion-services/) showing the addresses of some well known services, like The New York Times, Facebook etc. Some of these run over https, and I thought it wouldn't hurt to run my service over https as well for extra swag.

According to [this](https://kushaldas.in/posts/get-a-tls-certificate-for-your-onion-service.html), there are only 2 providers of certificates for .onion address: Digicert which is expensive through the roof, and HARICA (Hellenic Academic and Research Institutions Certificate Authority) which has more normal prices. The setup was very easy, I verified by generating a CSR signed with my service's key as described in their instructions using [an open tool they host](https://github.com/HARICA-official/onion-csr) and got my certificates. Then, uploading and associating them with the domain was a breeze through Nginx Proxy Manager's GUI.

In the end, you can also use Nginx Proxy Manager to hide certain headers that you do not want proxied, using directives like:

```
add_header Server custom;
proxy_hide_header ETag;
```

This is it, happy onioning!

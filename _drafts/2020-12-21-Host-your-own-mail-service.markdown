---
layout: post
title:  "Host your own mail service"
date:   2020-12-21 00:00:00 +0000
categories: 
excerpt: "desc"
---

No matter the reasons, hosting your own mail service is not a *simple* adventure. But it does not mean you should not do it. In fact, I'd argue you **should**. Email is a decentralized protocol and it is a shame nowadays, this field has been 'monopolized' by a few actors. It's time to take our mail *back*.

I reckon that my previous articles are becoming pretty large, which may make reading them unnecessarily difficult or too time consuming. Considering the complexity and the amount of things you have to do in order to achieve the goal of this article, if I were to write in the same style as before, it would easily reach 40 pages when printed on A4. And it does not make sense to write everything from scratch because there are already a lot of *very good* resources on the matter, plus, there are too many differences in this setup. For example, I have largely followed an excellent guide written by someone that has its setup running on FreeBSD, while I run mine on Raspbian, as the single low-power computer I had laying around that was available for such a task was a Raspberry Pi 4 2GB. Of course I had to adjust all the paths to ones corresponding to how stuff on Linux/Debian gets installed/set-up, and also slightly adjust some of the settings to cope with some customizations Debian does on the packages mentioned in that article. On the other hand, I don't want this guide to be specific for a particular UNIX-like OS, but to actually work with slight adjustments on any UNIX and even Windows (in WSL). If I had an x86 machine laying around, I would have chosen Arch Linux as my distro and deployed the whole thing there.

So, with these in mind, this article will have a different structure: I will highlight each major step of our journey, and provide a series of links and information about why I mention them where you can read how detailed steps on how to configure the stuff mentioned. Among these links I will include and mention any pitfalls or edge cases I have encountered and may have a lost a bit of time, as a heads up so you don't lose your time as well.

This being said, let's start with a few words on the philosophy of this: if you actually deploy this, depending on the nature of the machine you will install this on, you may decide to modularize the setup by containerizing everything or even by employing virtual machines. Of course, that's actually not a bad thing to do, and I recommend, if possible, to modularize stuff as much as possible because it makes management easier. For actual technologies to use for modularization, I have used KVM for VMs in the past, and LXD for containers, so I'd recommend them, but of course Docker or any other thing is more than fine. As for my actual setup, I am running this as the single service on my Raspberry Pi, so I think it is pretty modularized in that that Pi does only this. As a base OS, I chose Raspbian Lite Buster (or "Raspberry Pi OS Lite 32-bit" - it is pretty obvious I don't like the new name).

## Basics

It is good to actually decide on some basic aspects so that we know how to proceed.

Firstly, you need to own a domain name. Any domain name will do, you just need to own it so that you can register name servers with it. In the name servers that you enter, you need to be able to configure them and register a few entries. If you are looking for a service that provides name server services for free, I highly recommend [1984.is](https://www.1984.is/).

#### Public IP vs private (dynamic) IP

For my setup, I have decided to do the computing and store the actual data of the mail server in my local premises (in the closet in my dorm room) on that Raspberry Pi I mentioned (from now on, I will call it the *server*). The other alternative would be to host the data at a third party, namely a virtual server provider (VPS).

Also, the way I connect my server to the Internet is via a router provided by my ISP that does NAT. The router has a public IPv4 address of its own, and I am able to set up port forwarding on it for any port (they use some JavaScript on the router setup page to actually restrict some ports, but yeah, being only JavaScript and no server side limitation, I can configure any port I want using something like curl).

For this setup though, you DO NOT actually need a public IP address for your residential router. So, dynamic IP will do. I will explain why this is not a necessity in the following section.

#### Mail vs email

Email works exactly like real-life mail. The same principles apply. Nothing stops you from writing and sending a letter in the name of Donald Trump, for example. The recipient has to deal with actually figuring out whether the real Donald Trump wrote the letter or not. For sensitive mail, people authenticate it using pre-shared keys (mere mortals call them "seals"), self-signed certificates (mere mortals call them "handwritten signatures"), 2-factor authentication (mere mortals call it "use another medium to announce the arrival of mail to the recipient, like someone calls someone on the phone to tell them they have sent a mail"). All these methods and physical mail in general is inherently insecure, but this is due to its decentralized nature, as in order to authenticate the sender automatically, some database would have to be maintained, public post boxes would not be a thing, but it would require you to go to the post office and present an ID when wanting to send something etc. Also, mail is susceptible to man-in-the-middle attacks: the postman can open the letter and read it and even modify it, for example (of course it is illegal, but it is illegal because *it is a possibility*).

Furthermore, keep in mind that mail always gets to its recipient. In other words, the recipient does not have to authorize you to send them mail. Nothing stops you from paying for a stamp and mailing a letter to them.

Keeping this into consideration, email works **exactly** the same. No one stops you from setting up a mail  server (namely, a *mail transfer agent*) and sending emails in the name of anyone. Just try it, some distros come with a preconfigured "mail" command, use that to send an email to any address and override the from address and write anything in there, like *donald.trump@us.gov.* The mail will get delivered to your recipient if your MTA actually works properly. Also, as I said, it works exactly like physical mail: it automatically got into your recipient's mailbox, even if he/she has not given you prior consent to do so. Compare this to how any other online service that is deemed to be secure and/or private works nowadays: to talk to someone on Facebook, you have to send them a 'friend request', same with Discord, Steam, Skype etc. Some form of prior permission is required in order to get in touch with someone. Not to mention, to often get this consent, you need to authenticate with the service, which often implies personally identifying you: you cannot use Facebook without having a Facebook account.

With email, on the other hand, just like physical mail, there is no pre-authentication. You just send the message and it gets to your recipient. Only condition is to know their address, almost like in reality (I mean, in the physical world, the range of addresses is public and can be easily determined). I cannot emphasize how similar email and physical mail work from a conceptual point of view. Email really is the electronic counterpart to traditional mail.

> Funny story: recently, some public personality from Romania got incredibly mad when someone decided to troll them and sign up to various organizations using their email address. Most of these organizations would sent back automatic acknowledgment mail. The person got furious as to how that is possible, how can someone electronically attack them in such a way, and what are the authorities doing to combat this. Fortunately, they do nothing, as there is nothing to combat: maybe if this person knew how physical mail worked and how email is just 'electronic physical mail', they would have chilled out. Nothing stops anyone from impersonating you using *your* email address; the recipient decides how to treat your enquiry accordingly.

Of course, this nature of email makes it a pretty easy target for abuse. With physical mail, spamming, even though totally feasible, is actually hard because of the costs associated with it (you have to pay for stamps), also the labor required (someone has to print all those letters, put them in envelopes and place them in the mail box), the time involved and the actual effectiveness you get in the end. With email, physical deterrents are gone: you can write scripts that send emails en-masse to tons of addresses. With email, abusive behavior is called *spamming*.

As you imagine, this protocol would not be that successful if things were let to run in this Wild West style. So, because senders can be abusive, most if not all of the popular providers of addresses for recipients (think Gmail, Hotmail, Yahoo! etc) employ a complex array of mechanisms that decide whether the mail you send them ends up in spam or is discarded altogether, or it ends up where you want it to be, namely in your recipient's mail box. That's how security on email is done. It is mostly the recipient enforcing strong filtering policies and the sender coping with them. To protect against man in the middle attacks, encryption can be used when transporting email on the wire.

So yeah, technically, nothing stops you from impersonating anyone or doing all kinds of shenanigans using an MTA or by interposing between the sender and the recipient. Thus, I strongly believe that any guide regarding setting up an email service should begin with a comprehensive discussion of the security mechanisms you will have to employ in order to actually be able to send mail successfully to any major service provider (i.e. have it place your mail in inbox, not junk or simply discard it altogether).

#### Email security

There are a couple of mechanism that **you have to** deploy in order to make your mail server work properly and securely.

##### Reverse DNS

As I said, the sender can specify any address as "From" address. So, how can the recipient work towards determining whether the sender is authentic or an impersonator. One technique is by running a reverse DNS lookup on the IP address the mail was sent from. This lookup (PTR lookup) will produce a domain name, say "mail.valinet.ro". Then, the recipient can check whether "mail.valinet.ro" points to that same IP address in DNS (A lookup). This will tell the recipient whether that mail is allowed to end mail in the name of "valinet.ro" or any subdomain of it. If the check fails, the mail is discarded/marked as spam.

It is important to note this:

* the A record that maps "mail.valinet.ro" to an IP is set up on the name server associated with the domain that you own
* the PTR record that maps the IP to "mail.valinet.ro" is set up on the mail server associated with that IP; IPs are translated to a special domain, in the form of 4.3.2.1.in-addr.arpa and a regular query is done on that address; to edit DNS info for that address, you have to **own** that IP address

This way, if you specify the from address as "donald.trump@us.gov", you have to be able to both write a correct PTR record for your IP address, but also own the domain "us.gov", so that you can alter its DNS configuration too. So, as you can see, unless you are the actual Donald Trump, this is pretty much impossible so this mechanism is the first strong mechanism for protecting mail communication.

Take note of what I said above: you have to own the IP address in order to edit its DNS entries. So, even if your residential connection has a dedicated IP address, you still do not own that; it belongs to your ISP, and most of the times they won't allow you to change the PTR records for that IP. These is usually because they may provide that via a business subscription that is probably more costly than what you have at the moment, if they offer it at all. Also, even if they wanted to help you, they'd probably be incapable to effectively do so unless they transfer the actual ownership of the IP to you.

You see, IP stuff is pretty convoluted. As you know, IPv4 is a very limited resource, and so whoever gets the right to use particular addresses is highly regulated. Among these regulations, there is the necessity of the *owner* (more correct is to say the entity that is allowed usage rights on that IP, as they do not actually own it) to publish public details about themselves. These information, as is the case with information regarding regular domains, is public and is obtainable by issuing a *whois* query with the registrar. Because classful IP allocation is not a thing anymore, mail service providers use the whois information to determine a 'class' an IP belongs to. In other words, if they 'whois' query your IP and get the name of an ISP, they classify your IP as 'residential'. Then, all mail services outright reject any mail coming from residential IPs regardless of reverse DNS presence or not. The reason is simple: residential IPs are used by regular people to 'talk' on the Internet; people use computers that are highly likely to get infected with computer viruses, which in turn have a high risk of sending tons of spam. So, by default, mail from residential IPs is usually spamed. These associations are maintained by the mail service provider, so, in other words, even if the owner of an IP changes, you still have to call up any mail service provider and ask them to revise their records about your IP.

Then, even if you convince your ISP to give you the ownership of the IP you use and also convince all service providers to white list your IP, there is the next problem: how do you connect to the Internet? For me, it would be too cumbersome: I have a very competent modem+router box provided by the ISP. If I want to use this solution, I'd have to set the box to modem only and dedicate a computer to do at least basic routing (I could chain load an actual subsequent router to one of its NICs) and also maybe host my email server there.

So yeah, theoretically there is this route as well, although I don't recommend it. That's why I said before that you do not need a static IP. Just forget about modifying your home network setup so radically and you can use the solution I employed for this issue, which I describe in the next chapter.

##### SPF

The SPF record is a TXT record you set on your domain's DNS that tells recipients which of your hosts are allowed to send mail. Recipients will only accept mail from hosts whose IPs resolve to the domain names specified in the SPF record, and reject/spam all other mail. Most mail providers reject/spam mail if it comes from a domain without an SPF record set.

##### DKIM

DKIM helps against man in the middle attacks. See, even if you have rDNS+SPF, one can stop your mail midway on the wire, modify its contents (if sent without encryption at protocol level) and then forward that to the recipient. DKIM works around this by having your server sign all mail using private key. Then, the recipient will need to decrypt your mail using your public key. The way you share that with your recipient is by specifying it in a specific DNS TXT record. Most providers spam your message if you do not use DKIM.

##### DMARC

DMARC is also a TXT record that describes how you recommend the recipient to behave when validation for messages received in your name fails. Of course, the recipient is allowed to do whatever it pleases, but most providers, despite the behavior they actually employ, require DMARC to be set, or otherwise the message goes to spam.

##### Protocol encryption

Besides the mechanisms above, the protocols we use for transferring mail (SMTP), and for retrieving and managing the mailbox (IMAP) can themselves be encrypted. You may wonder, why is this necessary if we employ DKIM?

Well, the problem with DKIM is that there is no certification authority involved. In other words, even though DKIM guarantees that the mail was delivered untouched from us, it still does not guarantee we are trustworthy.

Encryption at protocol level means is done using a certificate issued by some certification authority. We could issue ourselves a certificate (*self-signing*), but then our certificate would not be a descendent of any of the certificates in the recipient's trusted certificate store, so they would notice this and reject us. So, you need an actual, reputable certification authority to "vouch" you. Fortunately, with the advent of HTTPS, this does not imply a fee anymore, as there are numerous services that quickly verify you and provide certificates for free.

## Obtaining a proper IP

With the basics outlined, let's begin addressing our problems.

For an IP address, what I recommend doing is buying a VPS service. Don't buy anything fancy, as we will use their VM just to replicate a router, and do all the processing and storage on our box. I recommend a VPS because it is the easiest solution compared to what I described above. Find the cheapest provider that gives you:

* an IPv4 address
* the possibility to register a reverse DNS entry (PTR record) on the IPv4 address they will provide you

As for my choice, I highly recommend [LaunchVPS](https://launchvps.com): not only did they have an awesome price (just $20/year), they also have very good technical support. The machine I got was the lowest tier option, which again, just go for that, it is already a million times more powerful than any apartment router and will get the job done: dual-core Intel Xeon E5-2620 (single core Pentium 4 is fine as well), 750MB RAM (maximum 100MB is fine as well), 20GB storage (4GB is more than enough as well). Just pick the cheapest lowest-tier option that gives you SSH access to a box having a public IPv4 address that you can somehow alter its rDNS config.

To set this up, I installed Arch Linux, of course. To set this up, follow the [Arch Installation Guide](https://wiki.archlinux.org/index.php/installation_guide) in order to install the distribution on your VPS' VM.

#### Connecting the VPS VM and the server

You need to establish a way to communicate with your actual server. There are two ways I can think of to do this; personally, I have used option #1 because that came to my mind first, more spontaneously; later, when I was implementing this, I realized #2 is an option as well and have looked into it:

1. Connect the two machines via a VPN tunnel. This will be exposed to applications as just another NIC connected to a common LAN. The underlaying architecture is transparent to the applications. To make them talk over the tunnel, one has to adjust the routing tables, which the VPN solution usually does automatically. With a bit of care, some apps can be made to talk on the regular LAN as well. All in all, this is VERY transparent to applications.
2. Connect the two machines by having one act as a proxy server. The way this goes is: you configure the VPS VM as a proxy server, and then, on the mail server, you configure each application to use the proxy server (this is done differently for each application); for applications that do not support proxies natively, you can use something like ProxyChains (a proxifier); the way this works is it injects into applications via LD_PRELOAD and intercepts network calls and forwards them to the proxy server. So, this solution is not transparent at all to applications, unless a proxifier is used. Also, with a proxy, you rely on the underlaying protocol to perform encryption, as most proxy protocols are not encrypted.

##### Setting up the tunnel

For setting up the tunnel, I cannot recommend anything else but WireGuard, which is a modern, very secure, VERY fast, robust and easy to configure VPN that is built into the Linux kernel. With it, it just works, you do not have to worry about any sort of problem should any of the machines restart, it really just works(TM). When you configure the tunnel, WireGuard sets up a network adapter on your hosts that makes them appear on the same network, which is exactly what we want. To install and configure a WireGuard server on your Arch install, look no further than [WireGuard](https://wiki.archlinux.org/index.php/WireGuard) on the official Arch wiki.

Similarly, follow some guide to install WireGuard on your server as a client that will connect to the WireGuard instance on your VPS VM. Again, just use a guide specific for your distro/OS; for my Raspberry Pi, I used this: [Installing and Configuring WireGuard on Raspberry Pi OS (August 2020)](https://www.sigmdel.ca/michel/ha/wireguard/wireguard_02_en.html) (there is also [pivpn](https://pivpn.io) but that automatically sets your Pi as server and you have to undo some config, plus, it is good to learn the manual way, in the spirit of this guide).

Once you have a tunnel between the two, it is time to put the VPS VM in 'router mode'. For this, you need to enable NAT for packets coming from your server to your VPS VM and going to the outside Internet:

```
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

This way, our mail packets will have their source IP changed to the VPS IP, for which we have proper reverse DNS set up.

Next, you have to do port forwarding: requests coming on some ports, like the SMTP port, should be forwarded from the VPS VM to your server via the WireGuard tunnel. To do so, use an iptables rule like this ('25' is port number, '10.6.0.2' is IP of your server via the WireGuard network):

```
sudo iptables -t nat -A PREROUTING -p TCP -i eth0 --dport 25 -j DNAT --to-destination 10.6.0.2
sudo iptables -A FORWARD -p TCP -i eth0 -d 10.6.0.2 --dport 25 -j ACCEPT
```

Rules set in iptables can be made persistent across reboots in Arch using 'iptables-save':

```
sudo iptables-save -f /etc/iptables/iptables.rules
```

Make sure to enable the iptables service so that they get applied at next boot:

```
sudo systemctl enable iptables
```

To drop all rules set above, do:

```
sudo iptables -D
sudo iptables -t nat -D
```

For the command above to work, be sure to enable IPv4 packet forwarding:

```
sudo sysctl net.ipv4.ip_forward=1
```

To make this persistent, per the [wiki](https://wiki.archlinux.org/index.php/Internet_sharing#Enable_packet_forwarding), add these to file "/etc/sysctl.d/30-ipforward.conf":

```
net.ipv4.ip_forward=1
net.ipv6.conf.default.forwarding=1
net.ipv6.conf.all.forwarding=1
```

If you have a static IP on your residential router, you may think that skipping WireGuard is possible. Just forward the packets to your residential router, deal with port forwarding and another level of NAT there and you should be good to go. Unfortunately, things are not so simple: see, the problem is that the gateway on your LAN cannot be the VPS VM, because its IP is out of the class you are probably using. The gateway has to be within your LAN, and most probably is your router. Then, you'd have to change the rules on your router and tell it to use your VPS VM as a gateway (more like next hop) for packets originating from your server, but the problem is, most residential routers do not let you do this (i.e. they do not allow altering the routing table at all). Even if they'd let you, your router's public IP, again, most probably is not in the same class as the VPS VM, so your routing table alteration would have to cascade though all the subsequent hops which is pretty much impossible. Because WireGuard presents the tunnel as just another LAN to the connected devices, we can have select applications bind and use the WireGuard interface to communicate with the Internet, so the gateway will be the WireGuard server, aka the VPS VM.

Plus, having an internal dedicated LAN composed of devices in these WireGuard VPN is advantageous because it allows you to connect to that network, so those devices, from any other device, pretty easily. So, say you also have a service that you use internally for managing some stuff (like LDAP), but we do not want exposed that to the Internet. Just connect from any machine that acts as a WireGuard peer via VPN and you have access to it.

##### Still, what about a proxy?

Based on the example above, I still think proxies can be 
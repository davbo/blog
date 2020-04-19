---
title: Connecting Home
date: 2020-04-18T10:00:40Z
draft: false
author:
 - Dave
summary: After working remotely from Finland for an unexpectedly long period I created a Tailscale private network to connect home.
---

We're lucky to have found ourselves visiting family in Finland as the physical
distancing measures for COVID-19 started in London. Finland appears to be
responding well and apparently has been [prepping][preppers] for some time.

However, what began as a trip for 2 weeks is now entering the 5th. We're already
up to the 3rd season of LOST, a rewatch for me, the ending now a distant memory.
I'm eager to be disappointed all over again. But there are a few other TV shows
we've been watching that are downloaded to a Raspberry Pi running
[LibreELEC][libreelec] at home.

[preppers]: https://www.nytimes.com/2020/04/05/world/europe/coronavirus-finland-masks.html

## ssh: port 22: Connection Refused

The Pi at home isn't accessible over the Internet, I'd need to forward a port to
my router to be able to SSH. Most routers I've owned in the past would need me
to be on the local network to do stuff like that.

Luckily the Google Wifi setup we have works a bit differently, it's is
configured via an app -- a restriction I'd previously grumbled about. The router
picks up it's configuration via a centrally managed service. This allows for
some other nice things such as automatic patch management, which is why I went
for Google Wifi in the first place.

So I added the port forwarding rule using a non-standard SSH port, because I'm a
coward. Then yay! I'm connected to my Pi at home. It felt a bit strange
connecting to home, we've been away for so long now. If someone has burgled us
at least they left the router plugged in.

Even with my public key configured I still wasn't comfortable leaving the Pi
this way. I wanted to setup some kind of VPN.

## Creating a private network with Tailscale

After reading about but never actually trying [WireGuard][wireguard] it was
seeing [Tailscale][tailscale] which made me feel confident to give it a try.

It's built in Go and the Linux client is [open source][tailscale-github].
Installation was extremely easy as they provide binaries built for all kinds of
platform architectures, including ARM for the Pi.

If you're also using LibreELEC I installed the following systemd unit to
`/storage/.config/system.d/tailscaled.service`:

```systemd
[Unit]
Description=Tailscale node agent
Documentation=https://tailscale.com/kb/
Wants=network-pre.target
After=network-pre.target
StartLimitIntervalSec=0
StartLimitBurst=0

[Service]
EnvironmentFile=/storage/tailscale/tailscaled.defaults
ExecStart=/storage/tailscale/tailscaled --state=/var/lib/tailscale/tailscaled.state --socket=/run/tailscale/tailscaled.sock --port $PORT $FLAGS

Restart=on-failure

RuntimeDirectory=tailscale
RuntimeDirectoryMode=0755
StateDirectory=tailscale
StateDirectoryMode=0750

[Install]
WantedBy=multi-user.target
```

On my laptop I added the Tailscale apt sources and installed the debian package.

Once each node is authorized to join the network they are assigned a fixed IP
address from the `100.64.0.0/10` block specified by [rfc6598][rfc6598] for
private networks.

The tailscale docs are good but a little tricky to navigate in the knowledgebase
tool they have. Once you're setup you find there is a preconfigured node in your
network `hello.ipn.dev` which runs an IRC server and you're invited to chat with
other Tailscale users. It's a great way to demo the types of internal services
you could easily create if Tailscale were adopted by an organisation.

That point is the one which makes Tailscale interesting beyond the traditional
VPN. In my experience VPN software is inevitably used to create a kind of
(forgive the analogy) crunchy shell protecting a gooey interior network.

It seems by linking user identity to the node Tailscale may be able to do
something far better than that. Thinking about it reminded me of when I first
encountered security groups in AWS.

For now my win with Tailscale is that I no longer need to forward any ports
directly to my pi. That makes me much more comfortable knowing it can't be
scanned from the big bad Internet.

I've added an entry to `/etc/hosts` to name the pi and can now happily SSH
over my private Tailscale network secured by WireGuard. Super easy to configure
and setup.

[libreelec]: https://libreelec.tv/
[salla]: https://www.visitsalla.fi/en/
[wireguard]: https://www.wireguard.com/
[tailscale]: https://tailscale.com/
[tailscale-github]: https://github.com/tailscale/tailscale
[rfc6598]: https://tools.ietf.org/html/rfc6598

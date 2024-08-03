---
title: "You don't need NordVPN (if you have SSH)"
date: 2024-08-03T09:47:56+02:00
tags: ["ssh", "bypass", "censorship", "vpn"]
draft: false
---

Let's say, _hypothetically_, you want to see the replay of Teddy Riner kicking ass at the Olympics.


But you're not in France, so ___France TV_ blocks you__ if your __IP is not French__.


![France TV player indicating that the content is blocked](/bypass_country/blocked.png)

> Your operator locates you in an area where this video is not available (Denmark).


At this point, you might think about taking a __VPN subscription__ or find a free shaddy VPN that will inject malware into unencrypted HTTP pages.


## SSH port forwarding to the rescue

Luckily I was already renting a __Linux server__ (_well, I forgot to cancel it_), __located__ in __France__.

![Meme heaviest objects in the universe](/bypass_country/meme.jpg)

You can use [the following __ssh__ command](https://explainshell.com/explain?cmd=ssh+-fND+127.0.0.1%3A3000+username%40server) to create a __SOCKS proxy__.

```bash
ssh -fND 127.0.0.1:3000 username@server
```

With `netstat`, we can see that `ssh` is listening on port `3000`.

```shell
$ sudo netstat -ltupn

Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address       
tcp        0      0 127.0.0.1:3000          0.0.0.0:*               LISTEN      27833/ssh
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      920/systemd-resolve
udp        0      0 127.0.0.53:53           0.0.0.0:*                           920/systemd-resolve
```

We can now use the __port `3000`__ as a ___proxy___.
Personally, I use [Switchy Omega](https://addons.mozilla.org/fr/firefox/addon/switchyomega/) to easily change between proxies. But you can modify directly the proxy settings of your web browser.

![Switchy Omega](/bypass_country/switchy_omega.png)

I create a new proxy in _Switchy Omega_ as a __Socks5 proxy__ defined as `127.0.0.1:3000`.

You can select it in your list of proxies. And _voilà_, we have access to _France TV_.

![Ippon](/bypass_country/bypassed.png)


## Awesome! What just happened?

Well, glad you ask. Let's look at the ssh command. (On a side note, [explainshell](https://explainshell.com/explain?cmd=ssh+-fND+127.0.0.1%3A3000+username%40server) is really nice to explore shell commands.)


```bash
ssh -fND 127.0.0.1:3000 username@server
```

* `-f` indicates to run `ssh` in background
* `-N` indicates that we don't want to run any commands (used for port forwarding).
* `-D` is the interesting stuff. Let's look at the __man page__.


> `-D [bind_address:]port`: Specifies a local __“dynamic” application-level port forwarding__.  This works by allocating a socket to __listen to port__ on the __local side__, optionally bound to the specified bind_address.

> Whenever a connection is made to this port, the connection is __forwarded over the secure channel__, and the application protocol is then used to determine where to connect to from the remote machine.

> Currently the __SOCKS4__ and __SOCKS5 protocols__ are supported, and ssh will act as a SOCKS server.  Only root can forward privileged ports.  Dynamic port forwardings can also be specified in the configuration file.



So __`-D`__ let us create a __SOCKS5 proxy__, and the packets will be __forwarded over SSH__.


### Nice, what's a SOCKS5 proxy exactly?

A __proxy__ is something that __sits between you and the internet__. It will _relay_ your requests to the different websites.

For example, a company might deploy a proxy to block adult websites, shady websites, check for malware, log connections, etc.

![proxy](/bypass_country/proxy.png)

Traditional proxy are only for the HTTP protocol, but soon enough there was a need to offer the same service for other protocols.

__SOCKS__ offer the same service at the __TCP level__. There are 2 major versions, _SOCKS4_ and _SOCKS5_. ___SOCKS5___ is more __modern__ and supports _UDP_, _IPv6_, and authentication.

SOCKS is a protocol, so the __support__ for the __protocol must be implemented__. But luckily for us, it's implemented in __all major browsers__. You can also _proxify_ some apps that don't implement SOCKS with tools like `proxychains`.


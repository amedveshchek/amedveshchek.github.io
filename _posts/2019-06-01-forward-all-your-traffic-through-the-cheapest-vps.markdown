---
title: "Own VPN: how to forward all your traffic through the cheapest VPS"
layout: post
date: 2019-06-01 19:48
image: /assets/images/markdown.jpg
headerImage: false
tag:
- ssh
- port forwarding
- vpn
- vps
category: blog
author: alexmedveshchek
description: "Own VPN: how to forward all your traffic through the cheapest VPS"
---

Today, the problem of restricting Internet traffic is quite relevant: perhaps you are a resident of a particular country such as Russia or just want to use certain services from work where traffic is curtailed -- you always have to think of workarounds.

Below I give a description of one of the simplest ways to solve this problem.

In short:

```bash
# Install sshuttle (Ubuntu):
you@localhost$> sudo apt-get install sshuttle
...
# OR other Linux OS:
you@localhost$> sudo pip3 install sshuttle
...

# Start forwarding:
you@localhost$> sshuttle -r root@<ip-of-your-VPS> 0.0.0.0/0
[local sudo] Password:           # sudo password of your local comp
root@<ip-of-your-VPS> password:  # user pass of your VPS
client: Connected.               # here we are!
```

Now go to any site like [get-myip.com](https://get-myip.com/) from your browser and make sure that it shows `ip-of-your-VPS`. 

Explanation:
0. It's hard to find better and easier tool to forward all the local traffic to your remote server than [`sshuttle`](https://sshuttle.readthedocs.io). Try it out yourself.
1. Obviously, we don't need a dedicated server, so simply rent the cheapest VPS (Virtual Private Server) with enough amount of Internet traffic. I've found `vpsserver.com` that has $5/month tariff with 1 TB of traffic, 1 GB RAM, 25 GB ssd which is more than enough for my purposes. Also I heard that there are even more cheaper VPSs.
2. Install `Ubuntu` on it or other OS your would prefer.
3. Insatll `sshuttle` on your local computer and run it as shown above.
4. Note: `sudo` password of your local machine is necessary to get enough permissions to forward all your local traffic.

`FIN!`

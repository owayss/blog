---
title: "SSH tunnels"
description: "How to route traffic through an encrypted channel"
tags: [
    "networks",
]
categories: [
    "networks",
]
menu: "main"
---

Recently I needed to access a machine located in a private network on my organization's wide area network. The WAN has a firewall that blocks all traffic coming from the outside (the internet) to access any of the machines in the internal cluster. This lead me to learn about port forwarding.

## Port forwarding
Port forwarding is a mechanism used for tunneling application ports form client to server (or vice versa) by routing traffic through an intermediate gateway. It can be used for adding encryption to insecure traffic coming from legacy applications, or to bypass firewalls allows remote computers. The latter is very useful for opening back-doors into internal networks.

Essentially, port forwarding can be divided into three types: **local**, **remote**, and **dynamic**.

Local port forwarding, as its name suggests, forwards traffic coming to a specific local port to a remote one. A typical use case of local forwarding is to have route incoming SSH access through a single _jump server_. 

Remote port forwarding does exactly the opposite; it forwards traffic coming to a remote port to a local one. This is frequently used to expose an internal web service to the public internet.

And last but not least, **dynamic port forwarding**. Where as both local and remote forwarding only allow interaction on a single port, dynamic forwarding allows a full range of TCP communication across a range of ports.


![Dynamic port forwarding](ssh-tunnels.png "Dynamic port forwarding")
## The gist

#### Local port forwarding
```sh
ssh -L 8080:youtube.com:80 home
```

The `L` option binds port 8080 of localhost to listen for local requests from port 80 of `home`. 
#### Remote port forwarding
```sh
ssh -R 9000:site.myorg.com:80 home
```
The `R` option creates a reverse tunnel, to allow forwarding of outgoing traffic. This way we can connect to in our internal organization site (site.myorg.com) from home through the SSH tunnel that gets created between the two machines. 

#### Dynamic port forwarding
```sh
ssh -DN 1080 user@gateway
```
The `N` option is to say that we are going to use this port to forward traffic. The `D` is for dynamic forwarding. 

The previous command turns our SSH client into a SOCKS proxy server. The SOCKS protocol is used to route any internet connection through a proxy server. For instance, we can tell our browser to request every webpage through the encrypted SSH tunnel:
```
open -a "Google Chrome" --args --proxy-server=socks5://localhost:1080
```
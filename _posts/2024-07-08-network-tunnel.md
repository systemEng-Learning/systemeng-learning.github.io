---
title: "Building a Network Tunnel"
last_modified_at: 2024-07-08
excerpt_separator: "<!--more-->"
categories:
  - Networking
tags:
  - rust
  - networking
  - wireguard
  - vpn
---

This project was an education in understanding a bit of low-level computer networking through the lens of wireguard. [Wireguard®](https://www.wireguard.com/) is an open-source VPN protocol and software that makes it easy for a user to set up their own VPN server and client.

Before we worked on this project, we weren't ignorant about computer networking. We were knowledgeable of [OSI layers](https://en.wikipedia.org/wiki/OSI_model#Layer_architecture), TCP, UDP, IPv4, IPv6, and some other application protocols. We have also built some [networking](https://github.com/goodyduru/nitrows) [applications](https://github.com/goodyduru/simpletorrent) in the past. Still, we felt that our knowledge was insufficient. We are avid readers of engineering blogs from companies like Cloudflare and Tailscale. Some of their networking articles were fully understandable because of our background knowledge, but some felt like we were reading a mix of English and Latin. Some of these barely understandable articles referenced [TUN](https://tailscale.com/blog/throughput-improvements) [devices](https://tailscale.com/blog/more-throughput), [iptables](https://blog.cloudflare.com/how-to-drop-10-million-packets/), and [routing tables](https://tailscale.com/blog/2021-05-life-of-a-packet). This project was an avenue to rectify our ignorance.

To fulfill the goal of this project, we created 3 milestones. The milestones are
* Create a simple VPN client and server.
* Support ipv6 and encryption.
* Implement localhost tunneling.

We implemented the project in Rust which in a way helped us gain an understanding of networking in Rust. The result of this project can be found [here](https://github.com/systemEng-Learning/simple-vpn). We worked separately on our own individual modules. *We don't advise anyone to use this project for production. It was strictly made for learning purposes.* 

The below progress log is retrospective, so some of the details might not be as accurate due to memory failure :-).

### Log
* Apr 14 - Apr 24, 2024: Didn't know anything about tunnel devices, tunnelling, and so on. We had to carry out some research. We played around with Wireguard and read several articles on it. We felt understanding network tunnels would give us a clearer understanding of Wireguard. We did, and we were right. Some of the articles that really helped us are found in this [GitHub Issue](https://github.com/systemEng-Learning/simple-vpn/issues/1).
* Apr 17 - May 10, 2024: Just reading amazing articles wouldn't be enough for us. We had to experiment with tun/tap devices on Linux and analyze network packets. This wasn't part of the milestones, they were just prototypical code. We used Python because of its ease of use and our familiarity with it. These are the [scripts](https://github.com/systemEng-Learning/simple-vpn/tree/main/playtun).
* May 20 - May 23, 2024: Steve implemented a simple tun device software that could process ping requests.
* May 24 - May 31, 2024: Goodness implemented his own simple tun device that could work as a VPN client and server. 
Encryption was also added. The major pain of adding encryption was recalculating the IPv4 packet header checksum. Thankfully, [etherparse](https://github.com/JulianSchmid/etherparse) provided helpful functions for that. Here's the [doc](https://github.com/systemEng-Learning/simple-vpn/tree/main/tunnel-cli) for the software.
* May 31, 2024: Steve upgraded his tunneling software to one that could act as a VPN client and server. It even works with docker 😎. Here's the [doc](https://github.com/systemEng-Learning/simple-vpn/tree/main/tunnel-indocker).
* Jun 3 - Jun 5, 2024: Goodness added ipv6 support to his tun package. 
* Jun 10 - Jun 20, 2024: Steve implemented an L4 localhost tunneling software using mio. Here's the [doc](https://github.com/systemEng-Learning/simple-vpn/tree/main/tunnel-indocker/localhost-tunnel).
* Jun 12 - Jun 17, 2024: Goodness implemented an L3 localhost tunneling software. Here's the [doc](https://github.com/systemEng-Learning/simple-vpn/tree/main/tunnel-local).
* Jun 17 - Jul 8, 2024: Play with Wireguard again, but with more knowledge this time :-), Fix bugs, add readme, and publish articles.

### Participants' Writeups
* [Goodness](https://goodyduru.github.io/networking/2024/07/08/what-i-learned-from-building-a-network-tunnel.html)
* [Steve](https://steveoni.github.io/network/2024/07/05/network-tunnel-in-rust.html)

### Conclusion
This project was a roundabout way of helping us understand Wireguard better. We did it! Now we can explain and talk about it at an admittedly nerdy dinner table. We learnt so much about low-level networking tools and concepts. Imagine dealing with packet checksum issues? It was fun and now we can say our understanding of those cool low-level networking blog articles has improved to 80%!
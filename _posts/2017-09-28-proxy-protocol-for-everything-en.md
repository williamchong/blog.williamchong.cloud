---
layout: post
title: "Docker + HAProxy = PROXY Protocol for Everything"
description: "A comprehensive guide to implementing PROXY protocol support for any TCP network service using Docker networking and HAProxy. Learn how to maintain source IP information across proxies, create transparent proxies without modifying application code, and configure cross-datacenter traffic routing."
date: 2017-09-28 00:00:00 +0800
lastmod: 2025-05-06
categories: english
image: /assets/images/2017-09-28-proxy-protocol-for-everything/1.png
tags: docker haproxy devops proxy transparent-proxy network-architecture tcp-services source-ip-preservation container-networking load-balancing
---

For the Cantonese version, please check [here]({% post_url 2017-09-28-proxy-protocol-for-everything %})

Originally posted on [Lakoo's Medium](https://m.lakoo.com/docker-haproxy-proxy-protocol-for-everyone-f0da0533d87b) on 2017-09-28, translated to English

## MMORPG Experience Sharing: Implementing PROXY Protocol Support for Containerized TCP Services

### Background
Our mobile MMORPG Teon server is hosted in AWS Japan. Recently, Taiwanese players reported network issues that appeared to be related to specific ISP routing problems. Since relocating the server would be overly complex, we decided to establish a proxy server in Taiwan to provide reliable and stable connections for all players.

### The Challenge
The game server needs to record players' IP addresses and cannot lose this client IP information through proxy NAT.

While HTTP proxies typically use headers like X-Forwarded-For to preserve the original connection's IP information, this approach isn't applicable in a pure TCP environment.

Traditional transparent proxy configurations require kernel-level TPROXY support and modification of the default gateway. However, even when using a cross-datacenter AWS-GCP VPN network, the EC2/VPC gateway settings don't allow specifying a server outside of AWS as the internet gateway.

### Solution

![HAProxy with PROXY protocol diagram](/assets/images/2017-09-28-proxy-protocol-for-everything/1.png)
*Using HAProxy with the PROXY protocol as a transparent proxy*

First, within the game server's Docker network, we launch an HAProxy container to act as the default gateway for the entire Docker network:

{% highlight yaml %}
haproxy:
  image: tombull/haproxy
  links:
    - game-server-container # game server's container name
  ports:
    - "8000:8000"
  cap_add:
    - ALL # Note: ALL is used here for demonstration purposes only
  environment:
    HAPROXY_PORTS: 8000
  networks:
    teon-net:
      ipv4_address: 172.20.0.10
  volumes:
    - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
networks:
  teon-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
          gateway: 172.20.0.1
{% endhighlight %}

The image uses [tombull/haproxy](https://hub.docker.com/r/tombull/haproxy/), which already includes the necessary iptable rules. You only need to configure `net.ipv4.ip_nonlocal_bind` and the NET_ADMIN capabilities.

Since Docker Compose doesn't currently support specifying individual container gateways, you need to enable NET_ADMIN within the game server's container and execute IP commands during container initialization to set the gateway:

{% highlight bash %}
ip route delete default
ip route add default via 172.20.0.10
{% endhighlight %}

In HAProxy, we configure a port that accepts the PROXY protocol and overwrite the original connection's IP when connecting to the game server, thereby achieving the effect of a transparent proxy:

{% highlight yaml %}
frontend game-proxy
  mode tcp
  option tcplog
  option clitcpka
  bind 0.0.0.0:8000 accept-proxy
  default_backend teon-servers

backend game-servers
  option tcplog
  mode tcp
  source 0.0.0.0 usesrc clientip # overwrite source IP
  server game game-server-container:8080
{% endhighlight %}

Since the game server's container specifies HAProxy as the default gateway, HAProxy handles the NAT issues for inbound connections. For outbound connections, additional iptable NAT rules are required:

{% highlight bash %}
# Rule for outbound traffic
# `! -d` means do not apply for destination 172.20.0.0/16
iptables -t nat -A POSTROUTING ! -d 172.20.0.0/16 -o eth0 -j MASQUERADE
{% endhighlight %}

For the reverse proxy configuration:

{% highlight yaml %}
backend reverse-proxy
  mode tcp
  server aws-haproxy aws-ip-address-here:8000 send-proxy
{% endhighlight %}

By connecting to the Taiwan HAProxy, clients are reverse-proxied to the AWS HAProxy and then connected to the game server. By controlling the HAProxy instances and backend targets, we can preserve the original IP while maintaining routing control.

### Advantages of the Docker Networking Implementation

This architecture offers significant portability advantages:

1. It eliminates the need to manage iptables and routing for each backend or proxy via separate VMs, reducing startup costs and enabling rapid deployment and modification.

2. The architecture can be managed on a per-docker-compose/stack basis. Conceptually, any TCP-based Docker application can be treated as a single application supporting the PROXY protocol, without requiring extensive network and proxy architecture considerations.

### Conclusion

The PROXY protocol, as a technology for preserving source IP information in proxies, has fewer limitations compared to traditional TPROXY techniques. However, support for the PROXY protocol beyond HTTP proxies remains limited, despite AWS ELB already supporting it.

The solution described above using Docker with HAProxy aims to help teams working with non-web server environments easily benefit from the advantages of the PROXY protocol.

### Additional Resources

[HAProxy and the PROXY Protocol](https://www.haproxy.com/blog/haproxy/proxy-protocol/)

[Preserving Source IP Address Despite Reverse Proxies](https://www.haproxy.com/blog/preserve-source-ip-address-despite-reverse-proxies/)

[Using HAProxy with the PROXY Protocol to Better Secure Your Database](https://www.haproxy.com/blog/using-haproxy-with-the-proxy-protocol-to-better-secure-your-database/)

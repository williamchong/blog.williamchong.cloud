---
layout: post
title: "魔術！乜叉嘢都支援 PROXY protocol！"
description: "全面解析如何使用 Docker 網絡同 HAProxy 令任何 TCP 服務支援 PROXY 協議，無需修改應用程式代碼。呢篇指南詳細介紹透明代理嘅實現步驟，點樣保留源 IP 資訊，以及跨數據中心嘅流量轉發技術。"
date: 2017-09-28 00:00:00 +0800
lastmod: 2025-05-06
categories: cantonese
locale: zh_HK
image: /assets/images/2017-09-28-proxy-protocol-for-everything/1.png
tags: docker haproxy devops proxy 透明代理 網絡架構 TCP服務 源IP保留 容器網絡 負載平衡
---
For english version, please click [here]({% post_url 2017-09-28-proxy-protocol-for-everything-en %})

Docker + HAProxy = 令所有服務都支援 PROXY protocol

Originally posted on [Lakoo's medium](https://m.lakoo.com/docker-haproxy-proxy-protocol-for-everyone-f0da0533d87b) on 2017-09-28

## MMORPG 經驗分享：點樣令容器入面嘅 TCP 網絡服務支援 PROXY protocol

### 背景
我哋嘅手機 MMORPG Teon 伺服器設置喺 AWS 日本區。最近有啲台灣玩家反映網絡唔順暢，懷疑係部分 ISP 線路有問題。因為搬遷伺服器太過麻煩，所以我哋決定喺台灣設立一個代理伺服器，為所有玩家提供穩定可靠嘅網絡連接。

### 問題
遊戲伺服器需要記錄玩家嘅真實 IP 地址，唔可以喺用 proxy NAT 之後就失去咗玩家嘅 IP 資訊。

一般嘅 HTTP Proxy 會用 X-Forwarded-for 之類嘅 Header 嚟保留原本連接嘅 IP 資訊，但係呢個方法喺純 TCP 環境之下係用唔到嘅。

而傳統嘅 Transparent proxy 設定需要 kernel 支援 TPROXY，仲要修改 default gateway。就算用咗跨 AWS GCP 嘅 VPN network，EC2/VPC 嘅 gateway setting 都唔支援指定 AWS 以外嘅伺服器做 Internet gateway。

### 解決方法

![HAProxy with PROXY protocol diagram](/assets/images/2017-09-28-proxy-protocol-for-everything/1.png)
*使用 HAProxy 同 PROXY protocol 做透明代理*

首先，喺遊戲伺服器嘅 docker network 入面，我哋起一個 HAProxy container 做成個 docker network 嘅 default gateway：

{% highlight yaml %}
haproxy:
  image: tombull/haproxy
  links:
    - game-server-container # 遊戲伺服器容器名稱
  ports:
    - "8000:8000"
  cap_add:
    - ALL # 注意：ALL 只係為咗示範方便，實際環境要限制權限
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

我哋用咗 [tombull/haproxy](https://hub.docker.com/r/tombull/haproxy/) 呢個 image，佢已經包含咗所需嘅 iptable rules，只需要設定好 `net.ipv4.ip_nonlocal_bind` 同 NET_ADMIN 權限就得。

因為 docker-compose 目前未支援為個別 container 指定 gateway，所以我哋要喺遊戲伺服器嘅 container 入面開啟 NET_ADMIN 權限，然後喺容器啟動時執行 IP 指令嚟設定 gateway：

{% highlight bash %}
ip route delete default
ip route add default via 172.20.0.10
{% endhighlight %}

喺 HAProxy 度，我哋設定一個接受 PROXY protocol 嘅埠，連接到遊戲伺服器時覆寫返原本連接嘅 IP，咁樣就可以實現透明代理嘅效果：

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
  source 0.0.0.0 usesrc clientip # 覆寫來源 IP
  server game game-server-container:8080
{% endhighlight %}

因為遊戲伺服器嘅 container 指定咗 HAProxy 做 default gateway，所以 HAProxy 會自動處理入站連接嘅 NAT 問題。至於出站連接，根據實際測試，我哋需要加多個 iptable NAT 規則嚟處理：

{% highlight bash %}
# 出站流量嘅規則
# `! -d` 表示唔應用於目標係 172.20.0.0/16 嘅流量
iptables -t nat -A POSTROUTING ! -d 172.20.0.0/16 -o eth0 -j MASQUERADE
{% endhighlight %}

最後喺台灣 HAProxy 嘅中繼設定：

{% highlight yaml %}
backend reverse-proxy
  mode tcp
  server aws-haproxy aws-ip-address-here:8000 send-proxy
{% endhighlight %}

咁樣一嚟，任何連接到台灣 HAProxy 嘅用戶都會被反向代理到 AWS 嘅 HAProxy，再連接到遊戲伺服器。透過控制 HAProxy 嘅數量同 backend 目標，我哋可以保留原始 IP 同時自由控制路由。

### Docker 網絡實現嘅優勢

呢個架構用 Docker 網絡嚟處理有幾個好處，主要係提升咗可移植性：

1. 避免咗每個 backend 或者 proxy 都要開獨立 VM 嚟管理 iptable 同埋 routing，減少咗開機成本，亦都方便快速部署同修改。

2. 可以以每個 docker-compose/stack 為單位嚟管理架構。換句話講，任何用 TCP 嘅 Docker 應用，只要用上面嘅設定方法，喺架構上都可以當成單個支援 PROXY protocol 嘅應用，唔使考慮複雜嘅網絡同 proxy 架構問題。

### 總結

PROXY protocol 作為一種保存源 IP 嘅代理技術，比傳統 TPROXY 技術嘅限制少好多。雖然 AWS ELB 早已支援 PROXY protocol，但喺 HTTP proxy 層面以外嘅支援似乎仲係唔夠全面。

上面講嘅用 Docker 配合 HAProxy 支援 PROXY protocol 嘅方案，希望可以幫到大家喺 web server 以外嘅環境都可以簡單咁享受到 PROXY protocol 嘅好處。

### 延伸閱讀

[HAProxy 同 PROXY Protocol](https://www.haproxy.com/blog/haproxy/proxy-protocol/)

[喺反向代理之下保留源 IP 地址](https://www.haproxy.com/blog/preserve-source-ip-address-despite-reverse-proxies/)

[用 HAProxy 同 PROXY Protocol 令你嘅資料庫更安全](https://www.haproxy.com/blog/using-haproxy-with-the-proxy-protocol-to-better-secure-your-database/)

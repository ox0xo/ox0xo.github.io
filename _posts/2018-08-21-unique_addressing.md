---
layout: post
title:  "IPアドレスの一意性"
date:   2018-08-21
permalink: /networking/:title
categories: TCP/IP
tags: ARP IP
excerpt: IPアドレスやMACアドレスの一意性に関するポストです
mathjax: false
---
 
* content
{:toc}

## L2/L3アドレスの一意性を確認する

### 前置き

2台のコンピュータが相互に通信するためにはお互いの位置を把握する必要があります。
Layer3のIPアドレスとLayer2のMACアドレスがその役目を担います。
これらのアドレスの一意性が確保できないと意図した通信が行われません。

![]({{site.baseurl}}/images/arp/addressing01.png)

![]({{site.baseurl}}/images/arp/addressing02.png)

![]({{site.baseurl}}/images/arp/addressing03.png)

### 目的

同一ネットワーク（ブロードキャストドメイン）内でIPアドレスやMACアドレスを重複させてpingが届かなくなることを確認します。
応用としてARPスプーフィングを用いてパケットを盗聴出来ることを確認します。

### 手順

2台以上の物理マシンを用意します。物理マシンは同一ネットワーク上に配置してお互いにpingで疎通確認してください。

その後お互いのNICのMACアドレスを確認します。

VirtualBoxのネットワーク設定から任意のMACアドレスを設定することが出来ます。
あらかじめ確認した通信相手のMACアドレスと同じ値を指定してください。

作業は仮想マシンを停止してから行います。

![]({{site.baseurl}}/images/arp/bridge_mac.png)

仮想マシンを起動してMACアドレスが指定した値になっている事を確認します。
この状態で通信相手にpingが届かなくなっていることを確認してください。

```
# ip address show enp0s8
```

![]({{site.baseurl}}/images/arp/mac2.png)

MACアドレスを元に戻し、その代りにIPアドレスが同一ネットワーク上で重複している状態を作り出してください。
この状態で通信相手にpingが届かなくなっていることを確認してください。

![]({{site.baseurl}}/images/arp/ip.png)

同一ネットワーク上のマシンが送信したARPパケットは逐一記録されます。
これを確認するためにARPテーブル（ネイバーテーブル）も参考にしてください。

```
ネイバーテーブルを確認する
# ip neighbor

５つのIPアドレス：MACアドレスの対応表が記録されていることがわかる
192.168.0.200 dev enp0s8 lladdr 08:00:27:fa:98:e5 STALE
10.0.2.2 dev enp0s3 lladdr 52:54:00:12:35:02 REACHABLE
192.168.0.205 dev enp0s8 lladdr 08:00:27:a1:72:f2 STALE
192.168.0.210 dev enp0s8 lladdr 08:00:27:fa:98:e5 STALE
192.168.0.254 dev enp0s8 lladdr 08:00:27:fa:98:e5 STALE
192.168.0.250 dev enp0s8 lladdr 08:00:27:fa:98:e5 STALE
```

### 応用

IPアドレスが正しくてもMACアドレスに問題があると通信に支障が出ることがわかりました。
悪意ある第三者が正規の通信を行っている2者のMACアドレスを収集し、自分のMACアドレスを騙ったらどうなるか試してみましょう。

```
# sysctl -w net.ipv4.conf.enp0s8.forwarding=1
# arpspoof -i enp0s8 -t 10.0.0.2 10.0.0.3
# arpspoof -i enp0s8 -t 10.0.0.3 10.0.0.2
```

### まとめ

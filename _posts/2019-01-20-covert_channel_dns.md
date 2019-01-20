---
layout: post
title:  "covert channel over DNS"
date:   2019-01-20
permalink: /security/:title
categories: Security
tags: DNS iodine covert_channel
excerpt: DNS通信に見せかけたC2通信を解析するためのアイデアを模索するものです
---

* content
{:toc}

# 初めに

IPSを運用しているとIodine DNS Tunnelというワードを目にすることがあります。
DNS通信にデータを埋め込むことで別の目的を持ったプロトコルを隠蔽する手法がDNS Tunnelです。
DNSの他にもICMPやHTTPの中に別の目的を持った通信を隠蔽することがあり、
このように通信を隠蔽する手法はCovert Channelと呼ばれています。
IodineはDNS Tunnnelを使って手軽にIPv4プロトコルを隠蔽できるVPNソフトです。

このポストでは実際にIodineを使ってDNS Tunnelを作成し、パケットをWiresharkで観察してDNS Tunnelを検出する手法を考えるきっかけを作ります。
また、Iodineに頼らないCovert Channelの実例を一つ取り上げて、脅威が身近に存在している事を認識するきっかけを作ります。

# Settings

検証環境は2台のUbuntuで構成されています。
2台は共通のネットワークセグメントである172.16.0.0/24に属していますが、Webサーバは123.0.0.1でHTTPリクエストを待ち受けており、このままではクライアントからアクセスできません。
クライアントがWebサーバにアクセスするには、DNS Tunnnelを使って123.0.0.0/27のVPNに接続する必要があります。

```
root@server:~# ip a s enp0s8
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:a8:1f:fb brd ff:ff:ff:ff:ff:ff
    inet 172.16.0.100/24 brd 172.16.0.255 scope global noprefixroute enp0s8
       valid_lft forever preferred_lft forever

root@server:~# cat /etc/nginx/conf.d/default.conf
server {
        listen  123.0.0.1:80;
        root /var/www/html/;
        index index.html;
}

root@server:~# cat /var/www/html/index.html
welcome to the iodine tunnel. have fun!
```

```
root@client:~# ip a s enp0s8
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:78:7a:9f brd ff:ff:ff:ff:ff:ff
    inet 172.16.0.200/24 brd 172.16.0.255 scope global noprefixroute enp0s8
       valid_lft forever preferred_lft forever
```

# VPN connection

サーバ側でIodinedを起動して123.0.0.0/27のVPNを用意します。
DNS TUnnelで利用するドメインはc2.netとします。

```
root@server:~# iodined -f 123.0.0.1 c2.net
Enter password:
Opened dns0
Setting IP of dns0 to 123.0.0.1
Setting MTU of dns0 to 1130
Opened IPv4 UDP socket
Listening to dns for domain c2.net
```

クライアント側でIodineを使ってサーバに接続します。
ドメインとパスワードが合っていればVPNが開通します。

```
root@client:~# iodine -f -r 172.16.0.100 c2.net
Enter password:
Opened dns0
Opened IPv4 UDP socket
Sending DNS queries for c2.net to 172.16.0.100
Autodetecting DNS query type (use -T to override).
Using DNS type NULL queries
Version ok, both using protocol v 0x00000502. You are user #0
Setting IP of dns0 to 123.0.0.2
Setting MTU of dns0 to 1130
Server tunnel IP is 123.0.0.1
Skipping raw mode
Using EDNS0 extension
Switching upstream to codec Base128
Server switched upstream to codec Base128
No alternative downstream codec available, using default (Raw)
Switching to lazy mode for low-latency
Server switched to lazy mode
Autoprobing max downstream fragment size... (skip with -m fragsize)
768 ok.. 1152 ok.. ...1344 not ok.. ...1248 not ok.. ...1200 not ok.. 1176 ok.. 1188 ok.. will use 1188-2=1186
Setting downstream fragment size to max 1186...
Connection setup complete, transmitting data.
```

```
root@server:~# ip a s dns0
9: dns0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1130 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none
    inet 123.0.0.1/27 scope global dns0
       valid_lft forever preferred_lft forever
```

```
root@client:~# ip a s dns0
10: dns0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1130 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none
    inet 123.0.0.2/27 scope global dns0
       valid_lft forever preferred_lft forever
```

# VPN over DNS

VPN経由でWebサーバのコンテンツにアクセスすると大量のDNSクエリが流れます。
アップストリームはDNSリクエストのドメイン名として送信されており、ダウンストリームはDNSレスポンスのNULLレコードとして返信されています。

```
root@client:~# curl 123.0.0.1
welcome to the iodine tunnel. have fun!
```

![](/images/iodine/wireshark01.png)

![](/images/iodine/wireshark02.png)

## Response Type

Iodineは利用される環境に応じてパケットのサイズやDNSレコードのタイプを自動的に決定しますが、
全ての環境で利用するためには微調整が必要です。
何もしなければNULLレコードにレスポンスが埋め込まれますが、
-Tオプションを使えばSRV、TXT、CNAME、MX、PRIVATE(Unknown)にレスポンスを埋め込むこともできます。

- SRV

  ![](/images/iodine/wireshark04.png)

- TXT

  ![](/images/iodine/wireshark05.png)

- CNAME

  ![](/images/iodine/wireshark07.png)

- MX

  ![](/images/iodine/wireshark08.png)

- PRIVATE

  ![](/images/iodine/wireshark09.png)

## Packet Length

Iodineがリクエストを送信する際はドメイン名の最大長である255byteに収まるようにデータを分割しますが、
255byteのドメイン名は一般的なドメイン名に比べてかなり長く、監視の目にかかりやすくなります。
-Mオプションを使えばドメイン名の最大長を指定することが出来ます。
マニュアルによると-Mで指定できる最小値は100byteとされていますが、ローカルの検証環境では43byteまで切り詰めることが出来ました。
対象の環境によってこの値は変化します。

![](/images/iodine/wireshark03.png)

ドメイン名から離れてパケット全体の長さに目を向けると、Iodineのパケットと通常のDNSパケットを見分けることは困難になります。
例えばgithub.comのDNSレスポンスは333byteでありIodineによるパケットよりも大きくなります。
大規模なサイトは耐障害性を考慮してDNSレコードが肥大化する傾向があるためです。

![](/images/iodine/wireshark06.png)

## Domain name

DNS Tunnelを構築するためのフェイクドメインは自由に指定できます。
AWSやAzureのドメインは長くなることが多いためフェイクドメインの候補になります。
次の画像はDNS Tunnelを利用した通信ですが一見するとamazonaws.comのDNSクエリに見えます。

![](/images/iodine/wireshark10.png)


# その他のCovert Channel

Iodineのような専用のソフトウェアを利用しなくても容易にCovert Channelを構築することが出来ます。
中でもDNSのTXTレコードは比較的大きなデータを埋め込めて、多くの組織で通信が許可されているためよく使われる手法です。

次の例ではcalc.exeのバイナリをHEXエンコードして、hidden.netドメインのTXTレコードに分割して書き込んであります。
calc.exeは圧縮しても11KBありますが、一つのTXTレコードに1024byteのデータを書き込めるので11レコードあれば全体を格納できます。

```
@       IN      SOA     hidden.net.     root.hidden.net. (
        2019011901
        10800
        3600
        604800
        86400 )
;
        IN      NS      dns.hidden.net.
;
hidden.net.    IN  TXT  "12"  ; record count
0.hidden.net.  IN  TXT  "789ced3d0b7413d7954fb60cb6b1b1686d62c227b263769d8f6d192159b22d23c796b14f31086c6c93406c218d2d51fd3a9a31360ba91de134ea40c266d313dad39c853ab46437a74db7d9f069b62b1312431a284dd386b6d9c6bb353d624d13daa6096928da7b6724ebe391200da561a3c7b9f3debbefbefb7bf7dd37a3996"
1.hidden.net.  IN  TXT  "30d9cdacddc4b8e836c644331486969998881dae2c5c4d84212e42936a5201ff8412ce11188a4500399f0198153b960bb011081430a6881b8bdef7f1c518d2cbd0dea940ba87e64a1fbe2be3cbdf4f9f9311a248237746abdd6973b69a6c4e523cc392629226cd7a25236bae6989fe796345d11d434726ee20ad3633edf2b87"
2.hidden.net.  IN  TXT  "fb9ae15ae86bcef175c97c03f953b3bc720208ce2d1f798f591c749704d992ceb6b541369f3bbd2ec8e604d9cc600126aee0cfb8d6cbddf72b4fdd17d809ccb8739b8e959057630a177c9157d17ba12ba26407a792427bb7e170e7ee1d87ad85d8eebf78c04aa061dd93af27c603070e28dff3a9e480f0a94ae01a2cc804812"
(snip)
11.hidden.net.  IN  TXT  "adf6f7daa0766ef5fcea25d5cbaaababebab77573f51fd64f5b7aadfaa3e57fd4ef51fabaf54a7d72caca9ab79ba66ac66bce64ccd1b352b6bfb6bcfd6fe57ed3bb5b9ba22dddfebded20574bfd565d76daeebab1ba9fb46dd5b7597ebc88a452be42b6a57e857e0678239e052297e38a810da7b13fc0db9bf75f93f0b95b20"
```

このTXTレコードからバイナリを再構築して実行するスクリプトの例は次の通りです。
特別なツールは使われていないのでjavascriptやpowershellで書くことも出来るはずです。

<script src="https://gist.github.com/ox0xo/49295570b361431165d384b1c1510d79.js"></script>

![](/images/iodine/covertchannel.png)

# まとめ

一見正常に見える通信に他の目的を持ったデータを埋め込むCovert Channelという攻撃手法があります。
Covert ChannelはIodineのようなツールを使ったり簡単なスクリプトを書く事で容易に実現することが出来ます。
ある一つの面だけを捉えるのではなく複数の観点から脅威の可能性を調査する事をお勧めします。

Covert Channelは本来の目的とは違う利用方法であるため通信効率が悪く、
通常よりも通信の頻度が高くなる傾向があります。
普段見かけない連続した通信ログがあれば疑いの目を向けた方が良いでしょう。

Covert Channelを構築するためには攻撃者の支配下にあるサーバが必要です。
通信先のIPアドレスが信頼できる組織に管理されているか確認した方が良いでしょう。
そのIPアドレスがレンタルサーバ等の不特定多数のユーザーに提供されるものならば、
より慎重に調査することをお勧めします。

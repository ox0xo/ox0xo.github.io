---
layout: post
title:  "IPv6 Getting Started"
date:   2018-11-28
permalink: /networking/:title
categories: TCP/IP
tags: IPv6
excerpt: 今までIPv4を利用していた人がIPv6をとりあえず使えるようにするために見るべき要素をまとめました
mathjax: false
---

* content
{:toc}
 
## はじめに

IPv6はこれまでのIPv4を代替するものなので、それ以外のアプリケーションプロトコル等をほぼそのまま利用できます。
とはいっても、IPv6に関わるプロトコルは非常に広範囲にわたっており、いきなり全てを理解することは困難です。
このポストは、今までIPv4で提供していたサービスをIPv6に対応する際に、どこから始めればよいのか判断する手助けをする事を目標にしています。

このポストをまとめるにあたって[Professional IPv6](https://professionalipv6.booth.pm/)を参照しています。

## IPアドレス体系
### 表記ルール

IPv6アドレスは128bitの長さを持ちます。
16進数表記されますが、それでも長いので省略表記ルールが定義されています。
代表的なルールは次の通りですがこの他にも細かなルールが[RFC5952](https://www.nic.ad.jp/ja/newsletter/No46/0800.html)に定義されています。

- 各16bitフィールドの先頭の連続する0を省略できる
```
2001:0bd8:11aa:22bb:33cc:44ee:55ff:0006
↓
2001:bd8:11aa:22bb:33cc:44ee:55ff:6
```

- 複数の16bitフィールドに渡って連続する0を::に置き換えできる
```
2001:0db8:0000:0000:0000:0002:0000:0001
↓
2001:0db8::2:0:1
```

IPv6アドレスはサブネットプレフィックスとインタフェースIDから構成されています。
異なるサブネットプレフィックスを持つインタフェースから送信されたパケットを受け取るためにはIPv4と同じようにルーティングが必要です。

![引用元　JPNIC インターネット10分講座：IPv6アドレス～技術解説～](https://www.nic.ad.jp/ja/newsletter/No32/images/090_01.jpg)


### リンクローカルアドレス

リンクローカルアドレスは同一セグメント間で通信するためのアドレスであり、このアドレスから送信されたパケットはルータを通過することが出来ません。
IPv6ネットワークインタフェースには必ずリンクローカルアドレスが割り当てられます。
リンクローカルアドレスの上位10bitは`fe80`です。

### ユニークローカルアドレス

ユニークローカルアドレスは組織内の複数のサブネットプレフィックス間に渡って通信するためのアドレスです。
IPv4におけるプライベートIPアドレスに該当します。
ユニークローカルアドレスの上位8bitは`fd00`です。

![引用元　JPNIC インターネット10分口座：IPv6アドレス～技術解説～](https://www.nic.ad.jp/ja/newsletter/No32/images/090_03.jpg)

グローバル識別子の算出方法は[RFC4193 3.2.2 Sample Code for Pseudo-Random Global ID Algorithm](https://tools.ietf.org/html/rfc4193#section-3.2.2)に定義されています。

1. 現在時刻を64bit NTP形式で取得する
2. インタフェース識別子として[EUI-64識別子](https://ja.wikipedia.org/wiki/IPv6%E3%82%A2%E3%83%89%E3%83%AC%E3%82%B9#Modified_EUI-64)を取得する
3. 1と2を連結してキーを作る
4. 連結したキーのSHA1ダイジェストを計算する
5. SHA1ダイジェストの最下位40bitをグローバル識別子として利用する

サブネット識別子はその組織内で重複しないように設定する必要があります。
営業部のサブネット識別子がAABBで技術部のサブネット識別子はCCDDのように設定します。

### マルチキャストアドレス

IPv6ではIPv4に存在したブロードキャストという仕組みがなくなり、ブロードキャストが担っていた役割をすべてマルチキャストが担うようになりました。
マルチキャストアドレスは先頭8bitが`ff00`です。

更にマルチキャストが配送されるスコープを以下の通り定義しています。
IPv4におけるブロードキャストは、IPv6ではリンクローカルスコープのマルチキャストが担います。
```
ff01::0/16  # インタフェースローカル
ff02::0/16  # リンクローカル
ff03::0/16  # レルムローカル
ff04::0/16  # アドミンローカル
ff05::0/16  # サイトローカル
ff08::0/16  # 組織ローカル
ff0e::0/16  # グローバルスコープ
```

## IPアドレスの割り当て
### SLAAC

IPv6にはIPアドレスを自動的に設定する機能が組み込まれておりSLAAC(State Less Address Auto Configuration)と呼ばれています。

1. EUI-64フォーマットのインタフェースIDを取得する
2. インタフェースIDからリンクローカルアドレスを仮設定する
3. NAパケットとNSパケットを使ってリンクローカルアドレスが重複していないか確認する
4. 重複していなければリンクローカルアドレスを正式採用する
5. リンクローカルアドレスを送信元にしてRSパケットをリンクローカルマルチキャストする
6. ルーターはRAパケットにプレフィックスとゲートウェイの情報を含めて返信する
7. プレフィックスとインタフェースIDからユニキャストアドレスを自動生成する

### NSパケットとNAパケット

IPv4においてARPメッセージが担っていた役割を、IPv6ではICMPv6 Neighborメッセージが担っています。
以下のネットワーク構成でNS/NAパケットの挙動を確認していきます。

```
# ip a s enp0s8
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:ec:e1:e7 brd ff:ff:ff:ff:ff:ff
    inet6 1920:1680::1/64 scope global noprefixroute
       valid_lft forever preferred_lft forever
    inet6 fe80::a550:a30:4e93:50bc/64 scope link
       valid_lft forever preferred_lft forever

 # ip a s enp0s8
 3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
     link/ether 08:00:27:87:f2:d4 brd ff:ff:ff:ff:ff:ff
     inet6 1920:1680::2/64 scope global noprefixroute
        valid_lft forever preferred_lft forever
     inet6 fe80::311c:887d:690:b5a6/64 scope link
        valid_lft forever preferred_lft forever

# ping -6 1920:1680::1
PING 1920:1680::1(1920:1680::1) 56 data bytes
64 bytes from 1920:1680::1: icmp_seq=1 ttl=64 time=0.941 ms
64 bytes from 1920:1680::1: icmp_seq=2 ttl=64 time=1.02 ms
64 bytes from 1920:1680::1: icmp_seq=3 ttl=64 time=0.952 ms
```

Neighbor Solicitationパケットは`ff02::1:ff**:****`宛てに送出されています。
アドレスの末尾24bitはエニーキャストアドレスから算出されており、自分宛のパケットを受け取ったインタフェースはNeighbor Advertisementパケットで応答します。

![](/images/ipv6/ping.png)

>以下の例はIPv4におけるDHCPのパケットです
>
>![](/images/ipv6/ipv4_ping.png)

NS/NAパケットを利用してDAD (Duplicate address detection)が行われます。
リンクローカルアドレスが重複していた場合はインタフェースのステータスがdadfailedとなり利用できません。

![](/images/ipv6/dad.png)

```
# ip a s enp0s8
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:87:f2:d4 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.2/24 brd 192.168.0.255 scope global noprefixroute enp0s8
       valid_lft forever preferred_lft forever
    inet6 1920:1680::1/64 scope global tentative noprefixroute dadfailed
       valid_lft forever preferred_lft forever
    inet6 fe80::311c:887d:690:b5a6/64 scope link
       valid_lft forever preferred_lft forever
```

NSパケットにNAパケットの応答がなければ、そのアドレスを利用している他のノードが居ないということなので、そのアドレスを継続利用できます。

![](/images/ipv6/linkup.png)

```
# ip a s enp0s8
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:87:f2:d4 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.2/24 brd 192.168.0.255 scope global noprefixroute enp0s8
       valid_lft forever preferred_lft forever
    inet6 1920:1680::10/64 scope global noprefixroute
       valid_lft forever preferred_lft forever
    inet6 fe80::311c:887d:690:b5a6/64 scope link
       valid_lft forever preferred_lft forever
```

### DHCPv6

IPv6ではIPv4におけるDHCPと似たDHCPv6というプロトコルが利用されます。
DHCPv6には大きく分けてステートレスDHCPv6、ステートフルDHCPv6、DHCPv6-PDの３種類の利用形態があります。

1. EUI-64フォーマットのインタフェースIDを取得する
2. インタフェースIDからリンクローカルアドレスを仮設定する
3. Neighbor AdversitementとNeighbor Solicitationパケットを使ってリンクローカルアドレスが重複していないか確認する
4. 重複していなければリンクローカルアドレスを正式採用する
5. リンクローカルアドレスを送信元にしてRouter Solicitationパケットをリンクローカルマルチキャストする
6. ルーターはRouter AdvertisementパケットにManaged address configurationフラグを付けて返信する
1. DHCPv6 Solicitパケットをリンクローカルマルチキャストする
1. DHCPv6サーバはDHCPv6 Replyメッセージを返信する
7. DHCPv6 Information requestパケットをリンクローカルマルチキャストする
8. DHCPv6 Replyパケットにユニキャストアドレスやネットワークに付随する情報を含めて返信する
9. Neighbor AdvertisementとNeighbor Solicitationパケットを使ってユニキャストアドレスが重複していないか確認する
10. 重複していなければユニキャストアドレスを正式採用する

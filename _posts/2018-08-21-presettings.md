---
layout: post
title:  "サービス制御"
date:   2018-08-21
permalink: /networking/:title
categories: TCP/IP
tags: Linux
excerpt: centos7のサービスを管理する方法に関するポストです
mathjax: false
---

* content
{:toc}

## 不要なサービスを停止する

配布した仮想マシンのLinuxには様々なサービスが導入されています。
IPアドレスやポートフォワードの設定ミスや意図しないブロードキャストによってネットワーク上の他のホストに影響が出る可能性があります。
今動いているサービスを把握して不要な物は停止しましょう。

![]({{site.baseurl}}/images/dhcp/network.png)

![]({{site.baseurl}}/images/dhcp/network2.png)

![]({{site.baseurl}}/images/dhcp/network3.png)

![]({{site.baseurl}}/images/dhcp/network4.png)

```
動作中のサービス一覧確認
# systemctl status --no-pager
```

今回の研修用にインストールしているサービスは次の通りです。
仮想ルータ、OSPF、BGPは後日取り扱います。

|サービス|モジュール名|
|---|---|
|DNS|named|
|HTTP|httpd|
|DHCP|dhcpd|
|SMTP|postfix|
|POP3/IMAP|dovecot|
|SSH|sshd|
|TELNET|telnet.socket|
|SNMP|snmpd|
|PROXY|squid|
|ファイアウォール|firewalld|
|仮想ルーター|zebra|
|OSPF|ospfd|
|BGP|bgpd|

```
例：DNSサービスの状態を確認する
# systemctl status named -a

例：DHCPサービスの状態を確認する
# systemctl status dhcpd -a
```

サービスの状態を変更するには以下のコマンドを使います。

```
例：DHCPサービスを停止する
# systemctl stop dhcpd

例：DHCPサービスを開始する
# systemctl start dhcpd

例：OSの起動と同時にサービスが起動するようにする
# systemctl enable dhcpd

例：OSの起動と同時にサービスが起動しないようにする
# systemctl disable dhcpd
```

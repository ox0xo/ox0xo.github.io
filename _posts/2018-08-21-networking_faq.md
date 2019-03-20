---
layout: post
title:  "FAQ"
date:   2018-08-21
permalink: /networking/:title
categories: TCP/IP
tags:
excerpt: ネットワーク研修のFAQです
mathjax: false
---
 
* content
{:toc}

## NAT-T
*IPsec-VPNでNATを超えるためのカプセル化にはなぜUDPが使われるの？*

IPsecはネットワーク層で動作するプロトコルなので、ネットワーク層以上のデータを暗号化して通信経路上の秘密を守ります。
TCP/UDPパケットの情報も暗号化されるので、通信経路上の機器がポート番号に依存する場合は支障があります。
そこで、暗号化されたパケットに仮のポート番号を付与するためにUDPヘッダを付与します。
これがNAT-Tと呼ばれるNAT超えの概要です。
NAT超えの技術には様々なバリエーションがあり、これとは違う仕組みで通信するVPNプロトコルも存在します。

TCPパケットをNAT-Tで送る場合を考えてみましょう。
VPNサーバはまず仮のUDPヘッダを外してESPパケットを復号してTCPパケットを取り出します。
TCPはコネクション型通信を提供するプロトコルなので3wayハンドシェイクが始まります。
VPNサーバはSYN+ACKを暗号化して仮のUDPヘッダを付与してVPNクライアントへ送り返します。
通信の途中でパケットが欠落した場合などはTCPの仕様に基づいて再送処理が行われます。

![]({{site.baseurl}}/images/networking-faq/ipsec1.png)

仮にNAT-TのヘッダがTCPだった場合はどうなるでしょうか。
VPNサーバからVPNクライアントへSYNを送る前に外側のTCPコネクションを成立させる必要があります。
TCPコネクションが2重に確立されることになりますが信頼性が2倍になるわけでもないので完全に無駄な処理です。

![]({{site.baseurl}}/images/networking-faq/ipsec2.png)

この問題はしばしばTCP over TCPとして取り上げられます。
IPsecライクなソフトウェアであるCIPEを開発したOlaf TitzはTCP over TCPについて次のようなコメントを残しています。

[なぜTCP Over TCPは悪いアイデアか](https://shugo.net/docs/tcp-tcp.html)

## 再帰問い合わせ
*演習用DNSから外部にDNSリクエストが送信されているのは何故？*

`www.sample.local`のような記述のことをFQDN(Fully Qualified Domain Name)と呼びます。
sample.localというドメイン（組織）に属しているwwwというサーバをこのように記します。
インターネット上に存在するホストにアクセスするときは宛先を間違えないように必ずFQDNで指定します。

FQDNは長いので入力する文字列をなるべく短くするためにDNSサフィックスという工夫が生まれました。
wwwのようなホスト名だけが与えられた場合に自動的にドメイン部分を補完して問い合わせをしてくれる機能です。
Windowsの以下のプロパティで設定することが出来ます。

![]({{site.baseurl}}/images/networking-faq/suffix.png)

この状態で`www.sample.local`の名前解決を行った時のパケットです。
FQDNとして認識されなかったようで組織のドメインが補完されていることが分かります。

![]({{site.baseurl}}/images/networking-faq/dnsquery_suffix_added.png)

DNSクエリの流れは以下の通りです。

![]({{site.baseurl}}/images/networking-faq/dnssuffix01.png)

![]({{site.baseurl}}/images/networking-faq/dnssuffix02.png)

DNSサーバは分散システムであり自分が知らないホスト名は外部に問い合わせる機能を持っています。
よりシンプルな挙動に着目して観察したい場合は外部に問い合わせる機能を無効化することも可能です。

```
再帰問い合わせを無効（recursion no）

# cp /etc/named.conf /etc/named.conf.old
# sed "s/recursion yes/recursion no/g" /etc/named.conf.old > /etc/named.conf
```

## SSHdelay
*sshでログインするのに時間がかかるのはなぜ？*

sshプロトコルは多機能な分、それらがうまく機能しない環境ではリトライを繰り返して処理に時間がかかることがあります。
以下のコマンドで不要な機能を無効化することで状況を改善することができます。

```
$ su -
パスワード入力

# echo "UseDNS no" >> /etc/ssh/sshd_config
# echo "AddressFamily inet" >> /etc/ssh/sshd_config
# cp /etc/ssh/sshd_config /etc/ssh/sshd_config.old
# sed "s/GSSAPIAuthentication yes/GSSAPIAuthentication no/g" /etc/ssh/sshd_config.old > /etc/ssh/sshd_config
# systemctl restart sshd
```

## promiscuous
*設定が正しいのにブリッジ接続できないのはなぜ？*

VirtualBoxのネットワークドライバには不具合がありブリッジ接続が出来なくなることがあるらしい。
仮想マシンから外部にpingを送信しているのにホストマシン側でパケットが観測できない場合はその可能性が高い。
以下の手順でVirtualBoxのブリッジを作り直して改善するか試してみる。

1. 対象の仮想マシンのパラメータを確認する

この先に必要なパラメータは`操作対象の仮想マシンの名前` `ブリッジアダプターの番号` `ブリッジアダプターの名前`。
画像の例ではそれぞれ`cent7` `2` `Realtek PCIe GBE Family Controller`となっている。

![]({{site.baseurl}}/images/networking-faq/vboxmanage.png)

2. VirtualBoxのインストールディレクトリにあるVBoxManage.exeを確認する

![]({{site.baseurl}}/images/networking-faq/vbox_installdir.png)

3. 対象(cent7)のブリッジアダプター(nic2)を初期化(null)する

```
Windowsコマンドプロンプト

> VBoxManage.exe controlvm cent7 nic2 null
```

4. NIC2が未割当てになったことを確認する

![]({{site.baseurl}}/images/networking-faq/vboxmanage2.png)

5. NIC2をブリッジアダプターとして再構成する

```
Windowsコマンドプロンプト

> VBoxManage.exe controlvm cent7 nic2 bridged "Realtek PCIe GBE Family Controller"
```

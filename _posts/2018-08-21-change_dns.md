---
layout: post
title:  "DNSレコードの登録"
date:   2018-08-21
permalink: /networking/:title
categories: DNS
tags: dns
excerpt: DNSレコードの登録に関するポストです
mathjax: false
---

* content
{:toc}

## DNSレコードの登録

- 既存設定の確認

手元の仮想マシンのDNSには`www.sample.local`の設定が既に登録されています。
DNSの設定は`ゾーン`という単位で管理します。
1台のDNSサーバが複数のゾーン情報を持つことが出来ます。

![]({{site.baseurl}}/images/dns/dns_zone.png)

以下の例ではsample.localドメインに3つのホスト名が登録されている事が分かります。

```
sample.localのゾーンを確認する

# cat /var/named/sample.local.zone
```
```
@     IN    SOA    sample.local.  root.sample.local. (
                           2018062901  
                           10800       
                           3600       
                           604800     
                           86400 )
;
      IN    NS     ns.sample.local.
;
      IN    MX     0 mail.sample.local
;
ns    IN    A      172.16.0.100
www   IN    A      172.16.0.100
mail  IN    A      172.16.0.100
```
ゾーンの情報を読み込むためにDNSサーバの設定も必要です。
```
DNSサーバの設定ファイルを確認する

# cat /etc/named.conf
```
```
...省略
zone "sample.local" IN {
        type master;
        file "sample.local.zone";
};
...省略
```

- ゾーンの追加

`www.practice.local`=172.16.0.200という設定を追加するケースを考えます。
```
入力待ち受け状態に遷移する
END という文字が渡されるまで入力された文字をファイルに書き込み続ける
# cat >> /var/named/practice.local.zone << END
```
```
渡す文字列（この下の行からENDまでコピー＆ペーストする）
@     IN    SOA    practice.local.  root.practice.local. (
                           2018062901  
                           10800       
                           3600       
                           604800     
                           86400 )
;
      IN    NS     ns.practice.local.
;
      IN    MX     0 mail.practice.local
;
ns    IN    A      172.16.0.200
www   IN    A      172.16.0.200
mail  IN    A      172.16.0.200
END
```
- DNSサーバ設定追加

`practice.local`のゾーン情報を読み込むための記述を追加します。
```
入力待ち受け状態に遷移する
# cat >> /etc/named.conf << END
```
```
渡す文字列（この下の行からENDまでコピー＆ペーストする）
zone "practice.local" IN {
        type master;
        file "practice.local.zone";
};
END
```

- DNSサーバ再起動

再起動することで設定を読み込みます。
もし誤りがあれば再起動に失敗するのでエラーメッセージからトラブルシューティングを行います。

```
# systemctl restart named
```

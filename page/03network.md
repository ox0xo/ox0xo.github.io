---
layout: page
title: TCP/IP
permalink: /networking/
icon: globe
type: page
---

* content
{:toc}

これは2018.07.03から某社で開催された新入社員研修のコンテンツです。

## 総合案内

* 目的
  - マスタリングTCP/IP入門編の範囲の会話が出来るようになる
  - TCP/IPプロトコルスタックを自分の言葉で説明出来るようになる
  - 各レイヤーの特性を生かしたネットワークを構築できるようになる（オプション）

* 対象者
  - 必須
    - 何らかの形でインターネットに触れたことがある人
    - IT業界でエンジニアとして仕事をしたいと考える人
  - 尚良し
    - パソコンで文章作成が可能なレベルのキーボード操作ができる人
    - 趣味などで日常的にパソコンを操作している人

* 期間
  - 2018-07-03（火）～2018-07-23（金）の13日間

* 受講条件
  - [マスタリングTCP/IP入門編](https://www.amazon.co.jp/dp/4274068765)を持参すること
  - 次の要件を満たすPCを持参すること
    - Wi-Fi経由でインターネットに接続出来る
    - 次のアプリケーションをインストール出来る
      - [Wireshark](https://www.wireshark.org/)
      - [VirtualBox](https://www.virtualbox.org/)
      - [RLogin](https://github.com/kmiya-culti/RLogin/)
      - [Becky!](http://www.rimarts.co.jp/becky-j.htm)
      - [TWSNMP](http://www.twise.co.jp/twsnmp.html)

* WiFiの設定

  研修中のインターネット通信はWiFi経由で行ってください。
  有線LANは研修用ネットワークに接続します。

* 講師との連絡手段

  研修に関する連絡事項のメールを受け取れるようにしてください。
  研修中に講師が席を外している場合はSkypeで質問のメッセージを送ってください。

* OneDriveの設定

  メールに添付できないファイルはOneDriveで共有します。

## フェーズ1
3日間の研修でTCP/IPプロトコルの概要と具体的なアプリケーションの扱い方を学びます。
3日目には5分程度のプレゼンテーションをしてもらいます。
1つ以上のアプリケーションをテーマに選び、実際に動作する環境と合わせてアプリケーションの目的や仕組みなどを紹介して下さい。
### [TCP/IPパケット概要](wireshark)
### [リモートログイン](remote_login)
### [サービス制御](presettings)
### [Mailの観測](mail)
### [HTTPの観測](http)
### [DNSの観測](dns)
### [Proxyの観測](proxy)
### [SNMPの観測](snmp)

## フェーズ2
3日間の研修でデータリンク層、ネットワーク層、トランスポート層の詳細を学びます。
3日目には受講者同士の仮想マシンを連携させたシンプルなネットワークを構築し、設計と実装のポイントを5分程度で簡潔に説明して頂きます。
ルーティングを駆使した複雑なネットワークはPhase3で取り扱うため今は考えなくても結構です。
### [サブネット](ipsubnet)
### [ブリッジを使う](bridge)
### [アドレスの一意性](unique_addressing)
### [DHCPの観測](dhcp)
### [VLANを使う](vlan)
### [スパニングツリー](stp)
### [ファイアウォール](firewall)

## フェーズ3
4日間（水、木、金、火）の研修でルーティングの詳細を学びます。
今回はルーティングの演習発表は行いません。
その代わり次のフェーズでこれまでの研修全体を踏まえたネットワークを構築し設計と実装のポイントを説明して頂きます。
### [static routing](routing)
### [DMZ](dmz)
### [OSPF](ospf)
### [BGP](bgp)
### [Network 環境構築](practice1)

## FAQ
* [IPsec-VPNでNATを超えるためのカプセル化にはなぜUDPが使われるの？](networking_faq#nat-t)
* [演習用DNSから外部にDNSリクエストが送信されているのは何故？](networking_faq#再帰問い合わせ)
* [sshでログインするのに時間がかかるのはなぜ？](networking_faq#sshdelay)
* [設定が正しいのにブリッジ接続できないのはなぜ？](networking_faq#promiscuous)

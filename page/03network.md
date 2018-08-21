---
layout: page
title: Networking
permalink: /networking/
icon: globe
type: page
---

* content
{:toc}

## WiFiの設定

研修中のインターネット通信はWiFi経由で行ってください。
有線LANは研修用ネットワークに接続します。

## 講師との連絡手段

研修に関する連絡事項のメールを受け取れるようにしてください。
研修中に講師が席を外している場合はSkypeで質問のメッセージを送ってください。

## OneDriveの設定

メールに添付できないファイルはOneDriveで共有します。

## 必須アプリケーション
- VirtualBoxのインストール + CentOS7.ovaのインポート
- Becky!のインストール
- Wiresharkのインストール
- RLoginのインストール
- TWSNMPのインストール

## フェーズ1
3日間の研修でTCP/IPプロトコルの概要と具体的なアプリケーションの扱い方を学びます。
3日目には5分程度のプレゼンテーションをしてもらいます。
1つ以上のアプリケーションをテーマに選び、実際に動作する環境と合わせてアプリケーションの目的や仕組みなどを紹介して下さい。
### [ロギング](markdown/logging_rlogin.md)
### [サービス制御](markdown/presettings.md)
### [TCP/IPパケット概要](markdown/wireshark.md)
### [Mailの観測](markdown/mail.md)
### [HTTPの観測](markdown/http.md)
### [HTTPSの観測](markdown/https.md)
### [TELNET/SSHの観測](markdown/remote_login.md)
### [ドメインの概要](markdown/dns_zone.md)
### [DNSの観測](markdown/dns.md)
### [Proxyの観測](markdown/proxy.md)
### [DNSレコードの作成](markdown/change_dns.md)
### [SNMPの観測](markdown/snmp.md)

## フェーズ2
3日間の研修でデータリンク層、ネットワーク層、トランスポート層の詳細を学びます。
3日目には受講者同士の仮想マシンを連携させたシンプルなネットワークを構築し、設計と実装のポイントを5分程度で簡潔に説明して頂きます。
ルーティングを駆使した複雑なネットワークはPhase3で取り扱うため今は考えなくても結構です。
### [サブネット](markdown/ipsubnet.md)
### [ブリッジを使う](markdown/bridge.md)
### [アドレスの一意性](markdown/unique_addressing.md)
### [DHCPの観測](markdown/dhcp.md)
### [VLANを使う](markdown/vlan.md)
### [スパニングツリー](markdown/stp.md)
### [ファイアウォール](markdown/firewall.md)

## フェーズ3
4日間（水、木、金、火）の研修でルーティングの詳細を学びます。
今回はルーティングの演習発表は行いません。
その代わり次のフェーズでこれまでの研修全体を踏まえたネットワークを構築し設計と実装のポイントを説明して頂きます。
## [構築の流れ](markdown/architect_tips.md)
## [スタティックルーティング](markdown/routing.md)
## [ファイアウォールマニアクス](markdown/firewall.md)
## [DMZ](markdown/dmz.md)
## [ファイアウォールマニアクス](firewall.md)
## [OSPF](markdown/ospf.md)
## [BGP](markdown/bgp.md)
## [大規模ネットワークサンプル1](markdown/practice1.md)
## [大規模ネットワークサンプル2](markdown/practice2.md)

---
layout: post
title:  "TCP/IPの概要"
date:   2018-08-21
permalink: /networking/:title
categories: TCP/IP
tags: Wireshark
excerpt: TCP/IPに関するポストです
mathjax: false
---
 
* content
{:toc}

## TCP/IPパケットの読み方

### 前置き

TCP/IPネットワークを流れるデータは`メッセージ` `パケット` `セグメント` `データグラム` `フレーム` 等と呼ばれています。
Webサイトを閲覧する場合を例にとって一連の処理を考えます。

1. アプリケーション層

Webブラウザがアプリケーション特有の情報をHTTPメッセージとして送信します。
ユーザーエージェントやエンコードなどアプリケーションの環境設定がHTTPヘッダに記述されます。

2. トランスポート層

HTTPメッセージをTCPセグメントとしてラッピングします。
TCPセグメントのヘッダには宛先ポート等の情報が記述されます。

3. ネットワーク層

TCPセグメントをIPパケットとしてラッピングします。
IPパケットのヘッダには宛先IPアドレス等の情報が記述されます。

4. データリンク層

IPパケットをデータリンク層のフレームとしてラッピングします。
フレームのヘッダには宛先MACアドレス等の情報が記述されます。

5. 受信側の処理

データリンク層からアプリケーション層まで逆操作することでHTTPメッセージを取り出します。

![]({{site.baseurl}}/images/wireshark/packet_structure.png)

### Wiresharkの場合

TCP/IPデータ構造を把握していればトラブルシューティングの時にWiresharkで見るべき場所が分かります。

例えば宛先ホストにpingが通じるならばトランスポート層以上に限定して見ていくのが良いでしょう。
pingが通じない場合はネットワーク層のIPアドレスかデータリンク層のMACアドレスが間違っているかもしれません。

![]({{site.baseurl}}/images/wireshark/view01.png)

アプリケーション層の情報は量が多いのでツリーを開きます。
解析する際は各アプリケーションプロトコルのRFC等を参考にするのが良いでしょう。

![]({{site.baseurl}}/images/wireshark/view02.png)

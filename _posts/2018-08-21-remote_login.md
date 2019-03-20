---
layout: post
title:  "リモートログインの概要"
date:   2018-08-21
permalink: /networking/:title
categories: TCP/IP
tags: ssh telnet RLogin
excerpt: リモートログインに関するポストです
mathjax: false
---
 
* content
{:toc}

## TELNET/SSHの観測
- VirtualBoxイメージのインポート

![]({{site.baseurl}}/images/vm_inport1.png)

![]({{site.baseurl}}/images/vm_inport2.png)

- VirtualBoxネットワーク設定

![]({{site.baseurl}}/images/vm_prop1.png)

![]({{site.baseurl}}/images/vm_prop2.png)

![]({{site.baseurl}}/images/vm_prop3.png)

![]({{site.baseurl}}/images/vm_prop4.png)

- TELNETログイン（アプリケーション）

![]({{site.baseurl}}/images/rlogin1.png)

![]({{site.baseurl}}/images/rlogin2.png)

![]({{site.baseurl}}/images/rlogin3.png)

![]({{site.baseurl}}/images/rlogin4.png)

- SSHログイン（アプリケーション）

![]({{site.baseurl}}/images/rlogin5.png)

![]({{site.baseurl}}/images/rlogin6.png)

## SSHの操作ログを保存する

複雑な変更を加える場合は途中のコマンドを一部間違えるだけで結果が大きく変わります。
自分がどこで間違えたのか把握するには`操作ログ`を取得することが重要です。

RLoginにも自分が操作したログを保存する機能が備わっています。
使い方を紹介します。

1. `ログファイルに保存`を有効化する

![]({{site.baseurl}}/images/rlogin/save_to_log.png)

2. ログファイルを保存する場所とファイル名を指定する

![]({{site.baseurl}}/images/rlogin/select_log_name.png)

3. SSHで操作する

![]({{site.baseurl}}/images/rlogin/run_command.png)

4. 必要な操作をすべて終了したら`ログファイルに保存`を無効化する

![]({{site.baseurl}}/images/rlogin/checkout.png)

5. Windows上に保存されたログファイルを確認する

![]({{site.baseurl}}/images/rlogin/saved_text.png)

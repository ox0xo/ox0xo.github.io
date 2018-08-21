---
layout: post
title:  "ロギング"
date:   2018-08-21
permalink: /networking/:title
categories: misc
tags: rlogin ssh
excerpt: RLoginでSSH接続作業のログを残す方法に関するポストです
mathjax: false
---

* content
{:toc}

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

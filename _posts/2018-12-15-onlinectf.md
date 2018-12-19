---
layout: post
title:  "オンラインCTFを開催する"
date:   2018-12-15
permalink: /ctf/:title
categories: CTF
tags: CTFd
excerpt: CTFdを使ってオンラインCTFを開催する際の一連の手順をまとめたものです
---

* content
{:toc}

このポストはインターネット上でCTFを開催する手順をまとめたものです。

# インフラの準備

CTFの規模や出題傾向に応じてインフラのスペックを考慮します。
今回はさくらのVPSで4GB/SSDのインスタンスを1つ建てます。
ベースOSはUbuntu16.04です。

## ssh

万が一にも第三者に操作されないようにSSH接続は公開鍵認証のみ許可しておきます。

```
sudo apt update && sudo apt upgrade

ssh-keygen -t rsa
mv ~/.ssh/id_rsa.pub ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# SSHクライアント側に必要な秘密鍵（id_rsa）を手元にダウンロードしておくこと
#$ ls -la .ssh
#total 20
#drwx------ 2 ubuntu ubuntu 4096 Dec 15 14:17 .
#drwxr-xr-x 4 ubuntu ubuntu 4096 Dec 15 14:17 ..
#-rw------- 1 ubuntu ubuntu  402 Dec 15 14:14 authorized_keys
#-rw------- 1 ubuntu ubuntu 1766 Dec 15 14:14 id_rsa

sudo sed -i -e 's/PasswordAuthentication yes/PasswordAuthentication no/g' /etc/ssh/sshd_config
sudo systemctl restart sshd
```
もう一度VPSにSSH接続を試みてパスワードログインが弾かれたらOKです。
これ以降は手元にダウンロードした秘密鍵を使ってVPSにSSH接続します。
公開サーバ側の秘密鍵（id_rsa）は削除しておきましょう。

> 本当はクライアント側で秘密鍵/公開鍵を作成して公開サーバに公開鍵をアップロードする手順が適切です。

## Docker

CTFdのデプロイにはDockerを使うのが簡単です。
[docker-ce]と[docker-compose]のマニュアルに従って作業します。

```
sudo apt-get install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository \
"deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get update
sudo apt-get install docker-ce
sudo curl -L \
https://github.com/docker/compose/releases/download/1.23.2/docker-compose-`uname -s`-`uname -m` \
-o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
sudo systemctl start docker
```

# CTFdのデプロイ

[ctfd]のマニュアルに従ってCTFdのDockerコンテナをビルドします。
ビルドが完了するまでには相応の時間がかかるので気長に待ちます。

```
sudo apt install vim git
git clone https://github.com/CTFd/CTFd.git
cd CTFd/
python -c "import os; f=open('.ctfd_secret_key', 'a+'); f.write(os.urandom(64)); f.close()"
vim docker-compose.yml
docker-compose up &
```

- 記述例：docker-compose.ymlにSECRET_KEYを追記する
  ```
  version: '2'

  services:
    ctfd:
      build: .
      restart: always
      ports:
        - "8000:8000"
      environment:
        - SECRET_KEY=.ctfd_secret_key
  ```

- 出力例：Dockerコンテナビルド完了
  ```
  ctfd_1   | Starting CTFd
  ```

# ドメインの準備

今回は無料ドメインを取得できる[Freenom]を利用します。
空いているドメインを確保したらWebサーバのIPアドレスをAレコードに登録します。

![](/images/ctfd/freenom.png)

Webサーバが名前解決できるようになったことを確認しておきます。

```
nslookup www.cctf2018.cf
```

# HTTPSの有効化

CTFdのサービスをHTTPSで提供するためにリバースプロキシを立てます。
サーバ証明書は[Let's Encrypt]で取得します。

## Let's Encrypt

Let's Encryptで証明書を取得するためには名前解決できるWebサーバが必要です。
ドメインは既に取得済みなのでファイアウォールとサーバアプリケーションを設定します。
今回は筆者の好みでnginxを使います。

```
sudo iptables -I INPUT 5 -p tcp -m tcp --dport 80 -j ACCEPT
sudo iptables -I INPUT 5 -p tcp -m tcp --dport 443 -j ACCEPT

sudo systemctl stop apache2 && sudo systemctl disable apache2
sudo apt install -y nginx
sudo vim /etc/nginx/conf.d/default.conf
sudo systemctl start nginx
```

- 記述例：default.conf
  ```
  server {
    listen 80;
    server_name www.cctf2018.cf;

    location / {
      root /usr/share/nginx/html;
    }
  }
  ```

Webサーバが準備出来たらLet's Encryptで証明書を取得します。

```
sudo apt-get install letsencrypt
sudo letsencrypt certonly --webroot --webroot-path /usr/share/nginx/html -d www.cctf2018.cf
```

- 出力例：秘密鍵(privkey1.pem)とサーバ証明書(cert1.pem)が生成された様子
  ```
  #
  #$sudo ls -la /etc/letsencrypt/archive/www.cctf2018.cf
  #total 24
  #drwxr-xr-x 2 root root 4096 Dec 15 15:42 .
  #drwx------ 3 root root 4096 Dec 15 15:42 ..
  #-rw-r--r-- 1 root root 1899 Dec 15 15:42 cert1.pem
  #-rw-r--r-- 1 root root 1647 Dec 15 15:42 chain1.pem
  #-rw-r--r-- 1 root root 3546 Dec 15 15:42 fullchain1.pem
  #-rw-r--r-- 1 root root 1704 Dec 15 15:42 privkey1.pem
  ```

## nginx

HTTPSのサービスを提供できるようにnginxでリバースプロキシを構築します。
nginxが受け取ったリクエストはローカルポート8000で動作しているCTFdに転送します。

```
sudo vim /etc/nginx/conf.d/ssl.conf
```

- 記述例：ssl.conf
  ```
  server {
    listen  443;

    ssl                   on;
    ssl_certificate       /etc/letsencrypt/archive/www.cctf2018.cf/cert1.pem;
    ssl_certificate_key   /etc/letsencrypt/archive/www.cctf2018.cf/privkey1.pem;

    ssl_session_timeout   5m;

    ssl_protocols              TLSv1.2;
    ssl_ciphers                HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers  on;

    location / {
      proxy_set_header Host             $host;
      proxy_set_header X-Real-IP        $remote_addr;
      proxy_set_header X-Forwarded-For  $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-User $remote_user;
      proxy_pass http://localhost:8000;
    }
  }
  ```

CTFd自身はHTTPで動作しているためリソースのリクエストにHTTPが利用されます。
これをHTTPSに書き換えるための設定も必要です。

```
sudo vim /etc/nginx/conf.d/default.conf
```

- 記述例：default.conf

  ```
  server {
    listen 80;
    server_name www.cctf2018.cf;
    rewrite ^(.*) https://www.cctf2018.cf$1 permanent;
  }
  ```

設定が完了したらnginxを再起動します。
WebブラウザからCTFdにアクセスしてSetupが表示されればOKです。
```
sudo systemctl restart nginx
```
![](/images/ctfd/ctfd.png)

[docker-compose]:https://github.com/docker/compose/releases
[docker-ce]:https://docs.docker.com/install/linux/docker-ce/ubuntu/
[ctfd]:https://github.com/CTFd/CTFd
[Freenom]:https://my.freenom.com
[Let's Encrypt]:https://letsencrypt.jp/usage/

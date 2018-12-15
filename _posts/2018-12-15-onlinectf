---
layout: post
title:  "オンラインCTFを開催する"
date:   2018-12-15
permalink: /ctf/:title
categories: CTF
tags: CTFd "Let's Encrypt" 
excerpt: CTFdを使ってオンラインCTFを開催する際の一連の手順をまとめたものです
---

* content
{:toc}

このポストはインターネット上でCTFを開催する手順をまとめたものです。

# インフラの準備

CTFの規模や出題傾向に応じてインフラのスペックを考慮します。
今回はさくらのVPSで4GB/SSDのインスタンスを1つ建てます。
ベースOSはUbuntu16.04です。

```
sudo apt update && sudo apt upgrade
sudo apt install vim git

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

sudo vim /etc/ssh/sshd_config
sudo sed -i -e 's/PasswordAuthentication yes/PasswordAuthentication no/g' /etc/ssh/sshd_config
sudo systemctl restart sshd
exit

# もう一度VPSにSSH接続を試みてパスワードログインが弾かれたらOK
# これ以降は手元にダウンロードした秘密鍵を使ってVPSにSSH接続する

sudo apt-get install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get update
sudo apt-get install docker-ce
sudo curl -L https://github.com/docker/compose/releases/download/1.23.2/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
sudo systemctl start docker

# HelloWorldで動作確認しておく
#$sudo docker run hello-world
#Hello from Docker!
```

# CTFdの準備

```
git clone https://github.com/CTFd/CTFd.git
cd CTFd/
python -c "import os; f=open('.ctfd_secret_key', 'a+'); f.write(os.urandom(64)); f.close()"
vim docker-compose.yml

# docker-compose.ymlにSECRET_KEYを追記する
# その他パラメータも必要に応じて書き換えておく
#
# 記述例
#----------------------------------------
#version: '2'
#
#services:
#  ctfd:
#    build: .
#    restart: always
#    ports:
#      - "8000:8000"
#    environment:
#      - SECRET_KEY=.ctfd_secret_key

docker-compose up &

# CTFd関連モジュールのビルドに相応の時間がかかるので気長に待つ
# 以下のメッセージが出力されたらブラウザからアクセスして初期設定する
#ctfd_1   | Starting CTFd
```

# ドメインの準備

今回は無料ドメインを取得できるFreenomを利用します。
名前解決できるようになったことを確認しておきます。

```
nslookup www.cctf2018.cf
```

# 証明書の準備

今回は無料証明書を取得できるLet's Encryptを利用します。

```
sudo iptables -I INPUT 5 -p tcp -m tcp --dport 80 -j ACCEPT
sudo iptables -I INPUT 5 -p tcp -m tcp --dport 443 -j ACCEPT

# HTTP/HTTPs用のエントリを追加しておく
#$sudo iptables -L
#Chain INPUT (policy ACCEPT)
#target     prot opt source               destination         
#f2b-sshd   tcp  --  anywhere             anywhere             multiport dports ssh
#ACCEPT     all  --  anywhere             anywhere             state RELATED,ESTABLISHED
#ACCEPT     icmp --  anywhere             anywhere            
#ACCEPT     all  --  anywhere             anywhere            
#ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:https
#ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:http

sudo systemctl stop apache2 && sudo systemctl disable apache2
sudo vim /etc/nginx/conf.d/default.conf

# 証明書の発行にWebサーバが必要なのでconfigを用意する
#
# 記述例
#----------------------------------------
#server {
#  listen 80;
#  server_name dev.cctf.cf;
#
#  location / {
#    root /usr/share/nginx/html;
#  }
#}

sudo systemctl start nginx
sudo apt-get install letsencrypt
sudo letsencrypt certonly --webroot --webroot-path /usr/share/nginx/html -d www.cctf2018.cf

# 秘密鍵(privkey1.pem)とサーバ証明書(cert1.pem)が生成されたことを確認する
#$sudo ls -la /etc/letsencrypt/archive/www.cctf2018.cf
#total 24
#drwxr-xr-x 2 root root 4096 Dec 15 15:42 .
#drwx------ 3 root root 4096 Dec 15 15:42 ..
#-rw-r--r-- 1 root root 1899 Dec 15 15:42 cert1.pem
#-rw-r--r-- 1 root root 1647 Dec 15 15:42 chain1.pem
#-rw-r--r-- 1 root root 3546 Dec 15 15:42 fullchain1.pem
#-rw-r--r-- 1 root root 1704 Dec 15 15:42 privkey1.pem

sudo vim /etc/nginx/conf.d/ssl.conf

# HTTPs用のconfigを用意する
#
# 記述例
#----------------------------------------
#server {
#  listen  443;
#
#  ssl                   on;
#  ssl_certificate       /etc/letsencrypt/archive/dev.cctf.cf/cert1.pem;
#  ssl_certificate_key   /etc/letsencrypt/archive/dev.cctf.cf/privkey1.pem;
#  
#  ssl_session_timeout   5m;
#  
#  ssl_protocols              TLSv1.2;
#  ssl_ciphers                HIGH:!aNULL:!MD5;
#  ssl_prefer_server_ciphers  on;
#
#  location / {
#    proxy_set_header Host             $host;
#    proxy_set_header X-Real-IP        $remote_addr;
#    proxy_set_header X-Forwarded-For  $proxy_add_x_forwarded_for;
#    proxy_set_header X-Forwarded-User $remote_user;
#    proxy_pass http://localhost:8000;
#  }
#}

sudo vim /etc/nginx/conf.d/default.conf

# ポート80への接続を内部でhttpsに書き換えるconfigを用意する
#
# 記述例
#----------------------------------------
#server {
#  listen 80;
#  server_name www.cctf2018.cf;
#  rewrite ^(.*) https://www.cctf2018.cf$1 permanent;
#}

sudo systemctl restart nginx
```

[docker-compose]:https://github.com/docker/compose/releases
[docker-ce]:https://docs.docker.com/install/linux/docker-ce/ubuntu/
[ctfd]:https://github.com/CTFd/CTFd
[Freenom]:https://my.freenom.com
[Let's Encrypt]:https://letsencrypt.jp/usage/

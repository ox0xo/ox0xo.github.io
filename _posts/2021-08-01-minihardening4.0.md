---
layout: post
title:  "Mini Hardening 4.0"
date:   2021-08-02
permalink: /security/:title
categories: Security
tags: ModSecurity Suricata Tripwire WAF IDS IPS
excerpt: 2021-07-31に開催されたMini Hardening 4.0のために準備した情報です.
mathjax: false
---

* content
{:toc}

2021-07-31に開催された[Mini Hardening 4.0](https://minihardening.connpass.com/)に参加しました.
色々な学びを得たので記憶を風化させないために記録に残します.
また今後Mini Hardeningに参加される方の参考になれば幸いです.

競技内容のネタばれは有りません.

# タイムライン

## 7月18日

まずはチームビルディングという事で各位プロフィールと今回やってみたい事を話し合いました.
話し合いと並行しながら光の速さで情報共有用のDiscordやGoogleスプレッドシートが用意されて行きます.
自分に出来る事を探して主張していかないと何もせず終わりそうだぞ.
明日までにGoogleスプレッドシートに想定される攻撃を列挙しておこうという事でこの日は終わりました.

## 7月19日

寝て起きたらGoogleスプレッドシートに情報が山盛り追記されていました.
つよい人たちと一緒になったなぁと思いながら私が追記したものは以下の通りです.

- WAFで防げる攻撃
XSSやSQLiやOGNLiあたりが使われそうですがいずれにしてもHTTPを経由して侵入してくるはず.
しかし競技時間中にそれらすべてにパッチを当てるのは大変そうです.
WAFを入れたらいいんじゃないかという発想に至ります.

- WAFで防げない攻撃
SSHやFTPを使った攻撃があるかもしれません.
これはWAFで検知できないのでIDS/IPSを導入するのが良さそうです.
IPSならDoSも防げるかもしれないという淡い期待を寄せます.

- どうやっても防げない攻撃
攻撃側がこちらを上回ってくるはず. 
やられた事に気付ける体制も必要です.
設定ファイルの改ざんやマルウェアの設置に気付くにはファイルシステムの変更を監視すれば良さそうです.
ファイルシステムの変更を伴わないメモリの改ざんは対処方法を思いつかなかったので見ないふりをします.
ユーザー目線で挙動がおかしくなった時に調査するしかないのでは？

このアイデアをシステム概要図に落とし込んだものは以下の通りです.
大部分の攻撃を自動的に防御してくれる仕組みと防御出来なかった攻撃のログを一元管理できる仕組みを作ろうというコンセプトです.

![](/images/minihardening40/2021-08-01-23-14-35.png)

![](/images/minihardening40/2021-08-01-23-15-57.png)


これ以外にもサーバの設定ミス等で見えてはいけないものが正規のアクセスで見える状況も考えられました.
どういう状況が正しいのか判断できるように検証環境を手元に用意しておいた方が良さそうです.

## 7月20日

Discordで初の顔合わせミーティングです.
想定される攻撃の一覧をもとにどんな対策を入れるか話し合いました.
私は主にWAF, IDS/IPS, 改ざん検知の導入と運用を担当することになりました.
ここから3日間でざっくり検証して情報をまとめています.
対象の環境が分からずWindows, CentOS, Ubuntuそれぞれ3パターンぐらい検証進めたのでこのフェーズが一番大変でした.

![](/images/minihardening40/2021-08-01-21-51-19.png)

## 7月24日

第二回のミーティング.
各位が担当している作業の進捗を共有します.

## 7月25日

運営から競技環境の詳細情報が提示されました.
やらなくても良くなった対策もあれば新たに検討しなければならない対策もあり.
OSが明らかになったので今後は事前検証の中から該当する手順を選んでブラッシュアップしていくことになりました.

## 7月29日

21:30から本番前の最後の打ち合わせ.
だったんですが仮眠のつもりが朝までぐっすり眠ってしまい不参加でした.
本番開始の1時間前に臨時で打ち合わせを設けてもらいどの順番で対策を打っていくかを決めました.

# 事前検証

## WAF

OSSのWAFといえばModSecurityです.
Apache HTTP ServerとNginxのどっちが来ても対応できるように検証しましたが最近のLinuxなら基本的な流れは殆ど変わらないようでした.
一方Windows版のModSecurityは過去存在していたようですが現在のリポジトリを見ると `Windows build is not ready yet.` です.
Windowsが来たら運用でカバーするしかないと諦めました.

Apacheに比べてNginxは動的モジュールを用意しないといけないので多少ハードルが高いかも.
私が動作確認したスクリプトを共有します.

[Ubuntu 18.04 + Nginx 1.14.0 で動作確認済みのスクリプト](https://gist.github.com/ox0xo/c581d63e327107239d8e9c5c9afe374e)


インストールが完了したら/etc/nginx/sites-available/default等にモジュールを使用する旨を追記してNginxを再起動すればModSecurityが有効になります.
```
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    # 以下2行を追加する
    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsec/main.conf;

    root /var/www/html;
```

リクエストクエリに?id=123を付けても200番が返されるところ?id=sleep(1)を付けたら403が返されたら成功です.

```
root@ubuntu18:~# curl -I http://localhost/?id%3D123
HTTP/1.1 200 OK
Server: nginx/1.14.0 (Ubuntu)
Date: Sun, 01 Aug 2021 12:40:40 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Thu, 29 Jul 2021 23:49:17 GMT
Connection: keep-alive
ETag: "61033e7d-264"
Accept-Ranges: bytes

root@ubuntu18:~# curl -I http://localhost/?id%3Dsleep%281%29
HTTP/1.1 403 Forbidden
Server: nginx/1.14.0 (Ubuntu)
Date: Sun, 01 Aug 2021 12:39:14 GMT
Content-Type: text/html
Content-Length: 178
Connection: keep-alive
```

あとはチューニングが必要です.
/var/log/nginx/error.log を見るとこんなログが大量に出てくるので実際にユーザー側の挙動を確認しながら過検知らしいログを探します.
過検知を引き起こしていると判断された場合は該当のconfを別の場所に移してNginxを再起動すればOK!

```
file "/usr/local/owasp-modsecurity-crs/rules/REQUEST-949-BLOCKING-EVALUATION.conf"] [line "80"] [id "949110"] [rev ""] [msg "Inbound Anomaly Score Exceeded (Total Score: 8)
```

## IPS/IDS

ネットワーク型IDS/IPSで有名なものはSnortやSuricataです.
軽く触ってみて簡単に使えそうだったSuricataで検証を進めることにしました.
Snortはちょっと幅広く検知しようと思ったらユーザー登録が必要になりそうで面倒くさかった.
時間がある時にもう一度試してみたいですね.

さてSuricataはaptで入れても良いんですが最新版は落ちてこないのでソースから最新版をコンパイルします.

[Ubuntu 18.04 で動作確認済みのスクリプト](https://gist.github.com/ox0xo/40e28d342796473887c2ddf19a421644)

コンパイルとインストールが終わったら自分が所属するネットワークをSuricataに教えてから動作確認します.
自分のネットワークを教えるには /etc/suricata/suricata.yaml のHOME_NETを書き換えるだけでOK.
```
vars:
# more specific is better for alert accuracy and performance
  address-groups:
    HOME_NET: "[192.168.0.0/16,10.0.0.0/8,172.16.0.0/12]"
```

動作確認には一工夫が必要です.
そのまま起動するとSuricataはIDSモードで動作します.
IDSモードでは攻撃パターンに一致する通信を検知してもアラートを上げるだけで通信は遮断されません.
IPSモードで通信を遮断するためにはNFQUEUEを経由してパケットを受け取る必要があります.

![](/images/minihardening40/2021-08-01-22-21-47.png)

通常LinuxはiptablesのINPUTチェインで処理されたパケットを受信します.
SuricataがIDSモードで動作する時は受信後のパケットを見るので通信を遮断できません.

INPUTチェインの中でNFQUEUEというバッファを経由するように指定することでこの問題を解決できます.
SuricataはNFQUEUEからパケットを取り出して問題が有ればLinuxに渡さずに破棄することが出来ます.

ただしNFQUEUEを使うときに注意することが１つあります.
iptablesはNFQUEUEにパケットを格納した後の面倒を見てくれません.
NFQUEUEからパケットを取り出してLinuxに渡す責任はSuricataに有るという事です.
Suricataが起動していない状態でiptablesだけ変更すると全てのパケットを受信できなくなります.

以上の事を踏まえてSuricataをIPSモードで動かす手順は以下の通りです.
iptablesにNFQUEUEを追加するときにサービスに必須の通信をACCEPTすることを忘れないように.
こうしておけばうっかりSuricataを停止してもSSHが遮断されることはありません.

```
# NFQUEUE 0 を使ってSuricataをバックグラウンドで起動する
suricata -c /etc/suricata/suricata.yaml -q 0 -D

# INPUTチェインにNFQUEUEを追加する
iptables -I INPUT 1 -p tcp --dport 22 -j ACCEPT
iptables -I INPUT 2 -j NFQUEUE
```

最後はチューニングです.
Suricataの検知ログは /var/log/suricata/fast.log に記録されているのでこれを良く見て遮断したい通信を特定します.


```
07/30/2021-09:55:56.606977  [**] [1:2010937:3] ET SCAN Suspicious inbound to mySQL port 3306 [**] [Classification: Potentially Bad Traffic] [Priority: 2] {TCP} 172.16.0.20:53999 -> 172.16.0.12:3306
07/30/2021-09:55:56.635475  [**] [1:2002911:6] ET SCAN Potential VNC Scan 5900-5920 [**] [Classification: Attempted Information Leak] [Priority: 2] {TCP} 172.16.0.20:53999 -> 172.16.0.12:5904
07/30/2021-09:55:56.641849  [**] [1:2010939:3] ET SCAN Suspicious inbound to PostgreSQL port 5432 [**] [Classification: Potentially Bad Traffic] [Priority: 2] {TCP} 172.16.0.20:53999 -> 172.16.0.12:5432
```

たとえば ET SCAN Potential VNC Scan 5900-5920 を遮断する場合.
今回の手順ではSuricataのルールは /va/lib/suricata/rules/suricata.rules に集約されています.
中身はこんな感じ.
先頭の alert を drop に書き換えればマッチした通信を遮断してくれます.

```
alert tcp $EXTERNAL_NET any -> $HOME_NET 5800:5820 (msg:"ET SCAN Potential VNC Scan 5800-5820"
```

通信が遮断されるようになったことは fast.log を確認すれば分かります.
Drop になっていますね.
```
07/30/2021-10:08:20.790069  [**] [1:2010935:3] ET SCAN Suspicious inbound to MSSQL port 1433 [**] [Classification: Potentially Bad Traffic] [Priority: 2] {TCP} 172.16.0.20:35580 -> 172.16.0.12:1433
07/30/2021-10:08:20.830601  [Drop] [**] [1:2002910:6] ET SCAN Potential VNC Scan 5800-5820 [**] [Classification: Attempted Information Leak] [Priority: 2] {TCP} 172.16.0.20:35580 -> 172.16.0.12:5811
07/30/2021-10:08:20.832920  [**] [1:2010936:3] ET SCAN Suspicious inbound to Oracle SQL port 1521 [**] [Classification: Potentially Bad Traffic] [Priority: 2] {TCP} 172.16.0.20:35580 -> 172.16.0.12:1521
```

## 改ざん検知

WAFやIPSで防げなかった攻撃はファイルの改ざん検知で対応します.
例えばアプリケーションの脆弱性を利用してシステムに侵入した攻撃者はそのシステムを永続的に侵害するためにユーザーアカウントを作ることがあります.
Linuxでユーザーアカウントを作ると/etc/passwdに新しいエントリが追加されます.
ここで/etc/passwdの変更を検知出来れば攻撃者の挙動に気づくことが出来ます.

ファイルの改ざんを検知できるアプリケーションは多数存在します.
今回は前から気になっていたTripwireを使う事にしました.
さらにTripwireで検知したアラートはGoogle App ScriptにPOSTしてGoogleスプレッドシートで一元管理する仕組みを作ります.

### Tripwireの準備

#### インストール
初期インストールは `apt install -y tripwire` だけで済みます.
メールでアラートを送信するつもりはないのでPostfixの設定変更は無し.
Site KeyとLocal Keyのパスフレーズは `TeamF!23` としておきます.

インストールが終わったら設定ファイルとポリシーファイルを用意します.

1. 設定ファイル作成
```
cd /etc/tripwire
sed 's/REPORTLEVEL   =3/REPORTLEVEL   =4/g' -i twcfg.txt
twadmin -m F -c tw.cfg -S site.key twcfg.txt
```

2. [ポリシーファイル変換スクリプト twpolmake.pl の準備](https://gist.github.com/ox0xo/d1a609d424e1a470ba6a1338beb883a4)

3. ポリシーファイル作成
```
perl twpolmake.pl twpol.txt > twpol.txt.new
twadmin -m P -c tw.cfg -p tw.pol -S site.key twpol.txt.new
tripwire -m i -s -c tw.cfg
```

設定ファイルとポリシーファイルが出来たら動作確認します.
検査対象ファイル数と改ざん検知されたファイル数が見えたらOK.
```
root@ubuntu20:~# tripwire -m c -s -c /etc/tripwire/tw.cfg | grep Total
Total objects scanned:  57709
Total violations found:  1
```

#### チューニング

Tripwireは初期状態で重要なファイルの改ざんを検知しますが状況によって検知対象を除外したり新たに追加したりする事も必要になります.
例えば/var/www/htmlを新たに監視対象に追加する手順は以下の通りです.

```
# ポリシーファイルをテキストに戻す
twadmin -m p -c /etc/tripwire/tw.cfg -p /etc/tripwire/tw.pol -S /etc/tripwire/site.key > /etc/tripwire/twpol.txt

# テキストのポリシーファイルに/var/www/htmlの監視を追記する
vim /etc/tripwire/twpol.txt
---
(
  rulename = "Invariant Directories",
  severity = $(SIG_MED)
)
{
        /               -> $(SEC_INVARIANT) (recurse = 0) ;
        /home           -> $(SEC_INVARIANT) (recurse = 0) ;
        /tmp            -> $(SEC_INVARIANT) (recurse = 0) ;
        /usr            -> $(SEC_INVARIANT) (recurse = 0) ;
        /var            -> $(SEC_INVARIANT) (recurse = 0) ;
        /var/tmp        -> $(SEC_INVARIANT) (recurse = 0) ;
        /var/www/html   -> $(SEC_INVARIANT) (recurse = -1) ;
}

# ポリシーファイルを作成する
twadmin -m P -c /etc/tripwire/tw.cfg -p /etc/tripwire/tw.pol -S /etc/tripwire/site.key /etc/tripwire/twpol.txt

# 新しいポリシーファイルを元にデータベースを更新する
tripwire -m i -s -c /etc/tripwire/tw.cfg
```

### Google App Scriptの準備

Slackを障害アラートの監視ポータルとして使うシーンは良くあります.
今回もそれを真似ようと思いましたがイベントの性質上攻撃が山ほど飛んできてSlackやDiscordでは見切れないだろう事が予想されました.
そこでとりあえずGoogleスプレッドシートにアラートを追記して後から追いかけられる仕組みを作ります.

1. Ｇoogleスプレッドシートを作ってスクリプトエディタを開く
![](/images/minihardening40/2021-08-02-00-02-46.png)

2. POSTリクエストを受け取るためのdoPostメソッドを定義する
```
function doPost(e) { 
  var sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName("シート1");
  var params = JSON.parse(e.postData.getDataAsString());
  sheet.appendRow([new Date(), params.host, params.log, params.message]);
}
```

3. スクリプトを保存したらウェブアプリとして公開する
![](/images/minihardening40/2021-08-02-00-15-57.png)

ここで表示されたURLに対して次のようにJSONをPOSTすれば動作します.

`curl -XPOST 'ウェブアプリのURL' -H 'Content-Type: applicaton/json' -d '{"host": "myhost", "log":"mylog", "message":"mymessage"}'`

### 自動化スクリプトの準備

定期的にTripwireを実行してアラートが見つかればGoogleにPOSTするスクリプトを書きます.
以下のスクリプトは動作確認済みですがPOST先を書き換える必要があるので注意.
Googleスプレッドシートにログが記録されたら成功です.

[send-tripwire-to-goole.sh](https://gist.github.com/ox0xo/f5ac0ff350b9afc7daff7265a8c2be24)

![](/images/minihardening40/2021-08-02-01-15-53.png)

スクリプトが動くことを確認したらcronを設定します.

```
echo "*/5 * * * * root /root/send-tripwire-to-google.sh" >> /etc/crontab
```

環境によってTripwireが走り終わるまでの時間が異なるので上手く調整してください.
そもそもTripwireは数分間隔で走らせるものではないので他にもっと良い方法があったかもしれません.
とりあえず使ってみたかったのです.



# 当日の反省点

CPUスペックやDiskの空きが足りなくて対策の導入に苦戦しました.
特にSuricataはシグネチャが1.9GBほどあるので空き容量を見てから展開した方が良いです.
競技中にDisk Fullの警告を何度も見る羽目になり結局ModSecurityは導入できませんでした.
SuricataとTripwireは基本が検知ツールなので状況を見ながらチューニングするつもりでした.
蓋を開けてみればインシデント対応とModSecurityの導入に忙殺されてしまいチューニングする暇などありま
んでした.

以上を踏まえると大がかりな対策より基本的な所を押さえた方が良かったような気がします.
何が正常な状態なのか始めに確認したうえでユーザー目線でサービスの異常を監視した方が良かったですね.

それでも強い人たちと一緒に作業出来て得るものは多かったので次があればリベンジしたいと思います.

# 参考情報

- [TechExpertTips: Nginx-ModSecurityのインストール](https://techexpert.tips/ja/nginx-ja/nginx-%E3%83%A2%E3%83%83%E3%83%89%E3%82%BB%E3%82%AD%E3%83%A5%E3%83%AA%E3%83%86%E3%82%A3%E3%81%AE%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB/)
- [ytyaru: libxml2-devel, libxslt-develというパッケージがみつからない](https://ytyaru.hatenablog.com/entry/2017/05/15/000000)
- [技術メモの壁: Ubuntu 14.04 + nginx で HTTP/2 サーバ](https://fsck.jp/?p=329)
- [SpiderLabs: Add ModSecurity to existing Nginx](https://github.com/SpiderLabs/ModSecurity-nginx/issues/117#issuecomment-495350465)
- [Tomorrow is always fresh with no mistake in it: ライブラリのパスを設定する](https://w.atwiki.jp/futoyama/pages/60.html)
- [To Linux and beyond!: Using NFQUEUE and libnetfilter_queue](https://home.regit.org/netfilter-en/using-nfqueue-and-libnetfilter_queue/)
- [Suricata User Guide](https://suricata.readthedocs.io/en/suricata-6.0.0/)
- [Atlantic.Net Blog: How to Install And Setup Suricata IDS on Ubuntu 20.04](https://www.atlantic.net/vps-hosting/how-to-install-and-setup-suricata-ids-on-ubuntu-20-04/)
- [CentOSで自宅サーバ構築: ファイル改ざん検知システム導入](https://centossrv.com/tripwire.shtml)
- [Server World: Tripwire](https://www.server-world.info/query?os=Ubuntu_18.04&p=tripwire)
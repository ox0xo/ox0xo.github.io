---
layout: post
title:  "TMCIT and 大和セキュリティ MAIR忍者チャレンジ writeup"
date:   2019-11-04
permalink: /ctf/:title
categories: Security
tags: malware
excerpt: 2019-11-03、2019-11-04の2日間に渡って開催されたMAIR忍者チャレンジのWriteupです
mathjax: false
---

* content
{:toc}

2019-11-03、2019-11-04 の2日間に渡って開催されたMAIR忍者チャレンジに参加しました.
MAIR忍者チャレンジは、マルウェアアナリストの立場からお客様のインシデントレスポンスを支援するスキルを磨くことを目的としたコンテストです.
アーティファクトを解析しつつ、与えられたシナリオに対応した提案に結びつけなければならず、さらにチームとして連携する必要がある難しい課題でしたが大変やりがいがありました.

MAIR (Malware Analysis Incident Response) は講師のザックさんの造語とのこと.
何となく意味が想像できるうえに語感がカッコいい略語ですね.

# シナリオ

```
里内の殆どのPCが急にブルースクリーンで⽌まってしまったという連絡が沢⼭の里⼈から来た
また、殆どの端末からのアンチウィルスアラートが来ている
アンチウィルスで駆除してみても、再起動したらまたすぐ感染してしまう
インシデント対応忍者達が来るまで、里⼈の端末を全部シャットダウンした
DCも感染しているが、業務が⽌まるのでシャットダウン・切断できない

端末：Windows XP ∼ Windows10まで感染確認 ※たけのこの里からリースで借りており、初期化できない
サーバ：2003∼2012
DCも含めて数年間アップデートしていないサーバ/端末が多い
クラウド上に複数のWindowsサーバを構築しており、VPNで里のネットワークと接続している
幾つかのサーバはグローバルIPアドレスが設定されている

ホストFW：全端末はグループポリシー設定でOFFになっている
セキュリティ対策：アンチウィルスソフトとPalo Alto（検知モードで誰もログを見ておらず導⼊しているだけの状態）
資産管理：SKYSEA
パスワード：グループポリシーで制限しておらず、パスワードが未設定の端末も存在する状態
ネットワーク制限：セグメント間の通信は全て許可
海外拠点（パンダの国、キムチの国、フォーの国、ウォッカの国等々）や関連企業と VPNで接続しており、拠点間の通信も全て許可
```

# アーティファクト

- 感染した端末のAutoruns結果：Infected-Computer.arn
- 不審なタスクファイル：DnsScan.xml
- 不審なファイル：mkatz.ini.malware、svchost.mlz.malware
- 不審なパワーシェルファイル：m.ps1
- Windows¥Tempにあった不審なファイル: windows_temp_svchost.exe.malware、TEMP-_MEI117922.zip.malware
- 不審な実⾏ファイル：dl.exe.malware、installed.exe.malware、svchost.exe.malware、taskmgr.exe.malware、wmiex.exe.malware、YqejbPXw.exe.malware

これらのファイルは不特定多数の端末から収集している

# Autorunsの解析

マルウェア感染が疑われる端末の情報保全に使えるツールの一つです. 
Windowsで自動実行されるプロセスを保存して一覧表示してくれます.
見た目が分かりやすいから初心者でも使いやすそうです.

![](/images/2019-11-04-tmcityamasec/2019-11-04-12-27-33.png)

既知のマルウェアなら Options > Hide VirusTotal Clean Entries にチェックを入れて Everything タブを確認するだけで目星がつきます.
タスクスケジューラから呼び出されている DnsScan (c:\windows\temp\svchost.exe) は明らかに怪しいので調査の起点にしてみます.

![](/images/2019-11-04-tmcityamasec/2019-11-04-12-32-28.png)

# タスクファイルの解析

そういえば DnsScan というタスクファイルがあったなと思い出しxmlを確認しました.
重要なのは以下の3点で、指定されたプログラムを60分間隔で永遠に実行し続けるタスクであることが分かります.
マルウェアの永続化機構っぽいので svchost.exe への疑いが深まりました.

```
<Interval>PT60M</Interval> : 60分間隔でこのタスクを繰り返す
<StopAtDurationEnd>false</StopAtDurationEnd> : 繰り返し期間に制限を設けない
<Exec><Command>c:\windows\temp\svchost.exe</Command> : 指定されたプログラムを実行する
```

![](/images/2019-11-04-tmcityamasec/2019-11-04-12-47-12.png)

# InfoStealerの解析

いよいよ実行ファイルに目を向けます.
まずはpestudioにかけてハッシュを入手しておきます.
VirusTotalの検索結果は[こんな感じで](https://www.virustotal.com/gui/file/bdbfa96d17c2f06f68b3bcc84568cf445915e194f130b0dc2411805cf889b6cc/behavior/) InfoStealerと呼ばれる既知のマルウェアでした.
dl.exe と installed.exe もハッシュが同じなので別名の同一マルウェアです.
このマルウェアは様々な経路で拡散されているのかもしれません.

![](/images/2019-11-04-tmcityamasec/2019-11-04-12-56-18.png)

VirusTotal にはサンドボックスで実行された結果が表示されていますが、詳細を確認するにはやはり手元で確認したいです.
Process Monitor を起動した状態で InfoStealer を実行してログを取得します.
マルウェア以外の無関係なログが含まれますが次のようなフィルタを追加すれば

```
Operation is WriteFile
Path ends with exe
```
![](/images/2019-11-04-tmcityamasec/2019-11-04-19-35-25.png)

exeファイルが出力されたログだけを表示させることが出来ます.
ttt.exe, svhost.exe, taskmgr.exe の3種類の実行ファイルがInfoStealerによって出力されています.
ttt.exeは生成されてすぐに削除されていますが svhost.exe, taskmgr.exe は残されています.
引き続きこの二つの実行ファイルを調査していきます.

![](/images/2019-11-04-tmcityamasec/2019-11-04-19-38-20.png)

svhost.exe のハッシュは InfoStealer と同一でした.
InfoStealer は 自分自身のコピーを C:\Windows\SysWOW64\svhost.exe にコピーする事が分かります.
![](/images/2019-11-04-tmcityamasec/2019-11-04-20-01-59.png)

ところで C:\Windows\SysWOW64\ にはもう一つ別の不審な実行ファイルがあります.
更新日時からInfoStealerによって作られたものだと推察できます.
VirusTotalによると[このハッシュも](https://www.virustotal.com/gui/file/b771267551961ce840a1fbcd65e8f5ecd0a21350387f35bbcd4c24125ec04530/behavior/Dr.Web%20vxCube) InfoStealer と判定されます.

![](/images/2019-11-04-tmcityamasec/2019-11-04-20-22-07.png)

ここで Process Monitor のログに戻って見ます.
Operation is Write では wmiex.exe は見つからなかったのでフィルタの制限を緩めます.
ここではフィルタに Path contains wmiex.exe を指定することにより ttt.exe が wmiex.exe を生成した事を確認できました.

![](/images/2019-11-04-tmcityamasec/2019-11-04-20-39-06.png)

次は taskmgr.exe を調べます.
VirusTotalによると[このハッシュは](https://www.virustotal.com/gui/file/de7dba8ef2f284e92f9ceec09599d7e4a31592b773c9642be5bcf18f2463a3a6/detection) Farfli と呼ばれるトロイの木馬だと判定されます.

![](/images/2019-11-04-tmcityamasec/2019-11-04-20-43-20.png)

ところで C:\Windows\SysWOW64\drivers\ にも InfoStealer が存在します.
Process Monitor のログによると最初に実行された InfoStealer によって作成されています.
InfoStealerは様々なパスに自己のコピーを保存することが分かりました.

![](/images/2019-11-04-tmcityamasec/2019-11-04-20-52-02.png)

![](/images/2019-11-04-tmcityamasec/2019-11-04-21-12-20.png)

実行ファイルの挙動をまとめると以下のような流れが明らかになりました.

![](/images/2019-11-04-tmcityamasec/2019-11-04-21-20-22.png)

# Mimikatzの解析

VirusTotalによると[このファイルは](https://www.virustotal.com/gui/file/60b6d7664598e6a988d9389e6359838be966dfa54859d5cb1453cbc9b126ed7d/detection) Mimikatzです.
アイコンからpythonで書かれたコードをPyInstallerでexe化したものだと推察できます.
現時点でこのファイルの侵入経路は定かではありません.

![](/images/2019-11-04-tmcityamasec/2019-11-04-21-37-27.png)

このファイルを実行すると %Temp%\\_MEI24642\ に各種 pyd ファイルが生成されます.
また C:\Windows\Temp\ に m.ps1 ファイルが生成されます.

![](/images/2019-11-04-tmcityamasec/2019-11-04-22-59-29.png)

pydファイル群は収集されたアーティファクトの一つである TEMP-_MEI117922.zip.malware の中身と同一です. 
したがって Mimikatz は %Temp%\\に_MEI{5桁のランダムな数字}\ディレクトリを作成し、その中にpythonバイトコードを生成する事が分かります.

![](/images/2019-11-04-tmcityamasec/2019-11-04-21-41-07.png)

m.ps1 は収集されたアーティファクトの一つと同一です.
巨大な難読化コードのため処理を追いかけるのは容易ではありませんが、ざっと眺めると末尾にBase64と推察できる文字列が見つかりました.
変数名からx64とx32に対応したデータだと推察できます.
Base64エンコードはCyberChefを使ってデコードします.

![](/images/2019-11-04-tmcityamasec/2019-11-04-21-44-16.png)

![](/images/2019-11-04-tmcityamasec/2019-11-04-21-52-23.png)

デコードされたデータはVirusTotalによると [32bit版](https://www.virustotal.com/gui/file/8433ba40d43c0bb661224190e1baf961272d766115113c8bf4b4ddd0dc88d4ea/detection) と [64bit版](https://www.virustotal.com/gui/file/95efe1994ddbe75ff322bf1cec08384b390cb8153b09f8d01ed928296d108cdb/detection) のMimikatzです.

![](/images/2019-11-04-tmcityamasec/2019-11-04-21-53-39.png)

![](/images/2019-11-04-tmcityamasec/2019-11-04-21-55-03.png)

# Radminの解析

VirusTotalによると[このファイルは](https://www.virustotal.com/gui/file/3c2fe308c0a563e06263bbacf793bbe9b2259d795fcc36b953793a7e499e7f71/detection) リモートアクセスツールのRADMINです.
実行中のProcess Monitorログを確認しましたが目ぼしい動きは確認できませんでした.
現時点でこのファイルの侵入経路は定かではありません.

![](/images/2019-11-04-tmcityamasec/2019-11-04-21-39-06.png)

# 永続化の解析

InfoStealer と Mimikatz は関連プログラムをタスクスケジューラに登録します.
アンチウイルスによってマルウェアが削除されても、タスクが残っている限り再び感染する仕組みになっています.

- InfoStealer
![](/images/2019-11-04-tmcityamasec/2019-11-05-00-04-16.png)

    タスク名: WebServers
    実行されるプログラム: C:\Windows\SysWOW64\wmiex.exe

    タスク名: Ddrivers
    実行されるプログラム: C:\Windows\SysWOW64\drivers\svchost.exe

- Mimikatz
![](/images/2019-11-04-tmcityamasec/2019-11-05-00-18-10.png)

    タスク名: DnsScan
    実行されるプログラム: C:\Windows\temp\svchost.exe

    タスク名: Bluetooths
    実行されるプログラム: 下記powershellスクリプト

    ![](/images/2019-11-04-tmcityamasec/2019-11-05-00-24-14.png)

# まとめ

以上の結果をまとめると、実行ファイルの関係から次の概要図が浮かび上がります.

![](/images/2019-11-04-tmcityamasec/2019-11-04-23-54-04.png)

本番中はとにかく情報量が多く自分が何を分析しているのか分からなくなる事がありました. 
練習を重ねて自分なりの解析フローを作り上げる必要がありそうでした.

そんな状況でも最後に[提出資料](https://docs.google.com/presentation/d/19iZOIiOqWt53NyWNAX-PJZaQlObqTciltn7oZ3XQAx0/edit?usp=sharing)をまとめられたのはチームメンバーのおかげです
ありがとうございました


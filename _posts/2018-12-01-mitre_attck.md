---
layout: post
title:  "MITRE ATT&CK 活用事例"
date:   2018-12-01
permalink: /security/:title
categories: Security
tags: MITRE ATT&CK
excerpt: InternetWeek2018 サイバー攻撃の実態を体系化する「ATT&CK」とふれ合うBoF でLTしたスライドを校正したものです。MITRE ATT&CKを実務レベルで活用する具体的な手法を提案します。
mathjax: false
---

* content
{:toc}

# ATT&CKの概要

[ATT&CK（アタック）](https://attack.mitre.org/)はMITRE（マイター）が開発している敵対的な戦術とテクニックのナレッジベースです。
サイバーセキュリティ脅威のモデリングや、検知・防衛方法論の開発の基礎として利用されます。

ATT&CKとよく似たセキュリティ・フレームワークとしてLockheed MartinによるCyber Kill Chainがあります。
日本ではマクニカネットワークスがとても分かりやすく[Cyber Kill Chainの情報](https://www.macnica.net/solution/security_apt.html/)をまとめています。

Cyber Kill Chainが戦術レベルで記述されているのに対して、ATT&CKは戦術を構成する技術レベルで記述されています。
プロダクトを組み合わせて多層防御を検討する段階ではCyber Kill Chainを参照し、各プロダクトの設計・運用の段階ではATT&CKを参照するという使い分けが考えられます。

ATT&CKはCyber Kill ChainにおけるExploit以降のステージをより具体的に細分化したものだと捉えることが出来ます。

![](https://attack.mitre.org/theme/images/Enterprise_Tactics.png)

ATT&CKの公式サイトには **PRE-ATT&CK**、**Enterprise**、**Mobile** の3種類のマトリクスが用意されています。
利用者は保護したい対象システムに応じて Enterprise か Mobile のマトリクスを参照します。
PRE-ATT&CKは攻撃者の組織立ち上げを含む事前活動を対象としているため、これを参照する利用者は限られるでしょう。

マトリクスから攻撃テクニックをドリルダウンすると、攻撃の事例を含む解説や、攻撃テクニックをカテゴライズするための様々なタグを確認できます。
利用者はタグを頼りにして必要な脅威情報をピックアップし、過去にそのテクニックを利用した攻撃者のレポートや技術解説を参照することが出来ます。

![](/images/attck/attck.png)

# サービス事例

既にいくつかのサンドボックスやエンドポイントプロテクションがATT&CKを組み込んでいます。
この動きは2018年に入って加速しました。
今後さらに多くのセキュリティプロダクトがATT&CKを採用しインシデント対応の効率化を図るであろうと予想できます。

- [CrowdStrike Falcon](https://www.crowdstrike.com/sites/jp/summer-product-release-jp-fnl/)

  >Falconの脅威検出にこのフレームワークを採用することで、アラートのトリアージが効率的になり、インシデント分析時間が短縮されます。これにより、セキュリティアナリストとインシデント対応者は、アラートと関係する影響やリスクを即座に把握できるだけでなく、攻撃の段階を瞬時に理解し、重要な質問に素早く回答できます。

  注：CrowdStrike Falconはサンドボックスの[Hybrid-Analysis](https://www.hybrid-analysis.com/)に採用されています

- [ANY RUN](https://any.run/)

  注：ATT&CKに関して特に触れられていませんが、CrowdStrike Falconのように効率的なトリアージを可能にしています

- [Joe Sandbox](https://www.joesandbox.com/)

  >We have completely mapped over 1,800 behavior signatures of Joe Sandbox to Mitre's adversary tactics and techniques. For each analysis you now get the Mitre ATT&ck matrix and can easily compare different malware samples based on their tactics.

- [Tanium](https://www.tanium.com/blog/getting-started-with-the-mitre-att-and-ck-framework-lessons-learned/)

  >At Tanium, our mission is to keep our customers safe and secure. As part of that mission, it’s crucial for the Tanium IT Security organization to operate according to the highest cybersecurity standards. To that end, our cybersecurity team has been working to adopt the MITRE Adversarial Tactics, Techniques & Common Knowledge (ATT&CK™) framework to help us improve and measure the effectiveness of our detection capabilities.

# データソースの入手

ATT&CKを実務で利用するためにはExcelやDBなどに取り込んで業務システムと連携する事になるはずです。
その場合は[MITREのGithubリポジトリ](https://github.com/mitre/cti)で公開されているSTIX2.0形式のデータソースを利用出来ます。

このデータソースを利用するpythonモジュールが[Cyb3rWard0gのGithubリポジトリ](https://github.com/Cyb3rWard0g/ATTACK-Python-Client)で公開されています。
Jupyter Notebookが同梱されておりpythonモジュールを利用してATT&CKのデータソースを活用するTipsが紹介されています。
必要に応じてデータソースを加工してビジネスに適した形で取り出すことが出来ます。

![](/images/attck/notebook.png)

または単純にATT&CKのデータソースを入手するだけで良いならエクスポートされた[Excelファイル](https://github.com/Cyb3rWard0g/ATTACK-Python-Client/tree/master/export_example)も公開されています。

# 事例：ハンティング能力向上

ATT&CKには様々なユースケースが考えられます。
ここでは筆者の主な業務であるパケット分析による脅威ハンティングの能力を向上させた事例を紹介します。

**1. 自組織のポテンシャルを把握する**

  まずは自組織のリソースを棚卸して、ATT&CKの戦術をどこまでカバーできるのか特定します。
  戦術詳細の Data Sources にはその戦術を検知するために必要な情報源が記述されています。
  筆者の組織ではネットワークパケットに痕跡が残る戦術には理論上ほぼすべて対応可能なので、Data SourcesにPacket captureやNetwork flowが含まれる戦術をピックアップします。

  ![](/images/attck/task1_5.png)

  リソースを完全に活用できていればピックアップした戦術はすべて検知可能だと仮定して次のフェーズに進みます..

  ![](/images/attck/task01.png)

**2. カバレッジを測定する**

  次に、実際にハンティングが出来ている戦術を特定してカバレッジを測定します。
  先ほどピックアップしなかった戦術は自組織のリソースでは対応できないので、カバレッジの測定プロセスから除外します。

  このフェーズでは自組織が運用しているセキュリティツールの検知・保護ルールを精査して、ATT&CKの戦術にマッピングする必要があります。

  ![](/images/attck/task10.png)

  ATT&CKの戦術に既存ルールをマッピングできるということは、既にその戦術に対応出来ていると捉えることが出来ます。
  次の図では対応できている戦術をグレーアウトさせています。

  ![](/images/attck/task02.png)

  既存ルールのマッピングが完了したら、自組織のポテンシャルに対する既存ルールのカバレッジを測定出来ます。
  Persistence、Defense Evasion、Collection の3つの戦術はカバレッジが特に低いので早急な手当てが望まれます。

  ![](/images/attck/task03.png)

**3. カバーされていない戦術を分析する**

このフェーズの目的はカバーできていない戦術を検知・防御する新たなルールを策定する事です。
そのためにATT&CKの戦術詳細を参照して攻撃手法を分析していきます。

MitigationやDetectionにそのまま利用できる検知・防御手法が記述されている事もありますが、ほとんどの場合それだけでは不十分です。
ExamplesやReferencesから過去実際にその戦術を採用したAPT攻撃を確認し、セキュリティベンダがリリースしている技術情報を精査します。

![](/images/attck/task20.png)

技術情報の精査が完了したら実際に戦術を検証します。
既存のルールをどこまで回避できるのか、またはどんな痕跡が残されるのか確認します。
その結果に応じて新たなルールを策定していくことになります。

![](/images/attck/task30.png)

今回の例ではわかりやすくBITSによるファイル転送を検証しました。

確認した限りではBITSによるファイル転送においてUser-Agentを変更することは出来ません。
User-Agentに着目すればこの戦術を検知することが出来そうです。
ただしBITSはWindowsUpdateなどにも利用される正規ツールなので、通信先URLなどの情報を併用して不審な通信だけを検知する工夫が必要です。

# まとめ

ATT&CKはサイバー脅威でよく利用される戦術のナレッジです。

サイバー脅威から組織を防衛しようとする人は、ATT&CKを利用して効率的にサイバー脅威に対応する策を立てることが出来ます。

既に十分な対策が出来ている組織では、ATT&CKを利用して日々のインシデントレスポンスの効率を上げることも出来るでしょう。

またATT&CKはよく体系化されているため、サイバーセキュリティに詳しくない人に脅威を説明するのにも役立つでしょう。

# 参考

- [MITRE ATT&CK](https://attack.mitre.org/)

- [SOCYETI ATT&CKについてとMatrixの日本語抄訳](https://www.socyeti.jp/posts/4873582)

- [McAfee 今知るべきATT＆CK｜攻撃者の行動に注目したフレームワーク徹底解説](https://blogs.mcafee.jp/mitre-attck)

- [MITRE Cyber Threat Intelligence Repository expressed in STIX 2.0](https://github.com/mitre/cti)

- [Cyb3rWard0g/ATTACK-Python-Client](https://github.com/Cyb3rWard0g/ATTACK-Python-Client)

- [TANIUM Getting started with the MITRE ATT&CK framework: Improving detection capabilities](https://www.tanium.com/blog/getting-started-with-the-mitre-attack-framework-improving-detection-capabilities/)

- [CrowdStrike 新しいデバイスコントロールモジュール、Dockerコンテナ保護強化によりエンドポイントプロテクションプラットフォームを拡張](https://www.crowdstrike.com/sites/jp/summer-product-release-jp-fnl/)

- [マクニカネットワークス 標的型攻撃対策](https://www.macnica.net/solution/security_apt.html/)

- [JOESecurity Scorch Malware with Joe Sandbox Fire Opal](https://www.joesecurity.org/blog/8594655092323822022)

- [ANY.RUN](https://any.run/)

- [Hybrid-Analysis](https://www.hybrid-analysis.com/)

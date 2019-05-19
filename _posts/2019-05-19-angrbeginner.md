---
layout: post
title:  "Angr Beginner"
date:   2019-05-19
permalink: /ctf/:title
categories: CTF
tags: HarekazeCTF Angr
excerpt: 2019-05-18に開催されたharekazeCTFでAngrを使っただけのWriteupです。
---

* content
{:toc}

Angr on python3を使ってscrambleを解きました。

scrambleをIDAで確認すると入力に応じて0x6FBと0x737に結果が分岐する事が分かります。
0x737は"Correct"を出力しているのでこちらが正解のようです。

![](/images/harekaze/angr_0.png)

0x737に到達する入力を調べたいのですが0x6B5のcall scrambleが曲者です。
scrambleの処理内容は0x888に書かれています。
長い。

![](/images/harekaze/angr_2.png)

全部読むのはとても大変そうなので[Angr](https://github.com/angr/angr)を使います。
到達したいアドレスに0x737を指定し回避したいアドレスに0x6fbを指定します。

```
# python 3.5.2
# angr 8.19.4.5

import angr

p = angr.Project("/root/Downloads/harekazeCTF/scramble")
state = p.factory.entry_state()
sim = p.factory.simulation_manager(state)
sim.explore(find=(0x400000+0x737,), avoid=(0x400000+0x6fb,))
if len(sim.found) > 0:
        print(sim.found[0].posix.dumps(0))
```

```
# python3 scramble.py
WARNING | 2019-05-19 14:36:50,212 | cle.loader | The main binary is a position-independent executable. It is being loaded with a base address of 0x400000.
WARNING | 2019-05-19 14:36:52,918 | angr.state_plugins.symbolic_memory | The program is accessing memory or registers with an unspecified value. This could indicate unwanted behavior.
WARNING | 2019-05-19 14:36:52,919 | angr.state_plugins.symbolic_memory | angr will cope with this by generating an unconstrained symbolic variable and continuing. You can resolve this by:
WARNING | 2019-05-19 14:36:52,920 | angr.state_plugins.symbolic_memory | 1) setting a value to the initial state
WARNING | 2019-05-19 14:36:52,921 | angr.state_plugins.symbolic_memory | 2) adding the state option ZERO_FILL_UNCONSTRAINED_{MEMORY,REGISTERS}, to make unknown regions hold null
WARNING | 2019-05-19 14:36:52,922 | angr.state_plugins.symbolic_memory | 3) adding the state option SYMBOL_FILL_UNCONSTRAINED_{MEMORY_REGISTERS}, to suppress these messages.
WARNING | 2019-05-19 14:36:52,923 | angr.state_plugins.symbolic_memory | Filling register r15 with 8 unconstrained bytes referenced from 0x400970 (__libc_csu_init+0x0 in scramble (0x970))
WARNING | 2019-05-19 14:36:52,932 | angr.state_plugins.symbolic_memory | Filling register r14 with 8 unconstrained bytes referenced from 0x400972 (__libc_csu_init+0x2 in scramble (0x972))
WARNING | 2019-05-19 14:36:52,938 | angr.state_plugins.symbolic_memory | Filling register r13 with 8 unconstrained bytes referenced from 0x400977 (__libc_csu_init+0x7 in scramble (0x977))
WARNING | 2019-05-19 14:36:52,942 | angr.state_plugins.symbolic_memory | Filling register r12 with 8 unconstrained bytes referenced from 0x400979 (__libc_csu_init+0x9 in scramble (0x979))
WARNING | 2019-05-19 14:36:52,953 | angr.state_plugins.symbolic_memory | Filling register rbx with 8 unconstrained bytes referenced from 0x40098a (__libc_csu_init+0x1a in scramble (0x98a))
WARNING | 2019-05-19 14:36:53,131 | angr.state_plugins.symbolic_memory | Filling register cc_ndep with 8 unconstrained bytes referenced from 0x4007e6 (register_tm_clones+0x26 in scramble (0x7e6))
b'HarekazeCTF{3nj0y_h4r3k4z3_c7f_2019!!}'
```

数分でフラグが入手できました。Angrすごい。

---

Product_keyもAngrで解けないか試してみました。
到達したいアドレスは0x9e8、回避したいアドレスは0x87d, 0x9b0, 0x9c3です。

![](/images/harekaze/angr_1.png)

```
sim.explore(find=(0x400000+0x9e8,), avoid=(0x400000+0x87d, 0x400000+0x9b0, 0x400000+0x9c3,))
```

こちらは解答が得られませんでした。
Angrには様々な使い方があるようなので書き方を工夫すれば解けるのかもしれませんが・・・

```
<SimulationManager with 1 avoid>
```

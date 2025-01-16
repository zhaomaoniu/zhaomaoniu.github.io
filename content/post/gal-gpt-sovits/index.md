---
title: 从零开始的柚子社角色语音养成计划
description: 
slug: gal-gpt-sovits
date: 2025-01-16 15:06:00+13:00
image: 
categories: 科技
tags:
    - 柚子社
    - GPT-SoVITS
weight: 1
---

Galgame 又称美少女游戏，美少女自然是其中不可或缺的一环。

在推完「她」的线后，我的内心十分满足，却好像又有些空虚。
我知道，在这个世界中，我会与她幸福永久地生活下去。
但现实中的我，却只能被剧本家画下的休止符挡在外面。
哪怕是一点也好，我想让她来到现实世界，真正地陪伴着我。

她的声音是十分重要的一部分。
因此，本篇文章的目的就是近乎完美地复刻「她」的声音。

> 每个人的「她」都不太一样，这里仅指柚子社的女主们

## 准备工作

本篇文章会涉及到许多计算机相关知识，推荐你先掌握下面几个技能点后再开始行动

- 基础的命令行用法
- 科学上网的方法

### 工具

- Python: 简洁美观的编程语言 \
    下载链接: <https://www.python.org/downloads/> \
    **注意**: 安装时勾选 `Add Python to PATH` 选项

- GARbro: 视觉小说资源浏览器 \
    下载链接: <https://github.com/morkt/GARbro/releases/tag/v1.5.44>

- FreeMote: Emote/PSB 管理工具 \
    下载链接: <https://github.com/UlyssesWu/FreeMote/releases/tag/v4.0.0>

- GPT-SoVITS: 声音克隆 \
    下载链接 (整合包): <https://www.yuque.com/baicaigongchang1145haoyuangong/ib3g1e/dkxgpiy9zb96hob4#KTvnO>

- 一个好用的文本编辑器 \
    推荐: [Sublime Text](https://www.sublimetext.com/), [Visual Studio Code](https://code.visualstudio.com/) \
    如果自信的话，用系统自带的记事本也没有问题。

上述软件安装好后，请打开命令行，输入下面的命令，安装一个立绘合成库：

```bash
pip install krkr-sprite-synth -i https://pypi.tuna.tsinghua.edu.cn/simple
```

如果出现错误，请尝试删除 `-i` 及以后的内容，重新运行命令。

顺利的话，你会看到命令输出 `Successfully installed Pillow-11.1.0 krkr-sprite-synth-0.1.7`，这表示你已经成功安装了立绘合成库。

> 顺带一提，这个库是我写的，所以有什么问题都是我的锅（

### 资源

资源，也就是数据，来源于游戏本身。

它可以通过录屏、截屏获取，不过这里我们选择解包。相比截屏，解包获取数据的效率更高，不需要对数据进行太多处理就能使用。

本文推荐使用民间汉化提供的游戏进行解包，因为它们通常是没有加密的（或者说汉化组已经帮我们解密好了）。

你需要在 GARbro 中进入游戏的目录，并把其中的 `scn.xp3`, `fgimage.xp3` 和 `voice.xp3` 提取到一个新建文件夹。

> 如果你发现 GARbro 打不开这些 `.xp3` 文件，可以尝试把它们从游戏目录复制出来后，重新用 GARbro 打开副本，加密方法选择 `no encryption` 即可。

完成后，你的新建文件夹应该差不多长这样。如果不是的话，最好整理一下，不然待会会变的一团糟。

```text
root/
├── scn/
│ ├── 【共通】01.ks.scn
│ ├── 【共通】02.ks.scn
│ ├── 【共通】03.ks.scn
│ └── ...
├── voice/
│ ├── ama_001_0001.ogg
│ ├── ama_001_0002.ogg
│ ├── ama_001_0003.ogg
│ └── ...
├── fgimage/
│ ├── かぐ耶.stand
│ ├── かぐ耶a.sinfo
│ ├── かぐ耶a.txt
│ ├── かぐ耶a_0.txt
│ ├── かぐ耶a_0_5096.png
│ ├── かぐ耶a_0_5097.png
│ ├── かぐ耶a_0_5151.png
│ ├── ...
│ ├── かぐ耶b.sinfo
│ ├── かぐ耶b.txt
│ ├── かぐ耶b_0.txt
│ ├── かぐ耶b_0_6423.png
│ ├── かぐ耶b_0_6529.png
│ ├── かぐ耶b_0_6530.png
│ └── ...
└── ...
```

## 操作

### 解码剧情

柚子社使用的是 Kirikiri 引擎，它的主体文件，包含对话、背景和动作，保存在了 `.scn` 文件中。

`.scn` 文件是一种 `.psb` 文件（大概？），所以我们可以使用 FreeMote 将它转换为 JSON，方便脚本读取。

1. 将 `scn` 文件夹中的文件全部选择，拖拽到 `FreeMoteToolkit/PsbDecompile.exe`，等待解码完成。

2. 解码过程中，一个命令行窗口会弹出来，等它消失之后，你会发现 `scn` 文件夹中多了一堆 `.json` 文件。

3. 在 `scn` 的同级目录下新建一个 `unparsed` 文件夹后，把这些 `.json` 文件剪切进去。

现在，你的文件夹应该长这样

```diff
  root/
  ├── scn/
  │ ├── 【共通】01.ks.scn
  │ ├── 【共通】02.ks.scn
  │ ├── 【共通】03.ks.scn
  │ └── ...
  ├── voice/
  │ ├── ama_001_0001.ogg
  │ ├── ama_001_0002.ogg
  │ ├── ama_001_0003.ogg
  │ └── ...
  ├── fgimage/
  │ ├── かぐ耶.stand
  │ ├── かぐ耶a.sinfo
  │ ├── かぐ耶a.txt
  │ ├── かぐ耶a_0.txt
  │ ├── かぐ耶a_0_5096.png
  │ ├── かぐ耶a_0_5097.png
  │ ├── かぐ耶a_0_5151.png
  │ ├── ...
  │ ├── かぐ耶b.sinfo
  │ ├── かぐ耶b.txt
  │ ├── かぐ耶b_0.txt
  │ ├── かぐ耶b_0_6423.png
  │ ├── かぐ耶b_0_6529.png
  │ ├── かぐ耶b_0_6530.png
  │ └── ...
+ ├── unparsed/
+ │ ├── 【共通】01.ks.json
+ │ ├── 【共通】01.ks.resx.json
+ │ ├── 【共通】02.ks.json
+ │ ├── 【共通】02.ks.resx.json
+ │ ├── 【共通】03.ks.json
+ │ ├── 【共通】03.ks.resx.json
+ │ └── ...
  └── ...
```

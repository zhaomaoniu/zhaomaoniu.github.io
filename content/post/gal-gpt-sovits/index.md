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
- 日语中五十音的基本发音

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

上述软件安装好后，请打开命令行，输入下面的命令，安装一些第三方库：

```bash
pip install krkr-sprite-synth tqdm -i https://pypi.tuna.tsinghua.edu.cn/simple
```

其中，`krkr-sprite-synth` 是用于合成 ~Kirikiri 引擎（未测试）~ 柚子社游戏的角色立绘的库，`tqdm` 是用于显示进度条的库。

如果出现错误，请尝试删除 `-i` 及以后的内容，重新运行命令。

顺利的话，你会看到命令的最后一行输出是 `Successfully installed ...`，这表示你已经成功安装了。

> 顺带一提，`krkr-sprite-synth` 是我写的，所以有什么问题都是我的锅（

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

### 脚本

脚本是本文的核心，用于数据提取、分类语音和生成 GPT-SoVITS 的 `.list` 文件。

将下面折叠起来的脚本分别保存为 `parser.py`, `mapper.py`, `list_generator.py` 和 `finder.py`，放在你刚才新建的文件夹中。

> （以防你其实不知道）其实 `.py` 文件就是文本文件，你只需要新建一个文本文件，把下面的内容复制进去，然后把文件后缀名改为 `.py` 即可。

<!-- markdownlint-disable MD033 -->
<details>
<summary>parser.py</summary>

```python
import os
import re
import json
from tqdm import tqdm
from pathlib import Path
from collections import Counter
from dataclasses import dataclass
from typing import List, Union, Callable, Dict, Any, Optional


@dataclass
class TextEntry:
    """存储文本条目信息的数据类"""

    character: str
    voice: str
    text: str
    options: Dict[str, Any]


class JapaneseTextProcessor:
    """处理日语文本的主类"""

    # 类级别常量
    HIRAGANA_RANGE = ("\u3040", "\u309F")
    KATAKANA_RANGE = ("\u30A0", "\u30FF")
    CHARACTER_MAPPINGS = {"乃愛": "乃爱"}
    # 用于将角色名映射到正确的名称
    # 汉化组可能会忘记翻译某些角色名，导致同一个角色因为两个不同的名字被当成不同的角色

    @staticmethod
    def contains_kana(text: str) -> bool:
        """检查文本是否包含假名"""
        return any(
            (
                JapaneseTextProcessor.HIRAGANA_RANGE[0]
                <= char
                <= JapaneseTextProcessor.HIRAGANA_RANGE[1]
            )
            or (
                JapaneseTextProcessor.KATAKANA_RANGE[0]
                <= char
                <= JapaneseTextProcessor.KATAKANA_RANGE[1]
            )
            for char in text
        )

    @staticmethod
    def clean_text(text: str) -> str:
        """清理文本中的特殊标记"""
        text = re.sub(r"\[.*?\]", "", text)
        text = re.sub(r"\\n", "", text)
        text = re.sub(r"%.*?;", "", text)
        return text

    @staticmethod
    def create_japanese_text_extractor(
        samples: List[Union[str, List[List[str]]]]
    ) -> Callable[[Union[str, List[List[str]]]], str]:
        """创建日语文本提取器"""
        position_counter = Counter()

        for sample in samples:
            if isinstance(sample, str):
                position_counter["direct"] += 1
            elif isinstance(sample, list):
                for inner_list in sample:
                    if not isinstance(inner_list, list):
                        continue
                    for i, text in enumerate(inner_list):
                        if isinstance(
                            text, str
                        ) and JapaneseTextProcessor.contains_kana(text):
                            # 字数小于 5 的文本权重为 1，否则为 2
                            # 防止定位到角色名
                            position_counter[i] += 1 if len(text) < 5 else 2

        if not position_counter:
            raise ValueError("No valid text position found in samples")

        most_common_position = position_counter.most_common(1)[0][0]

        def extractor(data: Union[str, List[List[str]]]) -> str:
            if isinstance(data, str):
                return data
            if isinstance(data, list) and data and isinstance(data[0], list):
                if most_common_position == "direct":
                    return data[0][0]
                try:
                    return data[0][most_common_position]
                except IndexError:
                    raise ValueError(f"Cannot access position {most_common_position}")
            raise ValueError("Unsupported data structure")

        return extractor


class SceneParser:
    """场景解析器类"""

    def __init__(self, base_path: str):
        self.base_path = Path(base_path)

    def parse_scene(
        self, scene_data: Dict[str, Any], extractor: Callable
    ) -> List[TextEntry]:
        """解析单个场景"""
        entries = []

        for text_data in scene_data.get("texts", []):
            try:
                entry = self._process_text_entry(text_data, extractor)
                if entry:
                    entries.append(entry)
            except (IndexError, KeyError) as e:
                print(f"Error processing text entry: {e}")
                continue

        return entries

    def _process_text_entry(
        self, text_data: List, extractor: Callable
    ) -> Optional[TextEntry]:
        """处理单个文本条目"""
        character_name = text_data[0]
        voice_data = text_data[2]
        action_data = text_data[4]["data"]

        if character_name is None or voice_data is None:
            return None

        voice = voice_data[0]["voice"].split("|")[0] if voice_data else None
        # kag_507_0013|DSP_ビデオ通話 -> kag_507_0013
        # NOTE: 天使骚骚是这样存的，脚本迁移到别的游戏是时候可能需要更改

        if not voice:
            return None

        scns = JapaneseTextProcessor.clean_text(extractor(text_data[1]))
        character_name = JapaneseTextProcessor.CHARACTER_MAPPINGS.get(
            character_name, character_name
        )

        for entry in action_data:
            if entry[1] == "msgwin" and entry[0] == "face":
                options = (
                    entry[2].get("redraw", {}).get("imageFile", {}).get("options", {})
                )
                if options:
                    return TextEntry(
                        character=character_name,
                        voice=voice,
                        text=scns,
                        options=options,
                    )
        return None

    def process_file(self, filename: str) -> List[TextEntry]:
        """处理单个文件"""
        file_path = self.base_path / filename
        try:
            with open(file_path, "r", encoding="utf-8") as json_file:
                data = json.load(json_file)

            result = []
            for scene in data.get("scenes", []):
                try:
                    if not scene.get("texts"):
                        continue

                    samples = [entry[1] for entry in scene["texts"]]
                    extractor = JapaneseTextProcessor.create_japanese_text_extractor(
                        samples
                    )
                    result.extend(self.parse_scene(scene, extractor))
                except (KeyError, IndexError) as e:
                    print(f"Error processing scene in {filename}: {e}")
                    continue

            return result

        except json.JSONDecodeError as e:
            print(f"Error decoding JSON from {filename}: {e}")
            return []
        except Exception as e:
            print(f"Unexpected error processing {filename}: {e}")
            return []


def main():
    """主函数"""
    parser = SceneParser("unparsed")
    all_results = []

    # 获取所有需要处理的文件
    files_to_process = [
        f for f in os.listdir(parser.base_path) if f.endswith(".ks.json")
    ]

    # 使用tqdm创建进度条
    for filename in tqdm(files_to_process, desc="Processing files", unit="file"):
        results = parser.process_file(filename)
        all_results.extend(results)

    print(f"\nTotal entries processed: {len(all_results)}")

    # 添加保存进度显示
    print("Saving results to data.json...")
    with open("data.json", "w", encoding="utf-8") as f:
        json.dump(
            [vars(entry) for entry in all_results], f, ensure_ascii=False, indent=4
        )
    print("Save completed!")


if __name__ == "__main__":
    main()

```

</details>

<details>
<summary>mapper.py</summary>

```python
import json
from pathlib import Path
import concurrent.futures
from collections import defaultdict
from krkr_sprite_synth import SpriteSynth


CHARACTER_NAME = "天音"
# 角色名，应该与 reboot.json 中的 character 字段一致
JAPANESE_CHARACTER_NAME = "天音"
# 角色日语名，应该与 fgimage 中的一致


PARSED_DIR = "data.json"
# 解析后的文件所在路径
OUTPUT_PATH = f"outputs/{CHARACTER_NAME}"
# 语音分类结果输出路径
VOICE_PATH = "voice"
# 语音文件所在路径
A_INFO_PATH = f"fgimage/{JAPANESE_CHARACTER_NAME}a.sinfo"
B_INFO_PATH = f"fgimage/{JAPANESE_CHARACTER_NAME}b.sinfo"
A_LAYERS_INFO_PATH = f"fgimage/{JAPANESE_CHARACTER_NAME}a.txt"
B_LAYERS_INFO_PATH = f"fgimage/{JAPANESE_CHARACTER_NAME}b.txt"
ASSETS_PATH = "fgimage"  # 图片所在的路径


data = []


synth = SpriteSynth(
    a_info_path=A_INFO_PATH,
    b_info_path=B_INFO_PATH,
    a_layers_info_path=A_LAYERS_INFO_PATH,
    b_layers_info_path=B_LAYERS_INFO_PATH,
    assets_path=ASSETS_PATH,
    character_name=JAPANESE_CHARACTER_NAME,
)


with open(PARSED_DIR, "r", encoding="utf-8") as json_file:
    data = json.load(json_file)


data = [entry for entry in data if entry["character"] == CHARACTER_NAME]

result = defaultdict(list)

for entry in data:
    key = f"{entry['options']['pose']}-{entry['options']['face']}"
    result[key].append(entry)


def process_entry(v):
    parse_result = synth.get_parse_result(**v[0]["options"])

    image = synth.draw(
        "私服", v[0]["options"]["face"], "1" if parse_result.info_type == "a" else "3"
    )
    name = (
        parse_result.info_type
        + "_"
        + v[0]["options"]["face"]
        + "#"
        + ",".join(parse_result.faces)
    )

    # Remove invalid characters
    name = "".join([c for c in name if c not in r'\/:*?"<>|'])

    output_k_path = Path(OUTPUT_PATH) / name
    output_k_path.mkdir(parents=True, exist_ok=True)
    image.save(f"{OUTPUT_PATH}/{name}.png")

    for entry in v:
        voice = (Path(VOICE_PATH) / (entry["voice"] + ".ogg")).read_bytes()
        # Copy to output_k_path
        (output_k_path / (entry["voice"] + ".ogg")).write_bytes(voice)
        print(f"Saved {output_k_path / (entry['voice'] + '.ogg')}")


with concurrent.futures.ThreadPoolExecutor() as executor:
    futures = [executor.submit(process_entry, v) for v in result.values()]
    concurrent.futures.wait(futures)

```

</details>

<details>
<summary>list_generator.py</summary>

```python
import json
from pathlib import Path


CHARACTER_NAME = "乃爱"
# 角色名，应该与 data.json 中的 character 字段一致
JAPANESE_CHARACTER_NAME = "乃愛"
# 角色日语名，应该与 fgimage 中的一致
USING_INDEXES = [
    "a_01",
    "a_02",
    "a_03",
    "a_04",
    "b_01",
    "b_02",
    "b_03",
    "b_04",
]
OUTPUT_PATH = "乃爱_普通.list"
# 数据集文件输出路径
PARSED_DIR = "data.json"
# 解析后的文件所在路径
MAPPER_OUTPUT_PATH = f"outputs/{CHARACTER_NAME}"
# 语音分类结果输出路径


with open(PARSED_DIR, "r", encoding="utf-8") as json_file:
    data = json.load(json_file)


voice_data = {entry["voice"]: entry["text"] for entry in data}


# {path}|{name}|{language:JP}|{text}
results = []

for index in USING_INDEXES:
    # 找到对应文件夹
    paths = list(Path(MAPPER_OUTPUT_PATH).glob(f"{index}#*"))
    if not paths:
        print(f"Index {index} not found")
        continue

    path = paths[0]

    files = list(path.glob("*.ogg"))

    for file in files:
        voice_name = file.stem
        text = voice_data.get(voice_name)
        if text is None:
            print(f"Text not found for {voice_name}, skipping")
            continue

        text = (
            text.replace("「", "")
            .replace("」", "")
            .replace("　", "")
            .replace("『", "")
            .replace("』", "")
        )
        # 如果只有符号，就不要了
        if all([not c.isalnum() for c in text]):
            continue

        if "DL" in voice_name or "DSP" in voice_name:
            continue

        path_str = str(file.absolute()).replace("\\", "/")

        results.append(f"{path_str}|{CHARACTER_NAME}|JA|{text}")

with open(OUTPUT_PATH, "w", encoding="utf-8") as f:
    f.write("\n".join(results))
    print(f"Saved to {OUTPUT_PATH}")

```

</details>

<details>
<summary>finder.py</summary>

```python
import json
from pathlib import Path


data = json.loads(Path("data.json").read_text("utf-8"))


if __name__ == "__main__":
    try:
        while True:
            voice = input("你要查询哪条语音？")

            results = [entry for entry in data if entry["voice"] == voice]

            if len(results) == 0:
                print("没有找到这条语音")
                continue

            print(f"找到了 {len(results)} 条对应数据")

            for result in results:
                print("=" * 50)
                print(f"    Character: {result['character']}")
                print(f"    Text: {result['text']}")
                print(f"    Dress: {result['options']['dress']}")
                print(f"    Pose: {result['options']['pose']}")
                print(f"    Face: {result['options']['face']}")
    except KeyboardInterrupt:
        pass

```

</details>

现在，你的文件夹应该差不多长这样

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
+ ├── parser.py
+ ├── mapper.py
+ ├── list_generator.py
+ └── finder.py
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
  ├── parser.py
  ├── mapper.py
  ├── list_generator.py
  └── finder.py
```

接下来，就该脚本上场了。

### 运行脚本

#### 数据提取 (parser.py)

假如你因为好奇而点开刚才那堆 JSON 文件看了眼，你会发现它们的结构十分复杂，完全不知道该怎么处理。

这时候， `parser.py` 就派上用场了。它会把这些复杂的结构解析成我们需要的文本、角色名、语音和立绘信息，方便我们后续的操作。

~在运行脚本前，我们需要对 `parser.py` 进行一些配置。~

这部分内容还在施工中，它有关脚本的通用性。目前而言，`parser.py` 仅在柚子社最新的几部作品上测试可行。具体而言，是它们的 **吉里吉里2模拟器** 版本。这里推荐到 [小鳥遊暁の会员制餐厅](https://t-satoru.top/) 获取。

现在，你可以打开命令行，输入下面的命令，运行脚本。

```bash
python parser.py
```

顺利的话，你会看到进度条在不断增长，最后输出 `Save completed!`，这表示数据提取完成。
同时，你的文件夹中会多出一个 `data.json` 文件，里面存储了所有的数据。

#### 语音分类 (mapper.py)

接下来，我们需要对语音使用立绘进行分类。

经过测试，这个方法能有效区分角色不同情感状态下的语音。这将有助于训练出优质的 GPT-SoVITS 模型，也会使参考音频的选择更加方便。

使用你安装的文本编辑器打开 `mapper.py`，修改其中的 `CHARACTER_NAME` 和 `JAPANESE_CHARACTER_NAME` 为你的角色名和日语角色名。

具体而言，`CHARACTER_NAME` 是 `data.json` 中的 `character` 字段，`JAPANESE_CHARACTER_NAME` 是 `fgimage` 文件夹中的立绘文件名。

> 要将它们对应上，你可能需要一点日语基础。如果不知道的话，就去百科上找到这个角色的日语名，再对照立绘文件名吧。

然后，你可以打开命令行，输入下面的命令，运行脚本。

```bash
python mapper.py
```

顺利的话，你会看到一堆 `Saved ...` 的输出，这表示语音文件正在被分类到对应的立绘文件夹中。

这时候，你的文件夹应该是这样的

```diff
  root/
  ├── scn/
  │   ├── 【共通】01.ks.scn
  │   ├── 【共通】02.ks.scn
  │   ├── 【共通】03.ks.scn
  │   └── ...
  ├── voice/
  │   ├── ama_001_0001.ogg
  │   ├── ama_001_0002.ogg
  │   ├── ama_001_0003.ogg
  │   └── ...
  ├── fgimage/
  │   ├── かぐ耶.stand
  │   ├── かぐ耶a.sinfo
  │   ├── かぐ耶a.txt
  │   ├── かぐ耶a_0.txt
  │   ├── かぐ耶a_0_5096.png
  │   ├── かぐ耶a_0_5097.png
  │   ├── かぐ耶a_0_5151.png
  │   └── ...
  │   ├── かぐ耶b.sinfo
  │   ├── かぐ耶b.txt
  │   ├── かぐ耶b_0.txt
  │   ├── かぐ耶b_0_6423.png
  │   ├── かぐ耶b_0_6529.png
  │   ├── かぐ耶b_0_6530.png
  │   └── ...
  ├── unparsed/
  │   ├── 【共通】01.ks.json
  │   ├── 【共通】01.ks.resx.json
  │   ├── 【共通】02.ks.json
  │   ├── 【共通】02.ks.resx.json
  │   ├── 【共通】03.ks.json
  │   ├── 【共通】03.ks.resx.json
  │   └── ...
+ ├── outputs/
+ │   ├── 辉耶/
+ │   │   ├── a_01#表情（耳無し）表情ベース/
+ │   │   │   ├── kag_006_0009.ogg
+ │   │   │   ├── kag_009_0031.ogg
+ │   │   │   ├── kag_013_0008.ogg
+ │   │   │   ├── kag_013_0012.ogg
+ │   │   │   └── ...
+ │   │   ├── a_02#表情（耳無し）3笑顔1/
+ │   │   │   ├── kag_006_0001.ogg
+ │   │   │   ├── kag_007_0008.ogg
+ │   │   │   ├── kag_008_0001.ogg
+ │   │   │   ├── kag_009_0024.ogg
+ │   │   │   └── ...
+ │   │   ├── ...
+ │   │   ├── a_01#表情（耳無し）表情ベース.png
+ │   │   ├── a_02#表情（耳無し）3笑顔1.png
+ │   │   ├── a_02h#表情（耳無し）3笑顔1,頬　弱.png
+ │   │   ├── a_03#表情（耳無し）5微笑み1.png
+ │   │   ├── a_03h#表情（耳無し）5微笑み1,頬　弱.png
+ │   │   └── ...
  ├── parser.py
  ├── mapper.py
  ├── list_generator.py
  └── finder.py
```

#### 生成 .list 文件 (list_generator.py)

一个受人喜爱的角色必然是立体的，她的语音也应该是多样的。

为了不丢失这些多样性，我们可以把一个角色的语音分成几个大类，然后分别训练 GPT-SoVITS 模型。
这样也能避免后续使用音调较高的参考音频时，角色的语音变得沙哑的问题。

操作如下：

1. 进入 `outputs` 文件夹，找到你的角色文件夹，里面应该有很多子文件夹，每个子文件夹对应一中立绘表情。
2. 鉴赏立绘，找到你想训练的种类的立绘（如平和、愤怒、悲伤等），记住它的文件编号。举个例子，`a_01#表情（耳無し）表情ベース` 的编号是 `a_01`。
3. 打开 `list_generator.py`，修改其中的 `CHARACTER_NAME` 和 `JAPANESE_CHARACTER_NAME` 为你的角色名和日语角色名。
4. 修改 `USING_INDEXES` 为你想要训练的立绘编号。编号本身被半角双引号包裹，用逗号分隔。
5. 修改 `OUTPUT_PATH` 为你想要输出的 `.list` 文件名，如 `乃爱_普通.list`。
6. （可选，不建议，效果提升不大）进入你选中编号的各个子文件夹，把语音全都听一遍，然后删除你认为偏离主题的语音。
7. 打开命令行，输入下面的命令，运行脚本。

```bash
python list_generator.py
```

顺利的话，你会看到 `Saved to ...` 的输出，这表示 `.list` 文件已经生成。

推荐使用文本编辑器打开 `.list` 文件，删除一些文字与语音不匹配的条目，以提高模型的训练效果。

举个例子，「……、ワタシも別にいいですけど」前面的省略号实际上在语音里可能是ん的发音，这时候就需要删除这个条目（如果你不嫌麻烦可以自己听一下修正过来）。

### 训练 GPT-SoVITS 模型

接下来，就可以使用刚才生成的 `.list` 文件，训练 GPT-SoVITS 模型了。

# UTAU_VChat

> 版本 **1.0.0**　|　Windows 10/11 x64

让 **UTAU 声库真正"开口说话"** 的桌面软件。

由 **DeepSeek** 驱动角色对话，用 **UTAU 声库**发声，**VOICEVOX** 提供自然语调，
支持角色切换、情绪语气、中日双语、以及从参考音频自动学习说话风格。

---

## ✨ 功能

| 功能 | 说明 |
|---|---|
| **多角色** | 重音テト / 足立レイ，各自独立声库、立绘、代表色、调音参数、人格卡 |
| **自然语调** | VOICEVOX（四国めたん）预测音高曲线与音节时长，喂给 UTAU 声库 |
| **情绪语气** | 自动判断回复情绪（开心/生气/悲伤/温柔/惊讶/傲娇），动态调整音高·音域·语速·气声·紧张度 |
| **角色卡** | 基于萌娘百科设定的人格提示词，含人物关系网，避免出戏 |
| **双语** | 语音语言（日语/中文）与显示语言可分离——可以「说日语、显示中文」 |
| **参考音频学习** | `/set learn <音频>` 提取声学特征 → 生成风格描述 → LLM 推荐参数并应用 |
| **立绘界面** | GUI 以角色立绘作 75% 透明背景，终端风输出，随窗口自动换行 |
| **说话人分离** | 分析多角色参考视频时，用声库指纹（jitter）区分说话人，避免学错角色 |

---

## 🏗 架构

### 数据流

```
  用户输入
     │
     ▼
① DeepSeek API ────────────── requests ──────► 角色回复文本
     │
     ▼
② strip_paren() 过滤括号内容
     │
     ├─►③ translate() 可选：显示语言 ≠ 语音语言时翻译（仅显示用）
     │
     ▼
④ detect_emotion() ──► apply_emotion()
     │                  调整：音高/音域/语速/气声/紧张度/辅音速度
     ▼
⑤ 文本 → 音符
     ├─ _voicevox_notes()   ← 主路径：VOICEVOX 预测时长 + 音高
     ├─ chinese_to_notes()  ← 中文：pypinyin → 假名近似 + 声调映射
     └─ text_to_notes()     ← 兜底：Sudachi 分词 + H/L 语调规则
     │
     ▼
⑥ 参数后处理
     _apply_pitch_range() 音域夸张度
     _apply_pause_scale() 句间停顿
     _apply_speed()       吐字速度（字与字间隔）
     │
     ▼
⑦ render() ─ 写音符 JSON ─ spawn(隐藏控制台) ─► ou_render.exe
     │
     ▼
⑧ C# 渲染器 (renderer/Program.cs)
     ├─ Ustx.Create()                     建 OpenUtau 项目
     ├─ ClassicSinger + 声库加载
     ├─ JapanesePresampPhonemizer         音素化 (CV/VCV)
     ├─ BREC(气声)/TENC(紧张度)/VEL(辅音)  表达式注入
     └─ PlaybackManager.RenderMixdown()   ─► WAV
     │
     ▼
⑨ play() ─ winsound ─► 扬声器
```

### 模块

| 文件 | 职责 |
|---|---|
| `gui.py` | GUI：立绘背景（75% 透明）、终端风输出、命令解析、多线程渲染 |
| `speak.py` | 核心库 + CLI：DeepSeek 对话、文本→音符、渲染调度、角色卡、情绪系统 |
| `zh.py` | 中文支持：拼音→假名近似表、声调→音高映射 |
| `refstyle.py` | 参考音频分析：声学特征 → 自然语言描述 → LLM 推荐参数 |
| `renderer/Program.cs` | C# 渲染器：驱动 `OpenUtau.Core` 完成无界面合成 |
| `characters.json` | 多角色配置：声库路径、立绘、代表色、调音参数、人格卡 |
| `config.json` | 全局配置：API Key、模型、渲染器路径、语言、显示语言 |

### 关键设计

1. **为什么要写 C# 渲染器？**
   OpenUtau 是纯 GUI 程序，没有命令行入口。`ou_render.exe` 直接引用
   `OpenUtau.Core.dll`，用 `DocManager` 无界面初始化，
   再调 `PlaybackManager.RenderMixdown()` 把音符渲染成 WAV。

2. **为什么需要 VOICEVOX？**
   UTAU 声库本身不具备"自然语调"能力。先由 VOICEVOX 预测一段
   真人风格语音的**音高曲线 + 音节时长**，再把这两样交给 UTAU 声库演唱。

3. **为什么参数分角色独立存储？**
   每个角色在 `characters.json` 中独立保存调音参数与人格卡，
   切换角色时整体切换，互不干扰。

---

## 📦 开源依赖

### 核心引擎

| 组件 | 版本 | 许可证 | 作用 |
|---|---|---|---|
| [OpenUtau](https://github.com/openutau/OpenUtau)（Core / Plugin.Builtin / worldline.dll） | 0.1.565 | **MIT** | 歌声合成引擎（实际发声） |
| [VOICEVOX CORE](https://github.com/VOICEVOX/voicevox_core) | 0.17.0 | **MIT** | 语调预测（音高曲线 + 时长） |

### Python 依赖

| 包 | 版本 | 许可证 | 作用 |
|---|---|---|---|
| [requests](https://github.com/psf/requests) | 2.34.2 | Apache-2.0 | DeepSeek API 调用 |
| [SudachiPy](https://github.com/WorksApplications/SudachiPy) | 0.6.11 | Apache-2.0 | 日语形态素解析 |
| [SudachiDict-core](https://github.com/WorksApplications/SudachiDict) | 20260723 | Apache-2.0 | 日语词典 |
| [pypinyin](https://github.com/mozillazg/python-pinyin) | 0.55.0 | MIT | 中文拼音（含声调） |
| [NumPy](https://numpy.org/) | 2.5.2 | BSD-3-Clause | 声学特征分析 |
| [Pillow](https://python-pillow.org/) | 12.3.0 | MIT-CMU | 立绘背景处理 |

Python 标准库：`tkinter`、`wave`、`subprocess`、`threading`、`json`、`ctypes`、`winsound`

### 数据组件

| 组件 | 许可证 | 作用 |
|---|---|---|
| [Open JTalk 词典](https://open-jtalk.sourceforge.net/) `open_jtalk_dic_utf_8-1.11` | BSD（NAIST） | 日语文本 → 音素 |
| VOICEVOX 语音模型 `0.vvm`（**四国めたん**） | ⚠️ 见免责声明 | 语调预测模型 |
| 重音テト / 足立レイ UTAU 声库 | ⚠️ 见免责声明 | 发声声源 |

### 仅开发/分析用（运行时不需要）

[faster-whisper](https://github.com/SYSTRAN/faster-whisper)（MIT）、
[pythonnet](https://github.com/pythonnet/pythonnet)（MIT）、
[FFmpeg](https://ffmpeg.org/)（LGPL/GPL）

---

## 🚀 安装

### 选哪个文件

| 文件 | 大小 | 适合 |
|---|---|---|
| `UTAU_VChat_Setup_1.0.0.exe` | 675 MB | **推荐**。有安装向导、开始菜单、卸载项 |
| `UTAU_VChat_1.0.0_portable.zip` | 772 MB | 绿色版。免安装，拷到 U 盘就能带走 |

两个内容完全一样，**目标机器都不需要装 Python / OpenUtau / VOICEVOX / .NET**。
包内已自带：OpenUtau 合成引擎（仅功能部分，无 GUI）、Worldline 声码器、
VOICEVOX CORE 语调引擎 + 语音模型、Open JTalk 日语词典、SudachiPy、
重音テト / 足立レイ 声库，以及 Python 运行时与全部依赖。

### 方式 A：安装程序（推荐）

1. 双击 **`UTAU_VChat_Setup_1.0.0.exe`**。
2. 会弹一次 **UAC 提示**（因为默认装到 `C:\Program Files`）→ 点「是」。
3. 阅读「**许可协议**」（MIT）→ 选「我同意此协议」→ 下一步。
4. 阅读「**信息**」页（即《首次使用必读》）→ 下一步。
5. **选择目标位置** —— 默认 `C:\Program Files\UTAU_VChat`，
   点「**浏览(R)...**」可以改成任意目录 → 下一步。
6. 选择附加任务（可选创建桌面快捷方式）→ 下一步 → 安装（约 1 分钟）。
7. 完成页可勾选「立即启动」和「查看首次使用必读」。

装完会创建开始菜单项：

```
UTAU_VChat                     主程序
自检 (检查组件是否齐备)          跑一遍 --selftest
首次使用必读                    说明文档
卸载 UTAU_VChat
```

**没有管理员权限？** 用命令行加 `/CURRENTUSER`，装到用户目录、不弹 UAC：

```bash
UTAU_VChat_Setup_1.0.0.exe /CURRENTUSER
```

**安装到 Program Files 后配置存哪？**

`C:\Program Files` 默认不可写，所以程序会**自动**把配置和输出改存到
`%LOCALAPPDATA%\UTAU_VChat`：

| 内容 | 位置 |
|---|---|
| 程序、引擎、声库 | `C:\Program Files\UTAU_VChat\`（只读） |
| `config.json` / `characters.json` | `%LOCALAPPDATA%\UTAU_VChat\` |
| 生成的音频 | `%LOCALAPPDATA%\UTAU_VChat\out\` |

用 `UTAU_VChat.exe --diag` 可以看到实际使用的数据目录。
装到可写目录（比如 `D:\UTAU_VChat`）时则是完全便携的，配置就在程序目录里。

### 方式 B：绿色版

1. 把 `UTAU_VChat_1.0.0_portable.zip` 解压到**有写入权限**的目录，
   例如桌面、文档、或 `D:\UTAU_VChat`。
2. 进入 `UTAU_VChat` 文件夹，双击 **`UTAU_VChat.exe`**。

> 解压到 `C:\Program Files` 也能用 —— 程序检测到目录不可写时会把配置
> 改存到 `%LOCALAPPDATA%\UTAU_VChat`，功能不受影响。

---

## 🎬 首次启动

不管用哪种方式安装，第一次运行都会走这三步：

**① 填写 API Key**

弹出「首次使用设置」窗口，要求填写**你自己的** DeepSeek API Key
—— 程序**不附带任何密钥**，`config.json` 里是空的。

| 字段 | 说明 |
|---|---|
| DeepSeek API Key | **必填**。在 <https://platform.deepseek.com/api_keys> 申请 |
| API Base URL | 默认 `https://api.deepseek.com`，一般不用改 |
| 模型名 | 默认 `deepseek-chat`，一般不用改 |

填好后点「保存并继续」。

**② 选择运行模式**

| 模式 | 说明 |
|---|---|
| **GUI 界面** | 角色立绘作 75% 透明背景，终端风格文字输出，可调窗口大小 |
| **CLI 命令行** | 纯控制台，适合脚本调用或低配机器 |

**③ 开始说话**

在底部输入框（GUI）或命令行（CLI）里打字，回车，角色就会**用 UTAU 声库念出来**。

---

## 💬 使用

### GUI 模式

```
 MATECH · UTAU_VChat 1.0.0                              ─  □  ✕
──────────────────────────────────────────────────────────────
         .                                  ┌──────────────┐
        ":"                                 │              │
      ___:____     |"\/"|                   │   角色立绘    │
    ,'        `.    \  /         | MATECH   │  (75% 透明)   │
    |  O        \___/  |                    │              │
  ~^~^~^~^~^~^~^~^~^~^~^~^~                 │              │
  |  MATECH · UTAU_VChat 1.0.0              │              │
  语音语言: 日本語   显示语言: 中文            │              │
  角色: 重音テト  基准音高: 63  音域: ×2.0     │              │
  语速: ×1.15  停顿: ×1.5  情绪: 开           │              │
  /voice teto|adachi 角色  /lang ja|zh 说话语言  /dlang zh|ja|auto 显示语言
  /set 设置  /help 帮助  q 退出                └──────────────┘

  你 > 你好呀
  思考中...
  重音テト > 我是重音Teto哦，这种事都不懂吗？
  (渲染完成: reply_1789291487515.wav, 35 拍 | 情绪: 平静×3)

  你 > ▊
──────────────────────────────────────────────────────────────
```

界面提示行里 `命令 可选值 中文标签` 三段式：前半段是能直接照抄的写法，
后半段是中文说明。所有调参项都收在 `/set` 里。

- 在输入框打字，**回车发送**。
- 角色回复会**显示出来**并**自动播放**语音。
- 窗口可以随意缩放，文字自动换行，立绘等比缩放不裁切。

### CLI 模式

控制台里同样的交互，提示符是 `你 > `。优点是启动快、输出可重定向。

```bash
UTAU_VChat.exe cli
```

### 说话与调参

直接输入文字就是聊天。**所有调参都收在一个 `/set` 里**——输入 `/set` 会列出
当前全部设置项、当前值、以及修改语法：

```
你 > /set
──────────────────────────────────────────────
设置　（角色: 重音テト）
──────────────────────────────────────────────
角色        重音テト  /set voice teto | adachi
说话语言    日本語    /set lang ja | zh
显示语言    中文      /set dlang zh | ja | auto
基准音高    63        /set tone 40 ~ 80
音域        ×2.0      /set pitch 0.5 ~ 4.0
语速        ×1.15     /set speed 0.5 ~ 3.0
停顿        ×1.5      /set pause 0.2 ~ 4.0
情绪语气    开        /set emo on | off
气声        30        /set breath -100 ~ 100
紧张度      -5        /set tension -100 ~ 100
辅音速度    100       /set velocity 0 ~ 150
参考音频学习  -        /set learn <音频路径>
──────────────────────────────────────────────
改法: /set <项目> <值>　　例: /set tone 60
```

改一项就写 `/set <项目> <值>`，**改完立即保存**，而且**各角色独立**
（给 Teto 调的音高不会影响 Adachi）：

```
你 > /set speed 1.3
语速 → ×1.3

你 > /voice adachi
角色 → 足立レイ

你 > 今天天气真好
思考中...
足立レイ > 嗯，这样的天气很适合出门呢。
(渲染完成: reply_1789291521883.wav, 28 拍 | 情绪: 开心×3)
```

生成的音频都放在程序目录的 `out\` 下，文件名形如 `reply_<时间戳>.wav`。

**几句实用建议**

| 想要的效果 | 怎么调 |
|---|---|
| 说话太慢、字黏在一起 | `/set speed 1.3`（数值越大越慢、字与字间隔越开） |
| 语调太平 | `/set pitch 2.5`（放大音高起伏） |
| 整体音太高/太低 | `/set tone 60`（降低基准音高） |
| 句子之间太赶 | `/set pause 2.0` |
| 不要情绪化演绎 | `/set emo off` |
| 想听中文发音 | `/set lang zh` |
| 说日语但想看懂 | `/set dlang zh` |

---

## ⌨️ 命令

界面提示行直接标出用法，形如 `命令 可选值 中文标签`：

```
/voice teto|adachi 角色  /lang ja|zh 说话语言  /dlang zh|ja|auto 显示语言  /set 设置  /help 帮助  q 退出
```

| 命令 | 作用 |
|---|---|
| `/voice teto\|adachi` | 切换角色（声库 + 立绘 + 人格 + 参数一起换） |
| `/lang ja\|zh` | 说话语言 |
| `/dlang zh\|ja\|auto` | 显示语言（可与说话语言不同） |
| `/set` | **打开设置面板**，列出全部调参项与当前值 |
| `/help` | 重新打印提示行 |
| `q` | 退出 |

其余调参项通过 `/set` 修改：

| 写法 | 作用 |
|---|---|
| `/set <项目>` | 只看该项当前值与取值范围 |
| `/set tone 40~80` | 基准音高（MIDI） |
| `/set pitch 0.5~4.0` | 音域夸张度 |
| `/set speed 0.5~3.0` | 吐字速度（>1 = 更慢、字间更宽） |
| `/set pause 0.2~4.0` | 句间停顿时长 |
| `/set emo on\|off` | 情绪语气开关 |
| `/set breath -100~100` | 气声量 |
| `/set tension -100~100` | 紧张度 |
| `/set velocity 0~150` | 辅音速度 |
| `/set learn <音频路径>` | 分析参考音频并自动调参 |

> `/voice`、`/lang`、`/dlang` 只是 `/set voice`、`/set lang`、`/set dlang`
> 的快捷写法，两者完全等价。
>
> `/set lang zh` 的中文发音仍在施工中，效果与最终版本有差异，切换到时会给出提示。

---

## 🔧 自检与诊断

安装后如果出问题，**先跑自检**——它会实际合成一句话，能直接判断是哪个环节坏了。

```bash
UTAU_VChat.exe --selftest    # 检查配置/渲染器/VOICEVOX/两个声库，并各渲染一句
UTAU_VChat.exe --diag        # 打印路径解析结果与编码环境
```

正常输出：

```
============================================================
UTAU_VChat 1.0.0 自检
============================================================
程序根目录: C:\...\UTAU_VChat
数据目录  : C:\...\UTAU_VChat
[1] config.json       OK
[2] 渲染器            OK   C:\...\UTAU_VChat\renderer\ou_render.exe
[3] VOICEVOX 语调引擎  OK
[4] 声库 teto         OK   C:\...\singers\重音テト OU用日本語統合ライブラリー
[4] 声库 adachi       OK   C:\...\singers\足立レイver3.5.0
[5] 实际渲染 teto     OK   168 KB  (重音テト)
[5] 实际渲染 adachi   OK   169 KB  (足立レイ)
============================================================
全部通过  [OK]  可以直接使用。
```

### 常见问题

| 现象 | 原因 / 处理 |
|---|---|
| 提示 `HTTP 400 ... lone leading surrogate` | 输入里混进了非法字符。1.0.0 已修，若复现请跑 `--diag` 反馈 |
| 提示 `HTTP 401` | API Key 填错或已失效，重新运行程序填写 |
| 提示 `HTTP 402 / Insufficient Balance` | DeepSeek 账户余额不足 |
| 没有声音 | 检查系统默认播放设备；音频文件在 `out\` 下，可手动播放确认 |
| 提示「没有可朗读的内容」 | 回复全是括号内容或符号，被朗读过滤器剥掉了 |
| 程序目录写入失败 | 已自动回退到 `%LOCALAPPDATA%\UTAU_VChat`，`--diag` 可看到 |

---

## 🗑 卸载

- **安装程序装的**：开始菜单 →「卸载 UTAU_VChat」，
  或「设置 → 应用 → 已安装的应用」里卸载。

  卸载会清掉**全部**相关内容，不留残留：

  | 清理项 | 说明 |
  |---|---|
  | 程序目录 | 整个安装目录，含引擎、声库、渲染缓存 |
  | `%LOCALAPPDATA%\UTAU_VChat` | 装到 Program Files 时配置与音频的存放处 |
  | `%TEMP%\utauchat_*.ico` | 运行时生成的角色图标缓存 |
  | 注册表 / 开始菜单 / 桌面快捷方式 | Windows 标准卸载流程 |

  > ⚠️ **卸载会连你的 API Key 一起删掉**（它在 `config.json` 里）。
  > 想保留的话，卸载前先备份 `%LOCALAPPDATA%\UTAU_VChat\config.json`。
  >
  > 如果你把程序装到了自定义目录（如 `D:\UTAU_VChat`），卸载会**删除该目录**，
  > 里面别放别的东西。

- **绿色版**：直接删掉 `UTAU_VChat` 文件夹，
  再把 `%LOCALAPPDATA%\UTAU_VChat`（若存在）一起删掉即可。

---

## 🧑‍💻 从源码运行（开发）

#### 前置要求

1. **Python 3.10+**（开发环境为 3.14.3）
2. **[OpenUtau](https://github.com/openutau/OpenUtau/releases)** 安装到默认路径 `C:\Program Files\OpenUtau`
3. **UTAU 声库**放入 `%USERPROFILE%\Documents\OpenUtau\Singers\`
4. **DeepSeek API Key**（[申请](https://platform.deepseek.com/)）

#### 安装依赖

```bash
pip install requests sudachipy sudachidict_core pypinyin numpy pillow voicevox_core
```

#### 配置

编辑 `config.json`：

```json
{
  "deepseek_api_key": "sk-你的密钥",
  "renderer_exe": "renderer\\ou_render.exe",
  "language": "ja",
  "display_lang": "zh"
}
```

编辑 `characters.json` 配置各角色的 `voice_folder`（声库路径）与 `portrait`（立绘路径）。

#### 运行

双击 **`run.bat`** → 选择模式，也可直接指定：`run.bat gui` / `run.bat cli`

编译渲染器（需 .NET 8 SDK）：
```bash
cd renderer && dotnet publish -c Release -r win-x64 --self-contained true
```

打包发布（安装程序 / 绿色版）见 [ARCHITECTURE.md](ARCHITECTURE.md) 第四章。

---

## ⚖️ 许可证

本项目采用 **MIT License**，详见 [LICENSE](LICENSE)。

随包分发的第三方组件（OpenUtau、.NET 运行时、NumPy、Pillow、requests、
VOICEVOX CORE、SudachiPy、Open JTalk 词典、Tcl/Tk 等）各自版权归其作者所有，
许可证原文与组件清单见 [licenses/00-索引.txt](licenses/00-索引.txt)。

声库（重音テト / 足立レイ）与 VOICEVOX 语音模型有**独立的使用条款**，
不适用本软件许可证，详见 singers/ 下各自的说明文件。

第三方组件遵循其各自许可证（见上文「开源依赖」，均为 MIT / Apache-2.0 / BSD 等宽松许可证）。

---

## ⚠️ 免责声明

**1. 软件按"原样"提供**
本软件为免费开源项目，不提供任何明示或暗示的担保，包括但不限于对适销性、
特定用途适用性及不侵权的担保。因使用本软件而产生的任何直接或间接损失，
作者不承担任何责任。

**2. 仅供学习、研究与个人同人创作**
请勿用于任何违法用途，包括但不限于：冒充他人、制作虚假音视频进行诈骗或诽谤、
侵犯他人名誉权或肖像权、规避平台审核等。

**3. 声库版权归原作者所有**
- **重音テト**：版权归其原作者及权利方所有，使用须遵守其使用条款。
- **足立レイ**：版权归 missile 39 / Mechanical Girl 所有。作者允许再配布与改造，
  但要求署名并保留原始 `log.txt` 更新记录；法人商业用途需另行联络作者。
- 本软件**不附带**上述声库，使用者需自行获取并遵守其条款。

**4. VOICEVOX 相关**
本软件调用 VOICEVOX CORE（MIT）进行语调预测。所使用的语音模型
（`0.vvm`，**四国めたん**）之著作权归 VOICEVOX 项目及各角色权利方所有，
其**利用条款独立于本软件**，商用或再分发前请自行确认并遵守。

**5. 开源组件版权归各自作者**
OpenUtau、VOICEVOX、SudachiPy、pypinyin、NumPy、Pillow、Open JTalk 等
组件的版权与许可归其各自作者所有，详见各项目仓库。

**6. DeepSeek API**
本软件需使用者自行提供 DeepSeek API 密钥。API 的**费用、额度、使用条款**
由使用者自行承担与遵守，与本软件作者无关。

**7. AI 生成内容**
角色回复与合成语音均由 AI 生成，内容仅供参考，
**不代表本软件作者的观点或立场**。使用者应对生成内容的传播与使用负责。

**8. 合规责任由使用者自负**
使用者应自行确认其使用场景（尤其是商业用途、公开发布）
是否符合相关声库、语音模型及开源组件的许可条款。

---

## 致谢

- [OpenUtau](https://github.com/openutau/OpenUtau) — 开源歌声合成引擎
- [VOICEVOX](https://voicevox.hiroshiba.jp/) — 开源语音合成引擎
- [SudachiPy](https://github.com/WorksApplications/SudachiPy) / [pypinyin](https://github.com/mozillazg/python-pinyin) — 中日文语言处理
- 重音テト、足立レイ 的原作者与所有同人创作者

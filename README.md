# UTAU_VChat

> 版本 **v1.08**　|　Windows 10/11 x64

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
| `UTAU_VChat_Setup_1.00.exe` | 673 MB | **推荐**。有安装向导、开始菜单、卸载项 |
| `UTAU_VChat_1.00_portable.zip` | 770 MB | 绿色版。免安装，拷到 U 盘就能带走 |

两个内容完全一样，**目标机器都不需要装 Python / OpenUtau / VOICEVOX / .NET**。
包内已自带：OpenUtau 合成引擎（仅功能部分，无 GUI）、Worldline 声码器、
VOICEVOX CORE 语调引擎 + 语音模型、Open JTalk 日语词典、SudachiPy、
重音テト / 足立レイ 声库，以及 Python 运行时与全部依赖。

### 方式 A：安装程序（推荐）

1. 双击 **`UTAU_VChat_Setup_1.00.exe`**。
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
UTAU_VChat_Setup_1.00.exe /CURRENTUSER
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

1. 把 `UTAU_VChat_1.00_portable.zip` 解压到**有写入权限**的目录，
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
 MATECH · UTAU_VChat v1.00                              ─  □  ✕
──────────────────────────────────────────────────────────────
         .                                  ┌──────────────┐
        ":"                                 │              │
      ___:____     |"\/"|                   │   角色立绘    │
    ,'        `.    \  /         | MATECH   │  (75% 透明)   │
    |  O        \___/  |                    │              │
  ~^~^~^~^~^~^~^~^~^~^~^~^~                 │              │
  |  MATECH · UTAU_VChat v1.00              │              │
  语音语言: 日本語   显示语言: 中文            │              │
  角色: 重音テト  基准音高: 63  音域: ×2.0     │              │
  语速: ×1.15  停顿: ×1.5  情绪: 开           │              │
  /voice teto|adachi 角色  /lang ja|zh 说话语言  /dlang zh|ja|auto 显示语言
  /set 设置  /help 帮助  q 退出                └──────────────┘

  你 > 你好呀
  思考中...
  重音テト > 我是重音Teto哦，这种事都不懂吗？
  (渲染完成: 我是重音Teto哦，这种事都不懂吗？_20260919_211530.wav, 35 拍 | 情绪: 平静×3)

  你 > ▊
──────────────────────────────────────────────────────────────
```

界面提示行里 `命令 可选值 中文标签` 三段式：前半段是能直接照抄的写法，
后半段是中文说明。所有调参项都收在 `/set` 里。

- 在输入框打字，**回车发送**。
- 角色回复会**显示出来**并**自动播放**语音。
- 窗口可以随意缩放，文字自动换行，立绘等比缩放不裁切。
- 输出变长后可以往回翻：**↑ ↓**（逐行）、**PgUp / PgDn**（翻页）、
  **鼠标滚轮**，**Ctrl+End** 回到底部。翻上去时底部会提示下方还有多少行。

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
(渲染完成: 嗯，这样的天气很适合出门呢。_20260919_211901.wav, 28 拍 | 情绪: 开心×3)
```

生成的音频都放在 `out\` 下，文件名是 **「朗读文字 + 说话时间」**，
形如 `こんにちは、テストです。_20260919_211530.wav`，一眼能认出哪句是哪句。

存放位置取决于安装方式：

| 安装方式 | `out\` 位置 |
|---|---|
| 绿色版 / 装在可写目录 | `<程序目录>\out\` |
| 安装程序装到 `C:\Program Files` | `%LOCALAPPDATA%\UTAU_VChat\out\` |

默认保留 **300 次回复 / 500 MB**，超出后自动从最旧的开始删
（用 `/set keep` 和 `/set keepmb` 调整）。

一次回复会按句拆成多个 `_pNN.wav` 文件（见「分句流水线渲染」），
它们**算作一组、整组一起删** —— 不会因为一次长回复就把保留额度用光。

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
/voice teto|adachi 角色  /lang ja|zh 说话语言  /dlang zh|ja|auto 显示语言  /set 设置  /about 关于  /help 帮助  q 退出
```

| 命令 | 作用 |
|---|---|
| `/voice teto\|adachi` | 切换角色（声库 + 立绘 + 人格 + 参数一起换） |
| `/lang ja\|zh` | 说话语言 |
| `/dlang zh\|ja\|auto` | 显示语言（可与说话语言不同） |
| `/set` | **打开设置面板**，列出全部调参项与当前值 |
| `/about` | **关于作者**：邮箱、问题反馈、版权说明、开源项目致谢 |
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
| `/set keep 10~10000` | 保留多少次回复的音频（超出删最旧的整组） |
| `/set keepmb 10~100000` | 渲染音频保留上限（MB） |
| `/set maxtok 100~8000` | 单次回复的 token 上限（默认 1500） |
| `/set font 8~24` | 界面正文字号（默认 12） |
| `/set artfont 4~20` | 启动横幅 ASCII 大字的字号（默认 7） |
| `/set capfont 8~32` | 署名行 `\| MATECH · …` 的字号（默认 16） |
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
UTAU_VChat v1.00 自检
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
| 提示 `HTTP 400 ... lone leading surrogate` | 输入里混进了非法字符。v1.00 已修，若复现请跑 `--diag` 反馈 |
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

## 📜 版本历史

> **编号规则**：两位小数 —— 小修小补 +0.01（`v1.00 → v1.01`），
> 大版本 +0.1（`v1.00 → v1.10`）。
>
> 本节**只追加、不删改**：每个版本发布时新增一段，旧版本的记录保持原样。
> 同时每个版本的说明文档会整份归档到 `D:\UTAU_VChat_pkg\releases\v<版本>\`，
> 新版不会覆盖旧版的文档。

### v1.08

- **逐字跟读显示**：以前是整段台词一次性贴出来，再开始播。现在改成
  **播到哪就显示到哪** —— 像实时字幕一样，文字比语音**提前 10 个字**冒出来，
  说完正好显示完整。读起来有「角色正在说这句话」的感觉，
  也能顺着文字判断进度。
  - 显示的是「显示语言」的文本（说日语、看中文时显示中文译文）
  - 刷新约每秒 18 次；播完会补全，不会漏字
- **系统消息精简为一行**：去掉了「渲染完成: 文件名, N 拍」那行（文件名又长
  又没用），把有用的信息并成一条：

  `
  (首句就绪 等待 2.1s ｜ 说话时长 18.3s ｜ 情绪: 傲娇×4)
  `

### v1.07

- **修复长回复被截断**：以前调用接口时把 `max_tokens` 写死成 **300**，
  长回复会被硬截断在半句话上，而且没有任何提示 —— 界面上看起来就是
  「说到一半没了」，渲染出来的音频自然也只有半句。
  现在默认放宽到 **1500**，可用 `/set maxtok 100~8000` 调整；
  万一还是撞到上限，会明确提示「回复达到长度上限被截断」，
  不会再让人莫名其妙。
- **分句流水线渲染（边渲边播）**：以前是整段渲染完才开始播，
  回复越长等得越久（20 秒音频要等 14 秒）。现在按停顿切成句子，
  **第一句渲染完就出声**，后面几句在播放的同时后台渲染，
  播放全程不断档。
  - 实测：21 秒音频，首字延迟从 **~14 秒降到 2.2 秒**
  - **音质零劣化**：切分点只在 ≥0.25 秒的停顿处，各段首尾相接与
    整段渲染**逐样本一致**（已用波形比对验证，最大差为 0）
  - 开播前先攒 2.5 秒缓冲，避免中途卡顿
  - 一次回复会生成多个 _pNN.wav；音频清理按「一次回复」为一组整组删除，
    不会因为一次长回复就把保留额度用光

### v1.06

- **字号可调，并调整了默认值**：
  - **署名行**（`| MATECH · UTAU_VChat vX.XX`）单独用 16 号字，
    比正文大一档，在横幅下面充当标题 —— `/set capfont`（8~32）
  - **横幅 ASCII 大字**单独用 7 号，不再"能放多大放多大" —— `/set artfont`（4~20）
  - **正文** 12 号 —— `/set font`（8~24）
  - 默认窗口相应加宽到 1120×760，保证命令提示行不折行
  - 三个值都存在 `config.json` 里，改完立即生效，不用重启

### v1.05

- **启动横幅换成「UTAU VChat」ASCII 大字**：原来的 DeepSeek 鲸鱼图标挪到
  `/about` 里（连同下面那行 `| MATECH · UTAU_VChat vX.XX`）。
  艺术字最宽 138 列，正文 11 号字放不下会被折行拆散，所以横幅会
  **按窗口宽度自动挑一个放得下的字号**（默认窗口下是 8 号）。

### v1.04

- **修复 GUI 下偶尔弹出的 Python 报错窗口**：渲染器（.NET）在中文 Windows 上
  按 GBK 输出，遇到日文声库路径时会写出非 UTF-8 字节，而取回输出的代码写死了
  `encoding="utf-8"` —— 解码线程当场抛 `UnicodeDecodeError`，界面上就冒出一个
  看不懂的报错窗。现在两侧都修了：渲染器固定输出 UTF-8，
  读取端也加了 `errors="replace"` 兜底。
- **GUI 模式的输出全部导向日志**：进程是 console 子系统的（CLI 模式共用同一个
  exe），控制台只能藏不能删，有未捕获异常时 Python 会往那儿打 traceback。
  现在 GUI 模式把 stdout/stderr 接到 `<数据目录>\logs\gui.log`，
  界面里只会出现软件自己的内容 —— 真出问题可以去日志里翻。
- **控制台隐藏更稳**：建窗之后再补藏一次，避免它在切前台时又冒出来。

### v1.03
- **音频文件名改为「朗读文字 + 说话时间」**：以前是 `reply_1789291487515.wav`，
  一堆文件根本认不出哪句是哪句。现在形如
  `こんにちは、テストです。_20260919_211530.wav` —— 朗读文字的开头几字接上说话时刻，
  在资源管理器里直接就能挑。会自动去掉文件名非法字符、避开 Windows 保留名、
  超长截断；整句都是符号时退化成 `voice_<时间>.wav`。

### v1.02

- **修复渲染的静默失败**：引擎缓存目录（`renderer\Cache`）不可写时 ——
  例如装在 `C:\Program Files` 而授权步骤没执行到 —— OpenUtau 会一路返回成功
  却不产出音频，用户只看到一句没头没脑的「渲染没有生成音频文件」。
  现在渲染器会提前检查并**明确报错**，直接给出原因和处理办法。
- **渲染音频保留策略**：`out\` 里的音频默认保留 **300 条 / 500 MB**，
  超出后自动从最旧的开始删，长时间挂着聊也不会把磁盘塞满。
  可用 `/set keep <条数>` 与 `/set keepmb <MB>` 调整（改小会立即清理一次）。

### v1.01

- **`/about` 关于作者**（GUI + CLI）：一行命令看到作者邮箱、
  问题反馈该附什么信息、版权说明、以及全部开源项目的致谢清单
- **GUI 输出可滚动**：`/about`、`/set` 这类长输出在窗口里放不下时，
  可以用 ↑↓ / PgUp·PgDn / 鼠标滚轮往回翻，Ctrl+End 回到底部
- **说明文件版本化**：README / 架构文档 / 项目介绍 / 必读 都带上版本号，
  并新增本节「版本历史」；每个版本的说明文档整份归档到 `D:\UTAU_VChat_pkg\releases\v<版本>\`

### v1.00

首个正式发布版。

- **让 UTAU 声库朗读对话**：DeepSeek 生成回复 → VOICEVOX 预测自然语调 →
  OpenUtau（WORLDLINE-R 声码器）合成，输出 44.1kHz 单声道 WAV
- **两个内置歌姬**：重音テト / 足立レイ，各自独立的声库、立绘、人格卡、调音参数
- **情绪语气**：自动判断回复情绪（开心/生气/悲伤/温柔/惊讶/傲娇），
  动态调整音高、音域、语速、气声、紧张度
- **中日双语**：语音语言与显示语言可分离（可以说日语、显示中文）
- **`/set` 统一设置面板**：12 项可调参数集中管理，改完立即保存，各角色独立
- **参考音频学习**：`/set learn <音频路径>` 提取声学特征并自动调参
- **参考视频说话人分离**：分析多角色素材时用声库指纹（jitter）区分说话人
- **发布形态**：中文安装程序（默认装到 `C:\Program Files\UTAU_VChat`，
  可自定义目录）+ 绿色版 zip
- **零残留卸载**：程序目录、用户数据、注册表、快捷方式、图标缓存一次清干净
- **合规**：`licenses\` 内含全部第三方组件的许可证原文与清单

### v1.0.0（开发期内部版本）

开发和打包阶段使用三段式版本号（`1.0.0`）的构建，
内容与 v1.00 基本一致，仅版本号格式不同，未作为正式版单独发布。
自 v1.00 起改用两位小数编号。

---

## 致谢

- [OpenUtau](https://github.com/openutau/OpenUtau) — 开源歌声合成引擎
- [VOICEVOX](https://voicevox.hiroshiba.jp/) — 开源语音合成引擎
- [SudachiPy](https://github.com/WorksApplications/SudachiPy) / [pypinyin](https://github.com/mozillazg/python-pinyin) — 中日文语言处理
- 重音テト、足立レイ 的原作者与所有同人创作者

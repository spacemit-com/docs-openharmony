<!--
 * Copyright 2022-2023 SPACEMIT. All rights reserved.
 * Use of this source code is governed by a BSD-style license
 * that can be found in the LICENSE file.
 * 
 * @Author: David(qiang.fu@spacemit.com)
 * @Date: 2026-03-04 11:39:35
 * @LastEditTime: 2026-07-10 10:30:00
 * @Description: TTS应用说明
-->
sidebar_position: 4

# TTS应用说明

## 修订记录

| 修订版本 | 修订日期   | 修订说明       |
|----------|------------|-------------|
| 001      | 2026-07-10 | 初始版本      |

## 概述

本地语音合成（TTS）应用基于 sherpa-onnx 推理框架，在 OpenHarmony 设备上实现完全离线的文字转语音功能。使用 Matcha-TTS 声学模型 + Vocos 声码器，支持中英文混合合成。

## 平台支持情况

|      平台 & 系统       |       是否支持     |
|-----------------------|-----------------------|
| K1 OpenHarmony5.0     | ❌ 不支持             |
| K1 OpenHarmony6.1    | ✅ 支持             |
| K3 OpenHarmony6.1     | ✅ 支持              |

## 技术栈

| 层次 | 技术 |
|------|------|
| 开发语言 | ArkTS（TypeScript 超集，OpenHarmony 专用） |
| UI 框架 | ArkUI（声明式 UI） |
| 平台 SDK | OpenHarmony SDK v12 |
| 构建工具 | Hvigor + DevEco Studio |
| 推理后端 | sherpa-onnx（通过 HAR 包提供 riscv64 native 库） |
| TTS 模型 | Matcha-TTS（matcha-icefall-zh-en，中英文） |
| 声码器 | Vocos（16kHz 通用） |
| 文本前端 | espeak-ng-data + FST 规则（日期/电话/数字） |
| 音频输出 | AudioRenderer（PCM 流式播放） |
| 多线程 | Worker（推理在独立线程执行） |

---

## 核心功能

- **中英文混合合成**：自动识别中英文并分别处理
- **流式生成**：逐句合成，边生成边播放
- **语速调节**：支持 0.5x ~ 2.0x 语速
- **多说话人**：支持 Speaker ID 切换
- **WAV 保存**：生成完成后自动保存到应用沙盒，也可手动保存
- **数字/日期/电话智能朗读**：通过 FST 规则将数字转为中文朗读形式

---

## 项目结构

```
sherpaonnxtts_spacemit/
├── AppScope/                           # 应用级资源与配置
│   ├── app.json5                       # 包名、版本号
│   └── resources/                      # 全局资源（图标等）
├── entry/                              # 主模块（HAP）
│   ├── sherpa_onnx.har                 # sherpa-onnx HAR 包（riscv64 native 库）
│   ├── src/main/ets/
│   │   ├── entryability/
│   │   │   └── EntryAbility.ets        # 应用入口
│   │   ├── pages/
│   │   │   └── Index.ets              # 主界面（TTS 控制 + 音频播放）
│   │   └── workers/
│   │       └── NonStreamingTtsWorker.ets  # TTS 推理 Worker
│   ├── src/main/resources/
│   │   └── rawfile/                    # 模型文件
│   │       ├── matcha-icefall-zh-en/   # Matcha-TTS 声学模型
│   │       │   ├── model-steps-3.onnx  # 模型文件（~75 MB）
│   │       │   ├── tokens.txt          # 词表
│   │       │   ├── lexicon.txt         # 词典
│   │       │   ├── date-zh.fst         # 日期 FST 规则
│   │       │   ├── number-zh.fst       # 数字 FST 规则
│   │       │   ├── phone-zh.fst        # 电话 FST 规则
│   │       │   └── espeak-ng-data/     # espeak-ng 前端数据（355 文件）
│   │       └── vocos-16khz-univ.onnx   # Vocos 声码器（~53 MB）
│   └── build-profile.json5            # 模块构建配置
├── build-profile.json5                 # 项目构建配置
├── oh-package.json5                    # 依赖声明
└── README.md
```

---

## 架构

```
┌───────────────────────────────────────────────────────┐
│                   OpenHarmony 设备                     │
│                                                       │
│  ┌────────────────────────────────────────────────┐   │
│  │             sherpaonnxtts HAP                  │   │
│  │                                                │   │
│  │  ┌────────────┐        ┌─────────────────┐    │   │
│  │  │ Index.ets  │        │  AudioRenderer  │    │   │
│  │  │ (UI+控制)  │ ─────→ │  (PCM 播放)    │    │   │
│  │  └─────┬──────┘        └─────────────────┘    │   │
│  │        │ postMessage                           │   │
│  │        ▼                                       │   │
│  │  ┌──────────────────────────────────────────┐  │   │
│  │  │     NonStreamingTtsWorker                │  │   │
│  │  │   ┌────────────┐  ┌──────────────────┐  │  │   │
│  │  │   │ Matcha-TTS │→ │   Vocos 声码器   │  │  │   │
│  │  │   │ (声学模型) │  │  (波形生成)      │  │  │   │
│  │  │   └────────────┘  └──────────────────┘  │  │   │
│  │  │   ┌────────────────────────────────────┐ │  │   │
│  │  │   │ espeak-ng + FST（文本前端处理）   │ │  │   │
│  │  │   └────────────────────────────────────┘ │  │   │
│  │  └──────────────────────────────────────────┘  │   │
│  └────────────────────────────────────────────────┘   │
│                                                       │
│  ┌────────────────────────────────────────────────┐   │
│  │    sherpa_onnx.har (riscv64 native libs)       │   │
│  │    libc++_shared.so | libonnxruntime.so.1      │   │
│  │    libsherpa-onnx-c-api.so | libsherpa_onnx.so│   │
│  └────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────┘
```

### 数据流

```
用户输入文本 → 点击"Start"
    ↓
Index.ets postMessage('tts-generate')
    ↓
NonStreamingTtsWorker:
    文本 → espeak-ng 音素化 → Matcha-TTS 推理 → Vocos 声码器
    ↓                                    ↓
tts-generate-partial (Float32Array)   tts-generate-done
    ↓                                    ↓
Index.ets: CircularBuffer → AudioRenderer 流式播放
                                         ↓
                                  savePcmToWav() 自动保存
```

---

## 代码获取

```bash
git clone https://gitee.com/spacemit-openharmony/sherpaonnxtts_spacemit.git
cd sherpaonnxtts_spacemit
```

---

## 模型文件获取

### Matcha-TTS 声学模型

从 GitHub Releases 下载：

```
https://github.com/k2-fsa/sherpa-onnx/releases/tag/tts-models
文件名：matcha-icefall-zh-en.tar.bz2
```

解压后将整个 `matcha-icefall-zh-en/` 目录放入 `rawfile/`。

### Vocos 声码器

```
https://github.com/k2-fsa/sherpa-onnx/releases/tag/tts-models
文件名：vocos-16khz-univ.onnx
```

放入 `entry/src/main/resources/rawfile/vocos-16khz-univ.onnx`。

### espeak-ng 数据

espeak-ng-data 目录（约 355 个文件）包含在 matcha 模型包中，解压后自动在 `matcha-icefall-zh-en/espeak-ng-data/` 下。

---

## 编译构建

### 环境要求

- **DevEco Studio** 5.x（推荐最新版）
- **OpenHarmony SDK v12**
- `entry/sherpa_onnx.har` 文件存在（约 98 MB，含 riscv64 native 库）
- 模型文件已放置到 `rawfile/` 目录

### 签名配置

使用 DevEco Studio 自动生成签名：

**File → Project Structure → Signing Configs → 勾选 "Automatically generate signature"**

### 构建命令

```bash
# 在项目根目录执行
node "<DevEco Studio 安装路径>/tools/hvigor/bin/hvigorw.js" \
  --mode module -p module=entry@default -p product=default \
  -p buildMode=release assembleHap --no-daemon
```

### 构建产物

```
entry/build/default/outputs/default/entry-default-signed.hap
```

---

## 设备部署

### 1. 设备连接确认

```bash
hdc list targets
# 预期输出设备序列号
```

### 2. TCM 设备权限（SpaceMiT 平台必须）

```bash
hdc shell "chmod 666 /dev/tcm"
```

### 3. 安装 HAP

```bash
hdc install entry-default-signed.hap
# 预期：install bundle successfully.
```

### 4. 启动应用

```bash
hdc shell "aa start -b com.k2fsa.sherpa.onnx.tts -a EntryAbility"
```

### 5. 验证初始化

```bash
# 等待模型提取和初始化（首次约 30-45 秒，后续约 10 秒）
hdc shell "hilog -x | grep -i 'JSAPP.*init-tts-done'"
# 预期输出：received msg from worker: init-tts-done
```

---

## 功能验证

### 生成语音

应用初始化完成后：
1. 在文本框中输入中文或英文（默认已有示例文本）
2. 点击 **Start** 按钮开始合成
3. 合成完成后音频自动保存到应用沙盒

```bash
# 使用 uitest 点击 Start 按钮
hdc shell "uitest uiInput click 721 263"

# 等待生成完成
hdc shell "hilog -x | grep -i 'JSAPP.*tts-generate-done'"

# 拉取生成的 WAV 文件
hdc file recv /data/app/el2/100/base/com.k2fsa.sherpa.onnx.tts/haps/entry/files/tts_output.wav tts_output.wav
```

### TTS ↔ ASR 互验

生成的 WAV 文件可直接推送给 ASR 应用进行识别验证：

```bash
# 推送给 ASR 应用
hdc file send tts_output.wav /data/app/el2/100/base/com.k2fsa.sherpa.onnx.vad.asr/haps/entry/files/test.wav

# ASR 解码验证
hdc shell "aa start -b com.k2fsa.sherpa.onnx.vad.asr -a EntryAbility"
# 等待 init 后点击 Decode
hdc shell "uitest uiInput click 960 308"
hdc shell "hilog -x | grep -i 'JSAPP.*partial result'"
```

### 验证结果（K3-Pico ITX 实测）

| 项目 | 结果 |
|------|------|
| 模型加载 | Matcha-TTS + Vocos + espeak-ng 初始化成功 |
| 中文合成 | "某某银行的副行长..." 合成清晰自然 ✅ |
| 英文合成 | "Friends fell out often..." 发音准确 ✅ |
| 数字朗读 | "2024年12月31号" → 正确朗读为中文数字 ✅ |
| 电话朗读 | "18920240522" → 逐位朗读 ✅ |
| 金额朗读 | "123456块钱" → "十二万三千四百五十六块钱" ✅ |
| 生成性能 | 默认文本（~80字）生成耗时约 10 秒，输出 24 秒音频 |
| 音频质量 | 16kHz，ASR 完整准确识别回原文 ✅ |

---

## 修改定制指导

### 更换 TTS 模型

修改 `NonStreamingTtsWorker.ets` 中模型路径配置，确保新模型文件放置在 `rawfile/` 对应目录下。

### 调整语速

应用 UI 提供语速输入框（默认 1.0），也可在代码中修改 `Index.ets` 的 `speechSpeed` 默认值。

### 切换说话人

修改 `sid` 参数（默认 0）。具体可用的 Speaker ID 范围取决于模型。

---

## 已知限制

| 限制 | 说明 | 解决方案 |
|------|------|----------|
| 音频播放延迟 | K3 设备 AudioRenderer 初始化约 10 秒 | 应用已采用懒加载，推理不受影响 |
| 首次启动慢 | 模型文件需从 rawfile 提取到沙盒（~130 MB） | 后续启动跳过提取 |

---

## FAQ

**Q: 应用启动后显示 "Extracting..." 很长时间？**

首次启动需要将 rawfile 中的模型文件（~130 MB）提取到应用沙盒，约需 30-45 秒。后续启动会跳过已提取的文件。

**Q: 点击 Start 后没有声音？**

K3 设备的 AudioRenderer 初始化较慢，首次播放可能有约 10 秒延迟。TTS 推理本身正常完成，音频文件已自动保存到沙盒，可通过 hdc 拉取验证。

**Q: 如何获取生成的音频文件？**

```bash
hdc file recv /data/app/el2/100/base/com.k2fsa.sherpa.onnx.tts/haps/entry/files/tts_output.wav tts_output.wav
```

**Q: sherpa_onnx.har 从哪里获取？**

HAR 包与 ASR 项目共用同一个，含相同的 riscv64 native 库（sherpa-onnx + onnxruntime）。

**Q: 支持哪些语言？**

Matcha-icefall-zh-en 模型支持中文和英文混合合成。输入文本中的中英文会自动识别和切换。

---
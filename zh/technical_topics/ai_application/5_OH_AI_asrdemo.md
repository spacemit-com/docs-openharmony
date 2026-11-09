<!--
 * Copyright 2022-2023 SPACEMIT. All rights reserved.
 * Use of this source code is governed by a BSD-style license
 * that can be found in the LICENSE file.
 * 
 * @Author: David(qiang.fu@spacemit.com)
 * @Date: 2026-03-04 11:39:35
 * @LastEditTime: 2026-11-07 10:00:00
 * @Description: ASR应用说明
-->
sidebar_position: 5

# ASR应用说明

## 修订记录

| 修订版本 | 修订日期   | 修订说明 |
|----------|------------|----------|
| 001      | 2026-07-10 | 初始版本 |
| 002      | 2026-11-07 | 精简部署流程并依据 K3 平台结果校正：模型随 HAP 打包，更新模型大小、耗时与文件选择器结论 |

## 概述

本地语音识别（ASR）应用基于 sherpa-onnx，在 OpenHarmony 设备上实现完全离线的语音转文字。默认使用 SenseVoice 多语言模型 + Silero VAD，支持中文、英文、粤语、日语、韩语，可用于麦克风实时识别和 WAV 文件解码。

推理库与模型都随 HAP 分发，安装即用，无需向设备推送模型；只有做端到端验证时，才需要通过 hdc 传一个 16 kHz 测试音频到应用沙箱。

## 平台支持情况

|      平台 & 系统       |       是否支持     |
|-----------------------|-----------------------|
| K1 OpenHarmony5.0     | ❌ 不支持             |
| K1 OpenHarmony6.1    | ✅ 支持             |
| K3 OpenHarmony6.1    | ✅ 支持              |

## 技术栈

| 层次 | 技术 |
|------|------|
| 开发语言 | ArkTS |
| UI 框架 | ArkUI（声明式 UI） |
| 平台 SDK | OpenHarmony SDK v12 |
| 推理后端 | sherpa-onnx（`sherpa_onnx.har`，含 riscv64 native 库） |
| ASR 模型 | SenseVoice int8（中/英/粤/日/韩） |
| VAD 模型 | Silero VAD（端点检测） |
| 多线程 | Worker（推理在独立线程执行） |

---

## 核心功能

- **麦克风实时识别**：VAD 检测语音段，自动触发识别
- **文件解码**：选择 16 kHz WAV 文件离线识别
- **沙盒文件解码**：通过 hdc 放入沙箱后一键解码，用于快速验证
- **多语言**：中文、英文、粤语、日语、韩语

---

## 项目结构

```
sherpaonnxvadasr_spacemit/
├── AppScope/                           # 应用级资源与配置
├── entry/
│   ├── sherpa_onnx.har                 # riscv64 native 库（与 TTS 相同）
│   ├── src/main/ets/
│   │   ├── entryability/EntryAbility.ets
│   │   ├── pages/
│   │   │   ├── Index.ets               # Tab：From file / From mic / Help
│   │   │   ├── NonStreamingAsrModels.ets  # 模型清单
│   │   │   └── Permission.ets          # 麦克风权限
│   │   └── workers/
│   │       └── NonStreamingAsrWithVadWorker.ets
│   └── src/main/resources/rawfile/
│       ├── sherpa-onnx-sense-voice-.../
│       │   ├── model.int8.onnx         # 约 228 MiB（需自行下载放入）
│       │   └── tokens.txt              # 已随仓库提供
│       └── silero_vad.onnx             # 已随仓库提供
├── build-profile.json5
└── oh-package.json5
```

---

## 架构

```
┌───────────────────────────────────────────────────────┐
│                    OpenHarmony 设备                    │
│                                                       │
│  ┌────────────────────────────────────────────────┐   │
│  │            sherpaonnxvadasr HAP                │   │
│  │  ┌─────────────┐        ┌──────────────────┐   │   │
│  │  │ EntryAbility│        │   Index.ets      │   │   │
│  │  │  生命周期   │        │  UI + 控制        │   │   │
│  │  └─────────────┘        └────────┬─────────┘   │   │
│  │                                  │ postMessage │   │
│  │                                  ▼             │   │
│  │  ┌──────────────────────────────────────────┐  │   │
│  │  │ NonStreamingAsrWithVadWorker             │  │   │
│  │  │ Silero VAD → SenseVoice int8             │  │   │
│  │  └──────────────────────────────────────────┘  │   │
│  └────────────────────────────────────────────────┘   │
│                                                       │
│  sherpa_onnx.har：libc++_shared / libonnxruntime /     │
│  libsherpa-onnx-c-api / libsherpa_onnx（随 HAP 打包）   │
└───────────────────────────────────────────────────────┘
```

---

## 代码获取

```bash
git clone https://gitee.com/spacemit-openharmony/sherpaonnxvadasr_spacemit.git
cd sherpaonnxvadasr_spacemit
```

---

## 模型准备（打包进 HAP）

仓库已包含 `tokens.txt`、`silero_vad.onnx`；SenseVoice 的 `model.int8.onnx` 体积较大，需自行下载：

| 文件 | 来源 | 大小 |
|------|------|------|
| `model.int8.onnx` | [sherpa-onnx asr-models](https://github.com/k2-fsa/sherpa-onnx/releases/tag/asr-models) 的 `sherpa-onnx-sense-voice-zh-en-ja-ko-yue-2024-07-17.tar.bz2` | 239,233,841 字节（约 228 MiB） |

解压后把 `model.int8.onnx` 放入 `rawfile/sherpa-onnx-sense-voice-zh-en-ja-ko-yue-2024-07-17/`，与已有的 `tokens.txt` 同级。模型随 HAP 打包，安装后直接从 `rawfile` 读取，不需要推送。

> 默认模型类型为 `15`（SenseVoice）；`NonStreamingAsrModels.ets` 还预置了 Whisper、Zipformer、Paraformer 等配置，切换时需把对应模型目录放入 `rawfile` 并修改 `NonStreamingAsrWithVadWorker.ets` 中的 `type`。

---

## 编译构建

### 环境要求

- DevEco Studio 5.x
- OpenHarmony SDK v12
- `entry/sherpa_onnx.har`（可从 TTS 项目复制，两者相同）
- 模型已放入 `rawfile/`

### 签名与构建

```bash
node "<DevEco Studio 安装路径>/tools/hvigor/bin/hvigorw.js" \
  --mode module -p module=entry@default -p product=default \
  -p buildMode=release assembleHap --no-daemon
```

产物：`entry/build/default/outputs/default/entry-default-signed.hap`。

---

## 设备部署

```bash
# 1. 安装
hdc install entry-default-signed.hap

# 2. 启动（首次加载模型约 8 秒）
hdc shell "aa start -b com.k2fsa.sherpa.onnx.vad.asr -a EntryAbility"

# 3. 查看初始化日志
hdc shell "hilog -x | grep -iE 'init.*asr|init.*vad|done'"
```

若镜像未放开 TCM 权限（一般已由 init 脚本配置），执行一次：

```bash
hdc shell "chmod 666 /dev/tcm"
```

首次启动时系统会请求麦克风权限，授权后可实时识别。

---

## 功能验证

### 沙盒文件解码

```bash
# 推送 16 kHz 单声道 WAV 到应用沙箱
hdc file send test.wav /data/app/el2/100/base/com.k2fsa.sherpa.onnx.vad.asr/haps/entry/files/test.wav

# 启动应用后点击「Decode test file (sandbox)」
hdc shell "uitest uiInput click 960 308"

# 查看识别结果
hdc shell "hilog -x | grep -i 'JSAPP.*partial result'"
```

验证结果（K3-Pico ITX，OpenHarmony 6.1.0.35 / API 23）：

| 项目 | 结果 |
|------|------|
| 模型加载 | SenseVoice int8 + Silero VAD 初始化成功，约 8 秒 |
| 中文识别 | 0.67 秒语音识别为「我在」✅ |
| 中英文混合 | 对 TTS 生成的 24.12 秒音频完整还原中英文 ✅ |
| 数字/电话/金额 | `2024年12月31号`、`110`、`18920240522`、`123456块钱` 均正确转写 ✅ |
| VAD 分段 | 正确检测 0.04–0.64、0.10–21.37、21.54–24.10 语音段 ✅ |
| 推理性能 | 24.12 秒音频解码约 4.7 秒 |

### TTS ↔ ASR 互验

TTS 生成 `tts_output.wav` 后拉取并推送到 ASR 沙箱，按上面的步骤解码；可完整识别 TTS 输出的中英文、数字与金额。

---

## 已知限制

| 限制 | 说明 | 应对 |
|------|------|------|
| 仅支持 16 kHz | 模型要求 16 kHz 单声道 WAV | 先转换采样率 |

> 文件选择器在 OpenHarmony 6.1.0.35 上可正常打开（早期镜像曾出现 IPC 超时）；「Decode test file (sandbox)」仍保留，便于脚本化、批量验证。

---

## FAQ

**Q: 启动后一直显示 "Initializing..."？**

SenseVoice 模型约 228 MiB，K3 上首次加载约 8 秒。查看日志：

```bash
hdc shell "hilog -x | grep -iE 'JSAPP.*init'"
```

**Q: 点击「Select .wav file」没有反应？**

在 OpenHarmony 6.1.0.35 上，文件选择器可正常打开。若做自动化或批量验证，建议仍使用「Decode test file (sandbox)」：把 WAV 放入应用沙盒后一键解码，无需操作文件选择器。

**Q: 如何更换 ASR 模型？**

在 `NonStreamingAsrWithVadWorker.ets` 中修改 `type`（对应 `NonStreamingAsrModels.ets` 的配置），并把模型目录放入 `rawfile/`；注意采样率需为 16 kHz。

**Q: `sherpa_onnx.har` 从哪里获取？**

可从 TTS 项目（`sherpaonnxtts_spacemit/entry/sherpa_onnx.har`）复制，两者使用相同的 riscv64 native 库。

**Q: 支持哪些语言？**

SenseVoice 模型支持中文、英文、粤语、日语、韩语；中英文混合语句也能识别。

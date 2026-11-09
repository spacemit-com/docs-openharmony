<!--
 * Copyright 2022-2023 SPACEMIT. All rights reserved.
 * Use of this source code is governed by a BSD-style license
 * that can be found in the LICENSE file.
 * 
 * @Author: David(qiang.fu@spacemit.com)
 * @Date: 2026-03-04 11:39:35
 * @LastEditTime: 2026-11-07 10:00:00
 * @Description: TTS应用说明
-->
sidebar_position: 4

# TTS应用说明

## 修订记录

| 修订版本 | 修订日期   | 修订说明 |
|----------|------------|----------|
| 001      | 2026-07-10 | 初始版本 |
| 002      | 2026-11-07 | 精简部署流程并依据 K3 平台结果校正：模型随 HAP 打包，修正 Vocos 下载地址与耗时 |

## 概述

本地语音合成（TTS）应用基于 sherpa-onnx，在 OpenHarmony 设备上实现完全离线的文字转语音。使用 Matcha-TTS 声学模型 + Vocos 声码器，支持中英文混合合成。

模型与推理库都随 HAP 一起分发：安装后首次启动时，应用把 `rawfile` 中的模型解压到沙箱，之后直接复用，不需要通过 hdc 向设备推送任何文件。

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
| TTS 模型 | Matcha-TTS（matcha-icefall-zh-en，中英文） |
| 声码器 | Vocos（16 kHz） |
| 文本前端 | espeak-ng-data + FST 规则（日期/电话/数字） |
| 音频输出 | AudioRenderer（PCM 流式播放） |
| 多线程 | Worker（推理在独立线程执行） |

---

## 核心功能

- **中英文混合合成**：自动识别中英文并分别处理
- **流式生成**：逐句合成，边生成边播放
- **语速调节**：0.5x ~ 2.0x
- **多说话人**：可切换 Speaker ID
- **WAV 保存**：生成完成后自动保存到沙箱，也可通过按钮另存
- **数字/日期/电话朗读**：通过 FST 规则转换为中文口语形式

---

## 项目结构

```
sherpaonnxtts_spacemit/
├── AppScope/                           # 应用级资源与配置
├── entry/
│   ├── sherpa_onnx.har                 # riscv64 native 库（约 95 MB）
│   ├── src/main/ets/
│   │   ├── entryability/EntryAbility.ets
│   │   ├── pages/Index.ets             # TTS 控制 + 播放
│   │   └── workers/
│   │       └── NonStreamingTtsWorker.ets   # TTS 推理 Worker
│   ├── src/main/resources/rawfile/     # 模型（需自行下载后放入）
│   │   ├── matcha-icefall-zh-en/
│   │   │   ├── model-steps-3.onnx      # 约 76 MB
│   │   │   ├── tokens.txt / lexicon.txt
│   │   │   ├── date-zh.fst / phone-zh.fst / number-zh.fst
│   │   │   └── espeak-ng-data/         # 约 355 个文件
│   │   └── vocos-16khz-univ.onnx       # 约 54 MB
│   └── build-profile.json5
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
│  │              sherpaonnxtts HAP                 │   │
│  │  ┌────────────┐       ┌──────────────────┐     │   │
│  │  │ Index.ets  │──────▶│  AudioRenderer   │     │   │
│  │  │ UI + 控制  │       │  PCM 流式播放     │     │   │
│  │  └─────┬──────┘       └──────────────────┘     │   │
│  │        │ postMessage                          │   │
│  │        ▼                                      │   │
│  │  ┌──────────────────────────────────────────┐  │   │
│  │  │ NonStreamingTtsWorker                    │  │   │
│  │  │ espeak-ng → Matcha-TTS → Vocos           │  │   │
│  │  └──────────────────────────────────────────┘  │   │
│  └────────────────────────────────────────────────┘   │
│                                                       │
│  sherpa_onnx.har：libc++_shared / libonnxruntime /     │
│  libsherpa-onnx-c-api / libsherpa_onnx（随 HAP 打包）   │
└───────────────────────────────────────────────────────┘
```

### 数据流

```
输入文本 → 点击 Start
    ↓
Index.ets postMessage('tts-generate')
    ↓
Worker：文本 → espeak-ng 音素化 → Matcha-TTS → Vocos
    ↓                              ↓
tts-generate-partial (Float32Array)  tts-generate-done
    ↓                              ↓
AudioRenderer 流式播放          自动保存 tts_output.wav
```

---

## 代码获取

```bash
git clone https://gitee.com/spacemit-openharmony/sherpaonnxtts_spacemit.git
cd sherpaonnxtts_spacemit
```

---

## 模型准备（打包进 HAP）

仓库不包含模型文件，编译前需下载并放入 `entry/src/main/resources/rawfile/`：

| 文件 | 来源 | 大小 |
|------|------|------|
| `matcha-icefall-zh-en/`（含 espeak-ng-data） | [sherpa-onnx tts-models](https://github.com/k2-fsa/sherpa-onnx/releases/tag/tts-models) 的 `matcha-icefall-zh-en.tar.bz2` | 模型 75,717,082 字节（约 76 MB），压缩包约 79 MB |
| `vocos-16khz-univ.onnx` | [sherpa-onnx vocoder-models](https://github.com/k2-fsa/sherpa-onnx/releases/tag/vocoder-models) | 53,882,848 字节（约 54 MB） |

> Vocos 不在 `tts-models` 页面，直链为 `https://github.com/k2-fsa/sherpa-onnx/releases/download/vocoder-models/vocos-16khz-univ.onnx`。

解压后将整个 `matcha-icefall-zh-en/` 目录和 `vocos-16khz-univ.onnx` 放入 `rawfile/`。模型随 HAP 打包，安装后首次启动自动解压到应用沙箱（合计约 144 MB）：

```
/data/app/el2/100/base/com.k2fsa.sherpa.onnx.tts/haps/entry/files/
├── matcha-icefall-zh-en/
└── vocos-16khz-univ.onnx
```

---

## 编译构建

### 环境要求

- DevEco Studio 5.x
- OpenHarmony SDK v12
- `entry/sherpa_onnx.har` 存在（约 95 MB，含 riscv64 native 库）
- 模型已放入 `rawfile/`

### 签名与构建

在 DevEco Studio 中配置自动签名后执行：

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

# 2. 启动（首次启动解压约 144 MB，通常 <1 秒；模型初始化约 8 秒）
hdc shell "aa start -b com.k2fsa.sherpa.onnx.tts -a EntryAbility"

# 3. 查看初始化日志
hdc shell "hilog -x | grep -iE 'TTS|init-tts|extracted'"
```

若镜像未放开 TCM 权限（一般已由 init 脚本配置），执行一次：

```bash
hdc shell "chmod 666 /dev/tcm"
```

---

## 功能验证

1. 等待界面显示模型初始化完成。
2. 输入中英文文本，设置 Speaker ID 与语速，点击 **Start**。
3. 合成完成后音频自动保存到沙箱，可拉取验证：

```bash
hdc file recv /data/app/el2/100/base/com.k2fsa.sherpa.onnx.tts/haps/entry/files/tts_output.wav tts_output.wav
```

验证结果（K3-Pico ITX，OpenHarmony 6.1.0.35 / API 23）：

| 项目 | 结果 |
|------|------|
| 模型加载 | Matcha-TTS + Vocos + espeak-ng 初始化成功，约 8 秒 |
| 中文合成 | 清晰自然 ✅ |
| 英文合成 | 发音准确 ✅ |
| 数字/电话/金额 | 按中文口语规则正确朗读 ✅ |
| 生成性能 | 默认文本生成 24.12 秒音频（385,918 采样点），点击到完成约 5 秒（推理约 2.6 秒） |
| 音频质量 | 16 kHz，ASR 完整识别回原文（中英文/数字/电话/金额均正确）✅ |

---

## 修改定制指导

### 更换 TTS 模型

修改 `NonStreamingTtsWorker.ets` 中的模型路径与文件名（`modelDir`、`acousticModelName`、`vocoder`、`ruleFsts`），把新模型放入 `rawfile/` 对应目录。

### 调整默认语速与说话人

界面提供语速输入框（默认 1.0）与 Speaker ID（默认 0）；也可修改 `Index.ets` 中 `speechSpeed`、`sid` 的初始值。

### 更换采样率

模型输出的采样率由声码器决定（Vocos 16 kHz）。更换声码器后需同步确认 `AudioRenderer` 的采样率设置。

---

## 已知限制

| 限制 | 说明 | 应对 |
|------|------|------|
| 音频播放延迟 | K3 上 `AudioRenderer` 初始化较慢（约 10 秒，视音频驱动而定） | 已改为懒初始化；合成与文件保存不受影响 |
| 首次启动慢 | 需把约 144 MB 模型从 rawfile 解压到沙箱；解压通常 <1 秒、模型初始化约 8 秒 | 后续启动跳过解压 |
| 仅 16 kHz | 当前声码器固定 16 kHz | 更换模型时同步调整音频参数 |

---

## FAQ

**Q: 启动后长时间显示 "Extracting..."？**

首次启动需要将 rawfile 中的模型文件（约 144 MB）解压到应用沙盒：解压通常 <1 秒，随后模型初始化约 8 秒。后续启动会跳过已解压的文件。

**Q: 点击 Start 后没有声音？**

K3 的 AudioRenderer 初始化较慢，首次播放可能延迟约 10 秒。TTS 推理与文件保存正常，可先拉取 `tts_output.wav` 验证。

**Q: 如何获取生成的音频？**

```bash
hdc file recv /data/app/el2/100/base/com.k2fsa.sherpa.onnx.tts/haps/entry/files/tts_output.wav tts_output.wav
```

**Q: `sherpa_onnx.har` 从哪里获取？**

仓库 `entry/` 下已提供；也可从 ASR 项目复制，两者使用相同的 HAR。

**Q: 支持哪些语言？**

Matcha-icefall-zh-en 支持中文和英文混合合成，输入中的中英文会自动识别与切换。

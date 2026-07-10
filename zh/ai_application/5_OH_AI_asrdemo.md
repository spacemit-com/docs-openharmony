<!--
 * Copyright 2022-2023 SPACEMIT. All rights reserved.
 * Use of this source code is governed by a BSD-style license
 * that can be found in the LICENSE file.
 * 
 * @Author: David(qiang.fu@spacemit.com)
 * @Date: 2026-03-04 11:39:35
 * @LastEditTime: 2026-07-10 10:30:00
 * @Description: ASR应用说明
-->
sidebar_position: 5

# ASR应用说明

## 修订记录

| 修订版本 | 修订日期   | 修订说明       |
|----------|------------|-------------|
| 001      | 2026-07-10 | 初始版本      |

## 概述

本地语音识别（ASR）应用基于 sherpa-onnx 推理框架，在 OpenHarmony 设备上实现完全离线的语音转文字功能。支持中文、英文、粤语、日语、韩语五种语言的实时识别和文件解码。

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
| ASR 模型 | SenseVoice（中/英/粤/日/韩多语言） |
| VAD 模型 | Silero VAD（端点检测） |
| 多线程 | Worker（推理在独立线程执行） |

---

## 核心功能

- **麦克风实时识别**：通过 VAD 检测语音段，自动触发 ASR 推理
- **文件解码**：选择本地 WAV 文件（16kHz）进行离线识别
- **沙盒文件解码**：通过 hdc 推送文件到应用沙盒，一键解码验证
- **多语言支持**：中文、英文、粤语、日语、韩语

---

## 项目结构

```
sherpaonnxvadasr_spacemit/
├── AppScope/                           # 应用级资源与配置
│   ├── app.json5                       # 包名、版本号
│   └── resources/                      # 全局资源（图标等）
├── entry/                              # 主模块（HAP）
│   ├── sherpa_onnx.har                 # sherpa-onnx HAR 包（riscv64 native 库）
│   ├── src/main/ets/
│   │   ├── entryability/
│   │   │   └── EntryAbility.ets        # 应用入口
│   │   ├── pages/
│   │   │   └── Index.ets              # 主界面（Tab 页：From file / From mic / Help）
│   │   └── workers/
│   │       └── NonStreamingAsrWithVadWorker.ets  # ASR+VAD 推理 Worker
│   ├── src/main/resources/
│   │   └── rawfile/                    # 模型文件
│   │       ├── sherpa-onnx-sense-voice-zh-en-ja-ko-yue-2024-07-17/
│   │       │   ├── model.int8.onnx    # SenseVoice 模型（~239 MB）
│   │       │   └── tokens.txt          # 词表
│   │       └── silero_vad.onnx         # Silero VAD 模型
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
│  │            sherpaonnxvadasr HAP                 │   │
│  │                                                │   │
│  │  ┌─────────────┐       ┌─────────────────┐    │   │
│  │  │ EntryAbility │       │   Index.ets     │    │   │
│  │  │  (生命周期)  │       │  (UI + 控制)   │    │   │
│  │  └──────────────┘       └───────┬─────────┘    │   │
│  │                                 │ postMessage  │   │
│  │                                 ▼              │   │
│  │  ┌──────────────────────────────────────────┐  │   │
│  │  │   NonStreamingAsrWithVadWorker           │  │   │
│  │  │   ┌──────────┐    ┌──────────────────┐  │  │   │
│  │  │   │ Silero   │    │   SenseVoice     │  │  │   │
│  │  │   │   VAD    │ →  │  ASR (int8)      │  │  │   │
│  │  │   └──────────┘    └──────────────────┘  │  │   │
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

---

## 代码获取

```bash
git clone https://gitee.com/spacemit-openharmony/sherpaonnxvadasr_spacemit.git
cd sherpaonnxvadasr_spacemit
```

---

## 模型文件获取

### SenseVoice ASR 模型

从 GitHub Releases 下载：

```
https://github.com/k2-fsa/sherpa-onnx/releases/tag/asr-models
文件名：sherpa-onnx-sense-voice-zh-en-ja-ko-yue-2024-07-17.tar.bz2
```

解压命令（Windows，需 7-Zip）：

```powershell
& "C:\Program Files\7-Zip\7z.exe" x sherpa-onnx-sense-voice-zh-en-ja-ko-yue-2024-07-17.tar.bz2
& "C:\Program Files\7-Zip\7z.exe" x sherpa-onnx-sense-voice-zh-en-ja-ko-yue-2024-07-17.tar
```

将 `model.int8.onnx` 和 `tokens.txt` 放入：
```
entry/src/main/resources/rawfile/sherpa-onnx-sense-voice-zh-en-ja-ko-yue-2024-07-17/
```

### Silero VAD 模型

`silero_vad.onnx` 已包含在仓库的 `rawfile/` 目录中。

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
hdc shell "aa start -b com.k2fsa.sherpa.onnx.vad.asr -a EntryAbility"
```

### 5. 验证初始化

```bash
# 等待 10-15 秒模型加载，查看日志
hdc shell "hilog -x | grep -iE 'JSAPP.*(init.*done)'"
# 预期输出：
# init vad done
# init non streaming ASR done
```

---

## 功能验证

### 沙盒文件解码验证

应用支持通过 hdc 推送 WAV 文件到应用沙盒，配合"Decode test file (sandbox)"按钮进行快速验证：

```bash
# 1. 推送测试 WAV 文件到应用沙盒（必须 16kHz）
hdc file send test.wav /data/app/el2/100/base/com.k2fsa.sherpa.onnx.vad.asr/haps/entry/files/test.wav

# 2. 启动应用，等待初始化完成

# 3. 点击 "Decode test file (sandbox)" 按钮
hdc shell "uitest uiInput click 960 308"

# 4. 查看识别结果
hdc shell "hilog -x | grep -i 'JSAPP.*partial result'"
```

### TTS ↔ ASR 互验

可使用 TTS 应用生成语音，然后推送给 ASR 识别，验证两个应用端到端正确性：

```bash
# TTS 生成后自动保存到沙盒
hdc file recv /data/app/el2/100/base/com.k2fsa.sherpa.onnx.tts/haps/entry/files/tts_output.wav tts_output.wav

# 推送给 ASR
hdc file send tts_output.wav /data/app/el2/100/base/com.k2fsa.sherpa.onnx.vad.asr/haps/entry/files/test.wav

# ASR 解码
hdc shell "uitest uiInput click 960 308"
hdc shell "hilog -x | grep -i 'JSAPP.*partial result'"
```

### 验证结果（K3-Pico ITX 实测）

| 项目 | 结果 |
|------|------|
| 模型加载 | SenseVoice int8 + Silero VAD 初始化成功 |
| 英文识别 | `the tribal chieftain called for the boy and presented him with fifty pieces of gold` ✅ |
| 中英文混合 | 某某银行...Friends fell out often...经济不断增长... ✅ |
| 数字/电话 | 2024年12月31号 → 二零二四年十二月三十一号 ✅ |
| VAD 分段 | 正确检测语音段起止时间 ✅ |
| 推理性能 | 24 秒音频解码耗时约 4 秒 |

---

## 已知限制

| 限制 | 说明 | 解决方案 |
|------|------|----------|
| 仅支持 16kHz | 模型要求 16kHz 单声道 WAV | 转换音频采样率后使用 |

---

## FAQ

**Q: 应用启动后一直显示 "Initializing..."？**

模型较大（~239 MB），在 K3 设备上首次加载约需 10-15 秒。查看日志确认：
```bash
hdc shell "hilog -x | grep -i 'JSAPP.*init'"
```

**Q: 如何验证文件解码功能？**

通过 hdc 推送 16kHz WAV 文件到应用沙盒，然后点击 "Decode test file (sandbox)" 按钮即可触发解码。详见上文「功能验证」章节。

**Q: 如何更换 ASR 模型？**

修改 `NonStreamingAsrWithVadWorker.ets` 中模型配置路径，确保新模型也放置在 `rawfile/` 对应目录下，采样率匹配 16kHz。

**Q: sherpa_onnx.har 从哪里获取？**

HAR 包可从 TTS 项目（`sherpaonnxtts_spacemit/entry/sherpa_onnx.har`）复制，两者使用相同的 HAR 包（含相同 riscv64 native 库）。

---
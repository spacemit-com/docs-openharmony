<!--
 * Copyright 2022-2023 SPACEMIT. All rights reserved.
 * Use of this source code is governed by a BSD-style license
 * that can be found in the LICENSE file.
 * 
 * @Author: David(qiang.fu@spacemit.com)
 * @Date: 2026-03-04 11:39:35
 * @LastEditTime: 2026-11-07 10:00:00
 * @FilePath: \doc\docs-openharmony\zh\ai_application\1_OH_AI_yolo.md
 * @Description: 
-->
sidebar_position: 1

# AI检测识别应用说明

## 修订记录

| 修订版本 | 修订日期   | 修订说明 |
|----------|------------|----------|
| 001      | 2026-04-27 | 初始版本 |
| 002      | 2026-11-07 | 对齐最新代码并依据 K3 平台结果校正：推理库复用系统镜像，推理移入 TaskPool，更新 EP/TCM 版本 |

## 概述

YOLODetection 是基于 OpenHarmony 的 AI 视觉检测应用，提供目标检测、人脸检测、人脸识别三个功能，使用 ONNX Runtime + SpaceMIT Execution Provider 在 RISC-V 设备上推理。

HAP 内只包含自己的 NAPI 模块 `libyolo_detect.so` 及 C++ 运行时（当前 HAP 为 `libc++_shared.so`），ONNX Runtime、SpaceMIT EP、TCM 支持库全部复用设备镜像中的 `/system/lib64`；模型随 HAP 打包在 `rawfile`，首次运行复制到应用沙箱后复用。推理在 TaskPool 线程中执行，不占用 UI 线程。

## 平台支持情况

|      平台 & 系统       |       是否支持     |
|-----------------------|-----------------------|
| K1 OpenHarmony5.0     | ✅ 支持               |
| K1 OpenHarmony6.1    | ✅ 支持             |
| K3 OpenHarmony6.1     | ✅ 支持              |

## 技术栈

| 层次 | 技术 |
|------|------|
| 操作系统 | OpenHarmony API 12 |
| UI 框架 | ArkUI (ArkTS / ETS) |
| 原生层语言 | C++17 |
| JS/Native 桥接 | NAPI (Node API) |
| 推理引擎 | ONNX Runtime 1.24.2（镜像提供） |
| 硬件加速 | SpaceMIT EP 2.0.7 + TCM 3.0.2（镜像提供） |
| 目标架构 | RISC-V 64-bit (riscv64) |
| 构建工具 | Hvigor + CMake 3.5+ |
| 模型格式 | ONNX (量化 int8) |
| 检测模型 | YOLOv5-nano (量化) |
| 人脸识别模型 | ArcFace (量化) |

---

## 项目架构

```
┌─────────────────────────────────────────────────────┐
│                ArkUI 页面层 (ETS)                     │
│  HomePage | DetectPage | FaceDetectPage |            │
│  FaceRecognizePage                                   │
└───────────────────┬─────────────────────────────────┘
                    │ taskpool.Task
┌───────────────────▼─────────────────────────────────┐
│           DetectTasks.ets（@Concurrent 任务）         │
└───────────────────┬─────────────────────────────────┘
                    │ NAPI
┌───────────────────▼─────────────────────────────────┐
│        libyolo_detect.so（随 HAP 打包）               │
│  detectRGBA | detectFace | recognizeFace             │
│  letterboxCHW | releaseModel                         │
└───────────────────┬─────────────────────────────────┘
                    │ 运行时从 /system/lib64 解析
┌───────────────────▼─────────────────────────────────┐
│  libonnxruntime.so.1 + libspacemit_ep.so.2           │
│  libspine_tcm.so.3（系统镜像提供，不随 HAP 打包）     │
└─────────────────────────────────────────────────────┘
```

### 模块说明

- **ArkUI 页面层**：加载图片、预处理、结果可视化。
- **TaskPool 任务层**：`DetectTasks.ets` 以 `@Concurrent` 任务调用原生接口，避免同步推理阻塞 UI 线程。
- **NAPI 层**：`yolo_detect.cpp` 负责模型加载、前后处理与 NMS，同一时刻只保留一个 ORT 会话。
- **推理引擎层**：ONNX Runtime 与 SpaceMIT EP 由系统镜像提供，HAP 不包含这两个库。

---

## 目录结构

```
yolo_detection/
├── AppScope/                        # 应用级资源
├── entry/
│   ├── libs/riscv64/                # 构建输出，按 ABI 打包
│   │   ├── libyolo_detect.so        # 本项目 NAPI 产物
│   │   └── libc++_shared.so         # C++ 运行时（随 HAP 打包）
│   ├── link_libs/riscv64/           # 仅链接期使用，不打包
│   │   ├── libonnxruntime.so(.1)
│   │   ├── libspacemit_ep.so(.2)
│   │   └── libspine_tcm.so
│   └── src/main/
│       ├── cpp/
│       │   ├── CMakeLists.txt
│       │   ├── yolo_detect.cpp      # 推理与后处理
│       │   ├── onnxruntime_*.h      # ORT 头文件
│       │   ├── spacemit_ort_env.h
│       │   └── types/libyolo_detect/
│       │       └── index.d.ts       # TS 类型声明
│       ├── ets/
│       │   ├── detect/
│       │   │   ├── CocoClasses.ets  # 80 个 COCO 类别
│       │   │   └── DetectTasks.ets  # TaskPool 并发任务
│       │   ├── pages/
│       │   │   ├── HomePage.ets
│       │   │   ├── DetectPage.ets
│       │   │   ├── FaceDetectPage.ets
│       │   │   └── FaceRecognizePage.ets
│       │   └── entryability/
│       └── resources/rawfile/       # 模型与测试图片
│           ├── yolov5nq.onnx        # 目标检测 (2.6 MB)
│           ├── yolov5nfaceq.onnx    # 人脸检测 (1.9 MB)
│           ├── arcfaceq.onnx        # 人脸识别 (1.1 MB)
│           ├── people.jpg
│           └── head.jpg
├── build-profile.json5
└── oh-package.json5
```

---

## 数据流程图

### 目标检测

```
people.jpg (rawfile)
        │ 首次运行时复制到沙箱（按文件大小判断，已存在则复用）
        ▼
解码为 PixelMap（RGBA_8888）→ scale 到 640×640
        ▼
detectRGBA(modelPath, rgba, {w, h})   ← C++ 内完成 RGBA→CHW、归一化
        ▼
ONNX 推理（yolov5nq.onnx），输出 [1, 25200, 85]
        ▼
Sigmoid + 置信度过滤（> 0.25）→ NMS（IoU > 0.45）
        ▼
坐标映射回原图 → 绘制检测框与类别
```

### 人脸检测

```
people.jpg → letterbox 到 640×640（保持宽高比，灰色填充 114）
        ▼
detectFace(modelPath, CHW buffer, {scale, padX, padY})
        ▼
解码 3 个检测头（stride 8 / 16 / 32）+ Anchor + Sigmoid + NMS
        ▼
去除 letterbox 偏移与缩放 → 人脸框列表
```

### 人脸识别

```
head.jpg / people.jpg
        ▼
人脸检测 → 裁剪 112×112 → 归一化到 [-1, 1]
        ▼
recognizeFace(arcfaceModelPath, query, candidates)
        ▼
ArcFace 提取 512 维特征 → 计算余弦相似度
        ▼
相似度 ≥ 0.5 判为同一人（绿框），否则红框
```

> 原生层对同一模型路径复用已缓存的 ORT 会话；页面隐藏时调用 `releaseModel()` 释放会话，保证 SpaceMIT EP 在进程内同一时刻只有一个 provider。

---

## 编译与构建

### 环境要求

| 工具 | 版本要求 |
|------|----------|
| DevEco Studio | 5.0+ |
| OpenHarmony SDK | API 12 |
| CMake | 3.5+ |
| 目标 ABI | riscv64 |

### 构建步骤

```bash
git clone https://gitee.com/spacemit-openharmony/yolo_detection.git
cd yolo_detection

# 配置签名后构建 release HAP
node "<DevEco Studio 安装路径>/tools/hvigor/bin/hvigorw.js" \
  --mode module -p module=entry@default -p product=default \
  -p buildMode=release assembleHap --no-daemon
```

`entry/link_libs/riscv64/` 只用于链接期符号解析，不会打进 HAP；运行时由动态链接器从 `/system/lib64`（应用可通过 `/system/lib64/ndk` 访问）解析同名库。若该目录缺失，可从设备拉取：

```bash
hdc file recv /system/lib64/libonnxruntime.so.1 entry/link_libs/riscv64/
hdc file recv /system/lib64/libspacemit_ep.so.2 entry/link_libs/riscv64/
```

构建产物：

```
entry/build/default/outputs/default/entry-default-signed.hap
```

---

## 部署与运行

```bash
# 安装（覆盖安装）
hdc install -r entry-default-signed.hap

# 启动
hdc shell aa start -a EntryAbility -b com.example.yolo_detection

# 查看日志
hdc shell hilog -T YoloDetect

# 卸载
hdc uninstall com.example.yolo_detection
```

无需向设备推送推理库或模型：推理库来自镜像，模型随 HAP 安装后在首次运行时复制到应用沙箱（`filesDir`）。

### 性能参考（K3 Pico，OpenHarmony 6.1.0.35 / API 23）

| 项目 | 结果 |
|------|------|
| 推理库 | `/system/lib64` 中 ORT 1.24.2+spacemit.a1、EP 2.0.7、spine_tcm 3.0.2 |
| 目标检测 | `people.jpg`，5 个目标，51 ms |
| 人脸检测 | 4 张人脸，40 ms |
| 人脸识别 | 4 张人脸中匹配 1 张，4 ms |
| HAP 内库 | `libyolo_detect.so` + `libc++_shared.so` |

> 仓库 `entry/libs/riscv64/` 里可能还留有工具链附带的运行时文件；以构建产物实际打包内容为准，当前 HAP 内为 `libc++_shared.so`。

---

## 定制与修改指导

### 替换检测模型

1. 将新的 `.onnx` 放入 `entry/src/main/resources/rawfile/`。
2. 修改页面中的模型文件名（如 `DetectPage.ets` 的 `copyAssetToSandbox('yolov5nq.onnx')`）。
3. 输入尺寸不是 640×640 时，同步修改 `DetectPage.ets` 的 `INPUT_SIZE` 与 `yolo_detect.cpp` 的 `INPUT_W` / `INPUT_H`。

### 调整阈值

```cpp
static const float CONF_THRESH = 0.25f;  // 置信度阈值
static const float NMS_THRESH  = 0.45f;  // NMS IoU 阈值
```

人脸匹配阈值在 `FaceRecognizePage.ets`：

```typescript
const SIMILARITY_THRESHOLD = 0.5;
```

### 自定义类别

修改 `CocoClasses.ets` 的类别数组，并同步 `yolo_detect.cpp` 中的 `NUM_CLASSES`（输出维度为 `5 + NUM_CLASSES`）。

### 更换测试图片

替换 `rawfile` 中的图片，并修改页面对应的 `copyAssetToSandbox(...)` 文件名。

### 适配其他架构

当前仅有 riscv64 预编译产物与镜像库。其他架构需替换 `link_libs` 下的 ORT/EP，并在 `build-profile.json5` 的 `abiFilters` 中增加目标架构；SpaceMIT EP 仅支持 RISC-V。

---

## 关键参数说明

| 参数 | 位置 | 默认值 | 说明 |
|------|------|--------|------|
| `INPUT_W` / `INPUT_H` | `yolo_detect.cpp` | 640 | 检测模型输入尺寸 |
| `CONF_THRESH` | `yolo_detect.cpp` | 0.25 | 目标检测置信度阈值 |
| `NMS_THRESH` | `yolo_detect.cpp` | 0.45 | NMS IoU 阈值 |
| `NUM_CLASSES` | `yolo_detect.cpp` | 80 | COCO 类别数 |
| `FACE_STRIDES` | `yolo_detect.cpp` | 8/16/32 | 人脸检测头步长 |
| `ARCFACE_W/H` | `yolo_detect.cpp` | 112 | ArcFace 输入尺寸 |
| `SIMILARITY_THRESHOLD` | `FaceRecognizePage.ets` | 0.5 | 人脸匹配余弦相似度阈值 |
| letterbox 填充值 | 页面预处理 | 114 | 灰色填充像素值 |

---

## FAQ

**Q: 点击检测后界面卡死？**

旧版本在 UI 线程同步推理会触发 `THREAD_BLOCK_3S/6S`。最新代码已将推理移入 TaskPool，并复用 ORT 会话，请确认使用最新代码。

**Q: 提示模型加载失败？**

确认 `rawfile` 中三个 `.onnx` 文件完整；首次运行时模型会复制到 `filesDir`，如之前复制过损坏文件，可清除应用数据或删除沙箱中的同名文件后重试。

```bash
hdc shell hilog -T YoloDetect | grep -i error
```

**Q: 推理速度明显变慢？**

确认镜像中存在 `/system/lib64/libonnxruntime.so.1`、`libspacemit_ep.so.2`、`libspine_tcm.so.3`。缺少 EP 时会回退到 CPU，耗时显著增加；EP 路径下目标检测约 51 ms。

**Q: 检测框坐标偏移？**

人脸检测使用 letterbox 预处理，坐标需要反变换。确认传入 native 的 `scale`、`padX`、`padY` 与预处理一致。

**Q: 如何换成自己训练的 YOLOv5 模型？**

导出 ONNX（可选 int8 量化）→ 替换 `rawfile` 中的模型 → 修改页面文件名；类别数不是 80 时同步修改 `NUM_CLASSES` 与 `CocoClasses.ets`。

**Q: 能否同时加载多个模型？**

不能。SpaceMIT EP 在进程内只允许一个 provider，原生层同一时刻只保留一个会话；页面隐藏时会释放模型，切页后再加载新模型。

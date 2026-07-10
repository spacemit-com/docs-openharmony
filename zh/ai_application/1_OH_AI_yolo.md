<!--
 * Copyright 2022-2023 SPACEMIT. All rights reserved.
 * Use of this source code is governed by a BSD-style license
 * that can be found in the LICENSE file.
 * 
 * @Author: David(qiang.fu@spacemit.com)
 * @Date: 2026-03-04 11:39:35
 * @LastEditTime: 2026-05-13 16:19:24
 * @FilePath: \doc\docs-openharmony\zh\ai_application\1_OH_AI_yolo.md
 * @Description: 
-->
sidebar_position: 1

# AI检测识别应用说明

## 修订记录

| 修订版本 | 修订日期   | 修订说明       |
|----------|------------|-------------|
| 001      | 2026-04-27 | 初始版本       |

## 概述
YOLODetection是基于 OpenHarmony 的 AI 视觉检测应用，集成目标检测、人脸检测、人脸识别三大功能，使用 ONNX Runtime 在 RISC-V 设备上进行推理。

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
| 推理引擎 | ONNX Runtime 1.x |
| 硬件加速 | SpaceMIT Execution Provider |
| 目标架构 | RISC-V 64-bit (riscv64) |
| 构建工具 | Hvigor + CMake 3.5+ |
| 模型格式 | ONNX (量化 int8) |
| 检测模型 | YOLOv5-nano (量化) |
| 人脸识别模型 | ArcFace (量化) |

---

## 项目架构

```
┌─────────────────────────────────────────────────────┐
│                   ArkUI 界面层 (ETS)                  │
│  ┌──────────┐  ┌──────────────┐  ┌────────────────┐  │
│  │ HomePage │  │  DetectPage  │  │FaceRecognizePage│  │
│  │  主菜单  │  │  目标检测页  │  │   人脸识别页   │  │
│  └──────────┘  └──────────────┘  └────────────────┘  │
│                ┌──────────────┐                       │
│                │FaceDetectPage│                       │
│                │  人脸检测页  │                       │
│                └──────────────┘                       │
├─────────────────────────────────────────────────────┤
│                  NAPI 桥接层 (C++)                    │
│            libyolo_detect.so (549 KB)                │
│   detect() | detectFace() | recognizeFace()          │
├─────────────────────────────────────────────────────┤
│               推理引擎层                              │
│  libonnxruntime.so.1 (34 MB)                         │
│  libspacemit_ep.so   (7.8 MB) — RISC-V 加速          │
├─────────────────────────────────────────────────────┤
│               模型层 (rawfile)                        │
│  yolov5nq.onnx (2.6MB) | yolov5nfaceq.onnx (1.9MB)  │
│  arcfaceq.onnx (1.1MB)                               │
└─────────────────────────────────────────────────────┘
```

### 模块说明

- **ArkUI 界面层**：负责图像加载、预处理、结果可视化，用 ETS 编写
- **NAPI 桥接层**：`yolo_detect.cpp` 实现 C++ 推理逻辑，通过 NAPI 暴露给 ETS 调用
- **推理引擎层**：ONNX Runtime 负责模型加载与推理，SpaceMIT EP 提供 RISC-V 硬件加速
- **模型层**：三个量化 ONNX 模型，存放于 `rawfile`，首次运行时复制到沙箱

---

## 目录结构

```
yolo_detection/
├── AppScope/                        # 应用级资源
│   ├── app.json5                    # 包名、版本、图标
│   └── resources/base/element/
│       └── string.json              # 应用名称字符串
├── entry/                           # 主模块
│   ├── build-profile.json5          # 构建配置（启用 native C++）
│   ├── oh-package.json5             # 包元数据与测试依赖
│   ├── libs/riscv64/                # 预编译 .so 库
│   │   ├── libonnxruntime.so.1      # ONNX Runtime (34 MB)
│   │   ├── libspacemit_ep.so        # SpaceMIT 执行提供器 (7.8 MB)
│   │   ├── libstdc++.so.6           # C++ 标准库 (17 MB)
│   │   ├── libgcc_s.so.1            # GCC 运行时 (786 KB)
│   │   ├── libatomic.so.1           # 原子操作库 (106 KB)
│   │   └── libyolo_detect.so        # 本项目编译产物 (549 KB)
│   └── src/main/
│       ├── cpp/                     # C++ 原生代码
│       │   ├── CMakeLists.txt       # CMake 构建脚本
│       │   ├── yolo_detect.cpp      # 核心推理实现 (1006 行)
│       │   ├── onnxruntime_*.h      # ONNX Runtime 头文件
│       │   ├── spacemit_ort_env.h   # SpaceMIT 环境头文件
│       │   └── types/libyolo_detect/
│       │       └── index.d.ts       # TypeScript 类型声明
│       ├── ets/                     # ArkTS UI 代码
│       │   ├── pages/
│       │   │   ├── HomePage.ets         # 主菜单
│       │   │   ├── DetectPage.ets       # 目标检测
│       │   │   ├── FaceDetectPage.ets   # 人脸检测
│       │   │   └── FaceRecognizePage.ets# 人脸识别
│       │   ├── detect/
│       │   │   └── CocoClasses.ets      # 80 个 COCO 类别名
│       │   └── entryability/
│       │       └── EntryAbility.ets     # 应用入口
│       └── resources/
│           ├── rawfile/                 # 模型与测试图片
│           │   ├── yolov5nq.onnx        # 目标检测模型 (2.6 MB)
│           │   ├── yolov5nfaceq.onnx    # 人脸检测模型 (1.9 MB)
│           │   ├── arcfaceq.onnx        # 人脸识别模型 (1.1 MB)
│           │   ├── people.jpg           # 测试场景图 (157 KB)
│           │   └── head.jpg             # 查询人脸图 (2.9 KB)
│           └── base/
│               ├── element/
│               │   ├── color.json       # 亮色主题颜色
│               │   └── string.json      # UI 字符串
│               └── profile/
│                   └── main_pages.json  # 页面路由配置
├── build-profile.json5              # 根构建配置（SDK 版本）
├── hvigorfile.ts                    # 根构建脚本
└── oh-package.json5                 # 根包配置
```

---

## 数据流程图

### 目标检测流程

```
people.jpg (rawfile)
        │
        ▼
  复制到沙箱目录
        │
        ▼
  解码为 PixelMap (RGBA_8888)
        │
        ▼
  缩放至 640×640
        │
        ▼
  读取像素 → Uint8Array
        │
        ▼
  RGBA uint8 → Float32 CHW，归一化 [0, 1]
        │
        ▼
  调用 native detect(modelPath, buffer, {w, h})
        │
        ▼
  ONNX 推理 (yolov5nq.onnx)
  输出形状: [1, 25200, 85]
        │
        ▼
  Sigmoid 激活 + 置信度过滤 (>0.25)
        │
        ▼
  NMS 去重 (IoU > 0.45)
        │
        ▼
  坐标映射回原图尺寸
        │
        ▼
  返回 [{x1,y1,x2,y2, confidence, classId}, ...]
        │
        ▼
  在原图上绘制彩色检测框 + 类别标签
        │
        ▼
  显示结果图 + 检测列表
```

### 人脸检测流程

```
people.jpg
        │
        ▼
  Letterbox 预处理
  （保持宽高比，灰色填充至 640×640，填充值 114）
        │
        ▼
  Float32 CHW，归一化 [0, 1]
        │
        ▼
  调用 native detectFace(modelPath, buffer, {w,h,scale,padX,padY})
        │
        ▼
  ONNX 推理 (yolov5nfaceq.onnx)
  解码 3 个检测头 (stride 8 / 16 / 32)
        │
        ▼
  Anchor 解码 + Sigmoid + NMS
        │
        ▼
  坐标反变换（去除 letterbox 偏移与缩放）
        │
        ▼
  返回人脸检测框列表
```

### 人脸识别流程

```
head.jpg (查询人脸)          people.jpg (场景图)
        │                           │
        ▼                           ▼
  人脸检测 (yolov5nfaceq)    人脸检测 (yolov5nfaceq)
                                    │
                                    ▼
                             裁剪每张人脸至 112×112
                             （双线性插值采样）
                             归一化至 [-1, 1]
        │                           │
        └──────────┬────────────────┘
                   ▼
  调用 native recognizeFace(arcfaceModelPath, queryBuf, [candidateBufs])
                   │
                   ▼
  ArcFace 推理 (arcfaceq.onnx)
  提取 512 维人脸嵌入向量
                   │
                   ▼
  计算余弦相似度 (query vs. 每个候选)
                   │
                   ▼
  相似度 ≥ 0.5 → 绿框（匹配）
  相似度 < 0.5 → 红框（不匹配）
```

---

## 编译与构建

### 环境要求

| 工具 | 版本要求 |
|------|----------|
| DevEco Studio | 5.0+ |
| OpenHarmony SDK | API 12 |
| CMake | 3.5+ |
| Node.js | 16+ |
| hvigorw | 随项目自带 |

### 构建步骤

**1. 克隆项目**

```bash
git clone https://gitee.com/spacemit-openharmony/yolo_detection.git
cd yolo_detection
```

**2. 用 DevEco Studio 打开项目**

File → Open → 选择项目根目录，等待 Hvigor 同步依赖。

**3. 配置签名**

在 `build-profile.json5` 的 `signingConfigs` 中填入你的签名信息，或在 DevEco Studio 中通过 `File → Project Structure → Signing Configs` 自动生成调试签名。

**4. 命令行构建**

```bash
# 构建 debug HAP
./hvigorw assembleHap --mode module -p module=entry@default -p buildMode=debug

# 构建 release HAP（需配置签名）
./hvigorw assembleHap --mode module -p module=entry@default -p buildMode=release
```

**5. 仅编译 C++ 原生库**

CMake 配置由 Hvigor 自动调用，也可手动验证：

```bash
cd entry/src/main/cpp
cmake -DCMAKE_TOOLCHAIN_FILE=<ohos_sdk>/native/build/cmake/ohos.toolchain.cmake \
      -DOHOS_ARCH=riscv64 \
      -B build
cmake --build build
```

编译产物 `libyolo_detect.so` 会输出到 `entry/libs/riscv64/`。

### 构建产物

```
entry/build/default/outputs/default/
└── entry-default-signed.hap    # 可安装的 HAP 包
```

---

## 部署与运行

**安装到设备**

```bash
hdc install entry-default-signed.hap
```

**启动应用**

```bash
hdc shell aa start -a EntryAbility -b com.example.yolo_detection
```

**查看日志**

```bash
# 查看应用日志（TAG: YoloDetect）
hdc shell hilog -T YoloDetect

# 查看所有日志
hdc shell hilog | grep yolo
```

**卸载**

```bash
hdc uninstall com.example.yolo_detection
```

---

## 定制与修改指导

### 替换检测模型

1. 将新的 `.onnx` 模型放入 `entry/src/main/resources/rawfile/`
2. 修改对应页面中的模型文件名，例如 `DetectPage.ets`：
   ```typescript
   // 修改这一行
   const modelFileName = 'your_new_model.onnx';
   ```
3. 如果模型输入尺寸不是 640×640，同步修改预处理中的缩放逻辑和 `yolo_detect.cpp` 中的 `INPUT_SIZE` 常量：
   ```cpp
   static const int INPUT_SIZE = 640; // 改为新尺寸
   ```

### 修改置信度阈值

在 `yolo_detect.cpp` 中修改：

```cpp
static const float CONF_THRESHOLD = 0.25f;  // 置信度阈值
static const float NMS_THRESHOLD  = 0.45f;  // NMS IoU 阈值
```

人脸识别匹配阈值在 `FaceRecognizePage.ets` 中修改：

```typescript
const SIMILARITY_THRESHOLD = 0.5; // 余弦相似度阈值
```

### 添加新的检测类别

如果使用自定义模型（非 COCO 80 类），修改 `CocoClasses.ets`：

```typescript
export const CLASS_NAMES: string[] = [
  'class_a', 'class_b', 'class_c', // 替换为你的类别
];
```

同时修改 `yolo_detect.cpp` 中的类别数量：

```cpp
static const int NUM_CLASSES = 80; // 改为实际类别数
// 输出维度也需对应修改: 5 + NUM_CLASSES
```

### 替换测试图片

将新图片放入 `entry/src/main/resources/rawfile/`，在对应页面修改文件名：

```typescript
// DetectPage.ets
const imageFileName = 'your_image.jpg';
```

### 增加新功能页面

1. 在 `entry/src/main/ets/pages/` 新建 `NewPage.ets`
2. 在 `entry/src/main/resources/base/profile/main_pages.json` 注册路由：
   ```json
   {
     "src": ["pages/HomePage", "pages/DetectPage", "pages/NewPage"]
   }
   ```
3. 在 `HomePage.ets` 添加导航按钮：
   ```typescript
   Button('新功能')
     .onClick(() => router.pushUrl({ url: 'pages/NewPage' }))
   ```

### 适配其他架构

当前预编译库仅支持 `riscv64`。若需支持 `arm64-v8a`：

1. 将对应架构的 `.so` 文件放入 `entry/libs/arm64-v8a/`
2. 在 `entry/build-profile.json5` 的 `abiFilters` 中添加目标架构：
   ```json
   "abiFilters": ["riscv64", "arm64-v8a"]
   ```

---

## 关键参数说明

| 参数 | 位置 | 默认值 | 说明 |
|------|------|--------|------|
| `INPUT_SIZE` | `yolo_detect.cpp` | 640 | 模型输入图像尺寸 |
| `CONF_THRESHOLD` | `yolo_detect.cpp` | 0.25 | 目标检测置信度阈值 |
| `NMS_THRESHOLD` | `yolo_detect.cpp` | 0.45 | NMS IoU 阈值 |
| `NUM_CLASSES` | `yolo_detect.cpp` | 80 | COCO 类别数量 |
| `ARCFACE_SIZE` | `FaceRecognizePage.ets` | 112 | ArcFace 输入尺寸 |
| `SIMILARITY_THRESHOLD` | `FaceRecognizePage.ets` | 0.5 | 人脸匹配相似度阈值 |
| `intraOpNumThreads` | `yolo_detect.cpp` | 4 | ONNX Runtime 推理线程数 |
| letterbox 填充值 | `FaceDetectPage.ets` | 114 | 灰色填充像素值 |

---

## FAQ

**Q: 安装后点击检测没有反应？**

检查日志中是否有模型加载失败的错误。模型首次运行时会从 rawfile 复制到沙箱，确保 `rawfile` 目录下的 `.onnx` 文件存在且完整。

```bash
hdc shell hilog -T YoloDetect | grep -i error
```

**Q: 推理速度很慢？**

- 确认 `libspacemit_ep.so` 已正确加载（日志中会有 `SpaceMIT EP loaded` 提示）
- 检查 `intraOpNumThreads` 是否设置合理（默认 4）
- 量化模型（`q.onnx`）比浮点模型快 2-4 倍，确认使用的是量化版本

**Q: 检测框坐标偏移？**

人脸检测使用了 letterbox 预处理，坐标需要反变换。确认传入 native 函数的 `scale`、`padX`、`padY` 参数与预处理时一致。

**Q: 如何更换为自己训练的 YOLOv5 模型？**

1. 导出为 ONNX 格式：`python export.py --weights best.pt --include onnx`
2. 可选：使用 `onnxruntime` 量化工具进行 int8 量化
3. 替换 `rawfile` 中对应的 `.onnx` 文件
4. 如果类别数不是 80，修改 `NUM_CLASSES` 和 `CocoClasses.ets`

**Q: 编译时提示找不到 ONNX Runtime 头文件？**

头文件已包含在 `entry/src/main/cpp/` 目录下，确认 `CMakeLists.txt` 中的 `include_directories` 路径正确：

```cmake
include_directories(${CMAKE_CURRENT_SOURCE_DIR})
```

**Q: 如何在非 RISC-V 设备上运行？**

需要替换 `entry/libs/` 下的所有 `.so` 为目标架构版本（arm64-v8a 等），并修改 `build-profile.json5` 中的 `abiFilters`。SpaceMIT EP 仅支持 RISC-V，其他架构需改用 CPU EP 或其他执行提供器。

**Q: 日志中出现 `YOLO_LOG_DOMAIN exceeding threshold`？**

这是 OpenHarmony hilog 的域值限制问题，已在 commit `50951e4` 中修复，确保使用最新代码。

**Q: 人脸识别误识别率高？**

调高 `SIMILARITY_THRESHOLD`（如从 0.5 改为 0.6），可以减少误匹配，但也会增加漏检。根据实际场景调整。

**Q: 如何添加摄像头实时检测？**

当前版本仅支持静态图片检测。接入摄像头需要：
1. 使用 OpenHarmony Camera Kit 获取帧数据
2. 将每帧转换为 PixelMap 后走相同的预处理流程
3. 注意控制推理频率（建议每隔 2-3 帧推理一次）以保证流畅度

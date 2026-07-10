<!--
 * Copyright 2022-2023 SPACEMIT. All rights reserved.
 * Use of this source code is governed by a BSD-style license
 * that can be found in the LICENSE file.
 * 
 * @Author: David(qiang.fu@spacemit.com)
 * @Date: 2026-03-04 11:39:35
 * @LastEditTime: 2026-05-13 16:19:54
 * @FilePath: \doc\docs-openharmony\zh\ai_application\6_OH_AI_opencvdemo.md
 * @Description: 
-->
sidebar_position: 6

# OpenCV应用说明

## 项目简介

本项目是一个运行在 **OpenHarmony** 系统、**RISC-V64** 架构设备上的图像处理演示应用。通过将 OpenCV 原生库集成到 HAP（HarmonyOS Ability Package）中，展示了在鸿蒙生态下调用 C++ 图像处理能力的完整链路。

应用包含三个功能场景：

| 场景 | 功能 |
|------|------|
| 场景一：图像滤波 | 灰度化、高斯模糊、锐化 |
| 场景二：边缘检测 | Canny 边缘检测、Sobel 梯度 |
| 场景三：图像变换 | 二值化阈值、直方图均衡化 |

## 平台支持情况

|      平台 & 系统       |       是否支持     |
|-----------------------|-----------------------|
| K1 OpenHarmony5.0     | ✅ 支持               |
| K1 OpenHarmony6.1    | ✅ 支持             |
| K3 OpenHarmony6.1     | ✅ 支持              |

## 技术栈

| 层次 | 技术 |
|------|------|
| UI 框架 | ArkTS + ArkUI（声明式 UI） |
| 原生桥接 | Node-API（NAPI） |
| 图像处理 | OpenCV 5.0（core / imgproc / imgcodecs） |
| 编译系统 | CMake 3.5+，hvigor |
| 目标平台 | OpenHarmony SDK 12，RISC-V64 |
| 开发语言 | ArkTS（前端）、C++17（原生层） |

---

## 架构设计

```
┌─────────────────────────────────────────────┐
│              ArkTS UI 层                     │
│  Index.ets → FilterPage / EdgePage /         │
│              TransformPage                   │
│                    │                         │
│           OpenCVHelper.ets                   │
│    processPixelMap()  +  opencvdemo 模块引用  │
└──────────────────┬──────────────────────────┘
                   │  ArrayBuffer (BGRA 像素)
                   ▼
┌─────────────────────────────────────────────┐
│           NAPI 桥接层（C++）                  │
│  napi_preload.cpp  —  预加载系统依赖库        │
│  napi_init.cpp     —  注册 JS 可调用函数      │
└──────────────────┬──────────────────────────┘
                   │  uint8_t* BGRA buffer
                   ▼
┌─────────────────────────────────────────────┐
│         OpenCV 原生实现层（C++）              │
│  opencv_native.cpp — 7 个图像处理函数         │
│  依赖：libopencv_core / imgproc / imgcodecs  │
└─────────────────────────────────────────────┘
```

**数据流：**

1. ArkTS 从 `PixelMap` 读取像素到 `ArrayBuffer`（BGRA_8888 格式）
2. 通过 NAPI 将 `ArrayBuffer` 指针传入 C++ 层
3. C++ 将 buffer 包装为 `cv::Mat`，调用 OpenCV 算法处理
4. 处理结果写入输出 `ArrayBuffer`，ArkTS 侧重建为新 `PixelMap` 并渲染

---

## 代码结构

```
opencvdemo/
├── AppScope/
│   ├── app.json5                  # 应用全局配置（bundleName、版本号）
│   └── resources/base/
│       ├── element/string.json    # 字符串资源
│       └── media/app_icon.png     # 应用图标
├── build-profile.json5            # 构建配置（SDK版本、签名、产品）
├── entry/
│   ├── build-profile.json5        # 模块构建配置
│   ├── hvigorfile.ts              # hvigor 构建脚本
│   ├── oh-package.json5           # 模块依赖声明
│   ├── libs/
│   │   └── riscv64/               # RISC-V64 预编译 OpenCV 动态库
│   │       ├── libopencv_core.so(.500)
│   │       ├── libopencv_imgproc.so(.500)
│   │       └── libopencv_imgcodecs.so(.500)
│   └── src/main/
│       ├── cpp/
│       │   ├── CMakeLists.txt     # 原生库构建脚本
│       │   ├── opencv_native.h    # 原生函数声明
│       │   ├── opencv_native.cpp  # OpenCV 算法实现
│       │   ├── napi_init.cpp      # NAPI 函数注册
│       │   ├── napi_preload.cpp   # 动态库预加载
│       │   └── include/opencv2/  # OpenCV 头文件
│       ├── ets/
│       │   ├── entryability/
│       │   │   └── EntryAbility.ets   # UIAbility 入口，加载首页
│       │   ├── pages/
│       │   │   ├── Index.ets          # 主页（场景列表）
│       │   │   ├── FilterPage.ets     # 场景一：图像滤波
│       │   │   ├── EdgePage.ets       # 场景二：边缘检测
│       │   │   └── TransformPage.ets  # 场景三：图像变换
│       │   └── utils/
│       │       └── OpenCVHelper.ets   # PixelMap ↔ ArrayBuffer 工具函数
│       └── module.json5           # 模块配置（Ability、页面路由）
```

---

## 算法实现

所有算法实现于 `entry/src/main/cpp/opencv_native.cpp`，统一接受 **BGRA_8888** 格式的像素 buffer，输出同格式结果。

### 公共辅助函数

```cpp
// BGRA buffer → cv::Mat（深拷贝，避免外部 buffer 生命周期问题）
static cv::Mat bgraToMat(const uint8_t* src, int width, int height);

// cv::Mat → BGRA buffer（自动处理单通道/三通道转换）
static void matToBgra(const cv::Mat& mat, uint8_t* dst, int width, int height);
```

### 场景一：图像滤波

**灰度化** `OpenCV_Grayscale`
- `cv::cvtColor(input, gray, COLOR_BGRA2GRAY)` 转为单通道灰度图
- 输出时再转回 BGRA 以保持格式一致

**高斯模糊** `OpenCV_GaussianBlur`
- 核大小强制为奇数（偶数自动 +1），最小为 3
- `cv::GaussianBlur(input, result, Size(k,k), 0)`，sigma 由核大小自动推导

**锐化** `OpenCV_Sharpen`
- 使用 Unsharp Masking：先高斯模糊（sigma=3），再与原图加权叠加
- `result = 1.5 * original - 0.5 * blurred`，等效于增强高频细节

### 场景二：边缘检测

**Canny 边缘检测** `OpenCV_CannyEdge`
- 先转灰度，再调用 `cv::Canny(gray, edges, threshold1, threshold2)`
- 双阈值可在 UI 侧通过滑块实时调节（低阈值 10–200，高阈值 50–400）
- 推荐比例：高阈值 ≈ 低阈值 × 3

**Sobel 梯度** `OpenCV_Sobel`
- 分别计算 X、Y 方向梯度（`CV_16S` 避免溢出）
- `convertScaleAbs` 取绝对值后，`addWeighted(absX, 0.5, absY, 0.5, ...)` 合并

### 场景三：图像变换

**二值化** `OpenCV_Threshold`
- 转灰度后 `cv::threshold(gray, result, thresh, 255, THRESH_BINARY)`
- 阈值范围 0–255，可通过滑块实时调节

**直方图均衡化** `OpenCV_EqualizeHist`
- 转灰度后 `cv::equalizeHist(gray, equalized)`
- 对低对比度图像效果显著（演示图像像素值刻意压缩在 [80, 180] 区间）

---

## NAPI 桥接层

### 函数注册（`napi_init.cpp`）

每个 C++ 函数通过 `napi_property_descriptor` 注册为 JS 可调用方法：

```
JS 调用名        →  C++ 实现
grayscale        →  OpenCV_Grayscale
gaussianBlur     →  OpenCV_GaussianBlur
sharpen          →  OpenCV_Sharpen
cannyEdge        →  OpenCV_CannyEdge
sobel            →  OpenCV_Sobel
threshold        →  OpenCV_Threshold
equalizeHist     →  OpenCV_EqualizeHist
```

模块名为 `opencvdemo`，对应 `libopencvdemo.so`，在 ArkTS 中通过 `import opencvdemo from 'libopencvdemo.so'` 引入。

### 依赖预加载（`napi_preload.cpp`）

RISC-V64 OpenHarmony 的动态链接器在某些情况下不能自动传递解析 OpenCV 的间接依赖，因此在模块 `__attribute__((constructor))` 阶段主动 `dlopen` 以下库：

- `libc++_shared.so`
- `libz.so`
- `/lib/libatomic.so.1`
- `/lib/libgcc_s.so.1`

### ArkTS 工具层（`OpenCVHelper.ets`）

`processPixelMap` 封装了完整的像素处理流程：

```
srcPixelMap
  → readPixelsToBuffer(srcBuf)   // PixelMap → ArrayBuffer
  → processFn(srcBuf, dstBuf)    // 调用 NAPI 函数
  → image.createPixelMap(dstBuf) // ArrayBuffer → 新 PixelMap
  → 返回新 PixelMap 供 Image 组件渲染
```

---

## 编译指南

### 环境要求

| 工具 | 版本要求 |
|------|---------|
| DevEco Studio | 4.x 及以上 |
| OpenHarmony SDK | API Level 12 |
| CMake | 3.5+（DevEco 内置） |
| 目标设备 | RISC-V64 OpenHarmony 设备或模拟器 |

### 编译步骤

1. **克隆仓库**

   ```bash
   git clone https://gitee.com/spacemit-openharmony/opencvdemo.git
   cd opencvdemo
   ```

2. **用 DevEco Studio 打开项目**

   File → Open → 选择项目根目录

3. **配置签名**

   File → Project Structure → Signing Configs，配置调试签名证书。
   或直接替换 `build-profile.json5` 中 `signingConfigs.material` 下的证书路径为本机路径。

4. **同步依赖**

   DevEco Studio 会自动触发 hvigor 同步。如需手动执行：

   ```bash
   hvigorw assembleHap
   ```

5. **连接设备并运行**

   Run → Run 'entry'，选择目标 RISC-V64 设备。

### 原生库说明

预编译的 OpenCV 动态库位于 `entry/libs/riscv64/`，已针对 RISC-V64 架构编译。CMake 构建时通过 `IMPORTED` 目标引入，无需重新编译 OpenCV。

如需为其他架构（如 arm64-v8a）构建，需替换对应架构的 OpenCV 库，并修改 `CMakeLists.txt` 中的 `OPENCV_LIBS_DIR` 路径。

---

## FAQ

**Q: 应用启动后图像显示空白？**

A: 检查 `napi_preload.cpp` 中的依赖库是否在目标设备上存在。可通过 hilog 查看 `preload failed` 日志定位缺失的库。

```bash
hdc shell hilog | grep opencvdemo
```

**Q: 编译时报 `libopencv_core.so` 找不到？**

A: 确认 `entry/libs/riscv64/` 目录下的 `.so` 文件存在，且 `CMakeLists.txt` 中 `OPENCV_LIBS_DIR` 路径正确。该路径是相对于 `CMakeLists.txt` 所在目录的相对路径。

**Q: 在非 RISC-V64 设备上能运行吗？**

A: 不能直接运行。`libs/riscv64/` 下的 OpenCV 库是 RISC-V64 专用二进制。需要替换为目标架构对应的 OpenCV 库，并更新 `CMakeLists.txt` 中的库目录。

**Q: 高斯模糊传入偶数核大小会怎样？**

A: `OpenCV_GaussianBlur` 内部会自动将偶数核大小加 1 转为奇数（OpenCV 要求高斯核必须为奇数），最小值为 3。

**Q: Canny 检测效果不明显？**

A: 调整双阈值参数。低阈值控制弱边缘的保留，高阈值控制强边缘的检测。通常高阈值设为低阈值的 2–3 倍效果较好。EdgePage 提供了实时滑块可直接调节。

**Q: 如何替换为真实图片而非内置测试图案？**

A: 在各 Page 的 `loadBuiltinImage()` 方法中，将程序生成的 buffer 替换为从相册或资源文件加载的 `PixelMap`。可使用 `@ohos.multimedia.image` 的 `createImageSource` 从文件路径创建图像源。

**Q: 处理大图时界面卡顿？**

A: `processPixelMap` 中的 NAPI 调用是同步的，大图处理会阻塞 JS 线程。可考虑将处理逻辑移至 Worker 线程（`@ohos.worker`）中执行，通过消息传递 ArrayBuffer。

---

## 许可证

本项目仅作演示用途。OpenCV 遵循 Apache 2.0 许可证。


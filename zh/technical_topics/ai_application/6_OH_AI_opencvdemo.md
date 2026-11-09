<!--
 * Copyright 2022-2023 SPACEMIT. All rights reserved.
 * Use of this source code is governed by a BSD-style license
 * that can be found in the LICENSE file.
 * 
 * @Author: David(qiang.fu@spacemit.com)
 * @Date: 2026-03-04 11:39:35
 * @LastEditTime: 2026-11-07 10:00:00
 * @FilePath: \doc\docs-openharmony\zh\ai_application\6_OH_AI_opencvdemo.md
 * @Description: 
-->
sidebar_position: 6

# OpenCV应用说明

## 项目简介

本项目是运行在 OpenHarmony、RISC-V64 设备上的图像处理演示应用，将 OpenCV 原生库集成进 HAP，展示 ArkTS → NAPI → C++ 图像处理的完整链路。

应用包含三个场景：

| 场景 | 功能 |
|------|------|
| 图像滤波 | 灰度化、高斯模糊、锐化 |
| 边缘检测 | Canny 边缘检测、Sobel 梯度 |
| 图像变换 | 二值化阈值、直方图均衡化 |

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
| 构建系统 | CMake 3.5+，Hvigor |
| 目标平台 | OpenHarmony SDK 12，RISC-V64 |

---

## 架构设计

```
┌─────────────────────────────────────────────┐
│              ArkTS UI 层                     │
│  Index → FilterPage / EdgePage /            │
│          TransformPage                      │
│         OpenCVHelper.ets                    │
└──────────────────┬──────────────────────────┘
                   │ ArrayBuffer（BGRA 像素）
                   ▼
┌─────────────────────────────────────────────┐
│           NAPI 桥接层（C++）                  │
│  napi_preload.cpp  预加载系统依赖库           │
│  napi_init.cpp     注册 7 个处理函数          │
└──────────────────┬──────────────────────────┘
                   ▼
┌─────────────────────────────────────────────┐
│        OpenCV 原生实现（C++）                 │
│  opencv_native.cpp                          │
│  依赖 libopencv_core / imgproc / imgcodecs   │
│  （随 HAP 打包在 entry/libs/riscv64）         │
└─────────────────────────────────────────────┘
```

**数据流：**

1. ArkTS 从 `PixelMap` 读取像素到 `ArrayBuffer`（BGRA_8888）。
2. 通过 NAPI 把 buffer 传入 C++，包装为 `cv::Mat`。
3. OpenCV 处理结果写回输出 buffer。
4. ArkTS 重建 `PixelMap` 并渲染。

---

## 代码结构

```
opencvdemo/
├── AppScope/                        # 应用级配置与资源
├── build-profile.json5
├── entry/
│   ├── libs/riscv64/                # 随 HAP 打包的 OpenCV 库
│   │   ├── libopencv_core.so(.500)
│   │   ├── libopencv_imgproc.so(.500)
│   │   └── libopencv_imgcodecs.so(.500)
│   └── src/main/
│       ├── cpp/
│       │   ├── CMakeLists.txt
│       │   ├── opencv_native.h / .cpp   # 算法实现
│       │   ├── napi_init.cpp            # NAPI 注册
│       │   ├── napi_preload.cpp         # 依赖库预加载
│       │   └── include/opencv2/         # 头文件
│       ├── ets/
│       │   ├── entryability/EntryAbility.ets
│       │   ├── pages/
│       │   │   ├── Index.ets
│       │   │   ├── FilterPage.ets
│       │   │   ├── EdgePage.ets
│       │   │   └── TransformPage.ets
│       │   └── utils/OpenCVHelper.ets
│       └── module.json5
```

---

## 算法实现

所有算法位于 `opencv_native.cpp`，统一接收 **BGRA_8888** 像素 buffer，输出同格式结果。

### 公共辅助

```cpp
static cv::Mat bgraToMat(const uint8_t* src, int width, int height);  // 深拷贝
static void matToBgra(const cv::Mat& mat, uint8_t* dst, int width, int height);
```

### 图像滤波

- **灰度化** `OpenCV_Grayscale`：`cvtColor(..., COLOR_BGRA2GRAY)`，输出时转回 BGRA。
- **高斯模糊** `OpenCV_GaussianBlur`：核大小强制为奇数（偶数 +1），最小 3。
- **锐化** `OpenCV_Sharpen`：Unsharp Masking，`1.5 × 原图 − 0.5 × 模糊图`。

### 边缘检测

- **Canny** `OpenCV_CannyEdge`：转灰度后 `cv::Canny`，双阈值可在界面用滑块调节（低 10–200，高 50–400）。
- **Sobel** `OpenCV_Sobel`：X/Y 方向分别求梯度（`CV_16S` 防溢出），取绝对值后等权合并。

### 图像变换

- **二值化** `OpenCV_Threshold`：转灰度后 `cv::threshold(..., THRESH_BINARY)`，阈值 0–255 可调。
- **直方图均衡化** `OpenCV_EqualizeHist`：转灰度后 `equalizeHist`。

---

## NAPI 桥接层

### 函数注册（`napi_init.cpp`）

| JS 调用名 | C++ 实现 |
|-----------|----------|
| `grayscale` | `OpenCV_Grayscale` |
| `gaussianBlur` | `OpenCV_GaussianBlur` |
| `sharpen` | `OpenCV_Sharpen` |
| `cannyEdge` | `OpenCV_CannyEdge` |
| `sobel` | `OpenCV_Sobel` |
| `threshold` | `OpenCV_Threshold` |
| `equalizeHist` | `OpenCV_EqualizeHist` |

模块名为 `opencvdemo`，对应 `libopencvdemo.so`，ArkTS 通过 `import opencvdemo from 'libopencvdemo.so'` 引入。

### 依赖预加载（`napi_preload.cpp`）

RISC-V64 上动态链接器不一定能自动解析 OpenCV 的间接依赖，因此在 `__attribute__((constructor))` 阶段主动 `dlopen`：

- `libc++_shared.so`
- `libz.so`
- `/lib/libatomic.so.1`
- `/lib/libgcc_s.so.1`

> 在 OpenHarmony 6.1.0.35 镜像中不存在 `/lib/libatomic.so.1` 与 `/lib/libgcc_s.so.1`，启动日志会打印两条 `preload failed`；这两个库实际位于 `/system/lib64`，预加载失败不影响 OpenCV 调用。

### ArkTS 工具层

`OpenCVHelper.processPixelMap` 封装处理流程：

```
srcPixelMap
  → readPixelsToBuffer(srcBuf)
  → processFn(srcBuf, dstBuf)
  → image.createPixelMap(dstBuf)
  → 返回新 PixelMap
```

---

## 编译指南

### 环境要求

| 工具 | 版本 |
|------|------|
| DevEco Studio | 4.x 及以上 |
| OpenHarmony SDK | API 12 |
| CMake | 3.5+（DevEco 内置） |
| 目标设备 | RISC-V64 OpenHarmony |

### 编译步骤

```bash
git clone https://gitee.com/spacemit-openharmony/opencvdemo.git
cd opencvdemo
```

1. 用 DevEco Studio 打开项目。
2. 配置签名：`File → Project Structure → Signing Configs`。
3. 构建：

```bash
node "<DevEco Studio 安装路径>/tools/hvigor/bin/hvigorw.js" \
  --mode module -p module=entry@default -p product=default \
  -p buildMode=release assembleHap --no-daemon
```

4. 安装并运行：

```bash
hdc install -r entry/build/default/outputs/default/entry-default-signed.hap
hdc shell aa start -a EntryAbility -b com.example.opencvdemo
```

### 原生库说明

OpenCV 动态库随 HAP 打包在 `entry/libs/riscv64/`，CMake 通过 `IMPORTED` 目标引入，无需重新编译 OpenCV。若需支持其他架构，替换对应架构的库并修改 `CMakeLists.txt` 中的 `OPENCV_LIBS_DIR`。

### 功能验证

K3 Pico（OpenHarmony 6.1.0.35 / API 23）：场景一正常打开；原图为彩色测试图，点击「灰度化」后红、蓝圆形均变为灰色，页面无白屏或错位（截图核验）。启动日志中的两条 `preload failed` 不影响功能。

---

## FAQ

**Q: 应用启动后图像显示空白？**

检查 `napi_preload.cpp` 预加载的库在设备上是否存在，用日志定位：

```bash
hdc shell hilog | grep opencvdemo
```

> 当前镜像会打印 `/lib/libatomic.so.1`、`/lib/libgcc_s.so.1` 两条 `preload failed`，属于已知噪音，不影响功能；若出现其他库加载失败，再按路径检查 `/system/lib64`。

**Q: 编译时报找不到 `libopencv_core.so`？**

确认 `entry/libs/riscv64/` 下库文件存在，且 `CMakeLists.txt` 中 `OPENCV_LIBS_DIR` 相对路径正确。

**Q: 能在非 RISC-V64 设备上运行吗？**

不能直接运行，`libs/riscv64` 下是 RISC-V64 二进制。需替换为目标架构的 OpenCV 库。

**Q: 高斯模糊传入偶数核大小会怎样？**

内部会自动加 1 转为奇数，最小值为 3。

**Q: Canny 效果不明显？**

调整双阈值：低阈值控制弱边缘保留，高阈值控制强边缘检测，通常高阈值取低阈值的 2–3 倍。

**Q: 如何替换为真实图片？**

把各页面 `loadBuiltinImage()` 中程序生成的 buffer 改为从相册或资源文件加载的 `PixelMap`（如 `@ohos.multimedia.image` 的 `createImageSource`）。

**Q: 处理大图时界面卡顿？**

`processPixelMap` 中的 NAPI 调用是同步的，会阻塞 JS 线程。可将处理移入 Worker（`@ohos.worker`），通过消息传递 `ArrayBuffer`。

---

## 许可证

本项目仅作演示用途。OpenCV 遵循 Apache 2.0 许可证。

<!--
 * Copyright 2022-2023 SPACEMIT. All rights reserved.
 * Use of this source code is governed by a BSD-style license
 * that can be found in the LICENSE file.
 * 
 * @Author: David(qiang.fu@spacemit.com)
 * @Date: 2026-03-04 11:39:35
 * @LastEditTime: 2026-11-07 10:00:00
 * @FilePath: \doc\docs-openharmony\zh\ai_application\3_OH_AI_rvvdemo.md
 * @Description: 
-->
sidebar_position: 3

# RVV应用说明

## 项目简介

RVV Demo 是一个基于 OpenHarmony 的原生应用，用于展示 RISC-V Vector Extension (RVV) 在常见算法中的性能优势。应用对同一算法分别提供标量与 RVV 向量化实现，并实时对比执行时间与加速比。

## 平台支持情况

|      平台 & 系统       |       是否支持     |
|-----------------------|-----------------------|
| K1 OpenHarmony5.0     | ✅ 支持               |
| K1 OpenHarmony6.1    | ✅ 支持             |
| K3 OpenHarmony6.1     | ✅ 支持              |

### 核心功能

- **向量加法**：两个浮点数组逐元素相加
- **点积**：两个向量的内积
- **矩阵向量乘**：矩阵与向量相乘
- **RGB 转灰度**：图像颜色空间转换

每个算法都提供标量和 RVV 两种实现，界面显示两种实现的耗时与加速比（数据规模可在 10⁴ ~ 10⁷ 之间调节）。

---

## 技术栈

| 层次 | 技术 |
|------|------|
| UI | ArkTS + ArkUI（声明式 UI） |
| 原生桥接 | N-API |
| 算法实现 | C++17 + RISC-V Vector Intrinsics 1.0 |
| 构建 | Hvigor + CMake |
| 目标架构 | riscv64（rv64gcv）；其他架构回退标量实现 |

---

## 架构设计

### 整体架构

```
┌─────────────────────────────────────────────────┐
│ ArkTS UI（Index / VecAddPage / DotProductPage …）│
└───────────────────┬─────────────────────────────┘
                    │ N-API
┌───────────────────▼─────────────────────────────┐
│ napi_init.cpp                                   │
│ benchVecAdd / benchDotProduct /                 │
│ benchMatVecMul / benchRgbToGray                 │
└───────────────────┬─────────────────────────────┘
                    │
        ┌───────────┴────────────┐
        ▼                        ▼
┌──────────────────┐    ┌──────────────────┐
│ 标量实现          │    │ RVV 实现          │
│ C++ for 循环      │    │ Intrinsics       │
└──────────────────┘    └──────────────────┘
```

### 编译策略

- **riscv64**：直接链接仓库内的预编译静态库 `entry/src/main/cpp/prebuilt/librvvalgo.a`（由 `precompile_rvv.sh` 用 SpaceMIT 工具链生成），确保生成 RVV 指令。
- **其他架构**：CMake 自动改为编译 `vec_add.cpp` 等源文件，RVV 函数通过 `#ifdef __riscv_vector` 回退到标量实现。

### 数据流

```
用户点击「开始测试」
    ↓
ArkTS 调用 rvv.benchVecAdd(n)
    ↓
N-API 分配内存并初始化数据
    ↓
标量实现计时 → RVV 实现计时
    ↓
返回 { scalarNs, rvvNs } → UI 显示结果与加速比
```

---

## 代码结构

```
rvvdemo/
├── AppScope/                      # 应用全局配置
├── entry/src/main/
│   ├── ets/
│   │   ├── pages/
│   │   │   ├── Index.ets          # 主页（算法列表）
│   │   │   ├── VecAddPage.ets
│   │   │   ├── DotProductPage.ets
│   │   │   ├── MatVecMulPage.ets
│   │   │   └── RgbToGrayPage.ets
│   │   └── components/
│   │       └── BenchmarkCard.ets  # 性能卡片
│   └── cpp/
│       ├── napi_init.cpp          # N-API 入口
│       ├── vec_add.cpp
│       ├── dot_product.cpp
│       ├── mat_vec_mul.cpp
│       ├── rgb_to_gray.cpp
│       ├── bench_utils.h          # 纳秒计时
│       ├── CMakeLists.txt
│       └── prebuilt/librvvalgo.a  # RVV 静态库（已随仓库提供）
├── precompile_rvv.sh              # 重新生成静态库时使用
└── build-profile.json5
```

---

## 算法实现

### 1. 向量加法

`C[i] = A[i] + B[i]`

```cpp
// 标量
for (size_t i = 0; i < n; i++) c[i] = a[i] + b[i];

// RVV
for (size_t vl; n > 0; n -= vl, a += vl, b += vl, c += vl) {
    vl = __riscv_vsetvl_e32m8(n);
    vfloat32m8_t va = __riscv_vle32_v_f32m8(a, vl);
    vfloat32m8_t vb = __riscv_vle32_v_f32m8(b, vl);
    __riscv_vse32_v_f32m8(c, __riscv_vfadd_vv_f32m8(va, vb, vl), vl);
}
```

要点：`vsetvl` 动态设置向量长度，`m8` 使用 8 个向量寄存器组提高吞吐。

### 2. 点积

`sum = Σ(A[i] × B[i])`

```cpp
// RVV
vfloat32m8_t acc = __riscv_vfmv_v_f_f32m8(0.0f, vsetvlmax);
for (size_t vl; n > 0; n -= vl, a += vl, b += vl) {
    vl = __riscv_vsetvl_e32m8(n);
    vfloat32m8_t va = __riscv_vle32_v_f32m8(a, vl);
    vfloat32m8_t vb = __riscv_vle32_v_f32m8(b, vl);
    acc = __riscv_vfmacc_vv_f32m8(acc, va, vb, vl);   // FMA
}
vfloat32m1_t red = __riscv_vfredusum_vs_f32m8_f32m1(acc, vzero, vsetvlmax);
return __riscv_vfmv_f_s_f32m1_f32(red);
```

要点：`vfmacc` 融合乘加；`vfredusum` 做归约求和。

### 3. 矩阵向量乘

`y = A × x`，A 为 `rows × cols`

```cpp
for (size_t r = 0; r < rows; r++) {
    const float* row = A + r * cols;
    const float* xp = x;
    vfloat32m8_t acc = __riscv_vfmv_v_f_f32m8(0.0f, vsetvlmax);
    for (size_t n = cols, vl; n > 0; n -= vl, row += vl, xp += vl) {
        vl = __riscv_vsetvl_e32m8(n);
        acc = __riscv_vfmacc_vv_f32m8(acc,
            __riscv_vle32_v_f32m8(row, vl),
            __riscv_vle32_v_f32m8(xp, vl), vl);
    }
    y[r] = reduce_sum(acc);
}
```

每行独立计算，内层循环向量化。

### 4. RGB 转灰度

`Gray = 0.299R + 0.587G + 0.114B`

```cpp
const uint16_t wr = 77, wg = 150, wb = 29;   // 定点系数，和 = 256
for (size_t vl; n > 0; n -= vl, src += vl * 3, dst += vl) {
    vl = __riscv_vsetvl_e8m2(n);
    vuint8m2x3_t segs = __riscv_vlseg3e8_v_u8m2x3(src, vl);  // 交错 RGB 一次加载
    vuint8m2_t vr = __riscv_vget_v_u8m2x3_u8m2(segs, 0);
    vuint8m2_t vg = __riscv_vget_v_u8m2x3_u8m2(segs, 1);
    vuint8m2_t vb = __riscv_vget_v_u8m2x3_u8m2(segs, 2);
    vuint16m4_t acc = __riscv_vwmulu_vx_u16m4(vr, wr, vl);
    acc = __riscv_vwmaccu_vx_u16m4(acc, wg, vg, vl);
    acc = __riscv_vwmaccu_vx_u16m4(acc, wb, vb, vl);
    __riscv_vse8_v_u8m2(dst, __riscv_vnsrl_wx_u8m2(acc, 8, vl), vl);
}
```

要点：`vlseg3e8` 处理交错数据；`vwmulu/vwmaccu` 扩展到 u16 防溢出；`vnsrl` 窄化回 u8。

---

## 编译指南

### 环境要求

| 工具 | 说明 |
|------|------|
| DevEco Studio 5.0+ | 构建 HAP |
| OpenHarmony SDK 12 | API 12 |
| SpaceMIT 工具链 | 仅在重新生成 `librvvalgo.a` 时需要，需支持 `-march=rv64gcv` |
| Linux 环境 | 运行 `precompile_rvv.sh` |

### 构建步骤

```bash
git clone https://gitee.com/spacemit-openharmony/rvvdemo.git
cd rvvdemo
```

仓库已包含 `entry/src/main/cpp/prebuilt/librvvalgo.a`，通常直接用 DevEco Studio 打开、配置签名并构建即可：

```bash
node "<DevEco Studio 安装路径>/tools/hvigor/bin/hvigorw.js" \
  --mode module -p module=entry@default -p product=default \
  -p buildMode=release assembleHap --no-daemon
```

### 修改算法后重新生成静态库

1. 修改 `precompile_rvv.sh` 顶部的 `TC` 为本机工具链路径。
2. 执行脚本：

```bash
./precompile_rvv.sh
```

脚本以 `-target riscv64-unknown-linux-musl -march=rv64gcv -O2 -fPIC` 编译 4 个算法文件，并用 `llvm-ar` 打包为 `librvvalgo.a`。

### 验证 RVV 指令

```bash
llvm-objdump -d entry/src/main/cpp/prebuilt/librvvalgo.a | grep -E "vle32|vfadd|vfmacc"
```

### 性能参考

K3 Pico（OpenHarmony 6.1.0.35 / API 23）向量加法、1,000,000 个 float32：标量 3.44 ms，RVV 1.55 ms，加速比 2.22x。

---

## RVV Intrinsic 要点

- **VLEN / LMUL / SEW**：`vfloat32m8_t` 表示 32 位浮点、LMUL=8；LMUL 越大吞吐越高，但寄存器压力也越大。
- **VLEN-agnostic**：通过 `__riscv_vsetvl_e32m8(n)` 动态取得单次可处理的元素数，代码不依赖具体 VLEN。
- **标准循环模式**：

```cpp
for (size_t vl; n > 0; n -= vl, ptr += vl) {
    vl = __riscv_vsetvl_e32m8(n);
    // 向量操作
}
```

- **常用操作**：`vle32/vse32` 加载存储、`vfadd/vfmul` 算术、`vfmacc` 融合乘加、`vfredusum` 归约、`vlseg3e8` 分段加载、`vwmulu/vnsrl` 扩展与窄化。
- **优化建议**：优先使用 FMA；数据量大时用 m8；处理交错数据时用 `vlseg`；尽量让数据留在向量寄存器中。

---

## FAQ

**Q1: 为什么需要预编译 RVV 库？**

DevEco 默认 NDK 工具链不一定支持 `-march=rv64gcv`。用 SpaceMIT 工具链预编译可确保生成 RVV 指令；仓库已提供预编译产物，只有修改算法后才需要重跑脚本。

**Q2: 如何确认 RVV 生效？**

反汇编静态库看到 `vle32/vfadd` 等指令；或在设备上对比加速比，若 `scalarNs ≈ rvvNs`，说明运行的是标量回退实现。

**Q3: ARM64 设备能运行吗？**

可以。CMake 会编译标量实现，RVV 函数回退为普通循环，但没有加速效果。

**Q4: 如何调整测试规模？**

各页面的 `dataSize` 默认 1,000,000，可通过 Slider 在 10⁴ ~ 10⁷ 之间调节。

**Q5: 测试结果波动大？**

关闭后台应用、多次运行取平均、增大数据规模以减少计时误差。

**Q6: 如何新增算法？**

1. 在 `entry/src/main/cpp/` 新增 `xxx.cpp`，导出 `xxx_run_scalar` / `xxx_run_rvv`。
2. 在 `napi_init.cpp` 注册 N-API 方法。
3. 新增 ArkTS 页面调用该方法。
4. 修改 `precompile_rvv.sh`，把新文件加入编译与打包列表。

**Q7: 报错 `undefined reference to __riscv_vle32_v_f32m8`？**

确认 `librvvalgo.a` 存在且已重新生成；清理构建缓存后重试。

---

## 参考资料

- [RISC-V Vector Extension Specification](https://github.com/riscvarchive/riscv-v-spec)
- [RVV Intrinsic API Reference](https://github.com/riscv-non-isa/riscv-rvv-intrinsic-doc)
- [OpenHarmony 开发文档](https://docs.openharmony.cn/)

## 许可证

本项目仅供学习和研究使用。

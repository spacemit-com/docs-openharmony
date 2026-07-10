<!--
 * Copyright 2022-2023 SPACEMIT. All rights reserved.
 * Use of this source code is governed by a BSD-style license
 * that can be found in the LICENSE file.
 * 
 * @Author: David(qiang.fu@spacemit.com)
 * @Date: 2026-03-04 11:39:35
 * @LastEditTime: 2026-05-13 16:19:38
 * @FilePath: \doc\docs-openharmony\zh\ai_application\3_OH_AI_rvvdemo.md
 * @Description: 
-->
sidebar_position: 3

# RVV应用说明

## 项目简介

RVV Demo 是一个基于 OpenHarmony 的原生应用，用于展示 RISC-V Vector Extension (RVV) 在常见算法中的性能优势。应用通过对比标量实现和 RVV 向量化实现的执行时间，直观展示向量化加速效果。

## 平台支持情况

|      平台 & 系统       |       是否支持     |
|-----------------------|-----------------------|
| K1 OpenHarmony5.0     | ✅ 支持               |
| K1 OpenHarmony6.1    | ✅ 支持             |
| K3 OpenHarmony6.1     | ✅ 支持              |

### 核心功能

- **向量加法 (Vector Add)**: 两个浮点数组逐元素相加
- **点积 (Dot Product)**: 计算两个向量的内积
- **矩阵向量乘 (Matrix-Vector Multiply)**: 矩阵与向量相乘
- **RGB 转灰度 (RGB to Grayscale)**: 图像颜色空间转换

每个算法都提供标量和 RVV 两种实现，并实时显示性能对比和加速比。

---

## 技术栈

### 前端技术

- **ArkTS**: OpenHarmony 官方应用开发语言（TypeScript 超集）
- **ArkUI**: 声明式 UI 框架
- **N-API**: 原生模块接口，用于 ArkTS 调用 C++ 代码

### 后端技术

- **C++17**: 算法实现语言
- **RISC-V Vector Intrinsics**: RVV 1.0 标准内联函数
- **CMake**: 构建系统

### 开发工具

- **DevEco Studio**: OpenHarmony 官方 IDE
- **SpacemiT Toolchain**: RISC-V 交叉编译工具链（支持 RVV）
- **Hvigor**: OpenHarmony 构建工具

### 目标平台

- **OpenHarmony 12**: 操作系统
- **RISC-V 64**: 目标架构（rv64gcv）
- **ARM64/x86_64**: 备用架构（标量实现）

---

## 架构设计

### 整体架构

```
┌─────────────────────────────────────────────────────────┐
│                    ArkTS UI Layer                       │
│  (Index.ets, VecAddPage.ets, DotProductPage.ets, ...)  │
└────────────────────┬────────────────────────────────────┘
                     │ N-API
                     ▼
┌─────────────────────────────────────────────────────────┐
│              Native Module (napi_init.cpp)              │
│  - benchVecAdd()    - benchDotProduct()                 │
│  - benchMatVecMul() - benchRgbToGray()                  │
└────────────────────┬────────────────────────────────────┘
                     │
        ┌────────────┴────────────┐
        ▼                         ▼
┌──────────────────┐    ┌──────────────────┐
│  Scalar Impl     │    │   RVV Impl       │
│  (C++ for loop)  │    │  (Intrinsics)    │
└──────────────────┘    └──────────────────┘
```

### 编译策略

项目采用**双编译策略**以支持多架构：

1. **RISC-V 平台**:
   - 使用 SpacemiT 工具链预编译 RVV 代码为静态库 `librvvalgo.a`
   - DevEco Studio 构建时链接该静态库
   - 启用 `-march=rv64gcv` 编译选项

2. **ARM64/x86_64 平台**:
   - 直接编译标量实现的 `.cpp` 文件
   - RVV 函数回退到标量版本（通过 `#ifdef __riscv_vector`）

### 数据流

```
用户点击"开始测试"
    ↓
ArkTS 调用 rvv.benchVecAdd(n)
    ↓
N-API 层分配内存并初始化数据
    ↓
调用 vec_add_run_scalar() → 计时 → 返回耗时 (ns)
    ↓
调用 vec_add_run_rvv()    → 计时 → 返回耗时 (ns)
    ↓
返回 {scalarNs, rvvNs} 到 ArkTS
    ↓
UI 更新显示结果和加速比
```

---

## 代码结构

```
rvvdemo/
├── AppScope/                      # 应用全局配置
│   ├── app.json5                  # 应用元数据
│   └── resources/                 # 全局资源
├── entry/                         # 主模块
│   ├── src/main/
│   │   ├── ets/                   # ArkTS 源码
│   │   │   ├── pages/             # 页面
│   │   │   │   ├── Index.ets      # 主页（算法列表）
│   │   │   │   ├── VecAddPage.ets # 向量加法页面
│   │   │   │   ├── DotProductPage.ets
│   │   │   │   ├── MatVecMulPage.ets
│   │   │   │   └── RgbToGrayPage.ets
│   │   │   └── components/        # 组件
│   │   │       └── BenchmarkCard.ets  # 性能卡片组件
│   │   ├── cpp/                   # C++ 原生代码
│   │   │   ├── napi_init.cpp      # N-API 入口
│   │   │   ├── vec_add.cpp        # 向量加法实现
│   │   │   ├── dot_product.cpp    # 点积实现
│   │   │   ├── mat_vec_mul.cpp    # 矩阵向量乘实现
│   │   │   ├── rgb_to_gray.cpp    # RGB 转灰度实现
│   │   │   ├── bench_utils.h      # 计时工具
│   │   │   ├── CMakeLists.txt     # CMake 构建脚本
│   │   │   └── prebuilt/          # 预编译产物
│   │   │       └── librvvalgo.a   # RVV 静态库
│   │   └── resources/             # 资源文件
│   └── build-profile.json5        # 模块构建配置
├── build-profile.json5            # 项目构建配置
├── precompile_rvv.sh              # RVV 预编译脚本
└── README.md
```

---

## 算法实现

### 1. 向量加法 (vec_add.cpp)

**功能**: 计算 `C[i] = A[i] + B[i]`

**标量实现**:
```cpp
for (size_t i = 0; i < n; i++) 
    c[i] = a[i] + b[i];
```

**RVV 实现**:
```cpp
for (size_t vl; n > 0; n -= vl, a += vl, b += vl, c += vl) {
    vl = __riscv_vsetvl_e32m8(n);           // 设置向量长度
    vfloat32m8_t va = __riscv_vle32_v_f32m8(a, vl);  // 加载 A
    vfloat32m8_t vb = __riscv_vle32_v_f32m8(b, vl);  // 加载 B
    vfloat32m8_t vc = __riscv_vfadd_vv_f32m8(va, vb, vl); // 向量加法
    __riscv_vse32_v_f32m8(c, vc, vl);       // 存储结果
}
```

**关键技术**:
- `vsetvl`: 动态设置向量长度（VLEN-agnostic）
- `vle32/vse32`: 向量加载/存储
- `vfadd_vv`: 向量-向量浮点加法
- `m8`: LMUL=8，使用 8 个向量寄存器组

---

### 2. 点积 (dot_product.cpp)

**功能**: 计算 `sum = Σ(A[i] × B[i])`

**标量实现**:
```cpp
float sum = 0.0f;
for (size_t i = 0; i < n; i++) 
    sum += a[i] * b[i];
```

**RVV 实现**:
```cpp
vfloat32m8_t acc = __riscv_vfmv_v_f_f32m8(0.0f, vsetvlmax);  // 初始化累加器
for (size_t vl; n > 0; n -= vl, a += vl, b += vl) {
    vl = __riscv_vsetvl_e32m8(n);
    vfloat32m8_t va = __riscv_vle32_v_f32m8(a, vl);
    vfloat32m8_t vb = __riscv_vle32_v_f32m8(b, vl);
    acc = __riscv_vfmacc_vv_f32m8(acc, va, vb, vl);  // FMA: acc += va * vb
}
// 归约求和
vfloat32m1_t red = __riscv_vfredusum_vs_f32m8_f32m1(acc, vzero, vsetvlmax);
return __riscv_vfmv_f_s_f32m1_f32(red);  // 提取标量
```

**关键技术**:
- `vfmacc`: 融合乘加（Fused Multiply-Accumulate）
- `vfredusum`: 向量归约求和
- `vfmv_f_s`: 向量到标量转换

---

### 3. 矩阵向量乘 (mat_vec_mul.cpp)

**功能**: 计算 `y = A × x`，其中 A 是 (rows × cols) 矩阵

**标量实现**:
```cpp
for (size_t r = 0; r < rows; r++) {
    float sum = 0.0f;
    for (size_t c = 0; c < cols; c++) 
        sum += A[r * cols + c] * x[c];
    y[r] = sum;
}
```

**RVV 实现**:
```cpp
for (size_t r = 0; r < rows; r++) {
    const float* row = A + r * cols;
    vfloat32m8_t acc = __riscv_vfmv_v_f_f32m8(0.0f, vsetvlmax);
    size_t n = cols;
    for (size_t vl; n > 0; n -= vl, row += vl, x += vl) {
        vl = __riscv_vsetvl_e32m8(n);
        vfloat32m8_t va = __riscv_vle32_v_f32m8(row, vl);
        vfloat32m8_t vx = __riscv_vle32_v_f32m8(x, vl);
        acc = __riscv_vfmacc_vv_f32m8(acc, va, vx, vl);
    }
    y[r] = reduce_sum(acc);  // 归约
}
```

**优化点**:
- 每行独立计算，便于并行
- 内层循环向量化，减少迭代次数

---

### 4. RGB 转灰度 (rgb_to_gray.cpp)

**功能**: `Gray = 0.299R + 0.587G + 0.114B`

**标量实现**:
```cpp
for (size_t i = 0; i < pixels; i++) {
    uint32_t r = rgb[i * 3 + 0];
    uint32_t g = rgb[i * 3 + 1];
    uint32_t b = rgb[i * 3 + 2];
    gray[i] = (r * 299 + g * 587 + b * 114 + 500) / 1000;
}
```

**RVV 实现**:
```cpp
const uint16_t wr = 77, wg = 150, wb = 29;  // 定点系数 (sum=256)
for (size_t vl; n > 0; n -= vl, src += vl * 3, dst += vl) {
    vl = __riscv_vsetvl_e8m2(n);
    // 分段加载 RGB
    vuint8m2x3_t segs = __riscv_vlseg3e8_v_u8m2x3(src, vl);
    vuint8m2_t vr = __riscv_vget_v_u8m2x3_u8m2(segs, 0);
    vuint8m2_t vg = __riscv_vget_v_u8m2x3_u8m2(segs, 1);
    vuint8m2_t vb = __riscv_vget_v_u8m2x3_u8m2(segs, 2);
    // 加权求和（扩展到 u16）
    vuint16m4_t acc = __riscv_vwmulu_vx_u16m4(vr, wr, vl);
    acc = __riscv_vwmaccu_vx_u16m4(acc, wg, vg, vl);
    acc = __riscv_vwmaccu_vx_u16m4(acc, wb, vb, vl);
    // 右移 8 位并窄化到 u8
    vuint8m2_t result = __riscv_vnsrl_wx_u8m2(acc, 8, vl);
    __riscv_vse8_v_u8m2(dst, result, vl);
}
```

**关键技术**:
- `vlseg3e8`: 分段加载（Segment Load），一次加载交错的 RGB 数据
- `vwmulu/vwmaccu`: 扩展乘法（u8 → u16），避免溢出
- `vnsrl`: 窄化右移（u16 → u8）

---

## 编译指南

### 前置条件

1. **DevEco Studio 5.0+**
   - 下载地址: https://developer.huawei.com/consumer/cn/deveco-studio/
   - 安装 OpenHarmony SDK 12

2. **SpacemiT RISC-V 工具链**（仅 RISC-V 平台需要）
   - 路径示例: `/data/home/fuqiang/workspace/ai/toolchain/spacemit-toolchain-linux-musl-x86_64-for-oh-20250425`
   - 包含 `clang++` 和 `llvm-ar`，支持 `-march=rv64gcv`

3. **Linux 开发环境**（用于预编译 RVV 库）

### 编译步骤

#### 方式一：RISC-V 平台（推荐）

**Step 0: 克隆项目**

```bash
git clone https://gitee.com/spacemit-openharmony/rvvdemo.git
cd rvvdemo
```

**Step 1: 预编译 RVV 静态库**

```bash
./precompile_rvv.sh
```

脚本会：
- 使用 SpacemiT 工具链编译 4 个算法文件
- 生成 `entry/src/main/cpp/prebuilt/librvvalgo.a`
- 编译选项: `-march=rv64gcv -O2 -fPIC`

**Step 2: 在 DevEco Studio 中构建**

1. 打开项目
2. 选择设备: RISC-V 真机或模拟器
3. 点击 **Run** 或 **Build > Build Hap(s)**
4. CMake 会自动检测 `CMAKE_SYSTEM_PROCESSOR` 为 `riscv` 并链接静态库

#### 方式二：ARM64/x86_64 平台

直接在 DevEco Studio 中构建，无需预编译：

1. CMake 检测到非 RISC-V 架构
2. 编译 `vec_add.cpp` 等源文件（标量实现）
3. RVV 函数自动回退到标量版本

### CMakeLists.txt 逻辑

```cmake
if(CMAKE_SYSTEM_PROCESSOR MATCHES "riscv" AND EXISTS "${PREBUILT_LIB}")
    # RISC-V: 链接预编译的 RVV 库
    target_link_libraries(rvvdemo PRIVATE "${PREBUILT_LIB}" ...)
else()
    # 其他架构: 编译标量实现
    target_sources(rvvdemo PRIVATE vec_add.cpp dot_product.cpp ...)
endif()
```

### 验证编译

```bash
# 检查静态库是否包含 RVV 指令
llvm-objdump -d entry/src/main/cpp/prebuilt/librvvalgo.a | grep -E "vle32|vfadd"

# 输出示例:
#   1234:  vle32.v  v8, (a0)
#   1238:  vfadd.vv v8, v8, v16
```

---

## RVV Intrinsic 编程指南

### 基础概念

#### 1. 向量寄存器

- **VLEN**: 向量寄存器位宽（硬件相关，如 128/256/512 位）
- **LMUL**: 寄存器组倍数（m1/m2/m4/m8），控制每次操作的数据量
- **SEW**: 元素位宽（e8/e16/e32/e64）

示例：`vfloat32m8_t` 表示：
- `float32`: 32 位浮点数
- `m8`: LMUL=8，使用 8 个寄存器组

#### 2. 向量长度 (VL)

RVV 采用 **VLEN-agnostic** 设计，通过 `vsetvl` 动态设置：

```cpp
size_t vl = __riscv_vsetvl_e32m8(n);  // 返回实际处理的元素数
```

- 当 `n >= VLEN/32*8` 时，`vl = VLEN/32*8`（满载）
- 当 `n < VLEN/32*8` 时，`vl = n`（处理剩余元素）

### 常用 Intrinsic 函数

#### 加载/存储

```cpp
// 加载
vfloat32m8_t v = __riscv_vle32_v_f32m8(ptr, vl);  // 从内存加载
vuint8m2x3_t segs = __riscv_vlseg3e8_v_u8m2x3(ptr, vl);  // 分段加载（交错数据）

// 存储
__riscv_vse32_v_f32m8(ptr, v, vl);  // 存储到内存
```

#### 算术运算

```cpp
// 向量-向量
vfloat32m8_t vc = __riscv_vfadd_vv_f32m8(va, vb, vl);  // 加法
vfloat32m8_t vc = __riscv_vfmul_vv_f32m8(va, vb, vl);  // 乘法

// 向量-标量
vfloat32m8_t vc = __riscv_vfadd_vf_f32m8(va, scalar, vl);

// 融合乘加
vfloat32m8_t acc = __riscv_vfmacc_vv_f32m8(acc, va, vb, vl);  // acc += va * vb
```

#### 归约操作

```cpp
// 求和归约
vfloat32m1_t sum = __riscv_vfredusum_vs_f32m8_f32m1(v, vzero, vl);
float result = __riscv_vfmv_f_s_f32m1_f32(sum);  // 提取标量
```

#### 类型转换

```cpp
// 扩展（u8 → u16）
vuint16m4_t wide = __riscv_vwmulu_vx_u16m4(v_u8, scalar, vl);

// 窄化（u16 → u8）
vuint8m2_t narrow = __riscv_vnsrl_wx_u8m2(v_u16, shift, vl);
```

### 编程模式

#### 标准循环模式

```cpp
for (size_t vl; n > 0; n -= vl, ptr += vl) {
    vl = __riscv_vsetvl_e32m8(n);  // 设置 VL
    // ... 向量操作 ...
}
```

#### 归约模式

```cpp
vfloat32m8_t acc = __riscv_vfmv_v_f_f32m8(0.0f, __riscv_vsetvlmax_e32m8());
for (size_t vl; n > 0; n -= vl, ptr += vl) {
    vl = __riscv_vsetvl_e32m8(n);
    vfloat32m8_t v = __riscv_vle32_v_f32m8(ptr, vl);
    acc = __riscv_vfadd_vv_f32m8(acc, v, vl);  // 累加
}
// 最后归约
vfloat32m1_t sum = __riscv_vfredusum_vs_f32m8_f32m1(acc, vzero, vsetvlmax);
```

### 性能优化技巧

1. **选择合适的 LMUL**
   - 数据量大 → 使用 m8（最大吞吐）
   - 寄存器压力大 → 使用 m1/m2

2. **使用 FMA 指令**
   - `vfmacc` 比 `vfmul + vfadd` 更快

3. **利用分段加载**
   - 处理交错数据（如 RGB）时使用 `vlseg`

4. **避免标量-向量转换**
   - 尽量保持数据在向量寄存器中

### 调试技巧

```cpp
// 打印向量内容（调试用）
vfloat32m8_t v = ...;
float tmp[256];
__riscv_vse32_v_f32m8(tmp, v, vl);
for (int i = 0; i < vl; i++) printf("%f ", tmp[i]);
```

---

## FAQ

### Q1: 为什么需要预编译 RVV 库？

**A**: DevEco Studio 默认的 NDK 工具链可能不支持 `-march=rv64gcv` 选项。使用 SpacemiT 工具链预编译可以确保 RVV 指令正确生成。

### Q2: 如何验证 RVV 指令是否生效？

**A**: 
```bash
# 方法 1: 反汇编静态库
llvm-objdump -d librvvalgo.a | grep -E "vle|vse|vfadd"

# 方法 2: 在设备上运行，对比加速比
# 如果 scalarNs ≈ rvvNs，说明 RVV 未启用
```

### Q3: ARM64 设备上能运行吗？

**A**: 可以。CMake 会自动编译标量实现，RVV 函数回退到 for 循环。但无法体验加速效果。

### Q4: 如何调整测试数据规模？

**A**: 在各个页面（如 `VecAddPage.ets`）中，通过 Slider 组件调整 `dataSize`：
```typescript
@State dataSize: number = 1000000;  // 默认 100 万元素
```

### Q5: 性能测试不准确怎么办？

**A**: 
- 关闭后台应用，减少干扰
- 多次运行取平均值
- 增大数据规模（如 1000 万），减少计时误差

### Q6: 如何添加新的算法？

**Step 1**: 在 `entry/src/main/cpp/` 下创建 `new_algo.cpp`：
```cpp
extern "C" int64_t new_algo_run_scalar(...) { /* 实现 */ }
extern "C" int64_t new_algo_run_rvv(...) { /* 实现 */ }
```

**Step 2**: 在 `napi_init.cpp` 中注册：
```cpp
static napi_value BenchNewAlgo(napi_env env, napi_callback_info info) { /* ... */ }
// 添加到 descs 数组
{"benchNewAlgo", nullptr, BenchNewAlgo, ...}
```

**Step 3**: 创建 ArkTS 页面 `NewAlgoPage.ets`，调用 `rvv.benchNewAlgo()`

**Step 4**: 更新 `precompile_rvv.sh`，添加编译命令

### Q7: 如何修改工具链路径？

**A**: 编辑 `precompile_rvv.sh`：
```bash
TC=/your/custom/toolchain/path
```

### Q8: 编译时报错 "undefined reference to `__riscv_vle32_v_f32m8`"

**A**: 
- 检查是否运行了 `precompile_rvv.sh`
- 确认 `librvvalgo.a` 存在于 `entry/src/main/cpp/prebuilt/`
- 清理构建缓存：DevEco Studio → Build → Clean Project

### Q9: 如何在模拟器上测试？

**A**: 
- 使用 QEMU RISC-V 模拟器（需支持 RVV 扩展）
- 或使用 ARM64 模拟器（仅测试功能，无加速效果）

### Q10: 代码中的 `bench_utils.h` 是做什么的？

**A**: 提供高精度计时函数 `now_ns()`，使用 `clock_gettime(CLOCK_MONOTONIC)` 获取纳秒级时间戳。

---

## 参考资料

- [RISC-V Vector Extension Specification](https://github.com/riscv/riscv-v-spec)
- [RVV Intrinsic API Reference](https://github.com/riscv-non-isa/rvv-intrinsic-doc)
- [OpenHarmony 开发文档](https://docs.openharmony.cn/)
- [ArkTS 语言规范](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/arkts-get-started-V5)

---

## 许可证

本项目仅供学习和研究使用。

## 联系方式

如有问题，请提交 Issue 或联系项目维护者。
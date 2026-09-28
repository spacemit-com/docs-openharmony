<!--
 * Copyright 2022-2023 SPACEMIT. All rights reserved.
 * Use of this source code is governed by a BSD-style license
 * that can be found in the LICENSE file.
 * 
 * @Author: David(qiang.fu@spacemit.com)
 * @Date: 2026-03-04 11:39:35
 * @LastEditTime: 2026-06-06 13:44:43
 * @FilePath: \doc\docs-openharmony\zh\skills\1_camera_skill.md
 * @Description: 
-->
sidebar_position: 1

# OH6.1 鸿蒙浏览器开发环境搭建

> 本文是可独立使用的 Codex skill 文档，也可作为人工开发指南。调用名：`$oh61-harmonyos-browser-developer`。
>
> 适用范围：Chromium 132、OpenHarmony 6.1 LTS、`riscv64` ArkWeb/NWeb。
>
> 重要边界：设备系统 HAP 安装、ArkWebCore 替换、远程 push 和厂商特权部署属于外部变更；执行前确认目标、备份和权限。不要绕过签名校验、删除设备数据或上传私钥。

## 使用方式

先确认当前阶段和已有条件：宿主系统、Linux 工作区、源码、SDK、目标架构、`hdc` 设备、浏览器包名/Ability、证书。按下文阶段逐步执行，每阶段完成后检查结果再继续。

**Skill 元数据**

- 显示名称：OH6.1 鸿蒙浏览器开发
- 简短描述：指导 Chromium 132/OH6.1 riscv64 ArkWeb 浏览器开发、构建、部署与验证
- 默认提示语：`请按 OH6.1 鸿蒙浏览器开发流程，先检查我的环境和当前阶段，再执行或指导下一步。`

## 组件关系

浏览器应用通过 ArkWeb API 使用系统 ArkWebCore，不直接启动 Chromium：

```text
ArkUI 应用
  └─ Web/ArkWeb API
      └─ com.ohos.arkwebcore（ArkWebCore/NWeb HAP）
          └─ Chromium/CEF（Blink、V8、网络、GPU、媒体）
```

`NWeb-riscv64.hap` 是内核 HAP；浏览器应用 HAP（如 `entry-default-signed.hap`）是 UI 应用。不要执行 `aa start -b com.ohos.arkwebcore`，应启动真实浏览器应用；Ability 以 `bm dump` 输出为准。

# OH6.1 Chromium/ArkWeb 操作手册

本文件整理了按阶段调用的可执行步骤。命令中的路径、主机、用户名、包名、Ability 和证书必须替换为用户实际值；示例值不能直接照用。

## 1. 前置检查

目标组合：Chromium 132、OpenHarmony 6.1 LTS、`riscv64`。建议 Linux 编译机为 Ubuntu 22.04 x86_64，至少 16 CPU 线程、64 GB 内存和 250 GB 可用磁盘；Windows 测试机准备 `hdc.exe` 和 OpenSSH `scp`。设备需开启 USB 调试并能被 `hdc` 识别。

Linux 依赖（这是主机变更，执行前告知用户）：

```bash
sudo apt-get update
sudo apt-get install -y \
  ca-certificates git git-lfs repo python3 python3-pip python-is-python3 \
  ninja-build pkg-config gperf bison flex cmake build-essential binutils \
  unzip zip rsync curl wget xz-utils file patch jq \
  libnss3-dev libnspr4-dev libdbus-1-dev libgtk-3-dev libglib2.0-dev \
  libpango1.0-dev libatk1.0-dev libcairo2-dev libgdk-pixbuf2.0-dev \
  libxcomposite-dev libxdamage-dev libxext-dev libxfixes-dev \
  libxrandr-dev libxtst-dev libopenjp2-7-dev libstdc++-11-dev \
  default-jdk-headless
git lfs install
pkg-config --modversion nss nspr
python --version
python3 --version
ninja --version
nm --version
java -version
```

构建优先使用源码树内置 Node.js、OHOS LLVM 和 Rust，不要让主机工具链覆盖它们。

## 2. 获取完整源码

```bash
mkdir -p ~/WorkSpace2/Projects/oh61-lts
cd ~/WorkSpace2/Projects/oh61-lts
repo init \
  -u https://code.ruyicommunity.cn/risc-verse/ruyi-desktop-os/manifest.git \
  -b dev-132_trunk_6.1-Release -m developer.xml \
  --no-repo-verify --no-clone-bundle
mkdir -p .repo/local_manifests
```

创建 `.repo/local_manifests/chromium-lts.xml`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <extend-project name="chromium_src" path="src" revision="dev-132_trunk_6.1-LTS" />
  <extend-project name="chromium_third_party" path="src/third_party"
                  revision="6f2c89d8e652648101603f0a57388ca1befad043" />
</manifest>
```

然后同步：

```bash
repo sync -c -j8 --no-clone-bundle --retry-fetches=5
git -C src log -1 --oneline
git -C src branch -r --contains HEAD | grep dev-132_trunk_6.1-LTS
git -C src lfs pull && git -C src lfs checkout && git -C src lfs fsck
test -f src/third_party/ohos_nweb_hap/BUILD.gn
test -f src/third_party/ohos_ndk/README.md
test -x src/third_party/node/linux/node-linux-x64/bin/node
test "$(wc -c < src/third_party/icu/common/icudtl.dat)" -gt 1000000
```

网络中断时重复 `repo sync`；不要删除 `.repo`。文件仍以 `version https://git-lfs.github.com/spec/v1` 开头时，先完成 LFS 拉取。

## 3. 下载并安装 OHOS SDK

下载完整归档（不要拼接单个压缩包 URL）：

`https://polyos.iscas.ac.cn/downloads/chromium/dev-132_trunk_6.1-Release/ohos_sdk/ohos_sdk.tar.xz`

```bash
cd ~/WorkSpace2/Projects/oh61-lts
mkdir -p downloads/ohos_sdk && cd downloads/ohos_sdk
wget --continue --timeout=30 --tries=20 --waitretry=10 \
  -O ohos_sdk.tar.xz.part \
  https://polyos.iscas.ac.cn/downloads/chromium/dev-132_trunk_6.1-Release/ohos_sdk/ohos_sdk.tar.xz
mv ohos_sdk.tar.xz.part ohos_sdk.tar.xz
sha256sum ohos_sdk.tar.xz
mkdir extracted && tar -xJf ohos_sdk.tar.xz -C extracted
test -d extracted/ohos_sdk/18
test -d extracted/ohos_sdk/openharmony
test -d extracted/ohos_sdk/rust-toolchain
```

将归档内 `18`、`ohos-sdk`、`openharmony`、`rust-toolchain`、`OpenHarmonyApplication.pem` 移入 `src/ohos_sdk`。目标目录已有内容时不要覆盖，改用干净工作区或经用户确认后备份旧目录。若 `src/ohos_sdk/.install` 存在，将其移出 SDK 目录。最后建立 `src/third_party/rust-toolchain -> ../ohos_sdk/rust-toolchain`，并检查：

```bash
src/ohos_sdk/openharmony/native/llvm/bin/clang --version | head -3
src/ohos_sdk/rust-toolchain/bin/rustc --version
test -f src/ohos_sdk/18/toolchains/lib/hap-sign-tool.jar
test -f src/ohos_sdk/18/toolchains/lib/OpenHarmony.p12
test -f src/ohos_sdk/18/toolchains/lib/OpenHarmonyProfileRelease.pem
test -f src/ohos_sdk/OpenHarmonyApplication.pem
```

## 4. 构建、签名和产物检查

```bash
cd ~/WorkSpace2/Projects/oh61-lts
nice -n 15 ionice -c 3 ./build_arkweb.sh -j 8 -t w -A riscv64
./sign.sh riscv64
test -s src/out/riscv64/NWeb-riscv64.hap
unzip -t src/out/riscv64/NWeb-riscv64.hap
sha256sum src/out/riscv64/NWeb-riscv64.hap
md5sum src/out/riscv64/NWeb-riscv64.hap
```

未签名产物通常为 `src/out/riscv64/ohos_nweb.hap`，签名产物为 `src/out/riscv64/NWeb-riscv64.hap`。必须使用设备认可证书；生产设备使用厂商签名流程。

## 5. Windows 传输、备份和安装

先替换 Linux 用户、构建主机和工作区：

```powershell
$windowsWorkspace = Join-Path $env:USERPROFILE "oh"
New-Item -ItemType Directory -Force $windowsWorkspace | Out-Null
Set-Location $windowsWorkspace
$linuxUser = "your-linux-user"
$buildHost = "your-build-host"
$workspace = "/path/to/oh61-lts"
scp "${linuxUser}@${buildHost}:${workspace}/src/out/riscv64/NWeb-riscv64.hap" .\NWeb-riscv64.hap
Get-FileHash .\NWeb-riscv64.hap -Algorithm SHA256
.\hdc.exe list targets
.\hdc.exe shell uname -a
.\hdc.exe shell bm dump -n com.ohos.arkwebcore
.\hdc.exe shell find /system /vendor /data/app -type f -iname '*arkwebcore*.hap'
```

从 `find` 输出选择实际路径，再备份并记录哈希：

```powershell
$arkwebPath = "/path/returned/by/the/previous/command"
.\hdc.exe file recv $arkwebPath .\ArkWebCore-before.hap
Get-FileHash .\ArkWebCore-before.hap -Algorithm SHA256
```

安装系统 HAP 前必须得到用户确认：

```powershell
.\hdc.exe install -r .\NWeb-riscv64.hap
.\hdc.exe shell bm dump -n com.ohos.arkwebcore
```

如需安装浏览器应用 HAP，使用实际包名查询 Ability：

```powershell
.\hdc.exe install -r .\entry-default-signed.hap
.\hdc.exe shell bm dump -n ohos.samples.browser
```

签名、权限或特权错误时停止，不绕过校验；改走设备厂商系统镜像/特权部署流程。

## 6. 启动和验证

Ability 以 `bm dump` 输出为准：

```powershell
.\hdc.exe shell aa force-stop ohos.samples.browser
.\hdc.exe shell aa start -a MainAbility -b ohos.samples.browser
```

验证地址栏软键盘、字符输入、访问 `https://www.bilibili.com/`、页面有内容且进程稳定。视频播放期间检查硬件解码：

```powershell
.\hdc.exe shell ls /sys/kernel/debug/amvx_if0/session/
```

非空会话名表示 AMVX 会话已建立；停止播放后目录为空是正常可能情况。目录不存在/为空时确认视频正在播放、debugfs 已挂载，并结合 MediaCodec/AVCodec 日志。可选日志：

```powershell
.\hdc.exe hilog -x > .\hilog-browser.txt
```

页面/GPU 问题搜索 `CreateNativeViewGLSurfaceEGLOhos`、`SwapBuffers`；硬解问题搜索 `MediaCodec`、`AVCodec`。

## 7. 可选源码提交

仅用户明确要求时执行。只上传公钥，绝不上传私钥：

```bash
ssh-keygen -t ed25519 -a 100 -f ~/.ssh/id_ed25519_ruyi -C "developer@build-host"
cat ~/.ssh/id_ed25519_ruyi.pub
ssh -T git@code.ruyicommunity.cn
cd ~/WorkSpace2/Projects/oh61-lts/src
git remote add upstream git@code.ruyicommunity.cn:risc-verse/ruyi-desktop-os/chromium_src.git
git remote add fork git@code.ruyicommunity.cn:your-group/chromium_src.git
git fetch upstream --prune
git switch --create feature/my-change upstream/dev-132_trunk_6.1-LTS
git status --short
git diff --check
git add path/to/changed_source.cc path/to/changed_source.h
git diff --cached --check
git commit -s -m 'OH6.1: describe the change'
git push -u fork HEAD:refs/heads/feature/my-change
```

MR 目标项目为 `risc-verse/ruyi-desktop-os/chromium_src`，目标分支为 `dev-132_trunk_6.1-LTS`。不提交 SDK、`out/`、HAP、日志、临时压缩包或私钥。

## 8. 常见故障

- `Package nss was not found`：安装 `libnss3-dev libnspr4-dev`，检查 `pkg-config --modversion nss nspr`。
- `check_ndk.py` 在 `nm -CD libarkweb_engine.so` 失败：检查 `command -v nm`、`nm --version`，安装 `binutils` 后重跑原 Ninja 目标；不要跳过 `check_ndk_symbol`。
- `specified ability does not exist`：用真实 bundle 名执行 `bm dump -n <bundleName>`，按输出使用真实 Ability；不要启动 ArkWebCore。
- HAP 签名/权限失败：使用设备认可证书或厂商特权部署流程，不绕过校验。
- 页面空白或浏览器退出：采集 `hilog -x`，分别检查网络、GPU/渲染进程和媒体解码日志。
- 网络中断：重复 `repo sync` 或下载命令；不要删除 `.repo`，不要从不完整 LFS 文件继续构建。

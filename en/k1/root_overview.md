sidebar_position: 1

# 1. K1 OH5.0简介

## 修订记录

| 修订版本 | 修订日期   | 修订说明                     |
|----------|------------|------------------------------|
| 001      | 2025-01-20 | 初始版本                     |
| 002      | 2025-03-18 | 添加FAQ                      |
| 003      | 2025-04-11 | 添加Muse Pi Pro              |
| 004      | 2025-05-28 | 添加更新说明                 |
| 005      | 2025-07-16 | 添加产品包框图               |
| 006      | 2025-09-10 | 添加OH5.0 V1.0RC2.4版本信息  |

## 1. 概述
`K1 OH5.0` 是由进迭时空推出的全球首个 `RISC-V` + `OpenHarmony5.0` 原生鸿蒙解决方案，CPU 采用新 `RISC-V AI CPU` 芯片 `K1`，操作系统采用 `OpenHarmony5.0.0.71`。同时，基于 RISC-V 同构融合 AI 算力和开源大模型，构建了新 AI 算力平台。方案已经经过完整的系统集成测试和可靠性测试，并提供了全套开发文档，量产工具和原厂支持通道，全方位助力开发者轻松上手，高效开发。

发布产品包框图如下所示：

![](./static/oh_structure.png)

## 2. 开发包发布清单

<table>
<tbody>
<tr>
<td>类别</td>
<td>文档名称</td>
<td>备注/下载地址</td>
</tr>
<tr>
<td rowspan=11 colspan=1>软件</td>
<td>K1 OH5.0下载编译烧录说明</td>
<td></td>
</tr>
<tr>
<td>K1 OH5.0方案添加移植说明</td>
<td></td>
</tr>
<tr>
<td>K1 OH5.0系统定制说明</td>
<td></td>
</tr>
<tr>
<td>K1 OH5.0系统调试说明</td>
<td></td>
</tr>
<tr>
<td>K1 OH5.0应用开发说明</td>
<td></td>
</tr>
<tr>
<td>K1 OH5.0 AI构建开发说明</td>
<td></td>
</tr>
<tr>
<td>K1 OH5.0驱动开发说明</td>
<td></td>
</tr>
<tr>
<td>K1 OH5.0方案OTA使用说明</td>
<td></td>
</tr>
<tr>
<td>K1 OH5.0三方包移植说明</td>
<td></td>
</tr>
<tr>
<td>K1 OH5.0工程测试应用使用说明</td>
<td></td>
</tr>
<tr>
<td rowspan=4 colspan=1>硬件</td>
<td>MUSE Paper方案框图</td>
<td rowspan=2 colspan=1><a href="https://cdn-resource.spacemit.com/file/product/K1/MUSEPaper_schematic-V1.1-20250701.pdf">点我查看</a></td>
</tr>
<tr>
<td>MUSE Paper原理图</td>
</tr>
<tr>
<td>MUSE Paper PCB</td>
<td><a href="https://cdn-resource.spacemit.com/file/product/K1/MUSEPaper_layout-V1.0-20241118.pdf">点我查看</a></td>
</tr>
<tr>
<td>MUSE Paper 用户使用指南</td>
<td><a href="https://www.spacemit.com/community/document/info?lang=zh&nodepath=hardware/eco/k1_muse_paper/paper_user_guide.md">点我查看</a></td>
</tr>
<tr>
<td rowspan=2 colspan=1>工具</td>
<td>刷机/写号工具：TitanFlasher</td>
<td><a href="https://www.spacemit.com/community/document/info?lang=zh&nodepath=tools/user_guide/flasher_user_guide.md">点我下载并获取使用说明</a></td>
</tr>
<tr>
<td>应用开发：devecostudio-windows-5.0.5.310.zip</td>
<td><a href="https://archive.spacemit.com/tools/openharmony/devecostudio-windows-5.0.5.310.zip">点我下载</a></td>
</tr>
<tr>
<td rowspan=10 colspan=1>固件</td>
<td>Muse Paper2固件（新版硬件）</td>
<td rowspan=9 colspan=1><a href="https://archive.spacemit.com/image/k1/version/openharmony5.0/">下载路径</a><br/></td>
</tr>
<tr>
<td>Muse Paper固件（旧版硬件）</td>
</tr>
<tr>
<td>Muse Paper Mini4G固件</td>
</tr>
<tr>
<td>Muse Pi Pro固件</td>
</tr>
<tr>
<td>Muse Book固件</td>
</tr>
<tr>
<td>Muse Card HDMI输出固件</td>
</tr>
<tr>
<td>Muse Card Mipi-DSI输出固件</td>
</tr>
<tr>
<td>Muse Pi HDMI输出固件</td>
</tr>
<tr>
<td>Muse Pi Mipi-DSI输出固件</td>
</tr>
<tr>
<td>deb1 HDMI输出固件</td>
<td></td>
</tr>
<tr>
<td></td>
<td>SDK包</td>
<td><a href="https://archive.spacemit.com/tools/openharmony/sdk/">下载路径</a></td>
</tr>
</tbody>
</table>

## 3. 支持的设备

| 生态产品       | MUSE Paper2（新版硬件）<br>MUSE Paper（旧版硬件）<br>[点我了解](https://www.spacemit.com/community/document/info?lang=zh&nodepath=hardware/eco/k1_muse_paper/root_overview.md) | MUSE Card<br>[点我了解](https://www.spacemit.com/community/document/info?lang=zh&nodepath=hardware/eco/k1_muse_card/root_overview.md) | MUSE Pi<br>[点我了解](https://www.spacemit.com/community/document/info?lang=zh&nodepath=hardware/eco/k1_muse_pi/root_overview.md) | MUSE Pi Pro<br>[点我了解](https://www.spacemit.com/community/document/info?lang=zh&nodepath=hardware/eco/k1_muse_pi_pro/root_overview.md) | MUSE Book<br>[点我了解](https://www.spacemit.com/community/document/info?lang=zh&nodepath=hardware/eco/k1_muse_book/root_overview.md) |
|------|---------|--------|-----|-----|------------|
| 产品形态       | 平板电脑  | 开发板    | 开发板 | 开发板   | 笔记本电脑  |
| 启动方式       | eMMC<br>TF卡    | TF卡   | eMMC   | eMMC  | NOR + SSD  |
| 显示输出       | MIPI DSI   | MIPI DSI<br>HDMI | MIPI DSI<br>HDMI   | MIPI DSI      | MIPI DSI 转 eDP |
| 屏幕触摸       | √   | √   | √  | √  | ×    |
| Wi-Fi          | √    | √   | √   | √     | √    |
| BT             | √   | ×   | ×  | ×  | ×    |
| Ethernet       | √（USB 转）  | √   | √    | √  | ×   |
| USB            | √    | √   | √ | √    | √   |
| 键鼠           | √   | √    | √  | √   | √ |
| 摄像头         | √（支持双摄）| ×   | ×  | ×  | × |
| 双屏显示       | ×    | ×    | ×   | ×   | √（接 HDMI）  |

## 4. 上手指南

### 4.1. 南向开发（系统）

- 根据文档 `K1 OH5.0下载编译烧录说明` 准备代码并编译烧录
- 根据文档 `K1 OH5.0方案添加移植说明` 添加自己的方案
- 根据文档 `K1 OH5.0系统定制说明` 和 `K1 OH5.0系统调试说明` 进行一些定制和调试

## 4.2. 北向开发（应用）

- 直接通过[下载路径](https://archive.spacemit.com/image/k1/version/openharmony5.0/)下载固件进行烧录，烧录请参考 `K1 OH5.0下载编译烧录说明` 的第 5 章节
- 根据文档 `K1 OH5.0应用开发说明` 进行应用环境的搭建和应用开发

## 5. 版本更新说明

<table>
<tbody>
<tr>
<td><strong>版本</strong></td>
<td><strong>发布时间</strong></td>
<td><strong>主要特性</strong></td>
</tr>
<tr>
<td>OH V1.0RC1</td>
<td>2024-12-25</td>
<td>1. 支持camera流畅预览拍照录像（480P）<br/>2. 支持浏览器<br/>3. 支持生态产品全家桶<br/>4. 支持蓝牙基本功能（不完善）</td>
</tr>
<tr>
<td>OHV1.0RC2<br/></td>
<td>2025-05-31</td>
<td>1. 支持camera流畅预览录像拍照（1080P30fps）<br/>2. 支持4K60fps H.264视频播放<br/>3. 支持AI标准框架与演示demo<br/>4. 支持MUSE Pi Pro<br/>5. 支持U盘/移动硬盘/固态硬盘挂载与文件管理器识别<br/>6. 支持USB摄像头<br/>7. 支持双摄<br/>8. 支持美格SLM790 4G通信模组</td>
</tr>
<tr>
<td>OHV1.0RC2.1</td>
<td>2025-06-12</td>
<td>1. 支持双屏（仅MUSE Book验证）<br/>2. 支持应用商城<br/>3. 支持ASR/TTS演示demo（从应用商城下载）</td>
</tr>
<tr>
<td>OHV1.0RC2.2</td>
<td>2025-06-20</td>
<td>1. 支持python3.10<br/>2. 支持中国移动MR880A 5G模组上网（deb1）<br/>3. 支持miracast sink端</td>
</tr>
<tr>
<td>OHV1.0RC2.3</td>
<td>2025-07-25</td>
<td>1. 支持miracast source端<br/>2. 支持卡刷机<br/>3. 支持A/B分区及A/B分区OTA升级（仅支持system/vendor）</td>
</tr>
<tr>
<td>OHV1.0RC2.4</td>
<td>2025-08-31</td>
<td>1. 支持震动马达<br/>2. 支持“工程测试”产测应用<br/>3. 支持指南针<br/>4. 支持闪光灯<br/>5. 支持GNSS定位</td>
</tr>
<tr>
<td>OHV1.0RC2.5</td>
<td>2025-09-26</td>
<td>1. 支持AI框架<br/>2. 支持光距感传感器<br/>3. 支持霍尔传感器</td>
</tr>
</tbody>
</table>

## 6. FAQ

### 6.1. MUSE Paper 和 MUSE Paper2 有什么差别？

`MUSE Paper` 和 `MUSE Paper2` 是同一款生态平板的不同硬件版本，目前市面上在售的版本都是新版硬件，也就是 `MUSE Paper2`，因此，在下载固件时，一般选择带有 `musepaper2` 字样的固件，在编译固件时，`--product-name` 也设置为 `musepaper2` 即可

### 6.2. 支持 K1 OH5.0 的设备的购买链接在哪里？

[点击购买 MUSE Paper](https://e.tb.cn/h.hHZVpHaqfgiw04T?tk=uRtQ48EtBaM)

[点击购买 MUSE Pi Pro](https://e.tb.cn/h.hskPVCg9vGaJXbn?tk=OJNO48ExGeF)

[点击购买 MUSE Book](https://m.tb.cn/h.gUvG9fgiJa4uJ43?tk=SrdHWBsSc5x)


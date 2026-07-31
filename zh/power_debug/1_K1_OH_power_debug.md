# OpenHarmony K1 平台功耗调试方法

| 修订版本 | 修订日期       | 修订说明 |
| ---- | ---------- | ---- |
| 001  | 2026-07-31 | 初始版本 |

> 本文用于 OpenHarmony 6.1 K1 平台的功耗定位与调试，覆盖电源域、CPU、DDR、GPU、VPU、DPU、AXI/CCI、USB 及常见外设。

## K1 组合 P1 电源域介绍

P1为我司自主开发的一款电源管理芯片，一共有6路buck。

各路默认的标准电压为

| 电源输出     | 电压  | 主要供电对象    |
| -------- | --- | --------- |
| BUCK1\+2 | 0V9 | K1 core   |
| BUCK3    | 1V8 | 外设        |
| BUCK4    | 3V3 | 外设        |
| BUCK5    | 2V1 | P1 DLDO输入 |
| BUCK6    | 1V1 | 外设        |

除测量整机功耗外，还可以分别测量各路 BUCK 的功耗，以定位主要耗电来源。

影响功耗的主要因素包括 CPU 频率、CPU 功能模块、DDR 频率以及外设状态。

建议按以下顺序进行调试：先检查并关闭 U-Boot 和内核 DTS 中不需要的模块；再调整 CPU 和 DDR 频率，观察 BUCK1+2 的功耗变化；最后以最小系统为基线，逐个恢复模块或增加外设，通过功耗差值定位问题来源。

## 1. 检查并关闭无用模块和外设

检查内核和 U-Boot DTS 中是否启用了不需要的模块或外设。尤其要注意 U-Boot DTS 已启用、但内核 DTS 未启用的情况：该模块可能在内核启动后仍处于耗电状态。

例如，产品不需要 PCIe，内核 DTS 中 PCIe 节点均为 disabled，但 U-Boot DTS 中仍为 enabled。系统启动后，PCIe 可能在没有业务使用的情况下持续占用功耗。

## 2. 调整 CPU 和 DDR 频率

### 2.1 CPU

cpu可以通过调整cpu的频率和核数来调整功耗，其中，cpu频率调整对功耗变化比较明显，尤其是从1600000降低到1228800是最明显的，因为这一小段会有降压，大概从1\.05V降低到0\.95V。从1228800往下电压不再变化。

cpu的默认性能策略是performance，默认最高频率，可以根据后续提到的策略表格自行调整。

K1 core的cpu组成：

- cpu0\-3是ai cpu，cluster0

- cpu4\-7是普通core，cluster1

注：ai cpu既可以跑一些ai推理，也可以当作普通cpu使用。

#### 2.1.1 CPU 调频

**可用频率：**614400、819000、1000000、1228800、1600000

查看当前所有核频率：

```shell
cat /sys/devices/system/cpu/cpu*/cpufreq/cpuinfo_cur_freq
```

查看所有cpu核当前策略：

```shell
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_governor
```

查看可用 CPU 频点：

```shell
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_available_frequencies
```

固定 CPU 频率（例如 1\.2GHz）：

```shell
echo userspace > /sys/devices/system/cpu/cpufreq/policy0/scaling_governor
echo 1228800 > /sys/devices/system/cpu/cpufreq/policy0/scaling_setspeed
```

恢复动态调频：

```shell
echo schedutil > /sys/devices/system/cpu/cpufreq/policy0/scaling_governor
```

补充，当前可支持的cpu频率策略介绍，可以根据产品使用场景在内核defconfig中进行配置。

#### 2.1.2 CPU 关闭

关闭cpu的指令（X为具体cpu号）：

```shell
echo 0 > /sys/devices/system/cpu/cpuX/online
```

CPU 下线/上线示例：

```shell
echo 0 > /sys/devices/system/cpu/cpu7/online
echo 1 > /sys/devices/system/cpu/cpu7/online
```

也可以通过查看寄存器来获取当前cpu状态，对应bit置位即为关闭：

```shell
devmem 0xd4282890 4
```

代码上，可以在dts的chosen节点中增加：

```Plain
maxcpus=4
```

来控制启动后最大启用的cpu核数量，写4意味启动后只启用4个cpu。

#### 2.1.3 查看 CPU 温度

该温度与实际cpu表面温度存在偏差，还是建议使用专业测温温枪测量

```shell
cat /sys/class/thermal/thermal_zone*/temp
```

### 2.2 DDR

ddr的功耗调试手段主要为调整频率，当前支持600，800，1066，1200，1600，2400几个挡位。

#### 2.2.1 DDR 静态调频

读取寄存器：

```shell
devmem 0xd4282980 4
```

- bit0 = 触发位，写 1 生效，读出来一般会变回 0

- bit\[3:1\] = DDR 频点档位

举例：写入2400（level 6）

```shell
# 写入level 6 2400
devmem 0xd4282980 4 0xD

# 写完读取
devmem 0xd4282980 4
# 返回 0x00000c，代表level 6，写入成功
```

常见档位：

有的设备在时钟里可以通过节点来获取：

```shell
cat /sys/kernel/debug/clk/clk_summary
```

#### 2.2.2 DDR 动态调频

暂无。当前除了调整寄存器，不支持ddr频率切换，开机起来后ddr频率是多少就一直是多少，一般默认最高。

## 3. 关闭模块和外设

### 3.1 GPU

gpu支持策略：userspace、powersave、performance、simple\_ondemand这四种工作策略。

默认的工作模式为simple\_ondemand，是动态调频模式。

100ms没有渲染任务时，gpu会自动关电。不过openharmony基本一直有渲染任务。

如果想关掉渲染任务，看gpu power off的情况，可以把 system/bin/render\_service 改个名字或者删掉的方式来不拉起render service，然后重启。

不要贸然在dts里关闭GPU，直接关闭会导致render\_service不断崩溃重启，4次后，foundation会主动触发内核panic。需要提前修改 /system/etc/init/foundation\.cfg，把"critical"这行删除掉，或者修改掉，参考命令如下：

```shell
hdc shell
mount -o rw,remount /
sed -i 's/"critical" : \[1, 4, 240\]/"critical" : [0, 4, 240]/' /system/etc/init/foundation.cfg
```

#### 3.1.1 GPU 静态调频

如果想手动调整gpu频率，需要修改内核配置

开启CONFIG\_POWERVR\_DVFS=y，

后重新编译内核，进入系统后在 `/sys/class/devfreq/cac00000.imggpu/` 路径下进行配置。

获取支持的频率：

```shell
cat /sys/class/devfreq/cac00000.imggpu/available_frequencies
```

支持频率：409000000、491000000、614000000、819000000

获取支持的调频策略：

```shell
cat /sys/class/devfreq/cac00000.imggpu/available_governors
```

支持策略：userspace、powersave、performance、simple\_ondemand

手动调整频率：

```shell
echo userspace > /sys/class/devfreq/cac00000.imggpu/governor
echo 491000000 > /sys/class/devfreq/cac00000.imggpu/userspace/set_freq
```

查看当前频率：

```shell
cat /sys/class/devfreq/cac00000.imggpu/cur_freq
```

切换回自动调频：

```shell
echo simple_ondemand > /sys/class/devfreq/cac00000.imggpu/governor
```

### 3.2 VPU

VPU没有编解码业务时，是处于power\-off的状态。可以通过power domain这个节点验证。

查看支持的频率：

```shell
cat /sys/devices/platform/soc/c0500000.linlon-v5/dvfs_available_freqency
```

查看当前状态：

```shell
cat /sys/devices/platform/soc/c0500000.linlon-v5/dvfs_stats
```

开关 VPU DVFS：

```shell
echo 1 > /sys/devices/platform/soc/c0500000.linlon-v5/dvfs_enable
echo 0 > /sys/devices/platform/soc/c0500000.linlon-v5/dvfs_enable
```

设置 VPU 频率上下限：

```shell
echo 307200000 > /sys/devices/platform/soc/c0500000.linlon-v5/dvfs_min_freq
echo 819200000 > /sys/devices/platform/soc/c0500000.linlon-v5/dvfs_max_freq
```

### 3.3 DPU

K1有两个DPU，所以理论最多接两个屏。

由于DPU的频率和屏幕绑定，所以DPU通常不能单独调整。

#### 3.3.1 DPU 频率查看

查看dpu时钟：

```shell
cat /sys/kernel/debug/clk/clk_summary | grep -iE "dpu|lcd|mclk|pxclk|bitclk|esc"
```

#### 3.3.2 DPU 关闭

dpu不接入屏幕时就会关闭，不会占用功耗，接入屏幕时可以在/sys/class/drm/下看到对应card。

也可以在/sys/kernel/debug/pm\_genpd/pm\_genpd\_summary节点下看，没接屏幕就不会有相应的dpu节点。

#### 3.3.3 DPU 图层问题

目前DPU最多支持4个图层，图层输出多少是由上层系统的图像框架决定。

例如，Openharmony在锁屏界面只有一个图层，进入桌面就会变成4个图层（状态栏\+导航栏\+桌面\+鼠标）。

### 3.4 AXI/CCI 调频

AXI 和 CCI 的时钟源及分频关系如下。

| 输出时钟                 | 可选时钟源                                                      | 默认时钟源                      |
| -------------------- | ---------------------------------------------------------- | -------------------------- |
| `cci550_clk`         | `pll1_d5_491p52`、`pll1_d4_614p4`、`pll1_d3_819p2`、`pll2_d3` | `pll1_d4_614p4`（614.4 MHz） |
| `pmua_aclk`（AXI/AHB） | `pll1_d10_245p76`、`pll1_d8_307p2`                          | `pll1_d8_307p2`（307.2 MHz） |

时钟路径可以概括为：

```text
CCI:
  pll1_d5_491p52 ─┐
  pll1_d4_614p4  ─┤
  pll1_d3_819p2  ─┼─> mux ─> div ─> cci550_clk
  pll2_d3        ─┘

AXI/AHB:
  pll1_d10_245p76 ─┐
  pll1_d8_307p2   ─┤─> mux ─> div ─> pmua_aclk
```

可通过下面的命令获取pll2\_d3的频率：

```shell
cat /sys/kernel/debug/clk/pll2_d3/clk_rate
```

**默认的频率：**

- CCI默认为pll1d\_4\_614p4，即614400000

- AXI/PMUA默认为pll1\_d8\_307p2，即307200000

如果想要修改AXI/PMUA的频率，首先需要修改文件：`kernel/linux/spacemit_kernel-6.6/drivers/clk/clk.c`

```c
-#undef CLOCK_ALLOW_WRITE_DEBUGFS
+#define CLOCK_ALLOW_WRITE_DEBUGFS
```

在 `kernel/linux/spacemit_kernel-6.6/drivers/clk/clk.c` 中，打开 debugfs 时钟频率写入功能：

```diff
-#undef CLOCK_ALLOW_WRITE_DEBUGFS
+#define CLOCK_ALLOW_WRITE_DEBUGFS
```

该宏启用后，相关时钟节点会提供可写的 `clk_rate` 属性。对应的核心实现如下：

```c
#define CLOCK_ALLOW_WRITE_DEBUGFS

static int clk_rate_set(void *data, u64 val)
{
    struct clk_core *core = data;
    int ret;

    clk_prepare_lock();
    ret = clk_core_set_rate_nolock(core, val);
    clk_prepare_unlock();

    return ret;
}

#define clk_rate_mode 0644
```

重新编译内核后，通过下面的操作即可修改频率：

```shell
# CCI频率
cat /sys/kernel/debug/clk/cci550_clk/clk_rate
echo 491520000 > /sys/kernel/debug/clk/cci550_clk/clk_rate

# AXI频率
cat /sys/kernel/debug/clk/pmua_aclk/clk_rate
echo 245760000 > /sys/kernel/debug/clk/pmua_aclk/clk_rate
```

### 3.5 关闭 USB

查看寄存器的方法

K1 有两个 USB 相关寄存器需要关注：`0xd428285c` 和 `0xd4282bcc`。

```shell
devmem 0xd428285c 4
```

power on reset value = 0x0

工作时值：

- USB3\.0 PortD: 0x\_XXX**F**\_XXXX

- USB3\.0 PortC: 0x\_XXXX\_**F**XXX

- USB3\.0 PortB: 0x\_XXXX\_X**F**XX （COM260 VL817 4 Port Hub）

- USB3\.0 PortA: 0x\_XXXX\_XX**F**X

- USB2\.0 Host: 0x\_XXXX\_XXX**F**

- 全是0就是关闭了

```shell
devmem 0xd4282bcc 4
```

当 bit0 到 bit6 均为 0 时，USB 相关模块全部关闭。

这两个寄存器可以直接写0关闭。

usb全关时可以看到寄存器状态如下

```Plain
# devmem 0xd428285 4
0x0
# devmem 0xd4282bcc 4
0x000480
```

除了寄存器，还可以通过修改DTS来关闭usb

把 DTS 里面这些节点的 status 全部改成 disabled：

```dts
&usbphy {
        status = "okay";
};

&udc {
        spacemit,udc-mode = <MV_USB_MODE_UDC>;
        status = "okay";
};

&ehci {
        spacemit,reset-on-resume;
        spacemit,udc-mode = <MV_USB_MODE_OTG>;
        status = "disabled";
};

&otg {
        usb-role-switch;
        role-switch-user-control;
        spacemit,reset-on-resume;
        role-switch-default-mode = "peripheral";
        vbus-gpios = <&gpio 123 0>;
        status = "disabled";
};

&usbphy1 {
        status = "okay";
};

&udc1 {
        spacemit,udc-mode = <MV_USB_MODE_OTG>;
        status = "okay";
};

&ehci1 {
        spacemit,udc-mode = <MV_USB_MODE_OTG>;
        spacemit,reset-on-resume;
        status = "okay";
};

&otg1 {
        usb-role-switch;
        role-switch-user-control;
        spacemit,reset-on-resume;
        role-switch-default-mode = "host";
        vbus-gpios = <&gpio 123 0>;
        status = "okay";
};

&usb2phy {
        status = "okay";
};

&combphy {
        status = "okay";
};

&usb3hub {
        vbus-gpios = <&gpio 79 0>;      /* gpio_79 for usb3 hub output vbus */
        status = "disabled";
};

&usbdrd3 {
        status = "okay";
        reset-on-resume;
        dwc3@c0a00000 {
                dr_mode = "peripheral";
                phy_type = "utmi";
                snps,hsphy_interface = "utmi";
                snps,dis_enblslpm_quirk;
                snps,dis_u2_susphy_quirk;
                snps,dis_u3_susphy_quirk;
                snps,dis-del-phy-power-chg-quirk;
                snps,dis-tx-ipgap-linecheck-quirk;
                snps,parkmode-disable-ss-quirk;
                usb-role-switch;
                role-switch-default-mode = "host";
        };
};
```

### 3.6 整体验证（可选）

如果所关注的模块向pm注册了，可以看pm\_genpd\_summary快速查看各个模块的工作状态。

#### 3.6.1 Power Domain 总体验证

```shell
cat /sys/kernel/debug/pm_genpd/pm_genpd_summary
```

示例输出如下：

```text
# cat /sys/kernel/debug/pm_genpd/pm_genpd_summary
domain                            status             children                         performance
    /device                                           runtime status
------------------------------------------------------------------------------------------------------
power-domain@SPT_PD_DUMMY         on                                                  0
    /devices/platform/soc/soc:usb3hub@0                    active                     0
    /devices/platform/soc/d4013400.mailbox                 active                     0
    /devices/platform/d4026000.i2s0                        active                     0
    /devices/platform/c0883900.spacemit_snd_sspa           suspended                  0
power-domain@SPT_PD_HDMI           off-0                                               0
power-domain@SPT_PD_GNSS           off-0                                               0
power-domain@SPT_PD_AUDIO          on                                                  0
    /devices/platform/soc/c088c000.rcpu_rproc              active                     0
power-domain@SPT_PD_ISP            off-0                                               0
    /devices/platform/soc/c02f8000.jpg                     suspended                  0
    /devices/platform/soc/d420a000.ccic                    suspended                  0
    /devices/platform/soc/d420a800.ccic                    suspended                  0
    /devices/platform/soc/d4206000.ccic                    suspended                  0
    /devices/platform/soc/c02f0000.cpp                     suspended                  0
    /devices/platform/soc/c0230000.isp                     suspended                  0
    /devices/platform/soc/c0230000.vi                      suspended                  0
power-domain@SPT_PD_LCD             on                                                  0
    /devices/platform/soc:soc:port@c0340000                active                     0
power-domain@SPT_PD_GPU             on                                                  0
    /devices/platform/soc/cac00000.imggpu                  active                     0
power-domain@SPT_PD_VPU             off-0                                               0
    /devices/platform/soc/c0500000.linlon-v5               suspended                  0
power-domain@SPT_PD_BUS             on                                                  0
    /devices/platform/soc/d4000000.pdma                    suspended                  0
    /devices/platform/soc/d4010800.i2c                     suspended                  0
    /devices/platform/soc/d4011000.i2c                     suspended                  0
    /devices/platform/soc/d4012000.i2c                     suspended                  0
    /devices/platform/soc/d4012800.i2c                     suspended                  0
    /devices/platform/soc/d401d800.i2c                     suspended                  0
    /devices/platform/soc/d4017000.serial                  active                     0
    /devices/platform/soc/f0612000.uart                    suspended                  0
    /devices/platform/soc/d4017300.uart                    suspended                  0
    /devices/platform/soc/f0613000.spi                     suspended                  0
    /devices/platform/soc/d401c000.spi                     suspended                  0
    /devices/platform/soc/d420c000.spi                     suspended                  0
    /devices/platform/soc/c0900100.otg                     active                     0
    /devices/platform/soc/c0980100.otg1                    active                     0
    /devices/platform/soc/d4282b8c.usb3                    active                     0
    /devices/platform/soc/c0900100.ehci                   active                     0
    /devices/platform/soc/c0980100.ehci1                  active                     0
    /devices/platform/soc/d4280800.sdh                    active                     0
    /devices/platform/soc/d4281000.sdh                    active                     0
```

#### 3.6.2 寄存器查看

或者查看该寄存器

```shell
devmem 0xd42828f0 4
```

寄存器各 bit 的含义如下。

**POWER STATUS REGISTER**  
**PMUA_PWR_STATUS_REG**  
Offset: `0xF0`

| Bits | Field                            | Field (Code)        | Type | Init | Reset | Description                                         |
| ---- | -------------------------------- | ------------------- | ---- | ----:| -----:| --------------------------------------------------- |
| 15:7 | Reserved                         | `RSVD`              | RO   | 0    | 0     | Reserved for future use.                            |
| 14   | GNSS HW Mode power state update  | `GNSS_HW_PWR_STAT`  | RO   | 0x0  | 0x0   | GNSS 进入或退出低功耗模式后，硬件电源模式更新状态。                        |
| 13   | Reserved                         | `RSVD`              | RO   | 0    | 0     | Reserved for future use.                            |
| 12   | LCD HW Mode power state update   | `LCD_HW_PWR_STAT`   | RO   | 0x0  | 0x0   | LCD 进入或退出低功耗模式后，硬件电源模式更新状态。                         |
| 11   | Audio HW Mode power state update | `AUDIO_HW_PWR_STAT` | RO   | 0x0  | 0x0   | Audio 进入或退出低功耗模式后，硬件电源模式更新状态。`0x0` 表示关闭，`0x1` 表示开启。 |
| 10   | ISP HW Mode power state update   | `ISP_HW_PWR_STAT`   | RO   | 0x0  | 0x0   | ISP 进入或退出低功耗模式后，硬件电源模式更新状态。                         |
| 9    | VPU HW Mode power state update   | `VPU_HW_PWR_STAT`   | RO   | 0x0  | 0x0   | VPU 进入或退出低功耗模式后，硬件电源模式更新状态。                         |
| 8    | GPU power state update           | `GPU_HW_PWR_STAT`   | RO   | 0x0  | 0x0   | GPU 进入或退出低功耗模式后的硬件状态更新。                             |
| 7    | Reserved                         | `RSVD`              | RO   | 0    | 0     | Reserved for future use.                            |
| 6    | GNSS power state                 | `GNSS_PWR_STATUS`   | RO   | 0x0  | 0x0   | GNSS 的硬件和软件电源状态：`1` 表示开启，`0` 表示关闭。                  |
| 5    | Reserved                         | `RSVD`              | RO   | 0    | 0     | Reserved for future use.                            |
| 4    | LCD power state                  | `LCD_PWR_STATUS`    | RO   | 0x0  | 0x0   | LCD 的硬件和软件电源状态：`1` 表示开启，`0` 表示关闭。                   |
| 3    | Audio power state                | `AUDIO_PWR_STATUS`  | RO   | 0x0  | 0x0   | Audio 的硬件和软件电源状态：`1` 表示开启，`0` 表示关闭。                 |
| 2    | ISP power state                  | `ISP_PWR_STATUS`    | RO   | 0x0  | 0x0   | ISP 的硬件和软件电源状态：`1` 表示开启，`0` 表示关闭。                   |
| 1    | VPU power state                  | `VPU_PWR_STATUS`    | RO   | 0x0  | 0x0   | VPU 的硬件和软件电源状态：`1` 表示开启，`0` 表示关闭。                   |
| 0    | GPU power state                  | `GPU_PWR_STATUS`    | RO   | 0x0  | 0x0   | GPU 的硬件和软件电源状态：`1` 表示开启，`0` 表示关闭。                   |

### 3.7 其他外设和模块

常见的功耗大户有wifi/bt 模组，屏幕，4/5G模块，pcie等，想要确定这些外设的功耗，可以采用最小系统法，即，在保证内核能正常工作时，关闭uboot dts的全部节点，和内核dts里的全部节点，逐个模块开启测量功耗差值。对某个模块功耗不满意再单独进行优化。

**最小系统测试配置参考**

- 内核：保留i2c8（P1所在），rcpu（保证cpu正常工作），uart0（串口交互）

- uboot：保留eeprom（选dtb），efuse（芯片基础数据），uart0（串口交互）

#### 3.7.1 Wi-Fi 内核模块动态加载

对于wifi/bt模块，wifi/bt 关闭时，只要驱动还在，还是会有一定的功耗。对于一些想去掉该功耗的场景，且wifi是以ko形式加载的情况下，进迭在Openharmony上做了开关wifi时同步装卸ko的特性，当前已在产品musepaper2上默认开启，新增产品可以按照如下方式增加：

#### 3.7.2 新产品接入方法

假设新产品名为 `<product>`，模块为 `<module>`，主接口为 `<iface>`。

##### 3.7.2.1 同步公共 Wi-Fi framework

同步 Wi\-Fi仓库 patch。该修改默认关闭，仅提供公共能力。

##### 3.7.2.2 在 device 仓增加产品配置

在 `device/board/<vendor>/<product>/cfg`：

1. 在产品 `default.para` 增加：

```Properties
const.wifi.driver_module.name=<module>
const.wifi.driver_module.path=/vendor/modules/<module>.ko
const.wifi.driver_interface.name=<iface>
wifi.driver_module.action=idle
```

1. 增加 `init.<product>.wifi.dynamic.cfg`：
   
   - load job 使用产品真实模块路径和 module parameter；
   
   - unload job 使用模块名；
   
   - job 完成后设置 action 为 idle。

2. 增加 `init.<product>.wifi.static.cfg`：
   
   - 内容保持该产品原有开机加载方式；
   
   - 用于 feature 关闭时兼容。

3. 增加 `wifi_module.para.dac`：

```Properties
wifi.driver_module.action="wifi:wifi:775"
```

1. 修改产品主 init：

```JSON
"/vendor/etc/init.${ohos.boot.hardware}.wifi.cfg"
```

1. 同时删除主 init 中原有的 Wi\-Fi模块直接 `insmod`。

2. 修改产品 cfg `BUILD.gn`：
   
   - 导入 Wi\-Fi `wifi.gni`；
   
   - 根据 feature 选择 dynamic/static 源文件；
   
   - 输出并安装为 `/vendor/etc/init.<product>.wifi.cfg`；
   
   - 安装产品 DAC。

##### 3.7.2.3 在 vendor 仓开启特性

在产品 `config.json` 的 Wi\-Fi component features 中增加：

```JSON
"wifi_feature_dynamic_driver_module = true"
```

不配置或配置为 false 时使用静态 cfg。

## 4. OpenHarmony 平台功耗测试的其他小 tips

1. **power\-shell setmode 602**开机后通过power\-shell setmode 602设置系统不睡眠，防止影响功耗数据。但是要注意，如果屏幕亮度调整功能已经调通，该命令同时会让屏幕亮度调整到最大。
   
   ```shell
   power-shell setmode 602
   ```

2. **uinput \-T \-m 100 100 500 500**该命令可以模拟input输入，触发一个手指滑动的效果，由锁屏进入桌面。
   
   ```shell
   uinput -T -m 100 100 500 500
   ```

3. 一般启动或调整数据后等待5\-10分钟等电压稳定再读数。

> **免责声明**：本文内容来自平台调试记录，部分结论和示例仍需结合具体硬件、SDK 版本及实测结果确认。

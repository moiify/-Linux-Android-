[TOC]

# Kernel

## 驱动开发

### DDR

#### 调整ddr最高频率

RK3588

```
cd rkbin/
cp bin/rk35/rk3588_ddr_lp4_2112MHz_lp5_2736MHz_v1.08.bin ./tools/
cd tools
调整ddrbin_param.txt以下：

保存退出，
执行
./ddrbin_tool ddrbin_param.txt rk3588_ddr_lp4_2112MHz_lp5_2736MHz_v1.08.bin
mv rk3588_ddr_lp4_2112MHz_lp5_2736MHz_v1.08.bin rk3588_ddr_lp4_1848MHz_lp5_2736MHz_v1.08.bin
cp rk3588_ddr_lp4_1848MHz_lp5_2736MHz_v1.08.bin ../bin/rk35/
拷贝到rkbin/bin/rk35 下，修改RKBOOT/RK3588MINIALL.ini 以下两项：

Path1=bin/rk35/rk3588_ddr_lp4_1848MHz_lp5_2736MHz_v1.08.bin
FlashData=bin/rk35/rk3588_ddr_lp4_1848MHz_lp5_2736MHz_v1.08.bin

保证合成新的ddr bin，重新编译UBOOT，烧录loader.
```

### EMMC

### 显示Display

#### LVDS

#### MIPI

#### HDMI

#### DP

#### EDP

#### V-BY-ONE

#### 背光

##### 调整显示器开关机时序

 [RK3399] [Linux4.19]

------

问题描述: 需要根据屏幕规格书，调整一下开关机时屏幕上下电的时序

分析过程:背光以及屏压的控制信号都接在了RK3399上，在原理图中找到相关引脚。RK的simple-plane驱动中有时序控制，查看源码找到dts中delay-time对应的时间代表什么意思，理清楚即可。

解决方法：

1. 首先找到规格书中关于上下电时序的描述

2. 从中得到以下关键信息

   * 屏供电(屏压)到发送有效数据时间 :T1+T2 
   * 从屏供电到开启背光时间 : T1+T2+T3 
   * 先关闭背光，再停止发送数据，最后关闭屏压 T4+T5+T7

3. 找到设备树和RK的驱动文件，panel_simple.c，虽然我们用的是MIPI转LVDS的芯片，但是RK驱动中提供的这几个延时，是通用的。下面就是找到对应关系以及赋上初值了。

   * prepare-delay-ms  先拉高屏压, 延时Nms，拉高reset脚 ①
   * enable-delay-ms  延时Nms,开背光
   * disable-delay-ms  关闭背光后延时Nms，发送关屏MIPI指令
   * unprepare-delay-ms 延时Nms，标记设备状态
   * reset-delay-ms 拉高reset脚，延时Nms，拉低reset脚 ② 
   * init-delay-ms  拉低reset脚后，延时Nms ，开始发送mipi指令 ③ 

   4.查找AI3399C的原理图，找到以下引脚

   * 屏压脚: GPIO4_D6_d   net:LVDS_POWON , dts中的enable脚,关键
   * 背光相关脚：GPIO4_D1 net:MIPI_ONOFF     GPIO3_C0 net:LCD_RST。由背光节点控制

   5.计算时间

   			prepare-delay-ms  +  reset-delay-ms + init-delay-ms  这三个参数都是控制MIPI转LVDS芯片的，对照芯片规格书填写。
   	
   			enable-delay-ms 大于500   T3
   	
   			disable-delay-ms 大于200  T4
   	
   			unprepare-delay-ms 大于1000 T7

##### 单双驱，重影问题

rk3588，android12

```
ltx@ltx-H510M-H:~/work/1.rk3588_12_r12/android/kernel-5.10$ git diff
diff --git a/arch/arm64/boot/dts/rockchip/rk3588-AIoT3588-display.dtsi b/arch/arm64/boot/dts/rockchip/rk3588-AIoT3588-display.dtsi
index 8a0b5b663466..645e869ec4ad 100755
--- a/arch/arm64/boot/dts/rockchip/rk3588-AIoT3588-display.dtsi
+++ b/arch/arm64/boot/dts/rockchip/rk3588-AIoT3588-display.dtsi
@@ -388,7 +388,7 @@
                         */
                        bus-format = "rgb888";
                        gvi,lanes = <8>;
-                       //"rockchip,division-mode";
+                       rockchip,division-mode;
                        //"rockchip, gvi-frm-rst";
                        status = "disabled";
                };
diff --git a/drivers/misc/rk628/rk628_gvi.c b/drivers/misc/rk628/rk628_gvi.c
index 62fa7d281041..a8669c512eab 100755
--- a/drivers/misc/rk628/rk628_gvi.c
+++ b/drivers/misc/rk628/rk628_gvi.c
@@ -133,6 +133,9 @@ static void rk628_gvi_pre_enable(struct rk628 *rk628, struct rk628_gvi *gvi)
        rk628_i2c_update_bits(rk628, GVI_SYS_CTRL1, SYS_CTRL1_DUAL_PIXEL_EN,
                              gvi->division_mode ? SYS_CTRL1_DUAL_PIXEL_EN : 0);

+       dev_info(rk628->dev, ">>> gvi patch ");
+       rk628_i2c_update_bits(rk628, GVI_SYS_CTRL3, 0xffffffff, 0x54761032);
+


uboot

diff --git a/drivers/video/rk628/rk628_gvi.c b/drivers/video/rk628/rk628_gvi.c
index b7e731c088..0223925b3f 100755
--- a/drivers/video/rk628/rk628_gvi.c
+++ b/drivers/video/rk628/rk628_gvi.c
@@ -128,7 +128,8 @@ static void rk628_gvi_pre_enable(struct rk628 *rk628, struct rk628_gvi *gvi)
                              gvi->division_mode ? SW_SPLIT_EN : 0);
        rk628_i2c_update_bits(rk628, GVI_SYS_CTRL1, SYS_CTRL1_DUAL_PIXEL_EN,
                              gvi->division_mode ? SYS_CTRL1_DUAL_PIXEL_EN : 0);
-
+       printk(">>> gvi patch ");
+       rk628_i2c_update_bits(rk628, GVI_SYS_CTRL3, 0xffffffff, 0x54761032);
        rk628_i2c_update_bits(rk628, GVI_SYS_CTRL0, SYS_CTRL0_FRM_RST_EN,
                              gvi->frm_rst ? SYS_CTRL0_FRM_RST_EN : 0);
        rk628_i2c_update_bits(rk628, GVI_SYS_CTRL1, SYS_CTRL1_LANE_ALIGN_EN, 0);

```

#### HDMI_IN

HDMI IN功能可以通过桥接芯片的方式实现，将**HDMI信号转换成MIPI信号**接收，RK3588芯片平台**自带**
**HDMI RX模块，可以直接接收HDMI信号**。本文档主要介绍在RK3588 Android 12平台通过HDMI RX模块
开发实现HDMI IN功能的方法。支持HDMI IN自适应分辨率：4K60、4K30、1080P60、720P60等，支持
HDMI IN热拔插，支持录像功能，支持EDID可配置，支持HDCP1.4/HDCP2.3，支持CEC

#### DRM

![img](https://img-blog.csdnimg.cn/img_convert/bb69466e8efba04e716bf9baaf20be86.png)

通常，一个普通的图形应用并不会直接通过 KMS 和内核进行交互，而是先和 display server (例如给予 X11 的 Xorg, 或者基于 Wayland 的 Weston，也称 display compositor) 进行交互：将显示的图像提交给 display server， 再由 display server 负责将多个 client 图形应用的图像合成成一张图像，并将这张图像通过 KMS 的接口提交给内核。



KMS 将硬件模块抽象成下面几个对象类型：

Planes：图层，例如在 rockchip 平台里对应 SOC 内部 VOP 模块的 win 图层;

CRTC：显示控制器，例如在 rockchip 平台里对应 SOC 内部的 VOP 模块;

Encoder：输出转换器，指 RGB、LVDS、DSI、eDP、HDMI、CVBS、VGA 等显示接口;

Connector：连接器，指 encoder 和 panel 之间交互的接口部分;

Bridge：桥接设备，一般用于注册 encoder 后面另外再接的转换芯片，如 DSI2HDMI 转换芯片;

Panel：泛指屏，各种 LCD、HDMI 等显示设备的抽象;

应用通过 KMS api 将这些对象连接成一条 display pipeline，最终将图像显示在屏幕上。

### USB

驱动框架：

![2022年6月14日170747](F:\note_doc\pic\2022年6月14日170747.png)

#### HID

将OTG口配置成usb_device模式，进行通讯

##### 修改HID的PID/VID/REV

*RK3288 Andorid7.1*

PID,VID在hid驱动中修改宏定义

REV修改方法如下：

```
--- a/include/linux/usb/composite.h
+++ b/include/linux/usb/composite.h
@@ -562,7 +562,8 @@ static inline u16 get_default_bcdDevice(void)

        bcdDevice = bin2bcd((LINUX_VERSION_CODE >> 16 & 0xff)) << 8;
        bcdDevice |= bin2bcd((LINUX_VERSION_CODE >> 8 & 0xff));
-       return bcdDevice;
+       return 0x0310;
+       //return bcdDevice;
 }
```

##### 动态可变包长

*rk3288,android7.1（需重启）*

```
--- a/init.rk3288.rc
+++ b/init.rk3288.rc
+on property:persist.sys.hid.packagelen=64
+    insmod /system/lib/modules/libcomposite.ko
+    insmod /system/lib/modules/usb_f_hid.ko
+    insmod /system/lib/modules/g_hid.ko package=64
+on property:persist.sys.hid.packagelen=128
+    insmod /system/lib/modules/libcomposite.ko
+    insmod /system/lib/modules/usb_f_hid.ko
+    insmod /system/lib/modules/g_hid.ko package=128

vendor/signway/device.mk:52:    device/rockchip/rk3288/hidmoudle/libcomposite.ko:system/lib/modules/libcomposite.ko\
vendor/signway/device.mk:53:    device/rockchip/rk3288/hidmoudle/usb_f_hid.ko:system/lib/modules/usb_f_hid.ko\
vendor/signway/device.mk:54:    device/rockchip/rk3288/hidmoudle/g_hid.ko:system/lib/modules/g_hid.ko
^C
```

rk3288 5.1

注意要配置CONFIG_USB_G_HID=m

```
hidmodule_ts.ko
module_param(package,int,0644);
system/core/iicwatchdog/iic_watchdog.c
    ERROR("DX199566: property_get start \n");
    property_get("persist.sys.package", tmp, NULL);
    ERROR("DX199566: property_get persist.sys.package = %s;\n", tmp);
    if (!strcmp(tmp,"64")) {
    system("insmod /system/lib/modules/hidmodule_ts.ko package=64");
    } else if(!strcmp(tmp,"128")) {
    system("insmod /system/lib/modules/hidmodule_ts.ko package=128");
    } else if(!strcmp(tmp,"256")) {
    system("insmod /system/lib/modules/hidmodule_ts.ko package=256");
    } else if(!strcmp(tmp,"512")) {
    system("insmod /system/lib/modules/hidmodule_ts.ko package=512");
    } else if(!strcmp(tmp,"1024")) {
    system("insmod /system/lib/modules/hidmodule_ts.ko package=1024");
    } else {
    LOGE("error package len\n");
    return;
    }
    
static int __init hidg_init(void)
{
	int status;
    printk("package = %d\n", package);
    ...
}
```



#### USB_HOST

##### 性能测试

假设我们要测试RK3399读写USB的速度

挂载后，U盘路径是:6170-12F0 （USB3.0的U盘 FAT32格式 插在2.0的口）

* step1. 使用 dd 命令创建一个 test 文件（1GB 大小），用于后续的拷贝测试使用

busybox dd if=/dev/zero of=/mnt/media_rw/6170-12F0/test bs=64K count=16K

* step2. 使用 cp 命令测试测试从 USB3 Disk 拷贝大文件到 3399 EMMC 的速率

time cp /mnt/media_rw/6170-12F0/1.mp4 /sdcard/. (test 为 1GB 文件) 测试的是U盘的读速率

  1m26.66s real     0m00.53s user     0m13.34s system

**测试 4K/64K/128K/512K 数据块的读速率**
busybox dd if=/mnt/media_rw/6170-12F0/1.mp4 of=/dev/null bs=4K count=256K

1073741824 bytes (1.0GB) copied, 34.484782 seconds, 29.7MB/s

2147483648 bytes (2.0GB) copied, 66.327777 seconds, 30.9MB/s

busybox dd if=/mnt/media_rw/6170-12F0/1.mp4 of=/dev/null bs=64K count=16K

1073741824 bytes (1.0GB) copied, 33.862399 seconds, 30.2MB/s

**测试 4K 数据块的写速率**

环境：2.0口  3.0金士顿

命令：

busybox dd if=/dev/zero of=/storage/6170-12F0/test bs=4K count=256K

busybox dd if=/dev/zero of=/mnt/media_rw/6170-12F0/test bs=4K count=256K

1073741824 bytes (1.0GB) copied, 87.578795 seconds, 11.7MB/s

busybox dd if=/dev/zero of=/mnt/media_rw/6170-12F0/test bs=64K count=32K

2147483648 bytes (2.0GB) copied, 190.684244 seconds, 10.7MB/s  

/dev/block/vold/public:8,1 on /mnt/media_rw/6170-12F0 type vfat (rw,nosuid,nodev,noexec,noatime,gid=1023,fmask=0007,dmask=0007,allow_utime=0020,codepage=437,iocharset=iso8859-1,shortname=mixed,utf8,errors=remount-ro)

busybox dd if=/dev/zero of=/mnt/media_rw/6170-12F0/test bs=4K count=256K
1073741824 bytes (1.0GB) copied, 88.350392 seconds, 11.6MB/s

测试结果：跟CVTE的测试结果类似

分析：瓶颈在USB3 Disk介质或者USB控制器，写很慢



测试环境：2.0的U盘

busybox dd if=/dev/zero of=/mnt/media_rw/9C1D-1F99/test bs=4K count=256K

1073741824 bytes (1.0GB) copied, 153.514933 seconds, 6.7MB/s

busybox dd if=/dev/zero of=/mnt/media_rw/9C1D-1F99/test bs=512K count=2K

1073741824 bytes (1.0GB) copied, 161.183985 seconds, 6.4MB/s

##### 分析strace

strace cp  /sdcard/1.mp4  /mnt/media_rw/6170-12F0/2.mp4

##### 分析文件系统的影响：

不同文件系统拷贝的速度也不一样，需要注意。

#### TYPE-C

![image-20221110162756545](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221110162756545.png)

fusb302 是pd芯片，物理通讯接口为iic

![image-20221110162836747](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221110162836747.png)

##### DP

USB PD是在CC(Configuration Channel) pin上传输，PD有个VDM (Vendor defined message)功能，定义了装置端ID，读到支持DP或PCIe的装置，DFP就进入替代（alternate）模式。

如果DFP认到device为DP，便切换MUX/Configuration Switch，让Type-C USB3.1信号脚改为传输DP信号。AUX辅助由Type-C的SBU1,SUB2来传。HPD是检测脚，和CC差不多，所以共用。

（1）DP Alt Mode 4Lane
DP有lane0-3四组差分信号， Type-C有RX/TX1-2也是四组差分信号，所以完全替代没问题

![image-20221110165717692](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221110165717692.png)



##### 正反插

通过CC1 CC2的上下拉电阻，来判断，给VBUS供电（接在fusb302上）

### 触摸

触摸外设一般是IIC或者USB的

##### IIC电磁屏触摸

有一个优先级的问题，需要在电磁屏读数据的时候，屏蔽TP的底层中断。

### 音频

查看声卡信息

cat /proc/asound/cards

### 视频

### RTC

### GMAC

androidboot.ethaddr: 用来配置以太网MAC地址

kernel里：

**3288 Android7.1**

驱动:drivers/net/ethernet/stmicro/stmmac

关键函数:stmmac_check_ether_addr

rk_get_boot_ethaddr



**3568 Android11** (在UBOOT中设置)

uboot:

（uboot中定义了CONFIG_ROCKCHIP_SET_ETHADDR, uboot中设置了以后，kernel里设置函数就不起效了）

rockchip_set_ethaddr

kernel:

/kernel-5.10/drivers/net/ethernet/stmicro/stmmac/dwmac-rk.c

include/uapi/linux/if_ether.h:32:#define ETH_ALEN       6

rk_get_eth_addr

#### USB 转以太网

动态修改mac地址

```
/drivers/net/usb/r8152.c
@@ -1524,10 +1546,16 @@ static int determine_ethernet_addr(struct r8152 *tp, struct sockaddr *sa)
        } else if (!is_valid_ether_addr(sa->sa_data)) {
                netif_err(tp, probe, dev, "Invalid ether addr %pM\n",
                          sa->sa_data);
-               eth_hw_addr_random(dev);
-               ether_addr_copy(sa->sa_data, dev->dev_addr);
-               netif_info(tp, probe, dev, "Random ether addr %pM\n",
+               if(!rk_get_eth_addr(tp, buf)){
+                       memcpy(sa->sa_data, buf, 6);
+                       netif_info(tp, probe, dev, "use RK vendor ether addr %pM\n",
                           sa->sa_data);
+               }else {
+                       eth_hw_addr_random(dev);
+                       ether_addr_copy(sa->sa_data, dev->dev_addr);
+                       netif_info(tp, probe, dev, "Random ether addr %pM\n",
+                               sa->sa_data);
+               }
                return 0;
        }

```

注意vendor分区加载时间要在USB转以太网之前

#### 调试

##### iperf命令



##### ethtool命令

强制百兆网：ethtool -s eth1 autoneg off speed 100 duplex full

### Debug调试

打开dev_dbg/dev_vdbg的两种方法：

第一种

在文件开头\#define DEBUG 或者在makefile中 增加KBUILD_CFLAGS+=DDEBUG

第二种

动态调试

内核定义CONFIG_DYNAMIC_DEBUG

```
echo -n "file demo.c +p" > /sys/kernel/debug/dynamic_debug/control  整个文件 打开
echo -n "func demo.c +p" > /sys/kernel/debug/dynamic_debug/control  函数

echo -n "func demo.c -p" > /sys/kernel/debug/dynamic_debug/control 
echo -n "func demo.c -p" > /sys/kernel/debug/dynamic_debug/control 
```

打开dev_vdeg的方法：在C文件开头 #define VERBOSE_DEBUG



### USB-CDC(ACM)

驱动文件: drivers/usb/class/*cdc-acm.c*



发送数据:acm_tty_write

```
struct acm *acm = tty->driver_data;//关键是获取这个结构体
```

接收数据:

https://blog.csdn.net/Luckiers/article/details/123586094

https://blog.csdn.net/tianxiawuzhei/article/details/7556106

https://blog.csdn.net/Luckiers/article/details/123586094

```
static void acm_read_bulk_callback(struct urb *urb) --> static void acm_process_read_urb(struct acm *acm, struct urb *urb) --> tty_flip_buffer_push(给上层)
```

```
[   48.961090] cdc_acm 3-1:1.1: submitted urb 0 //
[   48.961178] cdc_acm 3-1:1.1: submitted urb 1
[   48.961182] cdc_acm 3-1:1.1: submitted urb 2
[   48.961186] cdc_acm 3-1:1.1: submitted urb 3
[   48.961190] cdc_acm 3-1:1.1: submitted urb 4
[   48.961194] cdc_acm 3-1:1.1: submitted urb 5
[   48.961197] cdc_acm 3-1:1.1: submitted urb 6
[   48.961201] cdc_acm 3-1:1.1: submitted urb 7
[   48.961206] cdc_acm 3-1:1.1: submitted urb 8
[   48.961211] cdc_acm 3-1:1.1: submitted urb 9
[   48.961214] cdc_acm 3-1:1.1: submitted urb 10
[   48.961218] cdc_acm 3-1:1.1: submitted urb 11
[   48.961222] cdc_acm 3-1:1.1: submitted urb 12
[   48.961226] cdc_acm 3-1:1.1: submitted urb 13
[   48.961230] cdc_acm 3-1:1.1: submitted urb 14
[   48.961233] cdc_acm 3-1:1.1: submitted urb 15
[   48.961288] cdc_acm 3-1:1.1: got urb 0, len 8, status 0  //acm_read_bulk_callback
[   48.961358] acm_process_read_urb ACM callback readlen:8
```



## 外设调试适配

### GPIO

查看kernel已配置的GPIO

cat /d/gpio

/d/pinctrl/pinctrl-rockchip-pinctrl/pinmux-functions

/d/pinctrl/pinctrl-rockchip-pinctrl/pinmux-pins

echo 97 > /sys/class/gpio/export
echo 97 > /sys/class/gpio/gpio97/direction
echo 1 >/sys/class/gpio/gpio97/value



**需求**：原先DTS中提供的GPIO控制方法是针对RK主控直出的gpio口，现在由于硬件设计，将部分gpio引脚（仅作为GPIO功能使用）移动到了MCU上，通过IIC协议去控制。 相当于IO expander芯片

Linux内核中提供了pinctrl子系统，目的是为了统一各SoC厂商的pin脚管理，避免各SoC厂商各自实现相同的pin脚管理子系统，减少SoC厂商系统移植工作量。

#### 主要功能

1. 管理系统中所有可以控制的pin。在系统初始化的时候，枚举所有可以控制的pin，并标识这些pin。

2. 管理这些pin的复用（Multiplexing）。对于SOC而言，其引脚除了配置成普通GPIO之外，若干个引脚还可以组成一个pin group，形成特定的功能。例如pin number是{ 0, 8, 16, 24 }这四个引脚组合形成一个pin group，提供SPI的功能。pin control subsystem要管理所有的pin group。

3. 配置这些pin的特性。例如配置该引脚上的pull-up/down电阻，配置drive strength等。

4. 与GPIO子系统的交互

5. 实现pin中断


主要是用来创建iomux，初始化引脚，提供服务给gpio子系统使用

#### 核心驱动

kernel/drivers/pinctrl

kernel/drivers/gpio

pinctrl-names

pinctrl-0

有两种方法，一种是写pinctrl驱动+gpio驱动，另外一种是直接写gpio驱动，对于当前需求来说，因为不需要设置gpio外的其他复用功能，所以直接参考io扩展芯片，来写一个gpio子系统的驱动，实现linux标准io口功能。

#### 参考资料

http://www.wowotech.net/gpio_subsystem/io-port-control.html

https://www.freesion.com/article/18631078033/

### GM8775C

### ES8388

#### 调试步骤

* 先检查声卡有没有注册上去，注意I2S有没有打开。

* 看下i2c通讯正不正常。

* 在tinyalsa_hal下面，有es8388_config.h，它会控制读写非驱动接口中写好的控制。

#### 耳机插拔检测

耳机插入把PHE_DET1处的弹片弹开，然后实现headphone_dect直接连通到1.8v，检测端得到高电平1.8v；拔出耳机，弹片回去，只有0.159v电压。所以实现了没有耳机插入时这个“ADC_IN4”为低电位，插入耳机时这个“ADC_IN4”为高电位

### HYM8563

### PT7C4337

#### 调试记录

要实现两个RTC芯片的兼容，需要在probe函数中及时退出某一个未焊接的料的RTC驱动注册。

相关补丁:

### 4G模块

#### 			EC20

### 龙讯LT7911

#### 基础知识

#### 调试记录

### 异形屏/切割屏

### USB-PHY

### MIPI Camera

Sensor Driver -> RKISP2 -> V4L2-ctl -> camera_hal -> cameraManager ->app

Sensor Driver -> RKISP -> v4L2-ctl -> app or gstreamer

Sensor Driver

* v4l2_subdev_ops
* v4l2_ctrl_ops
* regisiter  注册media entity、v4l2 subdev、v4l2 controller
  * v4l2_i2c_subdev_init()，注册为一个 v4l2 subdev，参数中提供回调函数
  * ov5695_initialize_controls()，初始化 v4l2 controls
  * media_entity_init()，注册成为一个 media entity，OV5695 仅有一个输出，即Source Pad
  * v4l2_async_register_subdev()，声明 Sensor 需要异步注册。因为 RKISP1 及 RKCIF 都采用异步注册
    Sub Device，所以这个调用是必须的



### RK628d

HDMI转LVDS或者vbyone（gvi）

rk628工作时需要首先获得hdmi输出的信号，包含分辨率信息，其次如果要转成lvds或者gvi，还需要包含timing信息，两者要一一对应

## 编译

### 电源域 checklist的问题

./scripts/io-domain.sh

 一般来说第一次编译的时候，需要检查一下，后面就不用了。

## MISC

### 出工厂测试固件

3588

* 需要点mipi屏
* 需要将hdmi in 改为tv模式

# Android

## 开发基础

### 环境准备

### 编译

#### Android7,10

生成完整包

完整包所包含内容：system.img、recovery.img、boot.img

发布一个固件正确的顺序：

1、make -j4

2、make otapackage -j4

3、./mkimage.sh ota

发布固件必须使用./mkimage.sh ota,将 boot 与 kernel 打包，不需要单独烧 kernel，如果

量产固件是分开的，将会影响后面差异包升级，除非你不需要用差异升级。

在 out/target/product/rkxxxx/目录下会生成 ota 完整包 rkxxxx-ota-eng.root.zip，改成

update.zip 即可拷贝到 T 卡或者内置的 flash 进行升级。

6.8.3 生成差异包

OTA 差异包只有差异内容，包大小比较小，主要用于 OTA 在线升级，也可 T 卡本地升级。

OTA 差异包制作需要特殊的编译进行手动制作。

1、首先发布 v1 版本的固件，生成 v1 版本的完整包

2、保存

out/target/product/rkxxxx/obj/PACKAGING/target_files_intermediates/rkxxxx

target_files-eng.root.zip 为 rkxxxx-target_files-v1.zip，作为 v1 版本的基础素材包。

3、修改 kernel 代码或者 android 代码，发布 v2 版本固件，生成 v2 版本完整包

4、保存

out/target/product/rkxxxx/obj/PACKAGING/target_files_intermediates/rkxxxx

target_files-eng.root.zip 为 rkxxxx-target_files-v2.zip，作为 v2 版本的基础素材包。

5、生成 v1-v2 的差异升级包：

./build/tools/releasetools/ota_from_target_files -v -i rkxxxx-target_files

v1.zip -p out/host/linux-x86 -k build/target/product/security/testkey rkxxxx

target_files-v2.zip out/target/product/rkxxxx/rkxxxx-v1-v2.zip 

说明:生成差异包命令格式:

ota_from_target_files

-v -i 用于比较的前一个 target file

-p host 主机编译环境

-k 打包密钥

用于比较的后一个 target file

最后生成的 OTA 差异包

#### Android12

a)    整包编译，进入源码根目录下依次执行如下指令

source build/envsetup.sh

lunch rk3399_Android12-userdebug 或 lunch rk3399_Android12-user

./build.sh -UKAup

编译成功后会在rockdev\Image-rk3399_Android12生成固件包，烧录update.img即可

b)    单编译uboot

./build.sh -U

生成uboot.img，可不用烧录trust.img

c)    单编译kernel

cd kernel-4.19

make ARCH=arm64 rockchip_defconfig android-11.config

make ARCH=arm64 BOOT_IMG=../rockdev/ Image-rk3399_Android12/boot.img  rk3399-elo3399_elo_backpack_pvt.img -j24

（其中rk3399-elo3399_elo_backpack_pvt为需编译目标DTS名）

d)    单编译android

./build.sh -A

e)    重新生成固件包

./build.sh -up

##### uboot

##### kernel

make ARCH=arm64 BOOT_IMG=../rockdev/Image-rk3399_Android12/boot.img rk3399-elo3399_elo_iseries_10_mp.img

#### 问题汇总

* recipe for target 'silentoldconfig' failed

报错信息：

```
scripts/kconfig/Makefile:37: recipe for target 'silentoldconfig' failed
make[2]: *** [silentoldconfig] Error 1
Makefile:570: recipe for target 'silentoldconfig' failed
make[1]: *** [silentoldconfig] Error 2
make: *** No rule to make target 'include/config/auto.conf', needed by 'scripts'.  Stop.
```

解决方法：

执行以下命令，删除配置文件后，重新编译，不要带sudo.

如果还遇到rm权限报错，手动删除。

```
sudo make mrproper
```



## 系统知识

### SElinux权限

### 双屏异显

### 分区大小调整

device/rockchip/rk3288/parameter.txt

```
FIRMWARE_VER:7.1
2MACHINE_MODEL:rk3288
3MACHINE_ID:007
4MANUFACTURER:RK3288
5MAGIC: 0x5041524B
6ATAG: 0x60000800
7MACHINE: 3288
8CHECK_MASK: 0x80
9PWR_HLD: 0,0,A,0,1
10CMDLINE: console=ttyFIQ0 androidboot.baseband=N/A androidboot.selinux=permissive androidboot.hardware=rk30board androidboot.console=ttyFIQ0 init=/init initrd=0x62000000,0x00800000 mtdparts=rk29xxnand:0x00002000@0x00002000(uboot),0x00002000@0x00004000(trust),0x00002000@0x00006000(misc),0x00008000@0x00008000(resource),0x0000C000@0x00010000(kernel),0x00010000@0x0001C000(boot),0x00010000@0x0002C000(recovery),0x00038000@0x0003C000(backup),0x00040000@0x00074000(cache),0x00400000@0x000B4000(system),0x00008000@0x004B4000(metadata),0x00019000@0x004BC000(vendor0),0x00019000@0x004D5000(vendor1),-@0x004EE000(userdata)
```

比如system分区的信息是  0x00400000@0x000B4000(system),表示大小是0x00400000 * 512字节,开始地址是0x000B4000，另外分区对齐是4MB，所以调试的时候需要是4MB的整数倍。

此外 还需要注意在device/rockchip/common 目录下有个baseparameter_rk3288.txt , 会覆盖rk3288目录下的，所以需要改这边。另外还需要修改rockdev下面的parameter.txt，跟打包相关。否则会提示vendor0分区大小不足。

### 调试log打开方式



### API接口开发

#### 流程

##### Android7.1

signwayManger->ISignwayManager.aidl->SignwayManger.java->SignwayManagerServices.java（如果涉及到kernel底层，往下走）->HardwareControl.java->NativeApi.java->com_android_server_signwaymanager_NativeApi.cpp->Hal层（libhardware/modules/signway/signway.c）->fopen, ioctl



#### 功能

##### ModuleResult<Boolean> setResolution(int screen, String resolution) 

设置LVDS分辨率：



设置HDMI分辨率

3568 ：通过设置 *persist.vendor.resolution.HDMI-A-0*设置hdmi分辨率输出，以及persist.vendor.framebuffer.main / aux来设置fb大小，保证UI正常显示





### ADB调试命令

#### 查看系统内存大小

```
cat /proc/meminfo

```



## 定制修改

### 锁屏、息屏

 屏幕超时默认为永不休眠l

```xml
frameworks/base/packages/SettingsProvider/res/values/defaults.xml
<integer name="def_screen_off_timeout">2147483647</integer>
```

关闭锁屏设置

```xml
framework/base/core/res/res/values/config.xml
<bool name="config_disableLockscreenByDefault">true</bool>
```

### 时间、时区、语言

```java
NTP时间同步服务器
源码路径：frameworks/base/core/java/android/util/NtpTrustedTime.java
默认ntp地址是：frameworks/base/core/res/res/values/config.xml
<string translatable="false" name="config_ntpServer">2.android.pool.ntp.org</string>
国内的ntp地址是：
<string translatable="false" name="config_ntpServer">cn.pool.ntp.org</string>

系统设置里面同步时间默认打开
frameworks/base/packages/SettingsProvider/res/values/defaults.xml
 <bool name="def_auto_time">true</bool>

时间显示默认24小时制
frameworks/base/packages/SettingsProvider/res/values/defaults.xml
<string name="def_time_12_24" translatable="false">24</string>

frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/DatabaseHelper.java
private void loadSystemSettings(SQLiteDatabase db) {
	...
	loadStringSetting(stmt, Settings.System.TIME_12_24,R.string.def_time_12_24);
	/*
	* IMPORTANT: Do not add any more upgrade steps here as the global,
	* secure, and system settings are no longer stored in a database
	...
}

```

时区  

```xml
frameworks/base/packages/SettingsProvider/res/values/defaults.xml
系统设置里面同步时区默认关闭，另自动同步时区，需要接4G模块才会生效，之前查看android同步时区流程，需要用到移动网络相关服务
<bool name="def_auto_time_zone">false</bool> // 自动时区关闭

android标准设置：
persist.sys.timezone=Asia/Shanghai

AIoT3568：
vendor.persist.sys.timezone=Asia/Shanghai
```

语言

```
ro.product.locale=zh-CN 中文
ro.product.locale=en-US 英文
```

### 亮度、DPI 

DPI

修改prop属性，公司标准1920x1080及以下的为160，4K输出时为280

```makefile
ro.sf.lcd_density=160
```

亮度  

gamma liner算法  
android高版本后，系统在设置亮度时增加了gamma liner算法，默认需要去掉该功能  
rk3568-11, rk3588-12平台  
frameworks/base/packages/SettingsLib/src/com/android/settingslib/display/BrightnessUtils.java

```java
--- a/packages/SettingsLib/src/com/android/settingslib/display/BrightnessUtils.java
+++ b/packages/SettingsLib/src/com/android/settingslib/display/BrightnessUtils.java
@@ -23,6 +23,8 @@ public class BrightnessUtils {
     public static final int GAMMA_SPACE_MIN = 0;
     public static final int GAMMA_SPACE_MAX = 65535;

+    public static final boolean ENABLE_GAMMA = false;
+
     // Hybrid Log Gamma constant values
     private static final float R = 0.5f;
     private static final float A = 0.17883277f;
@@ -53,6 +55,9 @@ public class BrightnessUtils {
      */
     public static final int convertGammaToLinear(int val, int min, int max) {
         final float normalizedVal = MathUtils.norm(GAMMA_SPACE_MIN, GAMMA_SPACE_MAX, val);
+        if (!ENABLE_GAMMA) {
+            return Math.round(MathUtils.lerp(min, max, normalizedVal));
+        }
         final float ret;
         if (normalizedVal <= R) {
             ret = MathUtils.sq(normalizedVal / R);
@@ -76,6 +81,9 @@ public class BrightnessUtils {
      */
     public static final float convertGammaToLinearFloat(int val, float min, float max) {
         final float normalizedVal = MathUtils.norm(GAMMA_SPACE_MIN, GAMMA_SPACE_MAX, val);
+        if (!ENABLE_GAMMA) {
+            return MathUtils.lerp(min, max, normalizedVal);
+        }
         final float ret;
         if (normalizedVal <= R) {
             ret = MathUtils.sq(normalizedVal / R);
@@ -127,6 +135,10 @@ public class BrightnessUtils {
      * @return The corresponding slider value
      */
     public static final int convertLinearToGammaFloat(float val, float min, float max) {
+        final float normalizedValTemp = MathUtils.norm(min, max, val);
+        if (!ENABLE_GAMMA) {
+            return Math.round(MathUtils.lerp(GAMMA_SPACE_MIN, GAMMA_SPACE_MAX, normalizedValTemp));
+        }
         // For some reason, HLG normalizes to the range [0, 12] rather than [0, 1]
         final float normalizedVal = MathUtils.norm(min, max, val) * 12;
         final float ret;
```

​		1.3.2.2 修改默认亮度80%  

```xml
frameworks/base/packages/SettingsProvider/res/values/defaults.xml
<integer name="def_screen_brightness">205</integer>
```

​	AIoT3588上述修改不生效，修改如下：

```xml
frameworks/base/core/res/res/values/config.xml
<integer name="config_screenBrightnessSettingDefault">205</integer>
```

### 文件系统

新开发的主板，文件系统全部换成ext4，android默认是f2fs，掉电会丢数据。修改方法参考RK文档里面对应的平台开发手册。

rk3568-11平台  
device/rockchip/rk356x/rk3568_r/recovery.fstab

```makefile
--- a/rk3568_r/recovery.fstab
+++ b/rk3568_r/recovery.fstab
@@ -9,7 +9,8 @@
 /dev/block/by-name/system_ext            /system_ext          ext4             defaults                  defaults
 /dev/block/by-name/cache                 /cache               ext4             defaults                  defaults
 /dev/block/by-name/metadata              /metadata            ext4             defaults                  defaults
-/dev/block/by-name/userdata              /data                f2fs             defaults                  defaults
+#/dev/block/by-name/userdata              /data                f2fs             defaults                  defaults
+/dev/block/by-name/userdata              /data                ext4             defaults                  defaults
 /dev/block/by-name/cust                  /cust                ext4             defaults                  defaults
 /dev/block/by-name/custom                /custom              ext4             defaults                  defaults
 /dev/block/by-name/radical_update        /radical_update      ext4             defaults                  defaults
```

device/rockchip/common/scripts/fstab_tools/fstab.in

```makefile
--- a/scripts/fstab_tools/fstab.in
+++ b/scripts/fstab_tools/fstab.in
@@ -23,6 +23,6 @@ ${_block_prefix}system_ext /system_ext  ext4 ro,barrier=1 ${_flags},first_stage_
 # For sdmmc
 /devices/platform/${_sdmmc_device}/mmc_host*        auto  auto    defaults        voldmanaged=sdcard1:auto
 #  Full disk encryption has less effect on rk3326, so default to enable this.
-/dev/block/by-name/userdata /data f2fs noatime,nosuid,nodev${_sync_flags},discard,reserve_root=32768,resgid=1065 latemount,wait,check,fileencryption=aes-256-xts:aes-256-cts:v2+inlinecrypt_optimized,keydirectory=/metadata/vold/metadata_encryption,quota,formattable,reservedsize=128M,checkpoint=fs
+#/dev/block/by-name/userdata /data f2fs noatime,nosuid,nodev${_sync_flags},discard,reserve_root=32768,resgid=1065 latemount,wait,check,fileencryption=aes-256-xts:aes-256-cts:v2+inlinecrypt_optimized,keydirectory=/metadata/vold/metadata_encryption,quota,formattable,reservedsize=128M,checkpoint=fs
 # for ext4
-#/dev/block/by-name/userdata    /data      ext4    discard,noatime,nosuid,nodev${_sync_flags},noauto_da_alloc,data=ordered,user_xattr,barrier=1    latemount,wait,formattable,check,fileencryption=software,quota,reservedsize=128M,checkpoint=block
+/dev/block/by-name/userdata    /data      ext4    discard,noatime,nosuid,nodev${_sync_flags},noauto_da_alloc,data=ordered,user_xattr,barrier=1    latemount,wait,formattable,check,fileencryption=software,quota,reservedsize=128M,checkpoint=block
```

device/rockchip/common/scripts/fstab_tools/fstab_go.in

```makefile
--- a/scripts/fstab_tools/fstab_go.in
+++ b/scripts/fstab_tools/fstab_go.in
@@ -17,6 +17,6 @@ ${_block_prefix}system_ext  /system_ext ext4 ro,barrier=1 ${_flags},first_stage_
 # For sdmmc
 /devices/platform/${_sdmmc_device}/mmc_host*        auto  auto    defaults        voldmanaged=sdcard1:auto
 #  Full disk encryption has less effect on rk3326, so default to enable this.
-/dev/block/by-name/userdata /data f2fs noatime,nosuid,nodev,discard${_sync_flags},reserve_root=32768,resgid=1065 latemount,wait,check,fileencryption=aes-256-xts:aes-256-cts:v2+inlinecrypt_optimized,keydirectory=/metadata/vold/metadata_encryption,quota,formattable,reservedsize=128M,checkpoint=fs
+#/dev/block/by-name/userdata /data f2fs noatime,nosuid,nodev,discard${_sync_flags},reserve_root=32768,resgid=1065 latemount,wait,check,fileencryption=aes-256-xts:aes-256-cts:v2+inlinecrypt_optimized,keydirectory=/metadata/vold/metadata_encryption,quota,formattable,reservedsize=128M,checkpoint=fs
 # for ext4
-#/dev/block/by-name/userdata    /data      ext4    discard,noatime,nosuid,nodev,noauto_da_alloc${_sync_flags},data=ordered,user_xattr,barrier=1    latemount,wait,formattable,check,fileencryption=software,quota,reservedsize=128M,checkpoint=block
+/dev/block/by-name/userdata    /data      ext4    discard,noatime,nosuid,nodev,noauto_da_alloc${_sync_flags},data=ordered,user_xattr,barrier=1    latemount,wait,formattable,check,fileencryption=software,quota,reservedsize=128M,checkpoint=block
```

### 版本号、设备号

固件版本号  

用脚本编译时会自动生成版本号，对应的prop名称为ro.build.signway.board和ro.build.display.id。  

RK3568-11的标准版是多个主板使用同一个固件，因此在Build.java和SignwayManagerService里面会根据ro.boot.hwid信息，重新名称版本号，替换版本号里面的主板名称字符串。  
具体查看文档：《固件兼容设计文档》  

设备号  
根据ro.boot.hwid信息，显示主板名称  

### 系统方向

Android7.1:

```java
ro.sf.hwrotation=0/90/180/270
副屏HDMI的旋转ro.orientation.einit=0/90/180/270
```

android11:

```java
系统配置：
SF_PRIMARY_DISPLAY_ORIENTATION=0/90/180/270
副屏HDMI的旋转persist.sys.rotation.einit-1=0/1/2/3 对应方向0/90/180/270
副屏参数在助手设置旋转方向时，同步设置

标准代码配置：
persist.sdkapk.panel_rotation=0/90/180/270
```

旋转方向设计流程步骤：

因为系统方向是ro开头的属性，ro属性是不保存的，所以需要将方向的ro属性值保存下来，下次开机方向才能正确。  

AIoT3568，AIoT3588的修改代码如下：

a). 修改ro属性可以设置, 开机时通过获取的persist属性值重新设置ro系统属性

system/core/init/property_service.cpp

```c++
--- a/init/property_service.cpp
+++ b/init/property_service.cpp
@@ -188,10 +188,10 @@ static uint32_t PropertySet(const std::string& name, const std::string& value, s
     prop_info* pi = (prop_info*) __system_property_find(name.c_str());
     if (pi != nullptr) {
         // ro.* properties are actually "write-once".
-        if (StartsWith(name, "ro.")) {
-            *error = "Read-only property was already set";
-            return PROP_ERROR_READ_ONLY_PROPERTY;
-        }
+        //if (StartsWith(name, "ro.")) {
+        //    *error = "Read-only property was already set";
+        //    return PROP_ERROR_READ_ONLY_PROPERTY;
+        //}

         __system_property_update(pi, value.c_str(), valuelen);
     } else {
@@ -780,6 +780,28 @@ static void update_sys_usb_config() {
     }
 }
      
+static void update_vendor_signway_configs() {
+    std::string panel_rotation = android::base::GetProperty("persist.sdkapk.panel_rotation", "");
+    std::string last_rotation = "ORIENTATION_";
+    if (!panel_rotation.empty()) {
+        if ((panel_rotation=="0") || (panel_rotation=="90")
+            || (panel_rotation=="180") || (panel_rotation=="270")) {
+            last_rotation += panel_rotation;
+            InitPropertySet("ro.surface_flinger.primary_display_orientation", last_rotation);
+        }
+        InitPropertySet("persist.sys.rotation_test", last_rotation);
+    } else {
+        InitPropertySet("persist.sys.rotation_test", "none");
+    }
+    std::string lcd_density = android::base::GetProperty("persist.sdkapk.lcd_density", "");
+    if (!lcd_density.empty()) {
+        InitPropertySet("ro.sf.lcd_density", lcd_density);
+        InitPropertySet("persist.sys.lcd_density_test", lcd_density);
+    } else {
+        InitPropertySet("persist.sys.lcd_density_test", "none");
+    }
+}
+
 static void load_override_properties() {
     if (ALLOW_LOCAL_PROP_OVERRIDE) {
         std::map<std::string, std::string> properties;
@@ -1298,6 +1320,7 @@ static void HandleInitSocket() {
             }
             InitPropertySet("ro.persistent_properties.ready", "true");
             persistent_properties_loaded = true;
+            update_vendor_signway_configs();
             break;
         }
         default:
```

b). 延后启动SurfaceFlinger服务，该服务属于core核心服务，启动时间比加载persist数据早

```c++
--- a/services/surfaceflinger/SurfaceFlinger.cpp
+++ b/services/surfaceflinger/SurfaceFlinger.cpp
@@ -127,6 +127,8 @@
 #include "android-base/parseint.h"
 #include "android-base/stringprintf.h"

+#include <unistd.h>
+
 #define MAIN_THREAD ACQUIRE(mStateLock) RELEASE(mStateLock)

 #define ON_MAIN_THREAD(expr)                                       \
@@ -359,6 +361,7 @@ SurfaceFlinger::SurfaceFlinger(Factory& factory) : SurfaceFlinger(factory, SkipI

     useContextPriority = use_context_priority(true);

+    sleep(2); //for ro.surface_flinger.primary_display_orientation delay 1s, jian.xu 2021-11-30
     using Values = SurfaceFlingerProperties::primary_display_orientation_values;
     switch (primary_display_orientation(Values::ORIENTATION_0)) {
         case Values::ORIENTATION_0:

```

### 输入法

Android修改默认输入法，需要将对应的输入法预置到固件里面

配置参数和加载参数补丁如下：

```java
frameworks/base/packages/SettingsProvider/res/values/defaults.xml
<string name="def_input_method" translatable="false">com.android.inputmethod.pinyin/.PinyinIME</string>

frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/DatabaseHelper.java
private void loadSecureSettings(SQLiteDatabase db) {
	...
	loadStringSetting(stmt, Settings.Secure.DEFAULT_INPUT_METHOD,R.string.def_input_method );
	loadStringSetting(stmt, Settings.Secure.ENABLED_INPUT_METHODS,R.string.def_input_method );
	/*
	*IMPORTANT: Do not add any more upgrade steps here as the global,
	*
	...
}

市场上输入法的包名：
搜狗拼音输入法：com.android.inputmethod.pinyin/.PinyinIME
百度TV输入法：com.baidu.input_baidutv/.ImeService
百度：com.baidu.input/.ImeService
讯飞：com.iflytek.inputmethod/.FlyIME
腾讯：com.tencent.qqpinyin/.QQPYInputMethodService
谷歌：com.google.android.inputmethod.pinyin/.PinyinIME
搜狗：com.sohu.inputmethod.sogou/.SogouIME 或者 com.sohu.inputmethod.sogouoem/.SogouIME
触宝：com.cootek.smartinput5/.TouchPalIME
Android： com.android.inputmethod.latin/.LatinIME
```

### APN

apn代码里面的文件名称 apns-full-conf.xml

rk3568-11，rk3588-12

源码文件路径：vendor/rockchip/common/phone/etc/apns-full-conf.xml

adb调试路径：system/etc/apns-conf.xml

### 开机广播

frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java

```java
--- a/services/core/java/com/android/server/am/ActivityManagerService.java
+++ b/services/core/java/com/android/server/am/ActivityManagerService.java
@@ -12835,7 +12835,8 @@ public class ActivityManagerService extends IActivityManager.Stub
         }

         // By default broadcasts do not go to stopped apps.
-        intent.addFlags(Intent.FLAG_EXCLUDE_STOPPED_PACKAGES);
+        //intent.addFlags(Intent.FLAG_EXCLUDE_STOPPED_PACKAGES);
+        intent.addFlags(Intent.FLAG_INCLUDE_STOPPED_PACKAGES);

         // If we have not finished booting, don't allow this to launch new processes.
         if (!mProcessesReady && (intent.getFlags()&Intent.FLAG_RECEIVER_BOOT_UPGRADE) == 0) {
```

### 浏览器,webview

android升级webview版本

| Name            | PackageName                | 获取方式    | 自动更新 | 稳定性 |
| --------------- | -------------------------- | ----------- | -------- | ------ |
| Android WebView | com.android.webview        | Android自带 | 否       | 最高   |
| Chrome Stable   | com.android.chrome         | Chrome自带  | 可       | 高     |
| Google WebView  | com.google.android.webview | 随GMS包发布 | 可       | 高     |
| Custom Webview  | com.android.webview        | 自编译      | 否       | 中     |

修改系统webview的包名

For android 6.0 & before

/frameworks/base/core/res/res/values/confifig.xml

```xml
<string name="config_webViewPackageName"
translatable="false">com.android.webview</string>
```

For android 7.0 & after

frameworks/base/core/res/res/xml/confifig_webview_packages.xml

```xml
<webviewproviders>
 <!-- The default WebView implementation -->
 <webviewprovider description="Android WebView"
packageName="com.android.webview" availableByDefault="true">
</webviewprovider>
 <webviewprovider description="Chrome Stable"
packageName="com.android.chrome" availableByDefault="true" />
 <webviewprovider description="Google WebView"
packageName="com.google.android.webview" availableByDefault="true"
isFallback="true" />
</webviewproviders>
```

系统在开机过程中会自动根据这个配置文件中的顺序来搜索设备中已安装并启用的包信息，找到以后直接返回，例如上面配置中的三个发行版本如果都安装并启用了，则默认的包名是com.android.webview.



### 修改显示图像framebuffer的大小以及初始位置

​		./hardware/rockchip/hwcomposer

```C
diff --git a/hwcomposer.cpp b/hwcomposer.cpp
index 4061f73..277a30c 100755
--- a/hwcomposer.cpp
+++ b/hwcomposer.cpp
@@ -1011,6 +1011,10 @@ int DrmHwcLayer::InitFromHwcLayer(struct hwc_context_t *ctx, int display, hwc_la
   }else
 #endif
 
+  uint32_t x_offset= 0,y_offset =0;
+  char framebuffer_offset[PROPERTY_VALUE_MAX] = {0};
+  property_get("persist." PROPERTY_TYPE ".framebuffer.offset", framebuffer_offset, "0,0");
+  sscanf(framebuffer_offset, "%d,%d", &x_offset, &y_offset);
   {
     source_crop = DrmHwcRect<float>(
         sf_layer->sourceCropf.left, sf_layer->sourceCropf.top,
@@ -1039,8 +1043,8 @@ int DrmHwcLayer::InitFromHwcLayer(struct hwc_context_t *ctx, int display, hwc_la
       else
       {
         display_frame = DrmHwcRect<int>(
-          hd->w_scale * sf_layer->displayFrame.left, hd->h_scale * sf_layer->displayFrame.top,
-          hd->w_scale * sf_layer->displayFrame.right, hd->h_scale * sf_layer->displayFrame.bottom);
+          hd->w_scale * sf_layer->displayFrame.left + x_offset, hd->h_scale * sf_layer->displayFrame.top + y_offset,
+          hd->w_scale * sf_layer->displayFrame.right + x_offset, hd->h_scale * sf_layer->displayFrame.bottom + y_offset);
       }
   }
 
@@ -2634,8 +2638,10 @@ static int hwc_prepare(hwc_composer_device_1_t *dev, size_t num_displays,
     hd->rel_xres = mode.h_display();
     hd->rel_yres = mode.v_display();
     hd->v_total = mode.v_total();
-    hd->w_scale = (float)mode.h_display() / hd->framebuffer_width;
-    hd->h_scale = (float)mode.v_display() / hd->framebuffer_height;
+    //hd->w_scale = (float)mode.h_display() / hd->framebuffer_width;
+    //hd->h_scale = (float)mode.v_display() / hd->framebuffer_height;
+    hd->w_scale = 1.0;
+    hd->h_scale = 1.0;
     int fbSize = hd->framebuffer_width * hd->framebuffer_height;
     //get plane size for display
     std::vector<PlaneGroup *>& plane_groups = ctx->drm.GetPlaneGroups();
@@ -4125,8 +4131,10 @@ static int hwc_set_active_config(struct hwc_composer_device_1 *dev, int display,
     ALOGE("Could not find active mode for display=%d", display);
     return -ENOENT;
   }
-  hd->w_scale = (float)mode.h_display() / hd->framebuffer_width;
-  hd->h_scale = (float)mode.v_display() / hd->framebuffer_height;
+  //hd->w_scale = (float)mode.h_display() / hd->framebuffer_width;
+  //hd->h_scale = (float)mode.v_display() / hd->framebuffer_height;
+  hd->w_scale = 1.0;
+  hd->h_scale = 1.0;
 
   c->set_current_mode(mode);
   ctx->drm.UpdateDisplayRoute();

```

### service增加API接口

​	  service: .aidl -> serviceBinder.java -> xxxHandle.java -> InterfaceXXXX.java -> XXX.Base.java(Override)

### 设置界面增加一个按钮

数组型：

packages/apps/Settings/

./res/values-zh-rCN/strings.xml //中文按钮名

./res/values/strings.xml //英语按钮名

./res/values/arrays.xml //数组值 "navbar_entries"  key  "navbar_values" value

./res/xml/display_settings.xml // 描述按钮样式

./src/com/android/settings/display/xxx.java //按钮控制回调接口函数

### 以太网相关

安卓系统中以太网，WIFI，4G是有优先级关系的，在原生安卓系统中，以太网 >WIFI>4G，而且在插入网线后,wifi会断开连接。

现在要增加开关以及WIFI，以太网共存。

**增加单独控制开关：**

需要在Setting里调用相关manager

**共存：**

**双MAC地址**

### 适配一个耳麦类型的USB声卡

两个测试网站：

mictests.com

webcamtests.com

```cpp
shg@SERVER-CZXT-79:~/work2/rk3399_10/android/frameworks/av$ git diff
diff --git a/services/audiopolicy/enginedefault/src/Engine.cpp b/services/audiopolicy/enginedefault/src/Engine.cpp
index b702be087..8af369bc4 100644
--- a/services/audiopolicy/enginedefault/src/Engine.cpp
+++ b/services/audiopolicy/enginedefault/src/Engine.cpp
@@ -625,6 +625,9 @@ audio_devices_t Engine::getDeviceForInputSource(audio_source_t inputSource) cons
              * if usb mic is connected, use it first
              */
             device = AUDIO_DEVICE_IN_USB_DEVICE;
+        } else if (availableDeviceTypes & AUDIO_DEVICE_IN_USB_HEADSET) {
+            ALOGW(">> device = AUDIO_DEVICE_IN_USB_HEADSET");
+            device = AUDIO_DEVICE_IN_USB_HEADSET;
         } else if (availableDeviceTypes & AUDIO_DEVICE_IN_BACK_MIC) {
             device = AUDIO_DEVICE_IN_BACK_MIC;
         } else if (availableDeviceTypes & AUDIO_DEVICE_IN_BUILTIN_MIC) {
```

### SN号

关键属性:**ro.serialno 与 ro.boot.serialno**

system/core/drmservice/drmservice.c

SN工具写的是内部flash，在u-boot中读取，并且传给cmdline。最后自动赋值给ro.boot.serialno（注意只要是cmdline里是androidboot开头的，会在软件框架中自己生成ro.xxxx.节点）

SN写号工具如果勾选兼容模式，读的是IDB分区（老），如果不勾选，就是vendor分区。 暂时不用管RPMB

添加cmdline位置：

3288 7.1   uboot/common/cmd_bootrk.c

3399    uboot/common/android_bootloader.c



在system/core/drmservice/drmservice/main里，如果打开了SERIALNO_FROM_IDB，先读vendor分区里的，然后设置sys.serialno，最后由上层

`device/rockchip/common/init.rk30board.rc`中的：

```
on property:sys.serialno=*
    setprop ro.serialno ${sys.serialno}
```

同步给ro.serialno。

### HDMI栏里增加4K选项

思路：首先kernel层的edid要强制的加上4K的edid，其次在白名单xml里选择要显示的分辨率，当然包括4k的几个选项

rk3399最高支持4k 25hz的刷新，所以去掉一些50,60的分辨率。

经过测试，RK3399 Android10目前只支持将HDMI设置成主屏，并且将framebuffer.main配置成3840x2160。否则会在4k的分辨率下，屏幕画面只显示四分之一。

所以在HDMI设置栏里，在切换成4K分辨率的时候，需要加上判断逻辑，并且fb的设置重启后才能生效。

研究下packages/apps/Settings/src/com/android/settings/display 要把Hdmi设置成主屏，才能正确获取分辨率

出现桌面hotset 绿屏问题，需要把hwc.complicy =0或者打上RK的hwcomposer补丁(忘了从哪里搞来的  找了半天 rk文档 redmine QQ群 百度都找过了)

https://blog.csdn.net/Fighting4344/article/details/108172173

hwc.compose_policy=0 //改为GPU合成

最终应该是打上这个补丁生效的：

```
diff --git a/hwcomposer.cpp b/hwcomposer.cpp
index 277a30c..93d7bcc 100755
--- a/hwcomposer.cpp
+++ b/hwcomposer.cpp
@@ -1328,7 +1328,23 @@ int DrmHwcLayer::InitFromHwcLayer(struct hwc_context_t *ctx, int display, hwc_la
 #endif
     name = layername;
 #endif
-
+    // DrmHwc1 : Add a hardware restriction.
+    // The Vop1 cannot output the images of the following scenes:
+    //   1. src_w > 2560, h_scale_mul == 1, v_scale_mul != 1
+    //   2. src_w > 2560, h_scale_mul != 1, dst_w > 2560, v_scale_mul != 1
+    if(!is_yuv && src_w >= 2560){
+      if(h_scale_mul == 1.0){
+        if(v_scale_mul != 1.0){
+            bSkipLayer = true;
+        }
+      }else{
+        if(dst_w >= 2560){
+          if(v_scale_mul != 1.0){
+              bSkipLayer = true;
+          }
+        }
+      }
+    }

```



### 默认音量为50%

安卓声音是分成多个种类的，每种音量在源码中有一个封顶的值以及一个默认值，改那边即可

https://blog.csdn.net/CodingNotes/article/details/107287084

### 默认亮度80%

 frameworks\base\packages\SettingsProvider\res\values\defaults.xml

def_screen_brightness  255x0.8

如果设置不生效，有可能被overylay了注意检查

安卓11和12是 frameworks\base\res\  改config_screenBrightnessSettingDefault

对应的值需要通过adb shell get system brightness 调试屏幕来获取实际的值（80%对应94 原生非线性）

### TP触摸唤醒

services/inputflinger/InputReader.cpp
     // Initial downs on external touch devices should wake the device.
     // Normally we don't do this for internal touch screens to prevent them from waking
     // up in your pocket but you can enable it using the input device configuration.

mParameters.wake = true;

流程：inputreader -> eventhub -> inputdevice -> inputMapper -> touchmapper

### Audio Policy

![](F:\note_doc\pic\2022年6月27日144217.png)

1、es8388； 2、hdmi； 3、dp； 4、usb；5、bt;

几个关键文件查看log记录：

elo3399_10/android/hardware/rockchip/audio/tinyalsa_hal

frameworks/base/services/core/java/com/android/server/WiredAccessoryManager.java

frameworks/av/services/udiopolicy/enginedefault/src/Engine.cpp

相关问题：

* 261107  rk3399 android10 蓝牙音频问题
* 280802 rk3399 android10 蓝牙音频问题录音

流程梳理：

* kernel层具体codec驱动（例如es8388，rk817等）与DAI被simple-aduio-card驱动组合起来，形成一个声卡，供HAL调用
* HAL层指的就是hardware/rockchip/audio/tinyhal_audio，提供pcm方法给上层调用
* WiredAccessoryManager.java用来监听设备插拔事件，配置上层的输出通道
* Engine.cpp是来确定最终的输出通道，依靠上层的输出通道信息以及操作HAL层接口

修改前的音频测试状况

* Speaker和DP同时有声音
* Speaker+DP后，连接BT，只有DP有声音 （DP优先级比BT高了）
* Speaker+DP后，插入USB。 USB跟DP同时有声音
* Speaker+BT后，插入USB，没声音，再插DP没声音

归纳：BT优先级调到最高，USB第二，DP第三，SPKEAKER第四

修改后：

* 插上再拔掉USB后，BT没声音
* 切换DP policy以后，要切下一首歌才行
* 切换BT policy后，插拔USB后，声音到USB了
* 断开蓝牙以后，DP如果是开机后插拔的，则没声音，复现：开机+BT 然后插DP，断开BT。 偶现

* 有设置不保存的现象

### 8K分辨率

3588支持8k分辨率

需要把vp0 vp1都给hdmi0使用，确保其他port不占用

resolution_white.xml 上层过滤，确保没有屏蔽掉8k分辨率，即

```
	<resolution> <!-- 7680x4320P60 -->
		<clock>2376000</clock>
		<hdisplay>7680</hdisplay>
		<hsync_start>8232</hsync_start>
		<hsync_end>8408</hsync_end>
		<htotal>9000</htotal>
		<hskew>0</hskew>
		<vdisplay>4320</vdisplay>
		<vsync_start>4336</vsync_start>
		<vsync_end>4356</vsync_end>
		<vtotal>4400</vtotal>
		<vscan>0</vscan>
		<vrefresh>60</vrefresh>
		<flags>5</flags>
		<vic>109</vic>
	</resolution>
```



### 开机执行脚本文件

新云网有个BUG，需要开机录一次音

device/rockchip/common

```
diff --git a/init.rk30board.rc b/init.rk30board.rc
index a4aa0a7..e8b4b2e 100755
--- a/init.rk30board.rc
+++ b/init.rk30board.rc
@@ -313,3 +313,9 @@ service iso_operate /system/bin/iso
 service rk_store_keybox /system/bin/rk_store_keybox
     class main
     oneshot
+
+service customer_service /system/vendor/bin/customer_service.sh
+    class main
+    user root
+    group root
+    oneshot
\ No newline at end of file
diff --git a/sepolicy/file_contexts b/sepolicy/file_contexts
index 040d3c1..78d64cf 100755
--- a/sepolicy/file_contexts
+++ b/sepolicy/file_contexts
@@ -160,3 +160,5 @@

 #for multi-ethernet interface
 /system/bin/lan_bridge_on.sh   u:object_r:lan_bridge_on_exec:s0
+
+/system/vendor/bin/customer_service.sh u:object_r:customer_service_exec:s0
\ No newline at end of file

```

### 屏蔽WIFI

设置栏，下拉框里需要屏蔽wifi

Android 5.1

```
//设置选项
diff --git a/src/com/android/settings/SettingsActivity.java b/src/com/android/settings/SettingsActivity.java
index 3f646c095..40c90a9b7 100755
--- a/src/com/android/settings/SettingsActivity.java
+++ b/src/com/android/settings/SettingsActivity.java
@@ -1167,9 +1167,9 @@ public class SettingsActivity extends Activity
                     }
                 } else if (id == R.id.wifi_settings) {
                     // Remove WiFi Settings if WiFi service is not available.
-                    if (!getPackageManager().hasSystemFeature(PackageManager.FEATURE_WIFI)) {
+                    //if (!getPackageManager().hasSystemFeature(PackageManager.FEATURE_WIFI)) {
                         removeTile = true;
-                    }
+                    //}
                 } else if (id == R.id.bluetooth_settings) {
                     // Remove Bluetooth Settings if Bluetooth service is not available.
                     if (!getPackageManager().hasSystemFeature(PackageManager.FEATURE_BLUETOOTH)||(SystemPropert
     
//下拉框     
diff --git a/packages/SystemUI/src/com/android/systemui/statusbar/phone/QSTileHost.java b/packages/SystemUI/src/com/android/systemui/statusb
ar/phone/QSTileHost.java
old mode 100644
new mode 100755
index 37de036af21..a6cb0c73966
--- a/packages/SystemUI/src/com/android/systemui/statusbar/phone/QSTileHost.java
+++ b/packages/SystemUI/src/com/android/systemui/statusbar/phone/QSTileHost.java
@@ -257,7 +257,7 @@ public class QSTileHost implements QSTile.Host {
     }

     private QSTile<?> createTile(String tileSpec) {
-        if (tileSpec.equals("wifi")) return new WifiTile(this);
+        if (tileSpec.equals("wifi")) return new FlashlightTile(this);
         else if (tileSpec.equals("bt")) return new BluetoothTile(this);
         else if (tileSpec.equals("inversion")) return new ColorInversionTile(this);
         else if (tileSpec.equals("cell")) return new CellularTileForSlot(this, PhoneConstants.SIM_ID_1);

```



#### 存储分区大小

客户反馈在设置->存储 界面，读到的内部存储空间小了1个G

一开始以为这个内部存储空间是ddr，被误导了，其实是/data分区的大小。

通过追踪代码，最终在 StorageMeasurement.java文件的measureApproximateStorage方法中，将这个大小获取的方式弄了出来。

读的是/data分区大小的信息。

所以只需要在打包的时候，将parameter.txt中的分区调整一下就可以



### 设备参数信息初始化及导入

#### 目前设备参数加载流程：

宏：SIGNWAY_VENDOR_DTS

上电 ->uboot阶段， 从vendor分区读取不同类型的数据->结合dts宏，配置other_info，app_info，display_info，mipi_cmds_info四个结构体

->设置对应的env->写入cmdline，传给内核驱动使用->内核驱动中判断cmdline中的参数，如果support=1，则使用传入的值，否则用dts值或者默认值。

目前U盘导入参数类相关功能流程：

解析U盘中的Json文件->app调用不同的api接口->api接口去填充结构体或者直接调用signwaymanger接口

优点：

1.基础功能已经实现

缺点：

主要是扩展性

1.新加参数与设备已存在的配置参数信息，兼容性不好，

2.新增一个参数，api部分需要改动的部分较多，且目前好几个接口去操作vendor分区，有些混杂。

vendor_read（）， vendor_type_read（）backlightConfigsSave（）speakerConfigsRead（）...

3.hal层实现比较复杂，uboot改完结构体，对应hal层也要改

计划改进之后的参数加载流程：

简介：cJSON是一个使用C语言编写的JSON数据解析器，具有超轻便，可移植，单文件的特点。

在内存中的以标准json格式字符串的形式存储。

```JSON
{
"confVersion": "1.0.1",
"productName": "AI3399C",
"lvds0":{
    "timing_index":"1920x1080@60",
    "lvds_mode":0,
    "lvds_swap":0,
    "lvds_channel":1
},
"lvds1":{
    "timing_index":"1920x1080@60",
    "lvds_mode":1,
    "lvds_swap":0,
    "lvds_channel":0
},
"graphic":{
    "status":1,
    "x_pos":0,
    "y_pos":0,
    "width":1920,
    "height":540
},
"v-by-one":{
    "single-dual-mode":1
},
"backlight":{
    "bl_screentype":1,
    "bl_mode":1,
    "bl_inverse":0,
    "bl_min":0,
    "bl_max":100,
    "bl_freq":40000
},
"speaker":{
    "spk_power":2,
    "spk_resistance":4
},
"custom_panel_timing":{
    "panel_type":0,
    "panel_dclk":148500000,
    "panel_width":1920,
    "panel_height":1080,
    "panel_hfp":160,
    "panel_hbp":100,
    "panel_hsw":20,
    "panel_vfp":10,
    "panel_vbp":25,
    "panel_vsw":10
},
"custom_mipi_cmd":[
    "23", "00", "02", "27","AA",
    "23", "00", "02", "48","20",
    ...
    "05", "78","01","11",
    "05", "1E","01","29"
],
"display-port":{
    "vop0" =
}
}
```

### GMS（谷歌移动服务 google mobile service）

Google移动设备服务，包含Google服务框架，Play商店，Chrome浏览器等⼀系列应⽤。



### 移植谷歌服务

Android7.1:

RK-SDK自带，参考GMS文档打开配置选项即可

Android10：

已移植，参考rk3399-Android10相关提交

Android11：

已移植，参考rk3568-Android11相关提交

Android12：

**device/rockchip/common/device.mk**

\# GMS

$(call inherit-product, device/rockchip/common/modules/gms.mk)

**device/rockchip/common/modules/gms.mk**

 PRODUCT_PACKAGE_OVERLAYS += vendor/rockchip/common/gms/gms_overlay

 $(call inherit-product, vendor/partner_gms/products/$(TMP_GMS_VAR).mk)

 $(call inherit-product, vendor/partner_modules/build/$(TMP_MAINLINE_VAR).mk)





参考文档：Rockchip_User_Guide_Android_GMS_Configuration_CN.pdf

#### 其他：

RK有标准文档，所以先不谈标准的。

探讨下如何从别人已有gms的固件包中，把它的扒下来。

将众云世纪的整包通过瑞芯微下载工具2.84解包，得到img列表如下

![image-20220809151336861](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20220809151336861.png)

因为kernel层得适配我们自己的板子，于是在SDK的打包目录下，除了boot.img，uboot.img。  其他都替换掉。打包生成update.img，烧录到机器里，谷歌服务OK，但是上层都是众云的，缺少自己的apk和api支持，这种方法有一定的弊端。

将他们镜像中/vendor/分区下的apk拷贝到无谷歌服务的机器，发现可行。

## 问题记录

### USB摄像头无法录音

客户有一个USB摄像头，录像和录音时都没有声音。不过用其他USB摄像头是OK

直接可以tinycap录音，排查是否是kernel原因 

alsa可以放在最后进行排除 frameworks/av/services/audiopolicy/enginedefault/Engine.cpp里打点日志，排查下device类型。以及APP调用从logcat里看到的类型是否匹配。

AUDIO_DEVICE_IN_USB = 0x8001000

最终补丁，原因是device的在linux的auto suspend功能兼容不了。如果下面修改有问题可能uvc的quirks也要相应增加，你参考uvc的其它设备增加就好。或者在kernel/driver/core/usb/usb.c

```
diff --git a/sound/usb/quirks.c b/sound/usb/quirks.c
index ab9f3da..162a783 100644
--- a/sound/usb/quirks.c
+++ b/sound/usb/quirks.c
@@ -1852,6 +1852,8 @@ static const struct usb_audio_quirk_flags_table quirk_flags_table[] = {
                   QUIRK_FLAG_DISABLE_AUTOSUSPEND),
        DEVICE_FLG(0x17aa, 0x104d, /* Lenovo ThinkStation P620 Internal Speaker + Front Headset */
                   QUIRK_FLAG_DISABLE_AUTOSUSPEND),
+       DEVICE_FLG(0x0c45, 0x636b, /* Sonix Technology Co., Ltd. SN001 */
+                  QUIRK_FLAG_DISABLE_AUTOSUSPEND),
        DEVICE_FLG(0x1852, 0x5065, /* Luxman DA-06 */
                   QUIRK_FLAG_ITF_USB_DSD_DAC | QUIRK_FLAG_CTL_MSG_DELAY),
        DEVICE_FLG(0x1901, 0x0191, /* GE B850V3 CP2114 audio interface */
```



还有一种情况：摄像头在系统相机里没法录音，其他APP是好的，检查此处：

```
+++ b/services/audiopolicy/enginedefault/src/Engine.cpp
@@ -689,7 +694,9 @@ audio_devices_t Engine::getDeviceForInputSource(audio_source_t inputSource) cons
         }
         break;
     case AUDIO_SOURCE_CAMCORDER:
-        if (availableDeviceTypes & AUDIO_DEVICE_IN_HDMI) {
+        if (availableDeviceTypes & AUDIO_DEVICE_IN_USB_DEVICE) {
+            device = AUDIO_DEVICE_IN_USB_DEVICE;
+        } else if (availableDeviceTypes & AUDIO_DEVICE_IN_HDMI) {
```



### DP没有声音

https://max.book118.com/html/2020/0606/7061155042002140.shtm

HAL层探测到了系统具有哪些音频属性，以及默认声卡的加载。android上层再更根据不同事件选择不同通道的声音输出

插上DP显示器，系统播放没有声音

tinyplay /sdcard/SoundTest.wav -D 2 -d 0 

一开始是没有dp声卡，需要打开kernel里的宏定义.

然后是dp没有插拔检测。首先要再dp驱动里，加上extcon这个节点，提供给上层dp的插拔状态。

可以先把其他声卡屏蔽掉，但看下dp声卡有没有问题。

tinyplay 44100.wav

出现提示音没声音，其他音源可以



### HDMI没有声音

cat /sys/kernel/debug/asoc/dais 可以查看初始化了哪些DAI

DAI指的就是数字音频接口，比如I2S TDM等

simple audio card是meachine driver 负责将codec与 platform driver结合起来，最终形成一个声卡

 发现I2S2被人关掉了，导致声卡没出来



### 主副屏设置问题

如果出现分辨率，显示等问题，可以先检查一下主副屏设置

[vendor.hwc.device.aux]: DP
[vendor.hwc.device.extend]: DP
[vendor.hwc.device.main]: HDMI-A-1
[vendor.hwc.device.primary]: HDMI-A



```C
其中这两条是设置主副屏的：
vendor.hwc.device.primary=HDMI-A
vendor.hwc.device.extend=DP
```

```C
其中这两条是显示当前实际主副屏：
vendor.hwc.device.aux=DP 
vendor.hwc.device.main=HDMI-A-1 
```

### DP显示问题

#### 闪动

报错：

[  825.154969] [drm:vop_plane_atomic_update] *ERROR* crtc-0 win2 dest->x1[0] + dsp_w[1920] exceed mode hdisplay[1280]
[  825.155002] [drm:vop_plane_atomic_update] *ERROR* crtc-0 win2 dest->y1[0] + dsp_h[1080] exceed mode vdisplay[800]

代码跟踪：

dsp_w = drm_rect_width(dest);

struct drm_rect *dest = &vop_plane_state->dest;

struct vop_plane_state *vop_plane_state = to_vop_plane_state(state);

struct drm_plane_state *state = plane->state;

vop_plane_atomic_update(struct drm_plane *plane,  struct drm_plane_state *old_state)

```c
struct drm_plane {
	struct drm_device *dev;
 
    //挂载到&drm_mode_config.plane_list
	struct list_head head;
 
	char *name;
	struct drm_modeset_lock mutex;
 
	//表示plane的mode对象， 其包含了plane的各种属性
	struct drm_mode_object base;
 
	/**
	 * @possible_crtcs: pipes this plane can be bound to constructed from
	 * drm_crtc_mask()
	 */
    //绑定的crtc
	uint32_t possible_crtcs;
	//plane支持的fb像素format类型数组， format类型如DRM_FORMAT_ARGB8888
	uint32_t *format_types;
    //plane支持的fb像素format类型数组大小
	unsigned int format_count;
	bool format_default;
 
	//modifier数组，其存放的值如DRM_FORMAT_MOD_LINEAR/DRM_FORMAT_MOD_X_TILED等
	uint64_t *modifiers;
	unsigned int modifier_count;
 
	/**
	 * @crtc:
	 *
	 * Currently bound CRTC, only meaningful for non-atomic drivers. For
	 * atomic drivers this is forced to be NULL, atomic drivers should
	 * instead check &drm_plane_state.crtc.
	 */
    /*no-atomic drivers用来标识当前绑定的crtc。 对于atomic driver，该值应该为null
     *并使用 &drm_plane_state.crtc替代
     */
	struct drm_crtc *crtc;
 
    /*no-atomic drivers用来标识当前绑定的fb。 对于atomic driver，该值应该为null
     *并使用 &drm_plane_state.fb替代
     */
	struct drm_framebuffer *fb;
 
    /*对于non-atomic drivers， old_fb用于在modeset操作时跟踪老的fb
     * atomic drivers下，该值为Null
    */
	struct drm_framebuffer *old_fb;
 
    //plane funcs
	const struct drm_plane_funcs *funcs;
 
	/** @properties: property tracking for this plane */
    //plane的属性
	struct drm_object_properties properties;
 
    //如DRM_PLANE_TYPE_OVERLAY/DRM_PLANE_TYPE_PRIMARY/DRM_PLANE_TYPE_CURSOR
	enum drm_plane_type type;
 
    //mode_config.list中的序号
	unsigned index;
 
	const struct drm_plane_helper_funcs *helper_private;
 
    //表示plane的各种状态，如其绑定的crtc/fb等，用于atomic操作
	struct drm_plane_state *state;
 
    //这些属性待研究
	struct drm_property *alpha_property;
	struct drm_property *zpos_property;
	struct drm_property *rotation_property;
	struct drm_property *blend_mode_property;
	struct drm_property *color_encoding_property;
	struct drm_property *color_range_property;
	struct drm_property *scaling_filter_property;
};
```

跟分辨率有关系的，看下主副屏的配置

vendor.hwc.device.primary=HDMI-A,eDP,DSI,LVDS 

vendor.hwc.device.extend=DP

#### 黑屏

HDMI跟DP同显有问题，两者黑屏。最终确定跟主副屏配置有关系

#### 花屏

需要关闭HWC policy 不然拖动会有重影。最终确定跟主副屏配置有关系

### USB存储相关问题

#### 不识别

Type C接口插入U盘，部分型号的U盘无法再文件浏览器中显示，具体型号规格见附件

部分型号的U盘插入USB端口后，在文件浏览器中无法显示出来，具体型号见附件图片

根据现象和实验总结

1.不识别1T NTFS文件格式的硬盘 （应该是2.0和3.0口都不识别，未复测没条件）

2.USB 3.0和2.0口不识别部分U盘 （FAT32格式）， 可复现3.0口不识别一个U盘，kernel没有日志，排查如下：

3.TypeC口不识别部分U盘，没条件未复测

##### 忆捷CU20 64G  3.0U盘

现象：插2.0口，会出现大概率不识别的情况，已复现

分析：不识别时，kernel无枚举日志。

##### 兰科芯16G 2.0U盘

插3.0口

##### 联想F309 3.0移动硬盘

现象：插扩展2.0口，无法识别，设备灯亮。插在3.0口正常

分析：设备有识别日志，无枚举信息。

结果：改完2.0A电流后，正常了

#### 识别慢

3.0 U盘插入2.0或3.0口,kernel枚举日志有1秒左右的延迟

能否优化

对于usb2.0口，D+拉高是设备的行为，D+拉高之后host才会枚举的，因此这个与设备有关。
对于usb3.0的设备，一般会尝试先进行usb3.0通信，usb3.0没有检测到再进行usb2.0的枚举，不过一般也不会有3-4s那么长，换不同的U盘测试看看吧

#### 速率慢

见Kernel->驱动开发->USB章节

参考文档：《Rockchip USB Performance Analysis Guide》

### Android卡logo

* hj5188 没接电磁会卡

### GMS Test

#### CtsMediaTestCases

##### DecodeEditEncodeTest

​	android.media.cts.DecodeEditEncodeTest#testVideoEditQCIF

* junit.framework.AssertionFailedError: Found 4 bad frames

更新: frameworks/base/media/java/android/media

##### StreamingMediaPlayerTest

android.media.cts.StreamingMediaPlayerTest#testHTTP_H263_AMR_Video1

android.media.cts.StreamingMediaPlayerTest#testHTTP_H263_AMR_Video2

android.media.cts.StreamingMediaPlayerTest#testHTTP_H264Base_AAC_Video1

android.media.cts.StreamingMediaPlayerTest#testHTTP_H264Base_AAC_Video2

android.media.cts.StreamingMediaPlayerTest#testHTTP_MPEG4SP_AAC_Video1

android.media.cts.StreamingMediaPlayerTest#testHTTP_MPEG4SP_AAC_Video2

都是 java.lang.AssertionError: Remote connection to DynamicConfigService required for this test，推测和网络VPN设置有关系

### RK3588 HDMI点1366x768花屏

换成1360x768

```
diff --git a/drivers/gpu/drm/bridge/synopsys/dw-hdmi-qp.c b/drivers/gpu/drm/bridge/synopsys/dw-hdmi-qp.c
index 3c7e73cd061e..eb41b54f9708 100755
--- a/drivers/gpu/drm/bridge/synopsys/dw-hdmi-qp.c
+++ b/drivers/gpu/drm/bridge/synopsys/dw-hdmi-qp.c
@@ -134,6 +134,10 @@ static const struct drm_display_mode dw_hdmi_default_modes[] = {
                   4104, 4400, 0, 2160, 2168, 2178, 2250, 0,
                   DRM_MODE_FLAG_PHSYNC | DRM_MODE_FLAG_PVSYNC),
          .picture_aspect_ratio = HDMI_PICTURE_ASPECT_16_9, },
+       /* 1366x768@60Hz 16:9 */
+       { DRM_MODE("1366x768", DRM_MODE_TYPE_DRIVER, 74250, 1360, 1466,
+                  1516, 1566, 0, 768, 778, 784, 790, 0,
+                  DRM_MODE_FLAG_PHSYNC | DRM_MODE_FLAG_PVSYNC) },

@@ -2088,8 +2093,9 @@ static int dw_hdmi_connector_get_modes(struct drm_connector *connector)
                                hdmi->plat_data->right->sink_has_audio = true;
                        }
                }
-               if(strcmp("fdea0000.hdmi",dev_name(hdmi->dev)) == 0) {
+               if(strcmp("fdea0000.hdmi",dev_name(hdmi->dev)) == 0) {  //fdea0000.hdmi hdmi1  fde80000.hdm0
                        // Select resolution:lvds=0(1920x1080@60), vbyone=1(3840x2160@60)
+                       printk("%s : >>>>> HDMI1 use force resolution\n",__func__);
```



# U-boot

## 基础知识

## 调试修改

### HWID的适配与应用（Hardware ID）

#### 通过HWID来旋转屏幕方向

思路：

* uboot读取HWID signway_hwid
* 设置uboot 环境变量 setenv hardware_id#
* 在cmdline里传入androidboot.hwid，会自己生成ro.boot.hwid
* 在aosp/system/core/init 中 读取ro.boot.hwid
* 判断屏幕类型，修改对应旋转角度
* 注意property_set失效的问题，系统读取build.prop速度慢load_persist_props。通过删除build.prop所有的rotation相关字段解决。

### DTS fixup

先找到uboot中已经找到dtb之后的程序位置，uboot框架中为厂家自己的fdt_support.c



### 移植CJSON

不同产品参数需要灵活配置，json格式的配置文件正好可以满足这个需求。

uboot原生不支持Cjson调用，于是尝试移植Cjson进uboot

移植后总结：

1.一共需要两个文件，一个.c,一个.h。C文件可以放在/uboot/lib/下面，h文件可以放在include/linux/下

2.uboot中有些库函数不支持，需要自己实现或者修改头文件引用，包括 pow，floor函数。

3.源代码有些格式问题，会引发编译器报错，需要调整，一般是if后面的执行语句，没有换行。

# 格式模板

## 解决分析问题

 [RK3399] [Linux4.19]

------

问题描述:

分析过程:

解决方法：

------

参考资料:

## 学习记录

正文内容

------

参考资料:

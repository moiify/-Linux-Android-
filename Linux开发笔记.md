## 源码环境

### 3399

### 3568

### 3588

## 调试记录

### OpenHarmony鸿蒙



### MIPI摄像头

设备：RK3399   kernel4.4

#### 数据链路：

MIPI-RAW-Sensor -> MIPI 接口->ISP

Sensor输出数据流给ISP HW，ISP HW再输出经过一系列图像处理
算法后的图像。RkAiq不断从ISP HW获取统计数据，并经过3A等算法生成新的参数反馈给各硬件模块。
Tuning tool可在线实时调试参数，调试好后可保存生成新的iq参数文件。

#### 名词解释

ISP，Image Signal Processing，用以接收并处理图像。本文中既指硬件本身，也泛指ISP驱动

CIF，指RK芯片中的VIP模块，用以接收Sensor数据并保存到Memory中，仅转存数据，无ISP功能

#### 软件架构

ISP20 软件框图如图1-3所示。主要分成以下三层：
1. kernel layer。该层包含所有Camera系统的硬件驱动，主要有ISP驱动、sensor驱动、vcm驱动、
flashlight驱动、IrCutter驱动等等。驱动都基于V4L2及Media框架实现。
2. framework layer。该层为RkAiq lib的集成层，Rkaiq lib有两种集成方式：
IspServer 方式
该方式Rkaiq lib跑在 IspServer独立进程，客户端通过dbus与之通信。此外，该方式可为v4l-ctl等
现有第三方应用，在不修改源码的情况下，提供具有ISP调试效果的图像。
直接集成方式
RkAiq lib可直接集成进应用。
3. user layer。用户应用层。

media-ctl -d /dev/media0 --set-v4l2 '"m00_f_gc5035 7-0037":0[fmt:SBGGR10_1X10/1920x1080]'

#### 用gstreamer抓图以及拍照录像

export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/lib/gstreamer-1.0

export XDG_RUNTIME_DIR=/tmp/.xdg

##### 预览

gst-launch-1.0 v4l2src device=/dev/video0 io-mode=4 ! videoconvert ! video/x-raw,format=NV12,width=2592,height=1944 ! rkximagesink

##### 拍照

gst-launch-1.0 -v v4l2src device=/dev/video0 num-buffers=10 ! \
video/x-raw,format=NV12,width=1920,height=1080 ! mppjpegenc ! \
multifilesink location=/home/linaro/test%05d.jpg

gst-launch-1.0 v4l2src device=/dev/video0 num-buffers=100 ! \
video/x-raw,format=NV12,width=1920,height=1088,framerate=30/1 ! \
videoconvert ! mpph264enc ! h264parse ! mp4mux ! \
filesink location=/home/linaro/Videos/h264.mp4



### WIFI热点

工具： hostapd+dnsmasq+iptable

#### 准备工作:

* cat /sys/class/rkwifi/chip //获取WIFI型号为AP6210
* killall dnsmasq
* killall hostapd
* killall udhcpc



* ifconfig wlan1 down
*  rm -rf /userdata/bin/wlan1

* iw dev wlan1 del
* ifconfig wlan0 up
* update-alternatives --set iptables /usr/sbin/iptables-legacy
* iw phy0 interface add wlan1 type managed  //AP模组创建一个wlan1

#### 开始创建

* 修改hostapd.conf文件  /userdata/bin/hostapd.conf

  ```
  interface=wlan1
  ctrl_interface=/var/run/hostapd
  driver=nl80211
  ssid=EloLinux
  channel=6
  hw_mode=g
  ieee80211n=1
  ignore_broadcast_ssid=0
  auth_algs=1
  wpa=3
  wpa_passphrase=123456789
  wpa_key_mgmt=WPA-PSK
  wpa_pairwise=TKIP
  rsn_pairwise=CCMP
  ```

* ifconfig wlan1 up

* ifconfig wlan1 192.168.88.1 netmask 255.255.255.0

* 修改dnsmasq.conf文件 /userdata/bin/dnsmasq.conf

  ```
  user=root
  listen-address=192.168.88.1
  dhcp-range=192.168.88.50,192.168.88.150
  server=/google/8.8.8.8
  ```

* getpid dnsmasq

* kill *pid*

* sudo systemctl stop systemd-resolved

* dnsmasq -C  /etc/dnsmasq.conf --interface=wlan1

* hostapd /etc/hostapd/hostapd.conf &

```
sudo journalctl -fu NetworkManager
```



```
nmcli connection add type ethernet ifname eth0 ipv4.method shared con-name local
```

nmcli报iptable相关错误，可能和内核相关



### NVR功能



### RK628

Rockchip_DRM_RK628_Porting_Guide_CN.pdf --讲相关的驱动文件名和dts怎么配的

Rockchip_MCU_RK628D_Porting_Guide_CN.pdf --讲不用RK的主控，用MCU来驱动RK628的程序该怎么写，此应用场景暂未涉及，可忽略

Rockchip_RK628D_Application_Notes_CN.pdf-- 介绍628的接口，性能参数，应用场景，主要是HDMI-LVDS  Dual LVDS: 1080P 60Hz
LVDS: 720P 60Hz

Rockchip_RK628D_For_All_Porting_Guide_CN.pdf --将原理和配置，调试方法，为主要参考文档

#### 芯片简介

RK628 分为 Display 通路和 HDMI IN 通路，Display 通路的驱动于drivers/misc/rk628/下，
HDMI IN 通路的驱动于drivers/media/i2c/rk628/下

### WIFI/BT模组适配

#### AIC8800（AIOT3568/AIOT3588）

接口：sdio（wifi）+uart(蓝牙)

只要打入kernel驱动，然后再固件宏定义中，将蓝牙wifi固件放到对应目录下，开机加载ko驱动即可。

##### 流程

1.按照原理图，配置好引脚和总线驱动
2.将kernel配置中ko的方式变为内置（build in）的方式，因为debian框架不支持wifi兼容
3.检查wifi 32.768k时钟来源，aiot3588为pt7c4337输出，需要在RTC驱动中，将对应的时钟注册上去（重要）
4.打开AIC8800相关的内核配置，使得驱动编译的时候，能够正确编译



1.WIFI驱动有了 (ko文件)
kernel/drivers/net/wireless/rockchip_wlan/aic/

2.wifi/bt的固件
external/aic
文件在资料包aic8800_porting_package\SDIO\driver_fw\fw 

3.buildroot和debian 关于rkwifibt之间的关系

debian/binary/ 下面是有 external下的fw固件的，这个固件从哪来？ binary这个目录是如何生成的？
buildroot 下面没有包含fw固件，只有config选上了的才有

#------------------rkwifibt------------
echo -e "\033[36m Install rkwifibt.................... \033[0m"
\${APT_INSTALL} /packages/rkwifibt/*.deb
ln -s /system/etc/firmware /vendor/etc/
这个deb包可以参考文档<9.4 相关文件自动安装集成说明>中deb包制作

4.内核采用build-in方式或者动态加载ko方式的差别：
build-in方式扩展性较差，但是不需要修改上层，代码改动量基本为0
动态加载ko，需要了解ko加载原理，参考<9.4 相关文件自动安装集成说明>以及<3.4 开机自动加载Wi-Fi驱动KO的规则>,可以了解到
需要自己编写或者修改现有的脚本，并且让此脚本开机运行，关键是insmod相关的ko

5.关键步骤梳理：
①对照原理图，配置好相关dts
②编译出aic8800_bsp.ko以及aic8800_fdrv.ko
③将aic_8800压缩包内/fw/目录下的所有文件，拷贝到/vendor/lib/firmware/下
④手动insmod 两个ko文件，测试是 ifconfig -a否能能显示wlan 0
⑤若成功，则想办法集成到系统中( ko可采用build-in加载，固件拷贝参考《9.4 相关文件自动安装集成说明》)
⑥修改两个脚本，一个为了加载ko，一个为了拷贝文件
overlay/etc/init.d/rkwifibt.sh:15:sudo insmod /system/lib/modules/aic8800_bsp.ko aic8800_fdrv.ko

mk-rootfs-bullseye.sh
#for WIFI AIC8800
sudo cp overlay-signway/wifi/*.ko $TARGET_ROOTFS_DIR/system/lib/modules/
sudo cp overlay-signway/wifi/firmware/* $TARGET_ROOTFS_DIR/system/etc/firmware/

6.尝试build-in失败
经过研究，aic8800的驱动，没法直接在kernel启动时直接加载，RK给的build-in宏只是针对Realtek和正基模组的驱动的，如果通过修改makefile方式，
强行将aic8800驱动进行编译成=y的形式，会出现函数重复定义的报错，较多，可解决但是比较麻烦没有必要。

##### 问题记录：

###### ubuntu版本wifi能打开，但是扫描不到热点

一开始insmod异常，重新编译后，解决。

用debian正常的固件，替换ubuntu的rootfs，现象一样

正常的日志和insmod打印：

```
sudo insmod /system/lib/modules/aic8800_bsp.ko
[   13.594083] aicbsp_init
[   13.594109] RELEASE_DATE:2022_0622_1326
[   13.594705] aicbsp: aicbsp_set_subsys, subsys: AIC_WIFI, state to: 1
[   13.594718] aicbsp: aicbsp_set_subsys, power state change to 1 dure to AIC_WIFI
[   13.594723] aicbsp: aicbsp_platform_power_on
[   13.614083] aicbsp: aicbsp_sdio_probe:1
[   13.614181] aicbsp: aicbsp_sdio_probe:2
[   13.614185] aicbsp: aicbsp_sdio_probe after replace:1
[   13.614291] aicbsp: Set SDIO Clock 50 MHz
[   13.618500] aicbsp: aicbsp_driver_fw_init, chip rev: 7
[   13.621666] rwnx_load_firmware :firmware path = /vendor/etc/firmware/fw_adid_u03.bin
[   13.632374] file md5:cf3ee68167beda73aaa5cb7a17169b4d
[   13.632921] rwnx_load_firmware :firmware path = /vendor/etc/firmware/fw_patch_u03.bin
[   13.636306] file md5:c8c0bee8c31d166e72d11d8af0b7136f
[   13.649962] rwnx_load_firmware :firmware path = /vendor/etc/firmware/fw_patch_table_u03.bin
[   13.651404] file md5:32c60ebd1d220da2a36a07efe7e36a47
```

ubuntu异常insmod：

```
root@ubuntu:~# sudo insmod /system/lib/modules/aic8800_bsp.ko
[  177.758033] aicbsp: sdio_err:<aicwf_sdio_bustx_thread,1052>: sdio bustx thread stop
[  177.758185] aicbsp: sdio_err:<aicwf_sdio_busrx_thread,1071>: sdio busrx thread stop

root@ubuntu:~# sudo insmod /system/lib/modules/aic8800_fdrv.ko
[  210.538530] cmd timed-out
[  210.538684] tkn[0]  flags:0012  result: -4  cmd: 123-MM_SET_STACK_START_REQ   - reqcfm( 124-MM_SET_STACK_START_CFM)
[  210.538758] cmd queue crashed
[  210.539093] cmd queue crashed
[  210.539154] cmd queue crashed
[  210.539214] cmd queue crashed
[  210.539273] cmd queue crashed
[  210.539345] cmd queue crashed
[  210.539402] cmd queue crashed
[  210.539530] cmd queue crashed
[  210.540121] cmd queue crashed
[  210.540947] ieee80211 phy0:
[  210.540947] *******************************************************
[  210.540947] ** CAUTION: USING PERMISSIVE CUSTOM REGULATORY RULES **
[  210.540947] *******************************************************
[  210.541018] cmd queue crashed
[  210.576856] cmd queue crashed
[  210.576904] cmd queue crashed
[  210.576921] cmd queue crashed
[  210.576928] add_if: Status Error(255)
[  210.587348] cmd queue crashed
[  210.587374] cmd queue crashed
[  210.587383] cmd queue crashed

```

最后发现是wifi模组的fw路径没有匹配, debian 里做了映射

```
server01@server01-H610M-H-DDR4:~/work/1.rk3588_linux/RK3588_LINUX_SDK_RELEASE/debian/rootfs_mount/vendor/etc$ ls -al
lrwxrwxrwx 1 server01 root   20 11月 30 17:42 firmware -> /system/etc/firmware
```



### 适配4G模组

#### 域格CLM920-TD5m 

使用ppp拨号上网，按照手册上面配，注意要打开下面三个config。Gobinet和QML拨号不推荐。

```
CONFIG_PPP=y
CONFIG_PPP_ASYNC=y
CONFIG_PPP_SYNC_TTY=y
```

3568 - debian 

1.拨号脚本，添加到/etc/init.d/rockchip.sh里

2.sudo chroot  rootfs 然后安装ipppd

#### 移远EC20

3588 - ubuntu

使用QML拨号上网，按照手册上面配。最后要打开

```
#Add for 4G moudle EC20
CONFIG_USB_SERIAL_WWAN=y
CONFIG_USB_NET_RNDIS_WLAN=y
CONFIG_USB_WDM=y
CONFIG_USB_USBNET=y
CONFIG_USB_NET_QMI_WWAN=y
CONFIG_LTE=y
CONFIG_LTE_RM310=y
```

把移远提供的QMI拨号可执行文件，放到rootfs某一位置，在开机脚本里运行。注意要加20s左右延迟，等待底层usb识别。

#### 域格CL920-JC5

##### 3588 - ubuntu

使用RNDIS拨号，自动上网，比较方便，直接打开kernel中RNDIS支持的配置选项就可以。

参考文档《》



### 移植Ubuntu

#### 修改firefly固件

首先对应kernel版本，将ubuntu固件下载到服务器，通过mount命令，将镜像加载到文件夹。

##### 3588：

```
server01@server01-H610M-H-DDR4:~/work/1.rk3588_linux/RK3588_LINUX_SDK_RELEASE/debian/rootfs_mount$ sudo find -name "*firefly*"
[sudo] password for server01:
./var/lib/dpkg/info/firefly-fan.md5sums
./var/lib/dpkg/info/firefly-fan.list
./var/lib/dpkg/info/firefly-fan.prerm
./var/lib/dpkg/info/firefly-rk3588npu-driver.list
./var/lib/dpkg/info/firefly-rk3588npu-driver.prerm
./var/lib/dpkg/info/fireflydev.md5sums
./var/lib/dpkg/info/firefly-fan.postinst
./var/lib/dpkg/info/firefly-rk3588npu-driver.md5sums
./var/lib/dpkg/info/fireflydev.list
./var/lib/dpkg/info/firefly-rk3588npu-driver.postinst
./etc/apt/preferences.d/99-firefly-repo
./etc/systemd/system/local-fs.target.wants/firefly-fan.service
./usr/bin/firefly-fan-init
./usr/bin/firefly_fan_control
./usr/lib/systemd/system/firefly-fan.service
./usr/share/backgrounds/firefly-ubuntu-logo.png
./usr/share/doc/fireflydev
./home/firefly
```

壁纸：usr/share/backgrounds/firefly-ubuntu-logo.png

MOTD参考：https://blog.csdn.net/Cappuccino_jay/article/details/124826223

剩余问题：

1.开机默认需要登录 密码为firefly  

文件名：/etc/shadow   

ubuntu对应密码口令：

$6$YJDfTcFGlEXZDZ2C$Qlm/qBsL6h/AyWFGd3kZqxjvOhV3UWnyyhmQdpVf8HGM7I4GC/odkkjcI2DH/9xGVGqWqcbYZFrHTuAaIa5efi.

2.进入系统后串口打印firefly相关信息

3.把主机名也设置成ubuntu，不然sudo有问题，可能跟hosts相关，先不管

其他参考https://blog.csdn.net/john1337/article/details/116304824进行修改

##### 3568:

firefly验证：用纯净版的systemd去替换/usr/lib/systemd/下的同名文件，如果尝试将systemd通过objdump出来，再去修改30秒为00秒，会有一瞬间的logo显示，不可行。

壁纸：usr/share/lunbuntu/wallpaper/firefly-ubuntu-logo.png

自动登录：https://blog.csdn.net/flfihpv259/article/details/54863035

换源后，apt update 失败：https://blog.csdn.net/m0_62062850/article/details/125494765

### 调整用户分区

emmc颗粒是32G的，用户反馈root分区只有12个G，修改rockdev/parameter.txt

把0x01800000@0x00098000(rootfs)调整到0x03600000@0x00098000(rootfs)，调整到27G，重新打包。

最后用命令查看：

```
root@linaro-alip:~# df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/root        27G  2.5G   23G  10% /
devtmpfs        1.9G  8.0K  1.9G   1% /dev
tmpfs           1.9G     0  1.9G   0% /dev/shm
tmpfs           1.9G   17M  1.8G   1% /run
tmpfs           5.0M  4.0K  5.0M   1% /run/lock
tmpfs           1.9G     0  1.9G   0% /sys/fs/cgroup
tmpfs           372M  8.0K  372M   1% /run/user/1000
tmpfs           372M     0  372M   0% /run/user/0
/dev/mmcblk0p8  1.8G  5.5M  1.8G   1% /media/linaro/15f794ba-08e0-450b-ba49-2dfd1f636501
/dev/mmcblk0p6  126M   13M  107M  11% /media/linaro/727c6359-386f-4d50-9f7b-10a43a2b00e5

root@linaro-alip:~# fdisk -l
Disk /dev/ram0: 4 MiB, 4194304 bytes, 8192 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes


Disk /dev/mmcblk0: 29.1 GiB, 31268536320 bytes, 61071360 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 8A670000-0000-4A6C-8000-039700007699

Device            Start      End  Sectors  Size Type
/dev/mmcblk0p1    16384    24575     8192    4M unknown
/dev/mmcblk0p2    24576    32767     8192    4M unknown
/dev/mmcblk0p3    32768   163839   131072   64M unknown
/dev/mmcblk0p4   163840   294911   131072   64M unknown
/dev/mmcblk0p5   294912   360447    65536   32M unknown
/dev/mmcblk0p6   360448   622591   262144  128M unknown
/dev/mmcblk0p7   622592 57245695 56623104   27G unknown
/dev/mmcblk0p8 57245696 61071295  3825600  1.8G unknown

```

### Docker

#### docker支持

用参考文档/check-config.sh跑一下

```
info: reading kernel config from /proc/config.gz ...

Generally Necessary:
- cgroup hierarchy: properly mounted [/sys/fs/cgroup]
- CONFIG_NAMESPACES: enabled
- CONFIG_NET_NS: enabled
- CONFIG_PID_NS: enabled
- CONFIG_IPC_NS: enabled
- CONFIG_UTS_NS: enabled
- CONFIG_CGROUPS: enabled
- CONFIG_CGROUP_CPUACCT: enabled
- CONFIG_CGROUP_DEVICE: enabled
- CONFIG_CGROUP_FREEZER: enabled
- CONFIG_CGROUP_SCHED: enabled
- CONFIG_CPUSETS: enabled
- CONFIG_MEMCG: missing
- CONFIG_KEYS: enabled
- CONFIG_VETH: missing
- CONFIG_BRIDGE: missing
- CONFIG_BRIDGE_NETFILTER: missing
- CONFIG_IP_NF_FILTER: missing
- CONFIG_IP_NF_TARGET_MASQUERADE: missing
- CONFIG_NETFILTER_XT_MATCH_ADDRTYPE: missing
- CONFIG_NETFILTER_XT_MATCH_CONNTRACK: missing
- CONFIG_NETFILTER_XT_MATCH_IPVS: missing
- CONFIG_NETFILTER_XT_MARK: missing
- CONFIG_IP_NF_NAT: missing
- CONFIG_NF_NAT: missing
- CONFIG_POSIX_MQUEUE: missing
- CONFIG_NF_NAT_IPV4: missing
- CONFIG_NF_NAT_NEEDED: missing
- CONFIG_CGROUP_BPF: missing

Optional Features:
- CONFIG_USER_NS: enabled
- CONFIG_SECCOMP: enabled
- CONFIG_SECCOMP_FILTER: enabled
- CONFIG_CGROUP_PIDS: missing
- CONFIG_MEMCG_SWAP: missing
- CONFIG_MEMCG_SWAP_ENABLED: missing
- CONFIG_IOSCHED_CFQ: enabled
- CONFIG_CFQ_GROUP_IOSCHED: missing
- CONFIG_BLK_CGROUP: missing
- CONFIG_BLK_DEV_THROTTLING: missing
- CONFIG_CGROUP_PERF: missing
- CONFIG_CGROUP_HUGETLB: missing
- CONFIG_NET_CLS_CGROUP: missing
- CONFIG_CGROUP_NET_PRIO: missing
- CONFIG_CFS_BANDWIDTH: enabled
- CONFIG_FAIR_GROUP_SCHED: enabled
- CONFIG_RT_GROUP_SCHED: missing
- CONFIG_IP_NF_TARGET_REDIRECT: missing
- CONFIG_IP_VS: missing
- CONFIG_IP_VS_NFCT: missing
- CONFIG_IP_VS_PROTO_TCP: missing
- CONFIG_IP_VS_PROTO_UDP: missing
- CONFIG_IP_VS_RR: missing
- CONFIG_SECURITY_SELINUX: missing
- CONFIG_SECURITY_APPARMOR: missing
- CONFIG_EXT4_FS: enabled
- CONFIG_EXT4_FS_POSIX_ACL: enabled
- CONFIG_EXT4_FS_SECURITY: enabled
- Network Drivers:
  - "overlay":
    - CONFIG_VXLAN: missing
    - CONFIG_BRIDGE_VLAN_FILTERING: missing
      Optional (for encrypted networks):
      - CONFIG_CRYPTO: enabled
      - CONFIG_CRYPTO_AEAD: enabled
      - CONFIG_CRYPTO_GCM: enabled
      - CONFIG_CRYPTO_SEQIV: enabled
      - CONFIG_CRYPTO_GHASH: enabled
      - CONFIG_XFRM: enabled
      - CONFIG_XFRM_USER: enabled
      - CONFIG_XFRM_ALGO: enabled
      - CONFIG_INET_ESP: missing
      - CONFIG_INET_XFRM_MODE_TRANSPORT: missing
  - "ipvlan":
    - CONFIG_IPVLAN: missing
  - "macvlan":
    - CONFIG_MACVLAN: missing
    - CONFIG_DUMMY: missing
  - "ftp,tftp client in container":
    - CONFIG_NF_NAT_FTP: missing
    - CONFIG_NF_CONNTRACK_FTP: missing
    - CONFIG_NF_NAT_TFTP: missing
    - CONFIG_NF_CONNTRACK_TFTP: missing
- Storage Drivers:
  - "aufs":
    - CONFIG_AUFS_FS: missing
  - "btrfs":
    - CONFIG_BTRFS_FS: missing
    - CONFIG_BTRFS_FS_POSIX_ACL: missing
  - "devicemapper":
    - CONFIG_BLK_DEV_DM: missing
    - CONFIG_DM_THIN_PROVISIONING: missing
  - "overlay":
    - CONFIG_OVERLAY_FS: missing
  - "zfs":
    - /dev/zfs: missing
    - zfs command: missing
    - zpool command: missing

Limits:
- /proc/sys/kernel/keys/root_maxkeys: 1000000


```

把Generally Necessary中disabled的配置项在kernel/rockchip_linux_defconfig里打开即可

```

- CONFIG_MEMCG: missing
- CONFIG_VETH: missing
- CONFIG_BRIDGE: missing
- CONFIG_BRIDGE_NETFILTER: missing
- CONFIG_IP_NF_FILTER: missing
- CONFIG_IP_NF_TARGET_MASQUERADE: missing
- CONFIG_NETFILTER_XT_MATCH_ADDRTYPE: missing
- CONFIG_NETFILTER_XT_MATCH_CONNTRACK: missing
- CONFIG_NETFILTER_XT_MATCH_IPVS: missing
- CONFIG_NETFILTER_XT_MARK: missing
- CONFIG_IP_NF_NAT: missing
- CONFIG_NF_NAT: missing
- CONFIG_POSIX_MQUEUE: missing
- CONFIG_NF_NAT_IPV4: missing
- CONFIG_NF_NAT_NEEDED: missing
- CONFIG_CGROUP_BPF: missing
```

### QT

export QT_DEBUG_PLUGINS=1

### VUE

客户是在mac下开发的，用的electron

给过来一个源码，一个编译后的文件，运行后绿屏

首先检查板卡的node和vue版本

npm -v 8.19.3

node -v v16.19.1

electron@8.0.0

@vue/cli 5.0.8

然后跑npm run dev，有报错信息，基于报错信息分析。

第一个问题是electron模块乱码，直接删了重装，注意不能用cnpm，不好用，有依赖不一样或者直接找不到资源。

解决electron问题后，源码正常运行起来了，复现绿屏，从consle中发现是serialport这个库有问题，又是ELF问题又是版本号不匹配，删了重装，最后再重新编一下。

参考网址:

https://blog.csdn.net/weixin_36250061/article/details/103472978

https://stackoverflow.com/questions/46384591/node-was-compiled-against-a-different-node-js-version-using-node-module-versio

https://serialport.io/docs/guide-platform-support

https://blog.csdn.net/weixin_47513022/article/details/123105523 配置环境

https://zhuanlan.zhihu.com/p/489453403配置vue环境



### 常用Linux工具/命令

#### mount

tar 

ln

mv

dd

objdump

grep

sed

systemctl status

#### 串口调试

minicom


echo -e "AT+CSQ\r\n" > /dev/ttyUSB2

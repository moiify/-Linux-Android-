Android开发指南

## 文档更新说明
版本 | 日期 | 更新记录 | 作者 | 审核
:---:|:---:|:---|:---:|:---:
V1.0 | 2022-09-23 | 创建文档 | 徐剑 | 



## 1. android系统配置

### 1.1 锁屏、息屏

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



### 1.2 时间、时区、语言

​	1.2.1 时间  

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
​	1.2.2 时区  

```xml
frameworks/base/packages/SettingsProvider/res/values/defaults.xml
系统设置里面同步时区默认关闭，另自动同步时区，需要接4G模块才会生效，之前查看android同步时区流程，需要用到移动网络相关服务
<bool name="def_auto_time_zone">false</bool> // 自动时区关闭

android标准设置：
persist.sys.timezone=Asia/Shanghai

AIoT3568：
vendor.persist.sys.timezone=Asia/Shanghai
```
​	1.2.3 语言

```
ro.product.locale=zh-CN 中文
ro.product.locale=en-US 英文
```



### 1.3 亮度、DPI
​	1.3.1 DPI  

修改prop属性，公司标准1920x1080及以下的为160，4K输出时为280

```makefile
ro.sf.lcd_density=160
```



​	1.3.2 亮度  

​	1.3.2.1 gamma liner算法  
​		android高版本后，系统在设置亮度时增加了gamma liner算法，默认需要去掉该功能  
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



### 1.4 文件系统

  新开发的主板，文件系统全部换成ext4，android默认是f2fs，掉电会丢数据。修改方法参考RK文档里面对应的平台开发手册。

1.4.1 rk3568-11平台  
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



### 1.5 版本号、设备号

  1. 固件版本号  

用脚本编译时会自动生成版本号，对应的prop名称为ro.build.signway.board和ro.build.display.id。  

  RK3568-11的标准版是多个主板使用同一个固件，因此在Build.java和SignwayManagerService里面会根据ro.boot.hwid信息，重新名称版本号，替换版本号里面的主板名称字符串。  
具体查看文档：《固件兼容设计文档》  

  2. 设备号  
 根据ro.boot.hwid信息，显示主板名称  



### 1.6 系统方向

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



### 1.7 输入法

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



### 1.8 APN

apn代码里面的文件名称 apns-full-conf.xml

rk3568-11，rk3588-12

源码文件路径：vendor/rockchip/common/phone/etc/apns-full-conf.xml

adb调试路径：system/etc/apns-conf.xml



### 1.9 开机广播

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



### 1.10 浏览器FAQ

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



## 2. 功能列表

### 2.1 主板兼容同一个固件

#### 2.1.1 hwid适配

2.1.1.1 u-boot

rk3568-11

涉及signway_hwid.c，signway_hwid.h和resource_img.c

增加signway_hwid.c文件：arch/arm/mach-rockchip/signway/signway_hwid.c

```c
/*
 * (C) Copyright 2020-2025 Nanjing Signway Technology Co., Ltd
 *
 * SPDX-License-Identifier:	GPL-2.0+
 *
 * Author: Xiang.Ding <xiang.ding@signway.cn>
 */

//#define pr_fmt(fmt) "[signway-hwid]: " fmt

#include <common.h>
#include <errno.h>
#include <malloc.h>
#include <u-boot/crc.h>
#include <asm/arch/vendor.h>
#include <common.h>
#include <adc.h>

#include "signway_hwid.h"

/*
 * first field, product name
 * "01": AIoT3568       GPIO
 * "02": GV3568
 * 
 * second field, hardware version
 * "10" : V1.0          ADC
 * "21" : V2.1
 * 
 * three field, hardware bom id
 */
// first three auto read by hardware, after is used by software write, default '0'

typedef struct hwid_adc_info{
	uint32_t high_adc;
	uint32_t low_adc;
	uint8_t high_name;
	uint8_t low_name;
	char info_name[32];
}hwid_adc_info_t;


//AIOT3568-V2.1  		 11.30.11
//AIOT3568-V2.1-AD52068  11.30.12

hwid_adc_info_t product_info[] = {
	{1024, 1024, '1', '1', "AIoT3568"}, // 1.8V, 1.8V;   AIoT3568, "11"
	{1024,  922, '1', '2', "WHL3568"}, // 1.8V, 1.622V; WHL3568,  "12"
	{1024,  825, '1', '3', "AIoT3568-X"}, // 1.8V, 1.433V; AIoT3568-X,  "13"
	{1024,  704, '1', '4', "IVI3568"}, // 1.8V, 1.238V; IVI3568,  "13"
};

hwid_adc_info_t hardware_ver[] = {
	{1024, 0, '1', '0', "V1"}, // 1.8V, NC;   V1,X "10"
	{922,  0, '2', '0', "V2"}, // 1.622V, NC; V2, "20"
	{825,  0, '3', '0', "V2.1"}, // 1.622V, NC; V2.1, "20"
};

hwid_adc_info_t bom_id_info[] = {
	{1024, 1024, '1', '1'}, // 1.8V, 1.8V;   AIoT3568,AIoT3568-X "11"
	{1024,  922, '1', '2'}, // 1.8V, 1.622V; 
	{922,  1024, '2', '1'}, // 1.622V, 1.8V; 
};

uint8_t hwid_default[HWID_INFO_MAX] = "00.00.00.00.00.00.00.00";

static void insertion_sort(uint32_t arr[], int len){
	uint32_t i,j,temp;
	for (i=1;i<len;i++){
		temp = arr[i];
		for (j=i;j>0 && arr[j-1]>temp;j--)
			arr[j] = arr[j-1];
		arr[j] = temp;
	}
}

int get_adc_value(int adc_channel, uint32_t *adc_value)
{
	uint32_t adc_table[10] = {0};
	int i;

	for (i=0; i<10; i++) {
		if (adc_channel_single_shot("saradc", adc_channel, &adc_table[i])) {
			pr_err("---> %s read adc[%d] failed\n", __func__, adc_channel);
			adc_table[i] = 1024;
		}
		udelay(10);
	}
	insertion_sort(adc_table, 10);
	*adc_value = ((adc_table[3]+adc_table[4]+adc_table[5]+adc_table[6])/4);
	return 1;
}

int get_product_name(uint8_t *product_name, char *board_name)
{
	uint32_t high_adc;
	uint32_t low_adc;
	int i;

	if (!get_adc_value(PRODUCT_HIGH_ADC, &high_adc)) {
		pr_err("%s read high adc failed\n", __func__);
		return 0;
	}

	if (!get_adc_value(PRODUCT_LOW_ADC, &low_adc)) {
		pr_err("%s read low adc failed\n", __func__);
		return 0;
	}
	pr_err("%s read high adc = %d, low adc = %d\n", __func__, high_adc, low_adc);
	for (i=0; i<sizeof(product_info)/sizeof(hwid_adc_info_t); i++) {
		if (abs(high_adc - product_info[i].high_adc) > ADC_OFFSET)
			continue;
		if (abs(low_adc - product_info[i].low_adc) > ADC_OFFSET)
			continue;

		*(product_name + 0) = product_info[i].high_name;
		*(product_name + 1) = product_info[i].low_name;
		strcpy(board_name, product_info[i].info_name);
		return 1;
	}
	pr_err("%s: adc match failed!!!\n", __func__);
	return 0;
}

int get_hardware_version(uint8_t *hardware_version, char *board_version)
{
	uint32_t high_adc;
	int i;

	if (!get_adc_value(HARDWARE_HIGH_ADC, &high_adc)) {
		pr_err("%s read high adc failed\n", __func__);
		return 0;
	}
	pr_err("%s read high adc = %d, low adc = %d\n", __func__, high_adc, 0);
	for (i=0; i<sizeof(hardware_ver)/sizeof(hwid_adc_info_t); i++) {
		if (abs(high_adc - hardware_ver[i].high_adc) > ADC_OFFSET)
			continue;

		*(hardware_version + 0) = hardware_ver[i].high_name;
		*(hardware_version + 1) = hardware_ver[i].low_name;
		strcpy(board_version, hardware_ver[i].info_name);
		return 1;
	}
	pr_err("%s: adc match failed!!!\n", __func__);
	return 0;
}

int get_hardware_bom_id(uint8_t *bom_id)
{
	uint32_t high_adc;
	uint32_t low_adc;
	int i;

	if (!get_adc_value(BOM_ID_HIGH_ADC, &high_adc)) { //VIN3
		pr_err("%s read high adc failed\n", __func__);
		return 0;
	}

	if (!get_adc_value(BOM_ID_LOW_ADC, &low_adc)) { //VIN2
		pr_err("%s read low adc failed\n", __func__);
		return 0;
	}
	pr_err("%s read high adc = %d, low adc = %d\n", __func__, high_adc, low_adc);
	for (i=0; i<sizeof(bom_id_info)/sizeof(hwid_adc_info_t); i++) {
		if (abs(high_adc - bom_id_info[i].high_adc) > ADC_OFFSET)
			continue;
		if (abs(low_adc - bom_id_info[i].low_adc) > ADC_OFFSET)
			continue;

		*(bom_id + 0) = bom_id_info[i].high_name;
		*(bom_id + 1) = bom_id_info[i].low_name;
		return 1;
	}
	pr_err("%s: adc match failed!!!, %d, %d\n", __func__, high_adc, low_adc);
	return 0;
}

int get_hardware_id(uint8_t *hardware_id)
{
	int ret = 0;
	uint8_t product_name[2] = {0};
	uint8_t hardware_version[2] = {0};
	uint8_t hardware_bom_id[2] = {0};
	char board_name[32] = {0};
	char board_version[32] = {0};
	char buf[128] = {0};
	uint8_t hardware_info[16] = {'0'};
	int i;

	if (!hardware_id) {
		pr_err("--->hardware_id is NULL !!!, %d\n", ret);
		return 0;
	}

	ret = get_product_name(product_name, board_name);
	if (!ret) {
		pr_err("--->get_product_name failed !!!, %d\n", ret);
		return 0;
	}

	ret = get_hardware_version(hardware_version, board_version);
	if (!ret) {
		pr_err("--->get_hardware_version failed !!!, %d\n", ret);
		return 0;
	}

	ret = get_hardware_bom_id(hardware_bom_id);
	if (!ret) {
		pr_err("--->get_hardware_bom_id failed !!!, %d\n", ret);
		return 0;
	}

	*(hardware_id + 0) = product_name[0];
	*(hardware_id + 1) = product_name[1];
	*(hardware_id + 2) = '.';
	*(hardware_id + 3) = hardware_version[0];
	*(hardware_id + 4) = hardware_version[1];
	*(hardware_id + 5) = '.';
	*(hardware_id + 6) = hardware_bom_id[0];
	*(hardware_id + 7) = hardware_bom_id[1];

	pr_err("--->%s\n", __func__);
	for (i=0; i<8; i++)
		hardware_info[i] = *(hardware_id + i);
	sprintf(buf, "%s_%s_%s",
		board_name,
		board_version,
		hardware_info);
	env_set("hwid_info#", buf);
	pr_err("--->%s\n", buf);

	return 1;
}

int read_hardware_hwid_info(hwid_t *hwid)
{
	int ret = 0;
	int i;
	uint8_t local_checksum = 0;
	uint8_t hardware_id[HWID_HARDWARE_LEN] = {0};

	ret = get_hardware_id(hardware_id);
	if (!ret) {
		pr_err("--->get hardware id failed !!!, %d\n", ret);
		return 0;
	}

	hwid->status = 1;
	hwid->version = HWID_VERSION_V1;
	hwid->hwid_len = HWID_INFO_MAX;

	hwid->hwid_info=(uint8_t *)malloc(hwid->hwid_len);
	if (!hwid->hwid_info) {
		pr_err("--->malloc hwid info failed !!!, %d\n", ret);
		return 0;
	}

	for (i=0; i<hwid->hwid_len; i++) {
		if (i < HWID_HARDWARE_LEN) {
			*(hwid->hwid_info + i) = hardware_id[i];
		} else {
			*(hwid->hwid_info + i) = hwid_default[i];
		}
	}

	local_checksum ^= hwid->status;
	local_checksum ^= hwid->version;
	local_checksum ^= hwid->hwid_len;
	for (i=0; i<hwid->hwid_len; i++) {
		local_checksum ^= *(hwid->hwid_info + i);
	}
	hwid->checksum = local_checksum;
	return 1;
}

int get_signway_hwid_info(char *hwid_name)
{
	int ret = 0;
	hwid_t hwid;
	int key_words_len = strlen(KEY_WORDS_SIGNWAY_HWID);

	if (!hwid_name) {
		pr_err("--->%s, hwid_name is null\n", __func__);
		return 0;
	}

	// read hardware adc & gpio
	ret = read_hardware_hwid_info(&hwid);
	if (!ret) {
		pr_err("--->hwid info from hardware read error\n");
		return 0;
	}
	memcpy(hwid_name, KEY_WORDS_SIGNWAY_HWID, key_words_len);
	memcpy(hwid_name+key_words_len, hwid.hwid_info, hwid.hwid_len);
	return 1;
}

```

根据hwid加载dtb

arch/arm/mach-rockchip/resource_img.c

```c
#ifdef CONFIG_SIGNWAY_HWID_DTB
extern int get_signway_hwid_info(char *hwid_name);
static struct resource_file *signway_read_hwid_dtb(void)
{
        struct resource_file *file;
        struct list_head *node;
        char hwid_info[128] = {0};
        int ret = 0;

        pr_info("--->signway_read_hwid_dtb\n");
        ret = get_signway_hwid_info(hwid_info);
        if (!ret) {
                pr_info("---> get hwid error !!!\n");
                return NULL;
        }
        pr_err("---> hwid = %s\n", hwid_info);

        /* Find dtb file according to hardware id(GPIO/ADC) */
        list_for_each(node, &entrys_head) {
                file = list_entry(node, struct resource_file, link);
                pr_info("--->dtb file->name = %s\n", file->name);
                if (!strstr(file->name, ".dtb"))
                        continue;
                if (strstr(file->name, hwid_info)) {
                        return file;
                }
        }

        return NULL;
}
#endif

int rockchip_read_resource_dtb(void *fdt_addr, char **hash, int *hash_size)
{
        struct resource_file *file = NULL;
        int ret;

#ifdef CONFIG_ROCKCHIP_HWID_DTB
        file = resource_read_hwid_dtb();
#endif
#ifdef CONFIG_SIGNWAY_HWID_DTB
        file = signway_read_hwid_dtb();
#endif
        /* If no dtb matches hardware id(GPIO/ADC), use the default */
        if (!file)
                file = get_default_dtb();

        if (!file)
                return -ENODEV;

```

2.1.1.2 kernel

scripts/mkmultidtb.py

```python
DTBS['RK3568-AIoT3568'] = OrderedDict([('rk3568-AIoT3568-V1', '#hwid=11.10.11.00.00.00.00.00'),
                                    ('rk3568-AIoT3568-V2', '#hwid=11.20.12.00.00.00.00.00'),
                                    ('rk3568-AIoT3568-V21', '#hwid=11.30.11.00.00.00.00.00'),
                                    ('rk3568-AIoT3568-V21-AD52068', '#hwid=11.30.12.00.00.00.00.00'),
                                    ('rk3568-AIoT3568-V1-X', '#hwid=13.10.11.00.00.00.00.00'),
                                    ('rk3568-AIoT3568-V2-X', '#hwid=13.20.11.00.00.00.00.00')])

```

编译：

```shell
make ARCH=arm64 BOOT_IMG=../rockdev/Image-rk3568_r/boot.img rk3568-AIoT3568-V1.img -j8
make ARCH=arm64 BOOT_IMG=../rockdev/Image-rk3568_r/boot.img rk3568-AIoT3568-V2.img -j8
make ARCH=arm64 BOOT_IMG=../rockdev/Image-rk3568_r/boot.img rk3568-AIoT3568-V21.img -j8
make ARCH=arm64 BOOT_IMG=../rockdev/Image-rk3568_r/boot.img rk3568-AIoT3568-V21-AD52068.img -j8
make ARCH=arm64 BOOT_IMG=../rockdev/Image-rk3568_r/boot.img rk3568-AIoT3568-V1-X.img -j8
make ARCH=arm64 BOOT_IMG=../rockdev/Image-rk3568_r/boot.img rk3568-AIoT3568-V2-X.img -j8
./scripts/mkmultidtb.py RK3568-AIoT3568

```



#### 2.1.2 同一软件兼容不同主板

2.1.2.1 u-boot上报hwid信息

rk3568-11

 u-boot/arch/arm/mach-rockchip/signway/signway_hwid.c

```c
--- a/arch/arm/mach-rockchip/signway/signway_hwid.c
+++ b/arch/arm/mach-rockchip/signway/signway_hwid.c
@@ -36,6 +36,7 @@ typedef struct hwid_adc_info{
        uint32_t low_adc;
        uint8_t high_name;
        uint8_t low_name;
+       char info_name[32];
 }hwid_adc_info_t;


@@ -43,14 +44,14 @@ typedef struct hwid_adc_info{
 //AIOT3568-V2.1-AD52068  11.30.12

 hwid_adc_info_t product_info[] = {
-       {1024, 1024, '1', '1'}, // 1.8V, 1.8V;   AIoT3568, "11"
-       {1024,  922, '1', '2'}, // 1.8V, 1.622V; WHL3568,  "12"
+       {1024, 1024, '1', '1', "AIoT3568"}, // 1.8V, 1.8V;   AIoT3568, "11"
+       {1024,  922, '1', '2', "WHL3568"}, // 1.8V, 1.622V; WHL3568,  "12"
 };

 hwid_adc_info_t hardware_ver[] = {
-       {1024, 0, '1', '0'}, // 1.8V, NC;   V1, "10"
-       {922,  0, '2', '0'}, // 1.622V, NC; V2, "20"
-       {825,  0, '3', '0'}, // 1.433V, NC; V2.1, "30"
+       {1024, 0, '1', '0', "V1"}, // 1.8V, NC;   V1, "10"
+       {922,  0, '2', '0', "V2"}, // 1.622V, NC; V2, "20"
+       {825,  0, '3', '0', "V2.1"}, // 1.622V, NC; V2.1, "20"
 };
```

2.1.2.2 版本号自动适配

rk3568-11

注意：修改build.java是烧录启动后，系统设置里面的显示版本号概率显示编译的，和实际的prop信息不匹配 framework/base/core/java/android/os/Build.java

```java
/** A build ID string meant for displaying to the user */
-    public static final String DISPLAY = getString("ro.build.display.id");
+    private static String getNewDISPLAY() {
+        String hwid = SystemProperties.get("ro.boot.hwid", UNKNOWN);
+        if (!hwid.equals(UNKNOWN)) {
+            String ver = SystemProperties.get("ro.build.signway.board", UNKNOWN);
+            String[] hwidArray = hwid.split("_");
+            String hwidBoardName = hwidArray[0];
+            String[] verArray = ver.split("_");
+            ver = ver.replace(verArray[0], hwidBoardName);
+            Log.d(TAG, " ---> build new display ver = " + ver);
+            SystemProperties.set("ro.build.signway.board", ver);
+            SystemProperties.set("ro.build.display.id", ver);
+        }
+        return getString("ro.build.display.id");
+    }
+    public static final String DISPLAY = getNewDISPLAY();
+    //public static final String DISPLAY = getString("ro.build.display.id");

     /** The name of the overall product. */
     public static final String PRODUCT = getString("ro.product.name");
@@ -85,7 +102,13 @@ public class Build {
     public static final String BRAND = getString("ro.product.brand");

     /** The end-user-visible name for the end product. */
-    public static final String MODEL = getString("ro.product.model");
+    private static String getNewModel() {
+        String HWID = SystemProperties.get("ro.boot.hwid", UNKNOWN);
+        String[] hwidArray = HWID.split("_");
+        return hwidArray[0];
+    }
+    public static final String MODEL = getNewModel();
+    //public static final String MODEL = getString("ro.product.model");
```

framework/base/services/core/java/com/android/server/signway/SignwayManagerService.java

```java
 private SignwayNativeApi mSignwayNativeApi;
        private static int StorageType = 3;
+       private static final String UNKNOWN = "unknown";

        public static final String ACTION_SILENT_INSTALL = "android.intent.action.SIGNWAY_INSTALL";

@@ -70,7 +71,21 @@ public class SignwayManagerService extends ISignwayManager.Stub {

         mSignwayNativeApi = new SignwayNativeApi(mContext);
         Log.d(TAG, " new SignwayManagerService");
-               registerSignwayBroadcastReceiver();
+        registerSignwayBroadcastReceiver();
+
+        // for version rename
+        String hwid = getSystemProperties("ro.boot.hwid", UNKNOWN);
+        if (!hwid.equals(UNKNOWN)) {
+            String ver = getSystemProperties("ro.build.signway.board", UNKNOWN);
+            String[] hwidArray = hwid.split("_");
+            String hwidBoardName = hwidArray[0];
+            String[] verArray = ver.split("_");
+            Log.d(TAG, " ---> ver = " + verArray[0] + ", len = " + verArray[0].length());
+            ver = ver.replace(verArray[0], hwidBoardName);
+            Log.d(TAG, " ---> new ver = " + ver);
+            setSystemProperties("ro.build.signway.board", ver);
+            setSystemProperties("ro.build.display.id", ver);
+        }
     }

        protected void registerSignwayBroadcastReceiver() {
```

2.1.2.3 客户定制版本号不做自动适配

定制版本，有些客户需要固定版本信息，不需要自动适配版本号，可以在build.prop里面将persist.vendor.use_custom_version.enable设置为true

framework/base/core/java/android/os/Build.java

```java
String customVerEnable = SystemProperties.get("persist.vendor.use_custom_version.enable", UNKNOWN);
+        if (customVerEnable.equals("true")) {
+            return getString("ro.build.display.id");
+        }
```

framework/base/services/core/java/com/android/server/signway/SignwayManagerService.java

```java
// for version rename
+        String customVerEnable = SystemProperties.get("persist.vendor.use_custom_version.enable", UNKNOWN);
+        if (customVerEnable.equals("true")) {
+            return;
+        }
         String hwid = getSystemProperties("ro.boot.hwid", UNKNOWN);
```



### 2.2 壁纸
#### 2.2.1 壁纸路径

在frameworks/base/core下面搜索default_wallpaper.jpg，替换对应的文件

Tips：考虑兼容问题，不建议直接修改这里。

#### 2.2.2 壁纸缩放

在mk文件里面增加属性 persist.vendor.wallpaper.scaleZoom

默认壁纸是缩放模式：persist.vendor.wallpaper.scaleZoom = true

frameworks/base/packages/SystemUI/src/com/android/systemui/ImageWallpaper.java

```java
--- a/packages/SystemUI/src/com/android/systemui/ImageWallpaper.java
+++ b/packages/SystemUI/src/com/android/systemui/ImageWallpaper.java
@@ -46,6 +46,7 @@ import java.util.ArrayList;
 import java.util.List;

 import javax.inject.Inject;
+import android.os.SystemProperties;

 /**
  * Default built-in wallpaper that simply shows a static image.
@@ -203,6 +204,14 @@ public class ImageWallpaper extends WallpaperService {

         @Override
         public boolean shouldZoomOutWallpaper() {
+            /*
+            * for not zoom function;
+            * default: zoom
+            * jian.xu 2022-12-09
+            */
+            String isZoom = SystemProperties.get("persist.vendor.wallpaper.scaleZoom", "none");
+            if (isZoom.equals("false"))
+                return false;
             return true;
         }

```

#### 2.2.3 助手支持壁纸导入

按照助手提示，注意壁纸名称



### 2.3 开关机

#### 2.3.1 android标准开关机

将遥控器Power按键上报Key_Power，走系统Power流程



#### 2.3.2 公司信发和助手开关机

信发和助手需要在系统关机后，还有部分业务需要执行，例如：夜间下载，定时开关机等。

有些客户设备需要满足节能要求，所以待机机分两个模式：标准待机和低功耗待机

标准待机：只进行关闭屏幕，声音，控制指示灯等操作，系统不进入休眠

低功耗待机：系统断电，只有mcu工作。主板功耗在0.2w左右，这样整机功耗才能低于0.5w

待机流程：遥控Power按键（键值KEY_F11）-> framework层监听F11，发送广播com.assist.sleep.action ->助手监听广播做关机处理（信发App监听F11按键，不监听广播）->关机操作

rk3568-11：

frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java

```java
--- a/services/core/java/com/android/server/policy/PhoneWindowManager.java
+++ b/services/core/java/com/android/server/policy/PhoneWindowManager.java
@@ -2929,6 +2929,17 @@ public class PhoneWindowManager implements WindowManagerPolicy {
                 msg.sendToTarget();
             }
             return -1;
+        } else if (keyCode == KeyEvent.KEYCODE_F11) {
+            if (down) {
+                Log.e(TAG, "--->KeyCodeF11, send sleep action to assist app");
+                final String ACTION_NAME = "com.assist.sleep.action";
+                final String PACKAGE_NAME = "com.signway.assistapp";
+                Intent mIntent = new Intent();
+                mIntent.setAction(ACTION_NAME);
+                mIntent.setPackage(PACKAGE_NAME);
+                mContext.sendBroadcast(mIntent);
+            }
+            return 0;
         }

         // Toggle Caps Lock on META-ALT.

```



### 2.4 App安装白名单

1. 在`vendor/signway/aiot3568/customer/normal`

   下面添加文件`WhiteListAppFilter.properties`

​       这里的内容就是要加入白名单的app的包名

2. 然后进行文件拷贝

   ```makefile
   diff --git a/aiot3568/customers/normal/aiot3568.mk b/aiot3568/customers/normal/aiot3568.mk
   index b440b53..f707be5 100755
   --- a/aiot3568/customers/normal/aiot3568.mk
   +++ b/aiot3568/customers/normal/aiot3568.mk
   @@ -16,7 +16,8 @@ PRODUCT_PACKAGE_OVERLAYS += vendor/signway/$(PRODUCT_DIRS)/customers/$(PRODUCT_M
    # can and bluetooth
    PRODUCT_COPY_FILES += \
    	vendor/signway/$(PRODUCT_DIRS)/customers/$(PRODUCT_MODEL_DIRS)/can/can_bus_cfg.sh:vendor/bin/can_bus_cfg.sh \
   -	vendor/signway/$(PRODUCT_DIRS)/customers/$(PRODUCT_MODEL_DIRS)/bluetooth/bt_vendor.conf:$(TARGET_COPY_OUT_VENDOR)/etc/bluetooth/bt_vendor.conf
   +	vendor/signway/$(PRODUCT_DIRS)/customers/$(PRODUCT_MODEL_DIRS)/bluetooth/bt_vendor.conf:$(TARGET_COPY_OUT_VENDOR)/etc/bluetooth/bt_vendor.conf \
   + vendor/signway/$(PRODUCT_DIRS)/customers/$(PRODUCT_MODEL_DIRS)/WhiteListAppFilter.properties:system/etc/WhiteListAppFilter.prop
    
   ```

3. 添加属性开关（也是在normal/aiot3568.mk）

   ```makefile
    # build.prop info
    PRODUCT_PROPERTY_OVERRIDES += \
   @@ -41,10 +42,12 @@ PRODUCT_PROPERTY_OVERRIDES += \
    
    PRODUCT_PROPERTY_OVERRIDES += \
    	persist.statusbar.force_hide=true \
   	persist.sys.installapk.enable=false \
    	persist.navbar.touch_up_auto=false \
    	persist.sys.bootvideo.enable=true \
    	persist.navbar.display=1 \
    	persist.vendor.app.rotation_system=true \
   +	persist.vendor.whitelist.enable=true \
    	persist.vendor.eth_tether_enable=false
   ```

4. 在framework/base中添加白名单功能，在安装APP时检查是否在白名单中

   ```java
   diff --git a/services/core/java/com/android/server/pm/PackageManagerService.java b/services/core/java/com/android/server/pm/PackageManagerService.java
   index b352342c8d0b..437b96d9e168 100644
   --- a/services/core/java/com/android/server/pm/PackageManagerService.java
   +++ b/services/core/java/com/android/server/pm/PackageManagerService.java
   @@ -445,6 +445,9 @@ import java.util.function.Consumer;
    import java.util.function.Predicate;
    import java.util.function.Supplier;
    
   +import java.io.InputStreamReader;
   +import java.io.BufferedReader;
   +
    /**
     * Keep track of all those APKs everywhere.
     * <p>
   @@ -17334,6 +17337,35 @@ public class PackageManagerService extends IPackageManager.Stub
                return this;
            }
        }
   +    // jian.xu 22-07-26
   +    private boolean checkWhitelistPackageName(String packagename){
   +        ArrayList<String> whiteListApp = new ArrayList<String>();
   +        try{
   +            BufferedReader br = new BufferedReader(new InputStreamReader(
   +            new FileInputStream("/system/etc/WhiteListAppFilter.properties")));
   +
   +            String line ="";
   +            while ((line = br.readLine()) != null){
   +                whiteListApp.add(line);
   +            }
   +
   +            br.close();
   +        }catch(java.io.FileNotFoundException ex){
   +            return false;
   +        }catch(java.io.IOException ex){
   +            return false;
   +        }
   +
   +        Iterator<String> it = whiteListApp.iterator();
   +
   +        while (it.hasNext()) {
   +            String whitelisItem = it.next();
   +            if (whitelisItem.equals(packagename)) {
   +                return true;
   +            }
   +        }
   +        return false;
   +    }
    
        @GuardedBy("mInstallLock")
        private PrepareResult preparePackageLI(InstallArgs args, PackageInstalledInfo res)
   @@ -17702,6 +17734,19 @@ public class PackageManagerService extends IPackageManager.Stub
                throw new PrepareFailure(INSTALL_FAILED_INSUFFICIENT_STORAGE, "Failed rename");
            }
    
   +         // jian.xu 2022-07-05
   +        boolean whitelistResult = false;
   +        String whitelistEnable = SystemProperties.get("persist.vendor.whitelist.enable", "false");
   +        if (whitelistEnable.equals("true")) {
   +             Log.e(TAG, "---> install app name = " + pkgName);
   +             whitelistResult = checkWhitelistPackageName(pkgName);
   +             Log.e(TAG, "---> whitelistResult = " + whitelistResult);
   +             if (!whitelistResult) {
   +                throw new PrepareFailure(INSTALL_FAILED_INVALID_APK,
   +                                        "Could install app, install is disabled ");
   +           }
   +        }
   + 
            try {
                setUpFsVerityIfPossible(parsedPackage);
            } catch (InstallerException | IOException | DigestException | NoSuchAlgorithmException e) {
   
   ```
   
   


### 2.5 音频
#### 2.5.1 喇叭功率

在对应主板的codec驱动里面增加喇叭功率映射表

AIoT3568-11， kernel/sound/soc/codec/rk817_codec.c

AIoT3588-12,	kernel-5.10/sound/soc/codec/es8323.c

es8323.c的补丁如下：

spktosheet_DEFAULT这个表根据不同主板，单独调试

```c
--- a/sound/soc/codecs/es8323.c
+++ b/sound/soc/codecs/es8323.c
@@ -26,6 +26,8 @@
 #include <sound/initval.h>
 #include <linux/proc_fs.h>
 #include "es8323.h"
+#include <linux/of_gpio.h>
+#include <linux/gpio.h>

 #define NR_SUPPORTED_MCLK_LRCK_RATIOS 5
 static const unsigned int supported_mclk_lrck_ratios[NR_SUPPORTED_MCLK_LRCK_RATIOS] = {
@@ -85,6 +87,37 @@ static struct reg_default es8323_reg_defaults[] = {
        { 0x2c, 0x38 },
 };

+#ifdef CONFIG_SPEAKER_MISC
+#include <linux/miscdevice.h>
+
+#define VENDOR_STATUS_ON       1
+
+#define POWER_MAX 10
+#define RESISTANCE_MAX 4
+static int spktosheet_DEFAULT[POWER_MAX][RESISTANCE_MAX] = {
+//    2o,   4o,   6o,   8o
+       {0x15, 0x17, 0x18, 0x19},  //1W    17 x 0.8 x     x
+       {0x17, 0x19, 0x19, 0x1a},  //2W    18 x 1.1 1.2   x
+       {0x18, 0x19, 0x1a, 0x1b},  //3W    19 x 2.3 1.7  1.2
+       {0x19, 0x1a, 0x1b, 0x1c},  //4W    1a x 3.3 2.4  1.8
+       {0x19, 0x1b, 0x1c, 0x1d},  //5W    1b x 4.7 3.4  2.5
+       {0x1a, 0x1b, 0x1c, 0x1d},  //6W    1c x 6.6 4.7  3.6
+       {0x1a, 0x1c, 0x1d, 0x1e},  //7W    1d x 9.2 6.6  5.0
+       {0x1b, 0x1c, 0x1d, 0x1f},  //8W    1e x  x  8.5  6.5
+       {0x1c, 0x1d, 0x1e, 0x1f},  //9W    1f x  x  10.9 8.3
+       {0x1c, 0x1d, 0x1f, 0x1f}   //10W   20 x  x   x   9.3
+};
+
+struct spk_misc {
+       /* speaker power configs */
+       struct miscdevice misc_dev;
+       uint8_t spk_status;
+       uint8_t spk_power;
+       uint8_t spk_resistance;
+       int spk_volume;
+};
+#endif
+
 /* codec private data */
 struct es8323_priv {
        unsigned int sysclk;
@@ -93,7 +126,14 @@ struct es8323_priv {
        struct snd_pcm_hw_constraint_list sysclk_constraints;
        struct snd_soc_component *component;
        struct regmap *regmap;
+       #ifdef CONFIG_SPEAKER_MISC
+       struct spk_misc spk_misc;
+       #endif
 };
+#ifdef CONFIG_SPEAKER_MISC
+static void spk_misc_register(struct es8323_priv *es8323);
+static void spk_misc_update_codec_volume(struct es8323_priv *es8323);
+#endif

 static int es8323_reset(struct snd_soc_component *component)
 {
@@ -204,10 +244,12 @@ static const struct snd_kcontrol_new es8323_snd_controls[] = {
                       7, 1, bypass_tlv2),
        SOC_SINGLE_TLV("Right Mixer Right Bypass Volume", ES8323_DACCONTROL20,
                       3, 7, 1, bypass_tlv2),
:
+};
+
+struct spk_misc {
+       /* speaker power configs */
+       struct miscdevice misc_dev;
+       uint8_t spk_status;
+       uint8_t spk_power;
+       uint8_t spk_resistance;
+       int spk_volume;
+};
+#endif
+
 /* codec private data */
 struct es8323_priv {
        unsigned int sysclk;
@@ -93,7 +126,14 @@ struct es8323_priv {
        struct snd_pcm_hw_constraint_list sysclk_constraints;
        struct snd_soc_component *component;
        struct regmap *regmap;
+       #ifdef CONFIG_SPEAKER_MISC
+       struct spk_misc spk_misc;
+       #endif
 };
+#ifdef CONFIG_SPEAKER_MISC
+static void spk_misc_register(struct es8323_priv *es8323);
+static void spk_misc_update_codec_volume(struct es8323_priv *es8323);
+#endif

 static int es8323_reset(struct snd_soc_component *component)
 {
@@ -204,10 +244,12 @@ static const struct snd_kcontrol_new es8323_snd_controls[] = {
                       7, 1, bypass_tlv2),
        SOC_SINGLE_TLV("Right Mixer Right Bypass Volume", ES8323_DACCONTROL20,
                       3, 7, 1, bypass_tlv2),
+#ifndef CONFIG_SPEAKER_MISC
        SOC_DOUBLE_R_TLV("Output 1 Playback Volume", ES8323_DACCONTROL24,
                         ES8323_DACCONTROL25, 0, 33, 0, out_tlv),
        SOC_DOUBLE_R_TLV("Output 2 Playback Volume", ES8323_DACCONTROL26,
                         ES8323_DACCONTROL27, 0, 33, 0, out_tlv),
+#endif
 };

 static const struct snd_kcontrol_new es8323_left_line_controls =
@@ -732,8 +774,12 @@ static int es8323_resume(struct snd_soc_component *component)
        snd_soc_component_write(component, ES8323_CHIPPOWER, 0x00);
        snd_soc_component_write(component, ES8323_DACPOWER, 0x0c);
        snd_soc_component_write(component, ES8323_ADCPOWER, 0x59);
+       #ifdef CONFIG_SPEAKER_MISC
+       spk_misc_update_codec_volume(es8323);
+       #else
        snd_soc_component_write(component, 0x31, es8323_DEF_VOL);
        snd_soc_component_write(component, 0x30, es8323_DEF_VOL);
+       #endif
        snd_soc_component_write(component, 0x19, 0x02);
        return 0;
 }
@@ -792,8 +838,12 @@ static int es8323_probe(struct snd_soc_component *component)
        usleep_range(18000, 20000);
        snd_soc_component_write(component, 0x2E, 0x1E);
        snd_soc_component_write(component, 0x2F, 0x1E);
+       #ifdef CONFIG_SPEAKER_MISC
+       spk_misc_update_codec_volume(es8323);
+       #else
        snd_soc_component_write(component, 0x30, 0x1E);
        snd_soc_component_write(component, 0x31, 0x1E);
+       #endif
        snd_soc_component_write(component, 0x03, 0x09);
        snd_soc_component_write(component, 0x02, 0x00);
        usleep_range(18000, 20000);
@@ -867,6 +917,9 @@ static int es8323_i2c_probe(struct i2c_client *i2c,
        ret = devm_snd_soc_register_component(&i2c->dev,
                                              &soc_codec_dev_es8323,
                                              &es8323_dai, 1);
+       #ifdef CONFIG_SPEAKER_MISC
+       spk_misc_register(es8323);
+       #endif
        return ret;
 }

@@ -920,3 +973,162 @@ module_i2c_driver(es8323_i2c_driver);
 MODULE_DESCRIPTION("ASoC es8323 driver");
 MODULE_AUTHOR("Mark Brown <will@everset-semi.com>");
 MODULE_LICENSE("GPL");
+
+#ifdef CONFIG_SPEAKER_MISC
+static inline struct es8323_priv *
+                       device_to_codec_priv(struct device *device)
+{
+       return container_of(dev_get_drvdata(device), \
+               struct es8323_priv, spk_misc.misc_dev);
+}
+
+static ssize_t spk_parameters_store(struct device *device,
+                       struct device_attribute *attr,
+                       const char *buf, size_t count)
+{
+       struct es8323_priv *priv = device_to_codec_priv(device);
+       int spk_power;
+       int ret;
+       int spk_resistance;
+
+       ret = sscanf(buf, "%d %d", &spk_power, &spk_resistance);
+       pr_err("===> get spk_power=%d, resistance=%d, ret=%d\n", spk_power, \
+               spk_resistance, ret);
+       if (ret != 2) {
+               pr_err("===> spk_power_resistance param len invalid, %d\n", ret);
+               return -1;
+       }
+
+       if ((spk_power > 10) || (spk_power < 1)) {
+               pr_err("===> spk_power is invalid, %d\n", spk_power);
+               return -1;
+       }
+
+       if ((spk_resistance!=2) && (spk_resistance!=4) && (spk_resistance!=6) \
+               && (spk_resistance!=8)) {
+               pr_err("===> spk_resistance is invalid, %d\n", spk_resistance);
+               return -1;
+       }
+       priv->spk_misc.spk_power = spk_power;
+       priv->spk_misc.spk_resistance = spk_resistance;
+       spk_misc_update_codec_volume(priv);
+       return count;
+}
+
+static ssize_t spk_parameters_show(struct device *device,
+                       struct device_attribute *attr, char *buf)
+{
+       struct es8323_priv *priv = device_to_codec_priv(device);
+
+       sprintf(buf, "%d %d\n", priv->spk_misc.spk_power, priv->spk_misc.spk_resistance);
+       return strlen(buf);
+}
+
+static DEVICE_ATTR_RW(spk_parameters);
+
+static struct attribute *spk_misc_attrs[] = {
+       &dev_attr_spk_parameters.attr,
+       NULL
+};
+
+static const struct attribute_group spk_misc_group = {
+       .attrs = spk_misc_attrs,
+};
+
+static const struct attribute_group *spk_misc_groups[] = {
+       &spk_misc_group,
+       NULL
+};
+
+void spk_misc_update_codec_volume(struct es8323_priv *es8323)
+{
+       int (*powersheet)[RESISTANCE_MAX] = spktosheet_DEFAULT;
+
+       es8323->spk_misc.spk_volume = \
+                       *(*(powersheet+(es8323->spk_misc.spk_power - 1)) \
+                       + (es8323->spk_misc.spk_resistance/2 - 1));
+       pr_err("---> spk_volume = 0x%x\n", es8323->spk_misc.spk_volume);
+       snd_soc_component_write(es8323->component, ES8323_LOUT2_VOL,
+                               es8323->spk_misc.spk_volume);
+       snd_soc_component_write(es8323->component, ES8323_ROUT2_VOL,
+                               es8323->spk_misc.spk_volume);
+}
+
+#ifdef CONFIG_SPEAKER_VENDOR
+/*
+ * return: -1:failed, 0: ok
+ */
+static int vendor_other_get_spk_configs(struct es8323_priv *priv)
+{
+       int checksum, status, version;
+       int spk_status, spk_flag, spk_power, spk_resistance;
+       char *uboot_bl_ptr;
+
+       uboot_bl_ptr = strstr(saved_command_line, "androidboot.other=");
+       if (uboot_bl_ptr) {
+               sscanf(uboot_bl_ptr, "androidboot.other=%02x,%02x,%02x", &checksum,
+                       &status, &version);
+               pr_err("===> %s, checksum=%x, status=%x, ver=%x\n", __func__, checksum,
+                       status, version);
+       } else {
+               pr_err("===> %s, no other command line\n", __func__);
+               return -1;
+       }
+
+       if (status != VENDOR_STATUS_ON) {
+               pr_err("===> %s, other status not on, status=%x\n", __func__, status);
+               return -1;
+       }
+
+       uboot_bl_ptr = strstr(saved_command_line, "androidboot.o_spk=");
+       if (uboot_bl_ptr) {
+               sscanf(uboot_bl_ptr, "androidboot.o_spk=%02x,%02x,%02x,%02x",
+                       &spk_status, &spk_flag, &spk_power, &spk_resistance);
+               pr_err("===> %s, spk_status=%x, spk_flag = %x, spk_power=%x, spk_resistance=%x\n",
+                       __func__, spk_status, spk_flag, spk_power, spk_resistance);
+       } else {
+               pr_err("===> %s, no o_spk command line\n", __func__);
+               return -1;
+       }
+
+       if (spk_status != VENDOR_STATUS_ON) {
+               pr_err("===> %s, other_info.spk.status is off\n", __func__);
+               return -1;
+       }
+       if ((spk_resistance != 2) && (spk_resistance != 4) \
+          && (spk_resistance != 6) && (spk_resistance != 8)) {
+                       pr_err("===> priv->spk_spk_resistance is error\n");
+                       return -1;
+       }
+       if ((spk_power > 10) || (spk_power < 1)) {
+               pr_err("===> priv->spk_power is error\n");
+               return -1;
+       }
+       priv->spk_misc.spk_status = spk_status;
+       priv->spk_misc.spk_power = spk_power;
+       priv->spk_misc.spk_resistance = spk_resistance;
+       //spk_misc_update_codec_volume(priv);
+       return 0;
+}
+#endif
+
+static void spk_misc_init(struct es8323_priv *es8323)
+{
+       #ifdef CONFIG_SPEAKER_VENDOR
+       vendor_other_get_spk_configs(es8323);
+       #else
+       priv->spk_misc.spk_power = 2;
+       priv->spk_misc.spk_resistance = 4;
+       #endif
+}
+
+void spk_misc_register(struct es8323_priv *es8323)
+{
+       spk_misc_init(es8323);
+       es8323->spk_misc.misc_dev.minor = MISC_DYNAMIC_MINOR;
+       es8323->spk_misc.misc_dev.name  = "spk-power";
+       es8323->spk_misc.misc_dev.groups = spk_misc_groups;
+       misc_register(&es8323->spk_misc.misc_dev);
+}
+#endif

```



#### 2.5.2 音频输出、输入策略

修改音频输出，输入策略，基本就是engine.cpp文件

AIoT3588-12	frameworks/av/services/audiopolicy/enginedefault/src/Engine.cpp

```cpp
--- a/services/audiopolicy/enginedefault/src/Engine.cpp
+++ b/services/audiopolicy/enginedefault/src/Engine.cpp
@@ -36,6 +36,9 @@
 #include <utils/Log.h>
 #include <cutils/properties.h>

+#define AUDIO_DEV_DEF  "default"
+#define AUDIO_DEV_AUTO "auto"
+
 namespace android
 {
 namespace audio_policy
@@ -261,6 +264,19 @@ DeviceVector Engine::getDevicesForStrategyInt(legacy_strategy strategy,
                                               const SwAudioOutputCollection &outputs) const
 {
     DeviceVector devices;
+    char *targetdev = new char[16];
+    char *targetspeaker = new char[16];
+    property_get("persist.vendor.audiopolicy.out", targetdev, AUDIO_DEV_DEF);
+    property_get("persist.vendor.audiopolicy.speaker", targetspeaker, "false");
+    if (strcmp(targetdev, "auto") && strcmp(targetdev, "bt") && \
+        strcmp(targetdev, "usb")  && strcmp(targetdev, "dp") && \
+        strcmp(targetdev, "hdmi") && strcmp(targetdev, "speaker")) {
+        /* reset to default policy value */
+        strcpy(targetdev, AUDIO_DEV_DEF);
+    }
+
+    ALOGV("%s audiopolicy.out:%s,  audiopolicy.speaker:%s", __func__, targetdev, targetspeaker);
+

     switch (strategy) {

@@ -330,7 +346,28 @@ DeviceVector Engine::getDevicesForStrategyInt(legacy_strategy strategy,
     case STRATEGY_SONIFICATION_RESPECTFUL:
     case STRATEGY_REROUTING:
     case STRATEGY_MEDIA: {
+        /* signway audio policy */
+        /*auto : BT/USB > HDMI > Headset > Speaker */
+        /*speacial: "alway_speaker" means in all conditions, speaker have sound*/
         DeviceVector devices2;
+
+        bool isAvailibleBt =(!availableOutputDevices.getDevicesFromType(AUDIO_DEVICE_OUT_BLUETOOTH_A2DP).isEmpty())  || \
+                            (!availableOutputDevices.getDevicesFromType(AUDIO_DEVICE_OUT_BLUETOOTH_A2DP_HEADPHONES).isEmpty())  || \
+                            (!availableOutputDevices.getDevicesFromType(AUDIO_DEVICE_OUT_BLUETOOTH_A2DP_SPEAKER).isEmpty());
+
+        bool isAvailibleUsb = (!availableOutputDevices.getDevicesFromType(AUDIO_DEVICE_OUT_USB_HEADSET).isEmpty())  || \
+                            (!availableOutputDevices.getDevicesFromType(AUDIO_DEVICE_OUT_USB_ACCESSORY).isEmpty())  || \
+                            (!availableOutputDevices.getDevicesFromType(AUDIO_DEVICE_OUT_USB_DEVICE).isEmpty());
+
+        bool isAvailibleDp = (!availableOutputDevices.getDevicesFromType(AUDIO_DEVICE_OUT_SPDIF).isEmpty());
+
+        bool isAvailibleHeadset = (!availableOutputDevices.getDevicesFromType(AUDIO_DEVICE_OUT_ANLG_DOCK_HEADSET).isEmpty());
+
+        bool isAvailibleHdmi = (!availableOutputDevices.getDevicesFromType(AUDIO_DEVICE_OUT_HDMI).isEmpty())  || \
+                                (!availableOutputDevices.getDevicesFromType(AUDIO_DEVICE_OUT_HDMI_ARC).isEmpty())  || \
+                                (!availableOutputDevices.getDevicesFromType(AUDIO_DEVICE_OUT_AUX_LINE).isEmpty());
+
+        ALOGV("%s >> isAvailibleBt:%d isAvailibleUsb:%d isAvailibleDp:%d isAvailibleHeadset:%d isAvailibleHdmi:%d", __func__, isAvailibleBt, isAvailibleUsb, isAvailibleDp, isAvailibleHeadset, isAvailibleHdmi);
         if (strategy != STRATEGY_SONIFICATION) {
             // no sonification on remote submix (e.g. WFD)
             sp<DeviceDescriptor> remoteSubmix;
@@ -340,6 +377,78 @@ DeviceVector Engine::getDevicesForStrategyInt(legacy_strategy strategy,
                 devices2.add(remoteSubmix);
             }
         }
+        if (!strcmp(targetdev, "speaker")) {
+            ALOGD("%s audio policy : speaker", __func__);
+            if(devices2.isEmpty()){
+                ALOGD("%s finding speaker devices...",__func__);
+                devices2 = availableOutputDevices.getDevicesFromType(AUDIO_DEVICE_OUT_SPEAKER);
+            }
+            if(!devices2.isEmpty()) {
+                ALOGD("%s speaker policy apply",__func__);
+            }
+        }
+
+        if (!strcmp(targetdev, "hdmi")) {
+            ALOGD("%s audio policy : hdmi", __func__);
+            if(devices2.isEmpty()){
+                ALOGD("%s finding hdmi devices...",__func__);
+                devices2 = availableOutputDevices.getDevicesFromTypes({
+                     AUDIO_DEVICE_OUT_HDMI, AUDIO_DEVICE_OUT_HDMI_ARC,AUDIO_DEVICE_OUT_AUX_LINE,
+                    });
+            }
+            if(!devices2.isEmpty()) ALOGD("%s hdmi policy apply",__func__);
+        }
+
+        if (!strcmp(targetdev, "dp")) {
+            ALOGD("%s audio policy : dp", __func__);
+            if(devices2.isEmpty()){
+                ALOGD("%s isAvailibleDp = %d , finding dp devices...",__func__, isAvailibleDp);
+                devices2 = availableOutputDevices.getDevicesFromType(AUDIO_DEVICE_OUT_SPDIF);
+            }
+            if(!devices2.isEmpty()) ALOGD("%s dp policy apply",__func__);
+        }
+
+        if (!strcmp(targetdev, "bt")) {
+            ALOGD("%s audio policy : bt",__func__);
+            if (devices2.isEmpty() && (getLastRemovableMediaDevices().size() > 0)) {
+                ALOGD("%s isAvailibleBt = %d , finding bt devices...", __func__, isAvailibleBt);
+                devices2 = availableOutputDevices.getDevicesFromType(AUDIO_DEVICE_OUT_BLUETOOTH_A2DP);
+                if (devices2.isEmpty()){
+                    ALOGD("%s bt AUDIO_DEVICE_OUT_BLUETOOTH_A2DP failed",__func__);
+                    devices2 = availableOutputDevices.getDevicesFromType(AUDIO_DEVICE_OUT_BLUETOOTH_A2DP_HEADPHONES);
+                }
+                if (devices2.isEmpty()) {
+                    ALOGD("%s bt AUDIO_DEVICE_OUT_BLUETOOTH_A2DP_HEADPHONES failed",__func__);
+                    devices2 = availableOutputDevices.getDevicesFromType(AUDIO_DEVICE_OUT_BLUETOOTH_A2DP_SPEAKER);
+                }
+            }else{
+                ALOGD("%s policy bt warning",__func__);
+                devices.remove(devices2);
+            }
+            if(!devices2.isEmpty()) ALOGD("%s bt policy apply",__func__);
+        }
+
+        if (!strcmp(targetdev, "usb")) {
+            ALOGD("%s audio policy : usb",__func__);
+            if (devices2.isEmpty()) {
+                ALOGD("%s isAvailibleUsb = %d , finding usb devices...", __func__, isAvailibleUsb);
+                devices2 = availableOutputDevices.getDevicesFromType(AUDIO_DEVICE_OUT_USB_HEADSET);
+                if (devices2.isEmpty()) {
+                    ALOGD("%s usb AUDIO_DEVICE_OUT_USB_HEADSET failed",__func__);
+                    devices2 = availableOutputDevices.getDevicesFromType(AUDIO_DEVICE_OUT_USB_ACCESSORY);
+                }
+                if (devices2.isEmpty()) {
+                    ALOGD("%s usb AUDIO_DEVICE_OUT_USB_ACCESSORY failed",__func__);
+                    devices2 = availableOutputDevices.getDevicesFromType(AUDIO_DEVICE_OUT_USB_DEVICE);
+                }
+            }else{
+                ALOGD("%s policy usb warning",__func__);
+                devices.remove(devices2);
+            }
+            if(!devices2.isEmpty()) ALOGD("%s usb policy apply",__func__);
+        }
+
+        if(devices2.isEmpty()) ALOGD("%s no prior device regisiter, default OUT policy apply",__func__);

         if ((devices2.isEmpty()) &&
             (getForceUse(AUDIO_POLICY_FORCE_FOR_MEDIA) == AUDIO_POLICY_FORCE_SPEAKER)) {
@@ -418,6 +527,13 @@ DeviceVector Engine::getDevicesForStrategyInt(legacy_strategy strategy,
                     availableOutputDevices.getDevicesFromType(
                             AUDIO_DEVICE_OUT_SPEAKER_SAFE));
         }
+        if (!strcmp(targetspeaker, "true")) {
+            if (devices.getDevicesFromType(AUDIO_DEVICE_OUT_SPEAKER).isEmpty() &&
+                !availableOutputDevices.getDevicesFromType(AUDIO_DEVICE_OUT_SPEAKER).isEmpty()) {
+                ALOGD("always speaker, Add speaker device");
+                devices.add(availableOutputDevices.getDevicesFromType(AUDIO_DEVICE_OUT_SPEAKER));
+            }
+        }
         } break;

     case STRATEGY_CALL_ASSISTANT:
@@ -460,6 +576,10 @@ sp<DeviceDescriptor> Engine::getDeviceForInputSource(audio_source_t inputSource)
             : availableInputDevices.getDevicesFromHwModule(primaryOutput->getModuleHandle());
     sp<DeviceDescriptor> device;

+    /*default device audio IN policy (auto)*/
+    /* BT > HEADSET > USB > MIC */
+    char *targetdev = new char[16];
+    property_get("persist.vendor.audiopolicy.in", targetdev, AUDIO_DEV_DEF); // auto usb headset mic
     // when a call is active, force device selection to match source VOICE_COMMUNICATION
     // for most other input sources to avoid rerouting call TX audio
     if (isInCall()) {
@@ -484,6 +604,39 @@ sp<DeviceDescriptor> Engine::getDeviceForInputSource(audio_source_t inputSource)
     switch (inputSource) {
     case AUDIO_SOURCE_DEFAULT:
     case AUDIO_SOURCE_MIC:
+        ALOGV("input Source : AUDIO_SOURCE_MIC policy: \"%s\"", targetdev);
+
+        if (!strcmp(targetdev, "usb")) {
+            ALOGD("%s audio IN policy : usb", __func__);
+            device = availableDevices.getDevice(
+                    AUDIO_DEVICE_IN_USB_DEVICE, String8(""), AUDIO_FORMAT_DEFAULT);
+            if (device != nullptr) {
+                ALOGD("%s usb policy apply", __func__);
+                break;
+            }
+        }
+
+        if(!strcmp(targetdev, "headset")){
+            ALOGD("%s audio IN policy : headset", __func__);
+            device = availableDevices.getFirstExistingDevice({
+                    AUDIO_DEVICE_IN_WIRED_HEADSET, AUDIO_DEVICE_IN_BUILTIN_MIC});
+            if (device != nullptr) {
+                ALOGD("%s headset policy apply", __func__);
+                break;
+            }
+        }
+
+        // if(!strcmp(targetdev, "board_mic")){
+        //     ALOGD("%s audio IN policy : board_mic", __func__);
+        //     device = availableDevices.getDevice(
+        //             AUDIO_DEVICE_IN_BUILTIN_MIC, String8(""), AUDIO_FORMAT_DEFAULT);
+        //     if (device != nullptr) {
+        //         ALOGD("%s board_mic policy apply", __func__);
+        //         break;
+        //     }
+        // }
+
+        ALOGD("%s no prior device regisiter, \"auto\" IN apply", __func__);
         device = availableDevices.getDevice(
                 AUDIO_DEVICE_IN_BLUETOOTH_A2DP, String8(""), AUDIO_FORMAT_DEFAULT);
         if (device != nullptr) break;

```



### 2.6 蓝牙*

### 2.7 导航栏和状态栏

#### 2.7.1 导航栏内容

rk3588-12

frameworks/base/packages/SystemUI/res/values-sw900dp/config.xml

```xml
<!-- Nav bar button default ordering/layout -->
    <string name="config_navBarLayout" translatable="false">left;volume_sub,back,home,recent,volume_add,screenshot;right</string>
```

关闭任务栏

frameworks/base/packages/SystemUI/shared/src/com/android/systemui/shared/recents/utilities/Utilities.java

```java
--- a/packages/SystemUI/shared/src/com/android/systemui/shared/recents/utilities/Utilities.java
+++ b/packages/SystemUI/shared/src/com/android/systemui/shared/recents/utilities/Utilities.java
@@ -127,7 +127,7 @@ public class Utilities {

         float smallestWidth = dpiFromPx(Math.min(bounds.width(), bounds.height()),
                 context.getResources().getConfiguration().densityDpi);
-        return smallestWidth >= TABLET_MIN_DPS;
+        return false;//smallestWidth >= TABLET_MIN_DPS;
     }

     public static float dpiFromPx(float size, int densityDpi) {
```

#### 2.7.2 增加导航栏状态栏显示和隐藏

涉及frameworks/base和packages/apps/Settings

frameworks/base/core/res/AndroidManifest.xml

```xml
--- a/core/res/AndroidManifest.xml
+++ b/core/res/AndroidManifest.xml
@@ -26,6 +26,8 @@
     <!-- ================================================ -->
     <eat-comment />

+    <protected-broadcast android:name="android.intent.action.NAVIGATION_DISPLAY" />
+
     <protected-broadcast android:name="android.intent.action.SCREEN_OFF" />
     <protected-broadcast android:name="android.intent.action.SCREEN_ON" />
     <protected-broadcast android:name="android.intent.action.USER_PRESENT" />

```

frameworks/base/packages/SystemUI/src/com/android/systemui/navigationbar/NavigationBarController.java

```java
--- a/packages/SystemUI/src/com/android/systemui/navigationbar/NavigationBarController.java
+++ b/packages/SystemUI/src/com/android/systemui/navigationbar/NavigationBarController.java
@@ -278,6 +278,14 @@ public class NavigationBarController implements
         }
     }

+    //<E6><B7><BB><E5><8A><A0><E7><A7><BB><E9><99><A4><E7><9A><84><E9><80><BB><E8><BE><91>
+    public void hideNavigationBar() {
+        Display[] displays = mDisplayManager.getDisplays();
+        for (Display display : displays) {
+            removeNavigationBar(display.getDisplayId());
+        }
+    }
+
     /**
      * Adds a navigation bar on default display or an external display if the display supports
      * system decorations.

```

frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/StatusBar.java

```java
--- a/packages/SystemUI/src/com/android/systemui/statusbar/phone/StatusBar.java
+++ b/packages/SystemUI/src/com/android/systemui/statusbar/phone/StatusBar.java
@@ -680,6 +680,10 @@ public class StatusBar extends SystemUI implements
     private final ColorExtractor.OnColorsChangedListener mOnColorsChangedListener =
             (extractor, which) -> updateTheme();

+    /*  */
+    RegisterStatusBarResult result = null;
+    /* for status bar & navigation display boardcast */
+    private static final String OP_NAVIGATION = "android.intent.action.NAVIGATION_DISPLAY";

@@ -1178,7 +1182,22 @@ public class StatusBar extends SystemUI implements
         mHeadsUpManager.addListener(mVisualStabilityManager);
         mNotificationPanelViewController.setHeadsUpManager(mHeadsUpManager);

-        createNavigationBar(result);
+        //createNavigationBar(result);
+        try {
+            boolean showNav_Temp = true;
+            String display = SystemProperties.get("persist.navbar.display", "1");
+            Log.v(TAG, "persist.navbar.display = " + display);
+            SystemProperties.set("persist.navbar.display", display);
+            if("0".equals(display)) {
+                showNav_Temp = false;
+                mStatusBarWindowController.setStatusBarDisplay(false);
+            }
+            Log.v(TAG, "hasNavigationBar=" + showNav_Temp);
+            if (showNav_Temp) {
+                createNavigationBar(result);
+            }
+        } catch (Exception ex) {
+        }

         if (ENABLE_LOCKSCREEN_WALLPAPER && mWallpaperSupported) {
             mLockscreenWallpaper = mLockscreenWallpaperLazy.get();
@@ -1406,6 +1425,7 @@ public class StatusBar extends SystemUI implements
         filter.addAction(Intent.ACTION_CLOSE_SYSTEM_DIALOGS);
         filter.addAction(Intent.ACTION_SCREEN_OFF);
         filter.addAction(DevicePolicyManager.ACTION_SHOW_DEVICE_MONITORING_DIALOG);
+        filter.addAction(OP_NAVIGATION);
         mBroadcastDispatcher.registerReceiver(mBroadcastReceiver, filter, null, UserHandle.ALL);
     }

+            if("0".equals(display)) {
+                showNav_Temp = false;
+                mStatusBarWindowController.setStatusBarDisplay(false);
+            }
+            Log.v(TAG, "hasNavigationBar=" + showNav_Temp);
+            if (showNav_Temp) {
+                createNavigationBar(result);
+            }
+        } catch (Exception ex) {
+        }

         if (ENABLE_LOCKSCREEN_WALLPAPER && mWallpaperSupported) {
             mLockscreenWallpaper = mLockscreenWallpaperLazy.get();
@@ -1406,6 +1425,7 @@ public class StatusBar extends SystemUI implements
         filter.addAction(Intent.ACTION_CLOSE_SYSTEM_DIALOGS);
         filter.addAction(Intent.ACTION_SCREEN_OFF);
         filter.addAction(DevicePolicyManager.ACTION_SHOW_DEVICE_MONITORING_DIALOG);
+        filter.addAction(OP_NAVIGATION);
         mBroadcastDispatcher.registerReceiver(mBroadcastReceiver, filter, null, UserHandle.ALL);
     }

@@ -2667,6 +2687,28 @@ public class StatusBar extends SystemUI implements
             else if (DevicePolicyManager.ACTION_SHOW_DEVICE_MONITORING_DIALOG.equals(action)) {
                 mQSPanelController.showDeviceMonitoringDialog();
             }
+            else if (OP_NAVIGATION.equals(action)) {
+                if(intent.hasExtra("show_navigation_bar")) {
+                    boolean show = intent.getBooleanExtra("show_navigation_bar", true);
+                    Log.e("StatusBar", ">>>>>>>>>>>OP_NAVIGATION: " + show);
+                    SystemProperties.set("persist.navbar.display", show ? "1":"0");
+                    NavigationBarView mNavigationBarView = mNavigationBarController.getDefaultNavigationBarView();
+                    if (show) {
+                        if (mNavigationBarView != null)
+                            return;
+                        createNavigationBar(result);
+                        mStatusBarWindowController.setStatusBarDisplay(true);
+                    } else {
+                        if (mNavigationBarView == null)
+                            return;
+                        mNavigationBarController.hideNavigationBar();
+                        mStatusBarWindowController.setStatusBarDisplay(false);
+                    }
+                } else {
+                    Log.w(TAG,"didn't contain navigation_bar_show key");
+                }
+            }
+
             Trace.endSection();
         }

```

frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/window/StatusBarWindowController.java

```java
--- a/packages/SystemUI/src/com/android/systemui/statusbar/window/StatusBarWindowController.java
+++ b/packages/SystemUI/src/com/android/systemui/statusbar/window/StatusBarWindowController.java
@@ -315,4 +315,18 @@ public class StatusBarWindowController {
             mLpChanged.privateFlags &= ~PRIVATE_FLAG_FORCE_SHOW_STATUS_BAR;
         }
     }
+
+    public void setStatusBarDisplay(boolean display) {
+        if (mStatusBarWindowView == null) {
+            Log.e(TAG, "---> mStatusBarWindowView is null");
+            return;
+        }
+
+        if (display) {
+            mStatusBarWindowView.setVisibility(View.VISIBLE);
+        } else {
+            mStatusBarWindowView.setVisibility(View.GONE);
+        }
+    }
+
 }

```

packages/apps/Settings/res/values-zh-rCN/strings.xml

```xml
--- a/res/values-zh-rCN/strings.xml
+++ b/res/values-zh-rCN/strings.xml
@@ -5616,4 +5616,7 @@
	 <string name="ethernet_netmask"><E5><AD><90><E7><BD><91><E6><8E><A9><E7><A0><81></string>
     <string name="ethernet_gateway"><E7><BD><91><E5><85><B3></string>
     <string name="ethernet_mode_title"><E4><BB><A5><E5><A4><AA><E7><BD><91><E6><A8><A1><E5><BC><8F></string>
+
+    <!-- custom options -->
+    <string name="navbar_display"><E6><98><BE><E7><A4><BA><E5><AF><BC><E8><88><AA><E6><A0><8F></string>

```

packages/apps/Settings/res/xml/display_settings.xml

```xml
--- a/res/values/strings.xml
+++ b/res/values/strings.xml
@@ -13714,4 +13714,7 @@
     <string name="dialog_getting_screen_info">getting screen info...</string>
     <string name="dialog_update_resolution">Updating resolution...</string>
     <string name="dialog_wait_screen_connect">please wait while updating info to the device...</string>
+
+    <!-- custom options -->
+    <string name="navbar_display">"Display navigation"</string>
 </resources>

```

packages/apps/Settings/res/xml/display_settings.xml

```xml
--- a/res/xml/display_settings.xml
+++ b/res/xml/display_settings.xml
@@ -103,6 +103,11 @@
     <PreferenceCategory
         android:title="@string/category_name_display_controls">

+        <SwitchPreference
+            android:key="navbar_display"
+            android:title="@string/navbar_display"
+            settings:controller="com.android.settings.display.NavigationSwitchController" />
+
         <SwitchPreference
             android:key="auto_rotate"
             android:title="@string/accelerometer_title"

```

增加NavigationSwitchController.java文件

packages/apps/Settings/src/com/android/settings/display/NavigationSwitchController.java

```java
--- /dev/null
+++ b/src/com/android/settings/display/NavigationSwitchController.java
@@ -0,0 +1,70 @@
+/*
+ * Copyright (C) 2016 The Android Open Source Project
+ *
+ * Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file
+ * except in compliance with the License. You may obtain a copy of the License at
+ *
+ *      http://www.apache.org/licenses/LICENSE-2.0
+ *
+ * Unless required by applicable law or agreed to in writing, software distributed under the
+ * License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
+ * KIND, either express or implied. See the License for the specific language governing
+ * permissions and limitations under the License.
+ */
+package com.android.settings.display;
+
+import androidx.preference.Preference;
+import com.android.settings.core.PreferenceControllerMixin;
+import com.android.settings.core.TogglePreferenceController;
+import android.content.Context;
+import android.provider.Settings;
+
+import android.util.Log;
+import android.content.Intent;
+import android.os.SystemProperties;
+
+import com.android.settings.R;
+import com.android.settings.core.TogglePreferenceController;
+
+
+public class NavigationSwitchController extends TogglePreferenceController implements
+        PreferenceControllerMixin, Preference.OnPreferenceChangeListener {
+       private static final String TAG = "NavigationSwitchController";
+
+
+    public NavigationSwitchController(Context context, String key) {
+        super(context, key);
+    }
+
+    @Override
+    public boolean isChecked() {
+        String display = SystemProperties.get("persist.navbar.display");
+        Log.d(TAG, "---> isChecked, display = " + display);
+        if(display.equals("1")) {
+            return true;
+        }
+        return false;
+    }
+
+    @Override
+    public boolean setChecked(boolean isChecked) {
+        Log.d(TAG, "---> setChecked, isChecked = " + isChecked);
+        Intent mIntent = new Intent();
+        mIntent.putExtra("show_navigation_bar", isChecked);
+        mIntent.setAction("android.intent.action.NAVIGATION_DISPLAY");
+        mContext.sendBroadcast(mIntent);
+        return true;
+    }
+
+    @Override
+    @AvailabilityStatus
+    public int getAvailabilityStatus() {
+        Log.d(TAG, "---> getAvailabilityStatus\n");
+        return 0;
+    }
+
+    @Override
+    public int getSliceHighlightMenuRes() {
+        return R.string.menu_key_display;
+    }
+}

```

2.7.3 底部上划显示导航栏

此功能默认关闭，通过persist.navbar.touch_up_auto可以修改默认状态

frameworks/base/services/core/java/com/android/server/wm/DisplayPolicy.java

```java
--- a/services/core/java/com/android/server/wm/DisplayPolicy.java
+++ b/services/core/java/com/android/server/wm/DisplayPolicy.java
@@ -186,6 +186,8 @@ import java.util.HashMap;
 import java.util.Objects;
 import java.util.function.Consumer;

+import android.util.Log;
+
 /**
  * The policy that provides the basic behaviors and states of a display to show UI.
  */
@@ -509,6 +511,19 @@ public class DisplayPolicy {
                                     ? mNavigationBar
                                     : findAltBarMatchingPosition(ALT_BAR_BOTTOM);
                             requestTransientBars(bar, true /* isGestureOnSystemBar */);
+
+                            /* for display navbar; 20220929 */
+                            String touch_up = SystemProperties.get("persist.navbar.touch_up_auto", "false");
+                            if (touch_up.equals("true")) {
+                                String display = SystemProperties.get("persist.navbar.display");
+                                Log.d(TAG, "---> isChecked, display = " + display);
+                                if(display.equals("0")) {
+                                    Intent mIntent = new Intent();
+                                    mIntent.putExtra("show_navigation_bar", true);
+                                    mIntent.setAction("android.intent.action.NAVIGATION_DISPLAY");
+                                    mContext.sendBroadcast(mIntent);
+                                }
+                            }
                         }
                     }

```



### 2.8 系统默认root

标准版需要修改系统默认root

2.8.1 AIoT3568：

2.8.1.1 编译userdebug版本

2.8.1.2 关闭selinux

device/rockchip/common

```c++
diff --git a/BoardConfig.mk b/BoardConfig.mk
index 3706d7b7..8099919f 100755
--- a/BoardConfig.mk
+++ b/BoardConfig.mk
@@ -59,7 +59,7 @@ BOARD_BOOT_HEADER_VERSION ?= 2
BOARD_MKBOOTIMG_ARGS :=
BOARD_PREBUILT_DTBOIMAGE ?= $(TARGET_DEVICE_DIR)/dtbo.img
BOARD_ROCKCHIP_VIRTUAL_AB_ENABLE ?= false
-BOARD_SELINUX_ENFORCING ?= true
+BOARD_SELINUX_ENFORCING ?= false
```

2.8.1.3 修改su.cpp，注释用户组权限检测  

system/extras/su/su.cpp  

```c++
--- a/su/su.cpp
+++ b/su/su.cpp
@@ -80,8 +80,8 @@ void extract_uidgids(const char* uidgids, uid_t* uid, gid_t* gid, gid_t* gids, i
 }

 int main(int argc, char** argv) {
-    uid_t current_uid = getuid();
-    if (current_uid != AID_ROOT && current_uid != AID_SHELL) error(1, 0, "not allowed");
+    //uid_t current_uid = getuid();
+    //if (current_uid != AID_ROOT && current_uid != AID_SHELL) error(1, 0, "not allowed");

     // Handle -h and --help.
     ++argv;
```

2.8.1.4 给 su 文件默认授予 root 权限  

system/core/libcutils/fs_config.cpp  

```c++
--- a/libcutils/fs_config.cpp
+++ b/libcutils/fs_config.cpp
@@ -188,7 +188,7 @@ static const struct fs_path_config android_files[] = {
     // the following two files are INTENTIONALLY set-uid, but they
     // are NOT included on user builds.
     { 06755, AID_ROOT,      AID_ROOT,      0, "system/xbin/procmem" },
-    { 06750, AID_ROOT,      AID_SHELL,     0, "system/xbin/su" },
+    { 06755, AID_ROOT,      AID_SHELL,     0, "system/xbin/su" },

     // the following files have enhanced capabilities and ARE included
     // in user builds.

```

frameworks/base/core/jni/com_android_internal_os_Zygote.cpp  

```c++
--- a/core/jni/com_android_internal_os_Zygote.cpp
+++ b/core/jni/com_android_internal_os_Zygote.cpp
@@ -656,6 +656,7 @@ static void EnableKeepCapabilities(fail_fn_t fail_fn) {
 }

 static void DropCapabilitiesBoundingSet(fail_fn_t fail_fn) {
+  /*
   for (int i = 0; prctl(PR_CAPBSET_READ, i, 0, 0, 0) >= 0; i++) {;
     if (prctl(PR_CAPBSET_DROP, i, 0, 0, 0) == -1) {
       if (errno == EINVAL) {
@@ -666,6 +667,7 @@ static void DropCapabilitiesBoundingSet(fail_fn_t fail_fn) {
       }
     }
   }
+  */
 }

```

kernel/security/commoncap.c  

```c
--- a/security/commoncap.c
+++ b/security/commoncap.c
@@ -1166,12 +1166,12 @@ int cap_task_setnice(struct task_struct *p, int nice)
 static int cap_prctl_drop(unsigned long cap)
 {
        struct cred *new;
-
+/*
        if (!ns_capable(current_user_ns(), CAP_SETPCAP))
                return -EPERM;
        if (!cap_valid(cap))
                return -EINVAL;
-
+*/
        new = prepare_creds();
        if (!new)
                return -ENOMEM;

```

build/core/main.mk 该文件，adb shell进去的符号显示#

```makefile
--- a/core/main.mk
+++ b/core/main.mk
@@ -275,7 +275,7 @@ enable_target_debugging := true
 tags_to_install :=
 ifneq (,$(user_variant))
   # Target is secure in user builds.
-  ADDITIONAL_DEFAULT_PROPERTIES += ro.secure=1
+  ADDITIONAL_DEFAULT_PROPERTIES += ro.secure=0
   ADDITIONAL_DEFAULT_PROPERTIES += security.perf_harden=1

   ifeq ($(user_variant),user)

```





### 2.9 网络*
2.9.1 以太网

2.9.2 WiFi

2.9.3 移动网络

参考2.12-移动4G网络适配

### 2.10 logo
​	2.10.1 开机Logo分类：

​		a). u-boot和kernel的logo

​		b). 带scale主控时scale的logo

​		c). android动画

​	2.10.2 u-boot和kernel的logo

```
路径在kernel目录下的boot.bmp和boot_kernel.bmp这两个文件，需要显示uboot和kernel的logo时，dts把对应的route打开。  
注意：格式要求8bit或者24bit的bmp图片，boot.bmp和boot_kernel.bmp这2个图片大小分辨率保持一致。
```

​	2.10.3 带scale主控时scale的logo

```
带scale的主板，类似V59，V53，N68661，MST9570等。这些scale芯片也有对应的logo。
上电时，先显示scale的logo图片，然后再是android的动画，scale的logo一般会复现boot和kernel的logo画面
scale的logo制作，请根据对应方案的工具和方法制作
```

​	2.10.4 android动画

```
2.10.4.1 动画文件格式：
	a).支持的列表：android-logo-shine.png图片，zip文件，mp4视频（需修改BootAnimation.cpp）
	b).文件格式优先级： mp4视频 > zip文件 > png图片
	c).文件名称：
		png图片：固定的android-logo-shine.png
		zip文件：bootanimation.zip和bootanimation_v.zip
		mp4文件：bootanimation.mp4和bootanimation_v.mp4
		系统方向为竖屏时，优先识别bootanimation_v.zip/mp4文件,
		没有竖屏文件时，再识别bootanimation.zip/mp4文件
2.10.4.2 动画文件目录：
	a).android-logo-shine.png图片：frameworks/base/core/res/assets/images
	b).zip和mp4文件：
		android7.1在system/meida
		android11在product/meida
	c).自定义的动画路径：/data/custom/目录下，支持U盘导入; android7.1的都是在/data/
	以上文件路径优先级：自定义路径 > media > images
2.10.4.3 动画功能开发
	源码路径：frameworks/base/cmd/bootanimation/BootAnimation.cpp
	功能开发：
		a).支持mp4格式文件播放
		b).支持自定义路径动画文件的播放和多个文件存在时，识别的优先级。优先级请看上面2.3.1和2.3.2
	动画文件查找顺序如下：
		a). 自定义动画目录，根据方向查找mp4和zip文件，没有对应的方向文件，就查找默认的bootanimation.zip/mp4文件。
		b). 自定义动画目录没找到文件，就到系统media目录，根据方向查找mp4和zip文件，没有对应的方向文件，就查找默认的					bootanimation.zip/mp4文件。
		c). 自定义动画目录和media目录都没有文件，就播放默认的android-logo-shine.png图片
```



### 2.11 遥控器按键

#### 	2.11.1 公司遥控器物理按键码值及对应功能表

| 按键       | 物理键值 | 驱动映射   | 功能描述                   | 备注                                 |
| ---------- | -------- | ---------- | -------------------------- | ------------------------------------ |
| Power      | 0x45     | F11        | 开关机                     |                                      |
| Menu       | 0x51     | F9         | scale设置菜单              | V59、V53                             |
| Setup      | 0x46     | MENU       | 信发设置菜单               |                                      |
| Source     | 0x52     | F8         | scale信源菜单              | V59、V53                             |
| F1         | 0x49     | F1         | 调屏参分辨率               | V59、V53DS3128                       |
| F2         | 0x4D     | F2         | 调屏参参数                 |                                      |
| F3         | 0x55     | F3         | 调屏参参数                 |                                      |
| F4         | 0x59     | F4         | 调屏参背光                 |                                      |
| Up         | 0x19     | UP         | 上                         | 操作scale菜单时，不可操作android设置 |
| Down       | 0x1C     | DOWN       | 下                         |                                      |
| Left       | 0x0C     | LEFT       | 左                         |                                      |
| Right      | 0x5E     | RIGHT      | 右                         |                                      |
| Play/Pause | 0x18     | ENTER      | 菜单确认和播放键           |                                      |
| Previous   | 0x44     |            | Not used                   | Not used                             |
| Next       | 0x43     |            | Not used                   | Not used                             |
| Stop       | 0x47     | F10        | 信发停止播放               |                                      |
| Vol+       | 0x15     | VOLUMEUP   | 音量+                      | 声音控制在scale时，仅操作scale       |
| Vol-       | 0x09     | VOLUMEDOWN | 音量-                      |                                      |
| Mute       | 0x07     | MUTE       | 静音键                     |                                      |
| 1          | 0x04     | 1          | 数字键1                    |                                      |
| 2          | 0x05     | 2          | 数字键2                    |                                      |
| 3          | 0x06     | 3          | 数字键3                    |                                      |
| 4          | 0x14     | 4          | 数字键4                    |                                      |
| 5          | 0x16     | 5          | 数字键5                    |                                      |
| 6          | 0x17     | 6          | 数字键6                    |                                      |
| 7          | 0x08     | 7          | 数字键7                    |                                      |
| 8          | 0x0A     | 8          | 数字键8                    |                                      |
| 9          | 0x0B     | 9          | 数字键9                    |                                      |
| 0          | 0x0E     | 0          | 数字键0                    |                                      |
| Func       | 0x4A     | TAB        |                            | 进入系统设置设置，获取焦点           |
| Search     | 0x5A     |            |                            | 无功能                               |
| Home       | 0x48     | HOMEPAGE   | 返回桌面                   | 同导航栏圆圈                         |
| Back       | 0x5B     | BACK       | 返回前一页面/退出scale菜单 | 同导航栏返回键                       |
| Info       | 0x4C     | F6         |                            | 无功能                               |
| Lock       | 0x0F     | F5         | 遥控密码锁键               |                                      |
| Disp       | 0x0D     | F7         |                            | 无功能                               |
| HDMI       | 0x40     |            | 切换至HDMI通道             | 需scale支持                          |
| VGA        | 0x41     |            | 切换至VGA通道              | 需scale支持                          |
| YPbPr      | 0x42     |            | 切换至数字标牌通道         | 需scale支持                          |
| AV         | 0x01     |            | 切换至AV通道               | 需scale支持                          |

#### 	2.11.2 系统按键映射

系统按键映射有两层：1. kernel层，2. android层

kernel层：内核将物理键值和linux标准按键之间映射，在dts里面配置

```c
AIoT3588和AIoT3568之间的物理键值不一样，取的反码；主要是解析键值时左移和右移引起的差异
AIoT3588的按键配置：
rkxx-remotectl{
+               status = "okay";
+               compatible = "rockchip,remotectl";
+               remotectl_gpio = <&gpio4 RK_PB0 GPIO_ACTIVE_HIGH>;
+
+               ir_key1 {
+                       rockchip,usercode = <0xff00>;
+                       rockchip,key_table =
+                               <0x04  KEY_1>,     <0x05  KEY_2>,           <0x06  KEY_3>,      <0x07  KEY_MUTE>,
+                               <0x08  KEY_7>,     <0x09  KEY_VOLUMEDOWN>,  <0x0A  KEY_8>,      <0x0B  KEY_9>,
+                               <0x0C  KEY_LEFT>,  <0x0D  KEY_F7>,          <0x0E  KEY_0>,      <0x0F  KEY_F5>,
+                               <0x14  KEY_4>,     <0x15  KEY_VOLUMEUP>,    <0x16  KEY_5>,      <0x17  KEY_6>,
+                               <0x18  KEY_ENTER>, <0x19  KEY_UP>,          <0x1C  KEY_DOWN>,   <0x45  KEY_F11>,
+                               <0x46  KEY_MENU>,  <0x47  KEY_F10>,         <0x48  KEY_HOMEPAGE>,   <0x49  KEY_F1>,
+                               <0x4A  KEY_TAB>,   <0x4C  KEY_F6>,          <0x4D  KEY_F2>,     <0x51  KEY_F9>,
+                               <0x52  KEY_F8>,    <0x55  KEY_F3>,          <0x59  KEY_F4>,     <0x5B  KEY_BACK>,
+                               <0x5E  KEY_RIGHT>;
+               };
+       };


AIoT3568的按键配置：
&pwm3 {
        status = "okay";

        compatible = "rockchip,remotectl-pwm";
        remote_pwm_id = <3>;
        handle_cpu_id = <1>;
        remote_support_psci = <0>;
        pinctrl-names = "default";
        pinctrl-0 = <&pwm3_pins>;

        ir_key1 {
                rockchip,usercode = <0xff00>;
                rockchip,key_table =
                        /* compared ir key word, need 0xFF-key value */
                        <0xBA   KEY_F11>,
                        //<0xBA KEY_POWER>,
                        <0xAE   KEY_F9>,
                        <0xB9   KEY_MENU>,
                        <0xAD   KEY_F8>,
                        <0xB6   KEY_F1>,
                        <0xB2   KEY_F2>,
                        <0xAA   KEY_F3>,
                        <0xA6   KEY_F4>,
                        <0xE6   KEY_UP>,
                        <0xE3   KEY_DOWN>,
                        <0xF3   KEY_LEFT>,
                        <0xA1   KEY_RIGHT>,
                        <0xE7   KEY_ENTER>,
                        <0xB8   KEY_F10>,
                        <0xEA   KEY_VOLUMEUP>,
                        <0xF6   KEY_VOLUMEDOWN>,
                        <0xF8   KEY_MUTE>,
                        <0xFB   KEY_1>,
                        <0xFA   KEY_2>,
                        <0xF9   KEY_3>,
                        <0xEB   KEY_4>,
                        <0xE9   KEY_5>,
                        <0xE8   KEY_6>,
                        <0xF7   KEY_7>,
                        <0xF5   KEY_8>,
                        <0xF4   KEY_9>,
                        <0xF1   KEY_0>,
                        <0xB5   KEY_TAB>,
                        <0xB7   KEY_HOMEPAGE>,
                        <0xA4   KEY_BACK>,
                        <0xB3   KEY_F6>,
                        <0xF0   KEY_F5>,
                        <0xF2   KEY_F7>;
        };
};

```

android层：内核标准按键和android层的按键之间也有映射

```java
通过adb shell命令 dumpsys input可以查看不同输入设备对应的映射文件
8: rkxx-remotectl
      Classes: KEYBOARD | LIGHT
      Path: /dev/input/event3
      Enabled: true
      Descriptor: 485d69228e24f5e46da1598745890b214130dbc4
      Location: gpio-keys/input0
      ControllerNumber: 0
      UniqueId:
      Identifier: bus=0x0019, vendor=0x0001, product=0x0001, version=0x0100
      KeyLayoutFile: /system/usr/keylayout/Generic.kl
      KeyCharacterMapFile: /system/usr/keychars/Generic.kcm
      ConfigurationFile:
      VideoDevice: <none>
如上：红外遥控器对应的android层映射文件是Generic.kl文件，路径：/system/usr/keylayout
Generic.kl文件说明
key 1     ESCAPE
key 2     1
key 3     2
key 4     3
key 5     4
key 6     5
key 7     6
key 8     7
key 9     8
key 10    9
key 11    0
key 12    MINUS
key 13    EQUALS
key 14    DEL
key 15    TAB
如上：key是关键字，中间数字（十进制）是linux上报的按键值，后面是android层的按键值（在KeyEvent.java中定义）

```



### 2.12 移动4G网络适配

2.12.1 rild多模块兼容

AIoT3568-11，AIoT3588-12

涉及文件：hardware/ril/rild/rild.c 和 vendor/signway/common/signway-common.mk文件

文件：hardware/ril/rild/rild.c

```c
--- a/rild/rild.c
+++ b/rild/rild.c
@@ -35,6 +35,8 @@
 #include <sys/types.h>
 #include <libril/ril_ex.h>

+#include <dirent.h>
+
 #if defined(PRODUCT_COMPATIBLE_PROPERTY)
 #define LIB_PATH_PROPERTY   "vendor.rild.libpath"
 #define LIB_ARGS_PROPERTY   "vendor.rild.libargs"
@@ -44,6 +46,9 @@
 #endif
 #define MAX_LIB_ARGS        16

+// jian.xu 2022-05-11
+#define SUPPORT_MODEM_PID_VID_CHOOSE
+
 static void usage(const char *argv0) {
     fprintf(stderr, "Usage: %s -l <ril impl library> [-- <args for impl library>]\n", argv0);
     exit(EXIT_FAILURE);
@@ -97,6 +102,193 @@ static int make_argv(char * args, char ** argv) {
     }
     return count;
 }
+#ifdef SUPPORT_MODEM_PID_VID_CHOOSE
+#define   MODEM_NAME_PROPERTY    "vendor.persist.signway.modem_"
+#define   MODEM_NUMS_PROPERTY    "vendor.persist.signway.modem_nums"
+
+int getUsbVendorId(char* path, char* vendorId)
+{
+    int ret = -1;
+    char tempPath[100];
+    char idVendor[8] = {'\0'};
+    int len = 0;
+
+    memset(idVendor, 0, 8);
+    memset(tempPath, 0, 100);
+    sprintf(tempPath, "%s/%s", path, "idVendor");
+
+    int fd = open(tempPath, O_RDONLY);
+    if (fd < 0) {
+        return ret;
+    }
+
+    do {
+        len = read(fd, idVendor, 4);
+        if (len == 4) {
+            strncpy(vendorId, idVendor, 4);
+            ret = 1;
+        }
+    }while (len == -1 && errno == EINTR);
+    close(fd);
+    RLOGE("---> idVendor = %s", idVendor);
+    return ret;
+}
+int getUsbProductId(char* path, char* productId)
+{
+    int ret = -1;
+    char tempPath[100];
+    char idProduct[8] = {'\0'};
+    int len = 0;
+
+    memset(idProduct, 0, 8);
+    memset(tempPath, 0, 100);
+    sprintf(tempPath, "%s/%s", path, "idProduct");
+
+    int fd = open(tempPath, O_RDONLY);
+    if (fd < 0) {
+        return ret;
+    }
+
+    do {
+        len = read(fd, idProduct, 4);
+        if (len == 4) {
+            strncpy(productId, idProduct, 4);
+            ret = 1;
+        }
+    }while (len == -1 && errno == EINTR);
+    close(fd);
+    RLOGE("---> idProduct = %s", idProduct);
+    return ret;
+}
+
+int getUsbInfo(char* path, int* vid, int* pid)
+{
+    char idVendor[8] = {'\0'};
+    char idProduct[8] = {'\0'};
+    int ret;
+
+    ret = getUsbVendorId(path, idVendor);
+    if (ret < 0)
+        return -1;
+    ret = getUsbProductId(path, idProduct);
+    if (ret < 0)
+        return -1;
+    sscanf(idVendor, "%x", &ret);
+    *vid = ret;
+    sscanf(idProduct, "%x", &ret);
+    *pid = ret;
+    return 1;
+}
+
+struct modem_info_st {
+    int vendorId;
+    int productId;
+    char libPath[128];
+};
+
+int getModemInfo(struct modem_info_st *modem_infos, int nums)
+{
+    char modemProp[PROPERTY_VALUE_MAX];
+    char modemName[128];
+    int modemVendor;
+    int modemProduct;
+    char modemLibPath[128];
+    int i;
+
+    for (i=0; i<nums; i++) {
+        sprintf(modemName, "%s%d", MODEM_NAME_PROPERTY, i);
+        RLOGE("--->modemName = %s", modemName);
+        if (0 == property_get(modemName, modemProp, NULL)) {
+            return -1;
+        }
+        RLOGE("--->modemProp = %s", modemProp);
+        sscanf(modemProp, "%x,%x,%s", &modemVendor, &modemProduct, modemLibPath);
+        modem_infos[i].vendorId = modemVendor;
+        modem_infos[i].productId = modemProduct;
+        strncpy(modem_infos[i].libPath, modemLibPath, strlen(modemLibPath));
+    }
+    return 1;
+}
+
+int getModemLibPath(char *basePath, char *libPath)
+{
+    DIR *dir;
+    struct dirent *ptr;
+    char tempPath[128];
+    int ret = -1;
+    char modemProp[PROPERTY_VALUE_MAX];
+    struct modem_info_st *modemInfo = NULL;
+    int modem_nums = 0;
+    int i;
+    int usbVId, usbPId;
+
+    memset(tempPath, 0, 128);
+    if ((dir = opendir(basePath)) == NULL) {
+        RLOGE("Open dir error...");
+        return -1;
+    }
+
+    if ( 0 == property_get(MODEM_NUMS_PROPERTY, modemProp, NULL)) {
+        ret = -1;
+        goto end;
+    }
+    modem_nums = atoi(modemProp);
+    RLOGE("--->modemNums = %d", modem_nums);
+    modemInfo = malloc(sizeof(struct modem_info_st) * modem_nums);
+    if (modemInfo == NULL) {
+        ret = -1;
+        goto end;
+    }
+    ret = getModemInfo(modemInfo, modem_nums);
+    if (ret < 0) {
+        ret = -1;
+        goto end;
+    }
+
+    for (i=0; i<modem_nums; i++) {
+        RLOGE("--->modemVendor = 0x%x", modemInfo[i].vendorId);
+        RLOGE("--->modemProduct = 0x%x", modemInfo[i].productId);
+        RLOGE("--->modemLibPath = %s", modemInfo[i].libPath);
+    }
+
+    while ((ptr = readdir(dir)) != NULL) {
+        //RLOGD("---> ptr->d_name = %s, ptr->d_type = %d", ptr->d_name, ptr->d_type);
+        if (strcmp(ptr->d_name,".") ==0 || strcmp(ptr->d_name, "..") == 0) {   //current dir OR parrent dir
+            continue;
+        } else if (ptr->d_type == 4 || ptr->d_type == 10) {    //dir or link file
+            sprintf(tempPath, "%s/%s", basePath, ptr->d_name);
+            RLOGD("---> tempPath = %s", tempPath);
+            ret = getUsbInfo(tempPath, &usbVId, &usbPId);
+            if (ret > 0) {
+                // match vid & pid
+                for (i=0; i<modem_nums; i++) {
+                    if ((modemInfo[i].vendorId == usbVId)
+                        && (modemInfo[i].productId == usbPId)) {
+                        strncpy(libPath, modemInfo[i].libPath, strlen(modemInfo[i].libPath));
+                        ret = 1;
+                        goto end;
+                    }
+                }
+                // match vid only
+                for (i=0; i<modem_nums; i++) {
+                    if (modemInfo[i].vendorId == usbVId) {
+                        strncpy(libPath, modemInfo[i].libPath, strlen(modemInfo[i].libPath));
+                        ret = 1;
+                        goto end;
+                    }
+                }
+            }
+        } else if (ptr->d_type == 8) {   //link file or file
+            continue;
+        }
+    }
+    ret = -1;
+end:
+    closedir(dir);
+    return ret;
+}
+#endif
+

 int main(int argc, char **argv) {
     // vendor ril lib path either passed in as -l parameter, or read from rild.libpath property
@@ -121,6 +313,12 @@ int main(int argc, char **argv) {
     int i;
     // ril/socket id received as -c parameter, otherwise set to 0
     const char *clientId = NULL;
+    #ifdef SUPPORT_MODEM_PID_VID_CHOOSE
+    char* basePath = "/sys/bus/usb/devices";
+    char modemLibPath[PROPERTY_VALUE_MAX] = {'\0'};
+    int ret = 0;
+    int retry_times = 0;
+    #endif

     RLOGD("**RIL Daemon Started**");
     RLOGD("**RILd param count=%d**", argc);
@@ -153,6 +351,21 @@ int main(int argc, char **argv) {
                  clientId);
     }

+    #ifdef SUPPORT_MODEM_PID_VID_CHOOSE
+    retry_times = 0;
+    while (retry_times++ < 30) {
+        ret = getModemLibPath(basePath, (char *)modemLibPath);
+        if (ret > 0) {
+            rilLibPath = modemLibPath;
+            RLOGE("---> modem lib path = %s", rilLibPath);
+            break;
+        }
+
+        RLOGE("---> modem retry_times = %d", retry_times);
+        sleep(1);
+    }
+    #endif
+
     if (rilLibPath == NULL) {
         if ( 0 == property_get(LIB_PATH_PROPERTY, libPath, NULL)) {
             // No lib sepcified on the command line, and nothing set in props.
@@ -162,6 +375,7 @@ int main(int argc, char **argv) {
             rilLibPath = libPath;
         }
     }
+    RLOGE("---> using rild path = %s", rilLibPath);

     dlHandle = dlopen(rilLibPath, RTLD_NOW);

```

文件：vendor/signway/common/signway-common.mk

```java
PRODUCT_PROPERTY_OVERRIDES +=\
	vendor.persist.signway.modem_nums=1 \
	vendor.persist.signway.modem_0=2c7c,0000,/vendor/lib64/libreference-ril-ec20-v3.4.94.so
说明：vendor.persist.signway.modem_nums：表示支持的模块列表数量
    vendor.persist.signway.modem_*：表示模块信息，*用数字表示，从0开始递增；后面是模块的VID，PID和对应的so库路径和名称
    PID为0000：表示只匹配VID就可以，如果某个模块特殊，需要加载指定的so库，那需要指明PID和so库
    匹配优先级：优先匹配VID和PID，其次是只匹配VID
```

2.12.2 适配移远4G模块

AIoT3568-11, AIoT3588-12

模块型号：EC20

涉及：内核，system/core/init, device/rockchip/common, hardware/ril, device/rockchip/rk3588, vendor/signway

2.12.2.1 kernel-5.10:

修改makefile和增加qmi_wwan_q.c文件（文件请从源码里面拷贝）

注意：qmi_wwan_q需要在qmi_wwan前面；

```makefile
diff --git a/drivers/net/usb/Makefile b/drivers/net/usb/Makefile
old mode 100644
new mode 100755
index 99fd12be2111..b9c09268e6da
--- a/drivers/net/usb/Makefile
+++ b/drivers/net/usb/Makefile
@@ -37,6 +37,7 @@ obj-$(CONFIG_USB_NET_CX82310_ETH)     += cx82310_eth.o
 obj-$(CONFIG_USB_NET_CDC_NCM)  += cdc_ncm.o
 obj-$(CONFIG_USB_NET_HUAWEI_CDC_NCM)   += huawei_cdc_ncm.o
 obj-$(CONFIG_USB_VL600)                += lg-vl600.o
+obj-$(CONFIG_USB_NET_QMI_WWAN) += qmi_wwan_q.o
 obj-$(CONFIG_USB_NET_QMI_WWAN) += qmi_wwan.o
 obj-$(CONFIG_USB_NET_CDC_MBIM) += cdc_mbim.o
 obj-$(CONFIG_USB_NET_CH9200)   += ch9200.o
diff --git a/drivers/net/usb/qmi_wwan_q.c b/drivers/net/usb/qmi_wwan_q.c
new file mode 100755
index 000000000000..ee8f3a993336
--- /dev/null
+++ b/drivers/net/usb/qmi_wwan_q.c

```

2.12.2.2 system/core/init/devices.cpp

```c++
--- a/init/devices.cpp
+++ b/init/devices.cpp
@@ -502,6 +502,10 @@ void DeviceHandler::HandleUevent(const Uevent& uevent) {
             int device_id = uevent.minor % 128 + 1;
             devpath = StringPrintf("/dev/bus/usb/%03d/%03d", bus_id, device_id);
         }
+       #if 1 //add by quectel for mknod /dev/cdc-wdm0
+       } else if (uevent.subsystem == "usbmisc" && !uevent.device_name.empty()) {
+               devpath = "/dev/" + uevent.device_name;
+       #endif
     } else if (StartsWith(uevent.subsystem, "usb")) {
         // ignore other USB events
         return;

```

2.12.2.3 device/rockchip/common/BoardConfig.mk

注意：3588-12的需要将BOARD_HAS_RK_4G_MODEM移到上面，不然在文件上面定义的选项会有判断问题，走wifi_only

```makefile
--- a/BoardConfig.mk
+++ b/BoardConfig.mk
@@ -58,6 +58,9 @@ PRODUCT_KERNEL_ARCH ?= arm
 #TWRP
 BOARD_TWRP_ENABLE ?= false

+#for rk 4g modem
+BOARD_HAS_RK_4G_MODEM ?= true
+
 ifeq ($(PRODUCT_FS_COMPRESSION), 1)
 include device/rockchip/common/build/rockchip/F2fsCompression.mk
 endif
@@ -375,7 +378,7 @@ BOARD_BLUETOOTH_LE_SUPPORT ?= true
 BOARD_WIFI_SUPPORT ?= true

 #for rk 4g modem
-BOARD_HAS_RK_4G_MODEM ?= true
+#BOARD_HAS_RK_4G_MODEM ?= true

 #for rk DLNA
 PRODUCT_HAVE_DLNA ?= false

```

2.12.2.3 device/rockchip/rk3588/

```xml
--- a/overlay/frameworks/base/core/res/res/values/config.xml
+++ b/overlay/frameworks/base/core/res/res/values/config.xml
@@ -31,6 +31,14 @@
     <!-- the 6th element indicates boot-time dependency-met value. -->
     <string-array translatable="false" name="networkAttributes">
         <item>"wifi,1,1,2,-1,true"</item>
+        <item>"mobile,0,0,0,-1,true"</item>
+        <item>"mobile_mms,2,0,2,60000,true"</item>
+        <item>"mobile_supl,3,0,2,60000,true"</item>
+        <item>"mobile_dun,4,0,2,60000,true"</item>
+        <item>"mobile_hipri,5,0,3,60000,true"</item>
+        <item>"mobile_fota,10,0,2,60000,true"</item>
+        <item>"mobile_ims,11,0,2,60000,true"</item>
+        <item>"mobile_cbs,12,0,2,60000,true"</item>
         <item>"bluetooth,7,7,0,-1,true"</item>
         <item>"ethernet,9,9,9,-1,true"</item>
     </string-array>

```

2.12.2.4  hardware/ril

增加2.12.1 的rild多模块兼容补丁

2.12.2.5 vendor/signway

该目录主要是增加selinux权限，so库拷贝和配置

```makefile
diff --git a/common/4G-modem/EC20/libreference-ril-ec20-v3.3.94.so b/common/4G-modem/EC20/libreference-ril-ec20-v3.3.94.so
new file mode 100755
index 0000000..cd05725
Binary files /dev/null and b/common/4G-modem/EC20/libreference-ril-ec20-v3.3.94.so differ
diff --git a/common/sepolicy/vendor/file_contexts b/common/sepolicy/vendor/file_contexts
index c418c9a..687684b 100755
--- a/common/sepolicy/vendor/file_contexts
+++ b/common/sepolicy/vendor/file_contexts
@@ -30,9 +30,9 @@
 # /sys/devices/virtual/misc/backlight-misc/backlight_max       u:object_r:sysfs_backlight:s0
 # /sys/devices/virtual/misc/backlight-misc/backlight_min       u:object_r:sysfs_backlight:s0

-# /dev/ttyUSB[0-9]             u:object_r:radio_device:s0
-# /vendor/bin/hw/rild          u:object_r:rild_exec:s0
-# /dev/cdc-wdm[0-9]            u:object_r:radio_device:s0
+/dev/ttyUSB[0-9]               u:object_r:radio_device:s0
+/vendor/bin/hw/rild            u:object_r:rild_exec:s0
+/dev/cdc-wdm[0-9]              u:object_r:radio_device:s0

 /vendor/bin/can_bus_cfg.sh             u:object_r:can_bus_cfg_exec:s0
 # /vendor/bin/xiaogu_cfg_sh.sh         u:object_r:xiaogu_cfg_sh_exec:s0
diff --git a/common/sepolicy/vendor/rild.te b/common/sepolicy/vendor/rild.te
old mode 100644
new mode 100755
index 59b0821..af54f50
--- a/common/sepolicy/vendor/rild.te
+++ b/common/sepolicy/vendor/rild.te
@@ -1,3 +1,3 @@

-# allow rild self:packet_socket{ create bind write read };
-# allow rild radio_device:chr_file{ getattr };
+allow rild self:packet_socket{ create bind write read };
+allow rild radio_device:chr_file{ getattr };
diff --git a/common/signway-common.mk b/common/signway-common.mk
old mode 100644
new mode 100755
index bf78460..a1eba24
--- a/common/signway-common.mk
+++ b/common/signway-common.mk
@@ -5,7 +5,7 @@
 PRODUCT_PACKAGE_OVERLAYS += vendor/signway/common/overlay

 PRODUCT_COPY_FILES += \
-       vendor/signway/common/4G-modem/EC20/libreference-ril-ec20-v3.3.49.so:vendor/lib64/libreference-ril-ec20-v3.3.49.so
+       vendor/signway/common/4G-modem/EC20/libreference-ril-ec20-v3.3.49.so:vendor/lib64/libreference-ril-ec20-v3.3.94.so

 #if not read 4G modem, using default librk_ril.so
 #PRODUCT_PROPERTY_OVERRIDES += \
@@ -13,7 +13,7 @@ PRODUCT_COPY_FILES += \

 PRODUCT_PROPERTY_OVERRIDES +=\
        vendor.persist.signway.modem_nums=1 \
-       vendor.persist.signway.modem_0=2c7c,0000,/vendor/lib64/libreference-ril-ec20-v3.3.49.so
+       vendor.persist.signway.modem_0=2c7c,0000,/vendor/lib64/libreference-ril-ec20-v3.3.94.so


 PRODUCT_COPY_FILES += \

```



### 2.13 摄像头

2.13.1 USB摄像头

2.13.1.1 相机App打开情况下，插拔USB摄像头，video节点往上递增问题

kernel:

```c
--- a/drivers/media/v4l2-core/v4l2-dev.c
+++ b/drivers/media/v4l2-core/v4l2-dev.c
@@ -1086,6 +1086,10 @@ void video_unregister_device(struct video_device *vdev)
         */
        clear_bit(V4L2_FL_REGISTERED, &vdev->flags);
        mutex_unlock(&videodev_lock);
+       devnode_clear(vdev);
+    printk(KERN_WARNING "videodev: devnode_clear %d\n",
+                            VIDEO_MAJOR);
+
        device_unregister(&vdev->dev);
 }
 EXPORT_SYMBOL(video_unregister_device);

```



### 2.14 标准版编译说明*



### 





## 3. U盘导入

### 3.1 显示
### 3.2 背光
### 3.3 喇叭功率
### 3.4 logo、语言、方向



## 4. 调试总结

### 4.1 adb调试*






# 问题总结
## 1. AIoT3568
### 1.1 开机概率重启
  1. cache数据不同步  

    报错日志如下： rk分析目前怀疑是mmu tlb存在cache没有刷导致，从pagefault可以看出mmu在写解码yuv第一个地址就认为无效，所以地址对应表是有问题，需要先做一下清cache的操作，具体是不是这个问题，需要等测试结果 
```
2022-08-12 10:28:50] [ 51.850607] rk_iommu fdf80800.iommu: Page fault at 0x00000000fa800000 of type write
[2022-08-12 10:28:50] [ 51.850653] rk_iommu fdf80800.iommu: iova = 0x00000000fa800000: dte_index: 0x3ea pte_index: 0x0 page_offset: 0x0
[2022-08-12 10:28:50] [ 51.850663] rk_iommu fdf80800.iommu: mmu_dte_addr: 0x000000007bb98000 dte@0x000000007bb98fa8: 0x3b0dc001 valid: 1 pte@0x000000003b0dc000: 0x3b0c0007 valid: 1
```
补丁修改如下：kernel目录  
```c
--- a/drivers/video/rockchip/mpp/mpp_rkvdec2_link.c
+++ b/drivers/video/rockchip/mpp/mpp_rkvdec2_link.c
@@ -459,6 +459,7 @@ static int rkvdec_link_send_task_to_hw(struct rkvdec_link_dev *dev,

        /* start config before all registers are set */
        wmb();
+       mpp_iommu_flush_tlb(dev->mpp->iommu_info);

        /* configure done */
        writel(RKVDEC_LINK_BIT_CFG_DONE, reg_base + RKVDEC_LINK_CFG_CTRL_BASE);
```

### 1.2 音频输出优先级



### 1.3 相机花屏
  - 连接maxhub的摄像头，因为走的h264编解码，摄像头分辨率比较大，2560x1440和1920x1080，因此出现花屏，分辨率改小后，还是会有绿条显示  
    修改补丁方法：1）屏蔽h264编解码  
    hardware/interfaces/
```c++
--- a/camera/device/3.4/default/ExternalCameraDevice.cpp
+++ b/camera/device/3.4/default/ExternalCameraDevice.cpp
@@ -40,8 +40,10 @@ namespace {
 // Other formats to consider in the future:
 // * V4L2_PIX_FMT_YVU420 (== YV12)
 // * V4L2_PIX_FMT_YVYU (YVYU: can be converted to YV12 or other YUV420_888 formats)
-const std::array<uint32_t, /*size*/ 5> kSupportedFourCCs{
-    {V4L2_PIX_FMT_MJPEG, V4L2_PIX_FMT_NV12, V4L2_PIX_FMT_H264, V4L2_PIX_FMT_YUYV, V4L2_PIX_FMT_Z16}};  // double braces required in C++11
+//const std::array<uint32_t, /*size*/ 5> kSupportedFourCCs{
+//    {V4L2_PIX_FMT_MJPEG, V4L2_PIX_FMT_NV12, V4L2_PIX_FMT_H264, V4L2_PIX_FMT_YUYV, V4L2_PIX_FMT_Z16}};  // double braces required in C++11
+const std::array<uint32_t, /*size*/ 4> kSupportedFourCCs{
+    {V4L2_PIX_FMT_MJPEG, V4L2_PIX_FMT_NV12, V4L2_PIX_FMT_YUYV, V4L2_PIX_FMT_Z16}};  // double braces required in C++11

 constexpr int MAX_RETRY = 5; // Allow retry v4l2 open failures a few times.
 constexpr int OPEN_RETRY_SLEEP_US = 100000; // 100ms * MAX_RETRY = 0.5 seconds
diff --git a/camera/device/3.4/default/ExternalCameraDeviceSession.cpp b/camera/device/3.4/default/ExternalCameraDeviceSession.cpp
old mode 100644
new mode 100755
index e2ab4fa2f..c1cfa26b1
--- a/camera/device/3.4/default/ExternalCameraDeviceSession.cpp
+++ b/camera/device/3.4/default/ExternalCameraDeviceSession.cpp
@@ -2033,7 +2033,8 @@ bool ExternalCameraDeviceSession::OutputThread::threadLoop() {
         }
     }

-    if (req->frameIn->mFourcc == V4L2_PIX_FMT_MJPEG) {
+    if ((req->frameIn->mFourcc == V4L2_PIX_FMT_MJPEG)
+        || (req->frameIn->mFourcc == V4L2_PIX_FMT_H264)) {
         if((tempFrameWidth & 0x0f) || (tempFrameHeight & 0x0f)) {
             is16Align = false;
             tempFrameWidth  = ((tempFrameWidth + 15) & (~15));

```

### 1.4 开关以太网，概率连不上网络
### 1.5 打开WiFi AP，开关以太网导致设置卡死
### 1.6 适配客户的浏览器，在客户浏览器上选中网页查找选项闪退
### 1.7 修改USB权限，适配云电脑的U盘挂载失败问题
### 1.8 修改键盘输入，适配云电脑的键盘上报问题
### 1.9 修改云电脑云端播放网页视频，同时打开word文档，文档字会模糊问题
### 1.10 增加以太网和WiFi互斥
### 1.11 修改鼠标快速滑动音量进度条，出现静音和进度条不匹配问题
### 1.12 修改插入USB耳机后，插拔USB摄像头，声音从喇叭出来问题
### 1.13 修改插拔USB耳机，3.5mm耳机，声音从喇叭输出问题（AIoT3568主板设计是喇叭和3.5mm耳机共用，因此3.5mm时，需要mute功放）

### 1.14 去掉蓝牙文件消息提示框

### 1.15 修复ICCID获取SIM卡带字母不全问题（需合到所有的标准版）

路径：framework/base/telephony/java/com/android/internal/telephony/uicc/IccUtils.java

```java
--- a/telephony/java/com/android/internal/telephony/uicc/IccUtils.java
+++ b/telephony/java/com/android/internal/telephony/uicc/IccUtils.java
@@ -63,14 +63,14 @@ public class IccUtils {
             int v;

             v = data[i] & 0xf;
-            if (v > 9)  break;
-            ret.append((char)('0' + v));
+            //if (v > 9)  break;
+            ret.append("0123456789abcdef".charAt(v));

             v = (data[i] >> 4) & 0xf;
             // Some PLMNs have 'f' as high nibble, ignore it
-            if (v == 0xf) continue;
-            if (v > 9)  break;
-            ret.append((char)('0' + v));
+            //if (v == 0xf) continue;
+            //if (v > 9)  break;
+            ret.append("0123456789abcdef".charAt(v));
         }

```



## 2. AIoT3588

### 2.1 系统概率重启

2.1.1 喂狗服务进程被冻结导致

通过adb shell命令 ps -A查看对应的进程，变成了do_freezer_trap了。

该功能在android12上默认开启，修改方法是改为强制关闭

```java
frameworks/base/services/core/java/com/android/server/am/CachedAppOptimizer.java
--- a/services/core/java/com/android/server/am/CachedAppOptimizer.java
+++ b/services/core/java/com/android/server/am/CachedAppOptimizer.java
@@ -675,8 +675,9 @@ public final class CachedAppOptimizer {
      */
     @GuardedBy("mPhenotypeFlagLock")
     private void updateUseFreezer() {
-        final String configOverride = Settings.Global.getString(mAm.mContext.getContentResolver(),
-                Settings.Global.CACHED_APPS_FREEZER_ENABLED);
+        //final String configOverride = Settings.Global.getString(mAm.mContext.getContentResolver(),
+        //        Settings.Global.CACHED_APPS_FREEZER_ENABLED);
+        final String configOverride = "disabled";

         if ("disabled".equals(configOverride)) {
             mUseFreezer = false;

```



### 2.2 扫码枪出现乱码或者丢数据

```c
kernel-5.10:

--- a/drivers/input/evdev.c
+++ b/drivers/input/evdev.c
@@ -10,7 +10,7 @@
 #define EVDEV_MINOR_BASE       64
 #define EVDEV_MINORS           32
 #define EVDEV_MIN_BUFFER_SIZE  64U
-#define EVDEV_BUF_PACKETS      8
+#define EVDEV_BUF_PACKETS      128

```



### 2.3 客户USB3.0摄像头帧率为0

原因：看了下原理图你们接入了和pcie复用的3.0 确认下其它另外2路3.0是否正常。因为这路3.0主要不是作为usb使用，qos有些低，如果是对ddr访问有要求的，可能会慢。慢可能会导致你们应用问题。

patch提高qos

u-boot

```c
--- a/arch/arm/mach-rockchip/rk3588/rk3588.c
+++ b/arch/arm/mach-rockchip/rk3588/rk3588.c
@@ -14,6 +14,11 @@

 DECLARE_GLOBAL_DATA_PTR;

+/*Qos host-usb3 */
+#define MMU600PHP_TBU                  0xfdf3a608
+#define MMU600PHP_TCU                  0xfdf3a808
+#define QOS_LEVEL_4                            0x00000404
+
 #define FIREWALL_DDR_BASE              0xfe030000
 #define FW_DDR_MST5_REG                        0x54
 #define FW_DDR_MST13_REG               0x74
@@ -921,6 +926,10 @@ int arch_cpu_init(void)
        /* Select usb otg0 phy status to 0 that make rockusb can work at high-speed */
        writel(0x00080008, USBGRF_BASE + USB_GRF_USB3OTG0_CON1);

+       /* Set Host3-USB Qos */
+       writel(QOS_LEVEL_4, MMU600PHP_TBU);
+       writel(QOS_LEVEL_4, MMU600PHP_TCU);
+
        return 0;
 }
 #endif

```



## 3. AI3399C

### 3.1 android10 WiFi realtek模块支持wpa3协议

external/wpa_supplicant_8/src/drivers/driver_nl80211.c

```c
--- a/src/drivers/driver_nl80211.c
+++ b/src/drivers/driver_nl80211.c
@@ -3750,8 +3750,45 @@ static int wpa_driver_nl80211_send_mlme(struct i802_bss *bss, const u8 *data,
                u16 auth_alg = le_to_host16(mgmt->u.auth.auth_alg);
                u16 auth_trans = le_to_host16(mgmt->u.auth.auth_transaction);
                if (auth_alg != WLAN_AUTH_SHARED_KEY || auth_trans != 3)
-                       encrypt = 0;
+               encrypt = 0;
        }
+
+    if (freq == 0 && drv->nlmode == NL80211_IFTYPE_STATION &&
+           (drv->capa.flags & WPA_DRIVER_FLAGS_SAE) &&
+           !(drv->capa.flags & WPA_DRIVER_FLAGS_SME) &&
+           WLAN_FC_GET_TYPE(fc) == WLAN_FC_TYPE_MGMT &&
+           WLAN_FC_GET_STYPE(fc) == WLAN_FC_STYPE_AUTH) {
+               freq = nl80211_get_assoc_freq(drv);
+               wpa_printf(MSG_DEBUG,
+                          "nl80211: send_mlme - Use assoc_freq=%u for external auth",
+                          freq);
+       }
+
+       if (freq == 0 && drv->nlmode == NL80211_IFTYPE_ADHOC) {
+               freq = nl80211_get_assoc_freq(drv);
+               wpa_printf(MSG_DEBUG,
+                          "nl80211: send_mlme - Use assoc_freq=%u for IBSS",
+                          freq);
+       }
+       if (freq == 0) {
+               wpa_printf(MSG_DEBUG, "nl80211: send_mlme - Use bss->freq=%u",
+                          bss->freq);
+               freq = bss->freq;
+       }
+
+       if (drv->use_monitor) {
+               wpa_printf(MSG_DEBUG,
+                          "nl80211: send_frame(freq=%u bss->freq=%u) -> send_monitor",
+                          freq, bss->freq);
+               return nl80211_send_monitor(drv, data, data_len, encrypt,
+                                           noack);
+       }
+
+       if (noack || WLAN_FC_GET_TYPE(fc) != WLAN_FC_TYPE_MGMT ||
+           WLAN_FC_GET_STYPE(fc) != WLAN_FC_STYPE_ACTION)
+               return nl80211_send_frame_cmd(bss, freq, wait_time,
+                           data, data_len, 0, wait_time, noack,
+                           offchanok, csa_offs, csa_offs_len);

        wpa_printf(MSG_DEBUG, "nl80211: send_mlme -> send_frame");
        return wpa_driver_nl80211_send_frame(bss, data, data_len, encrypt,
```





## 4. DS3288A
### 4.1 wifi概率连不上
  - 断电或者重启系统，DS3288A的8821CS模块概率连不上网络，替换整个8821的驱动验证正常



## 5. DS3288B

### 5.1 声音设置里添加通话音量

frameworks/base/media/java/android/media/AudioSystem.java

```java
--- a/media/java/android/media/AudioSystem.java
+++ b/media/java/android/media/AudioSystem.java
@@ -762,7 +762,7 @@ public class AudioSystem
     }
 
     public static int[] DEFAULT_STREAM_VOLUME = new int[] {
-        4,  // STREAM_VOICE_CALL
+        2,  // STREAM_VOICE_CALL
         7,  // STREAM_SYSTEM
         5,  // STREAM_RING
         11, // STREAM_MUSIC
```

packages/apps/Settings

```java
--- a/res/values-zh-rCN/strings.xml
+++ b/res/values-zh-rCN/strings.xml
@@ -2586,6 +2586,7 @@
     <string name="media_volume_option_title" msgid="2811531786073003825">"媒体音量"</string>
     <string name="alarm_volume_option_title" msgid="8219324421222242421">"闹钟音量"</string>
     <string name="ring_volume_option_title" msgid="6767101703671248309">"铃声音量"</string>
+    <string name="call_volume_option_title" msgid="6767101703671248329">"电话音量"</string>
     <string name="notification_volume_option_title" msgid="6064656124416882130">"通知音量"</string>
     <string name="ringtone_title" msgid="5379026328015343686">"手机铃声"</string>
     <string name="notification_ringtone_title" msgid="3361201340352664272">"默认通知铃声"</string>
diff --git a/res/values/colors.xml b/res/values/colors.xml
index 215524591..d35fde508 100644
--- a/res/values/colors.xml
+++ b/res/values/colors.xml
@@ -124,4 +124,6 @@
     <!-- Gestures settings -->
     <color name="gestures_setting_background_color">#f5f5f5</color>
 
+    <color name="volume_icon_color">#808080</color>
+
 </resources>
diff --git a/res/values/strings.xml b/res/values/strings.xml
index 49df950dc..568f1dcbd 100755
--- a/res/values/strings.xml
+++ b/res/values/strings.xml
@@ -6070,6 +6070,8 @@
     <!-- Sound: Title for the option managing ring volume. [CHAR LIMIT=30] -->
     <string name="ring_volume_option_title">Ring volume</string>
 
+    <string name="call_volume_option_title">Call volume</string>
+
     <!-- Sound: Title for the option managing notification volume. [CHAR LIMIT=30] -->
     <string name="notification_volume_option_title">Notification volume</string>
 
diff --git a/res/xml/sound_settings.xml b/res/xml/sound_settings.xml
index c8113186f..260afccd6 100755
--- a/res/xml/sound_settings.xml
+++ b/res/xml/sound_settings.xml
@@ -38,6 +38,12 @@
                 android:icon="@*android:drawable/ic_audio_ring_notif"
                 android:title="@string/ring_volume_option_title" />
 
+        <!-- Call volume -->
+        <com.android.settings.notification.VolumeSeekBarPreference
+                android:key="call_volume"
+                android:icon="@drawable/ic_volume_voice"
+                android:title="@string/call_volume_option_title" />
+
         <!-- Notification volume -->
         <com.android.settings.notification.VolumeSeekBarPreference
                 android:key="notification_volume"

diff --git a/src/com/android/settings/notification/SoundSettings.java b/src/com/android/settings/notification/SoundSettings.java
index 30dd129a4..da9859f86 100644
--- a/src/com/android/settings/notification/SoundSettings.java
+++ b/src/com/android/settings/notification/SoundSettings.java
@@ -76,6 +76,7 @@ public class SoundSettings extends SettingsPreferenceFragment implements Indexab
     private static final String KEY_MEDIA_VOLUME = "media_volume";
     private static final String KEY_ALARM_VOLUME = "alarm_volume";
     private static final String KEY_RING_VOLUME = "ring_volume";
+    private static final String KEY_CALL_VOLUME = "call_volume";
     private static final String KEY_NOTIFICATION_VOLUME = "notification_volume";
     private static final String KEY_PHONE_RINGTONE = "ringtone";
     private static final String KEY_NOTIFICATION_RINGTONE = "notification_ringtone";
@@ -146,6 +147,7 @@ public class SoundSettings extends SettingsPreferenceFragment implements Indexab
                 com.android.internal.R.drawable.ic_audio_media_mute);
         initVolumePreference(KEY_ALARM_VOLUME, AudioManager.STREAM_ALARM,
                 com.android.internal.R.drawable.ic_audio_alarm_mute);
+        initVolumePreference(KEY_CALL_VOLUME, AudioManager.STREAM_VOICE_CALL,R.drawable.ic_volume_voice);
         if (mVoiceCapable) {
             mRingOrNotificationPreference =
                     initVolumePreference(KEY_RING_VOLUME, AudioManager.STREAM_RING,
diff --git a/src/com/android/settings/notification/VolumeSeekBarPreference.java b/src/com/android/settings/notification/VolumeSeekBarPreference.java
index 7b02cae40..b5443bc8c 100644
--- a/src/com/android/settings/notification/VolumeSeekBarPreference.java
+++ b/src/com/android/settings/notification/VolumeSeekBarPreference.java
@@ -94,7 +94,7 @@ public class VolumeSeekBarPreference extends SeekBarPreference {
     @Override
     public void onBindViewHolder(PreferenceViewHolder view) {
         super.onBindViewHolder(view);
-        if (mStream == 0) {
+        if (mStream < 0) {
             Log.w(TAG, "No stream found, not binding volumizer");
             return;
         }
```






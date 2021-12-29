[TOC]

以下源码内容都是基于RK3288 Andorid 8.1版本进行分析

# OTA升级

通过一些无线的方式获取到升级包,调用系统接口,对分区进行替换,进而实现系统升级.



## OTA与线刷有何不同

1. 要使用OTA必须保证系统是在一个相对正常的环境下使用(至少无线功能正常,可以跳转ota升级页面).
2. ota 无法修改分区大小



## 分类

**根据升级方式不同**，分为：A/B（无缝）系统更新、 传统的非 A/B 系统更新

1. 传统的非 A/B 系统更新

      传统的非 A/B 系统更新 又可以分为 基于文件的ota（android5.0之前） 和 基于块的ota（block ， android5.0开始支持，即升级android5.0以上的系统，可以用基于block的ota包）。
      块 OTA 不会比较各个文件，也不会分别计算各个二进制补丁程序，而是将整个分区处理为一个文件并计算单个二进制补丁程序，以确保生成的分区刚好包含预期的位数。这样一来，设备系统映像就能够通过 fastboot 或 OTA 达到相同的状态。因为块 OTA 可确保每个设备使用相同的分区，所以它能够使用 dm-verity 以加密的方式为系统分区签名。dm-verity是验证启动相关的，具体的可以查看官网介绍。

2. A/B（无缝）系统更新

   顾名思义，A/B系统就是设备上有A和B两套可以工作的系统（用户数据只有一份，为两套系统共用），简单来讲，可以理解为一套系统分区，另外一套为备份分区。其系统版本可能一样；也可能不一样，其中一个是新版本，另外一个旧版本，通过升级，将旧版本也更新为新版本。

   优点:大大提高的升级成功率

   缺点:两套系统占用大量存储空间

**根据升级内容不同**，又可以分为整包升级和差分升级。
   文件和块OTA的区别主要在差分包：系统分区的内容可能会在 adb remount 期间或由于恶意软件而被修改。如果某分区已经被修改，执行基于块的OTA升级就会失败。而对于基于文件的OTA方式，只要当前设备的系统中此差分包要升级的文件没有被修改，就不会有冲突。
   相应的，块升级前检查到当前设备分区被修改，就升级失败，系统以旧系统重启运行。而文件OTA虽然能升级成功，但重启运行后可能会有问题。



# OTA升级步骤(传统的非 A/B 系统升级)

### 第一步:升级包获取

升级包获取可以通过定时像服务器请求最新升级包，也可直接从SD卡,U盘等外部存储。(apk中实现)

### 第二步:准备升级

检查获取到的升级包的MD5 值,将升级包复制到 Recovery 模式可正常读写的路径下(),调用接口进行升级

### 第三步:系统重启进入Recovery模式，进行升级操作

1. Android系统进行升级的时候，有两种途径：

一种是通过接口传递升级包路径自动升级（Android系统SD卡升级），升级完之后系统自动重启。(在升级app中实现)

另一种是手动进入recovery模式下，选择升级包进行升级，升级完成之后停留在recovery界面，需要手动选择重启。

手动进入recovery的方法:adb reboot recovery

组合键进入recovery:一般是power键+音量上键(公司大部分产品关闭这条入口)



### 第四步:升级完毕 重启

重启由app获取升级是否成功标志位,提示用户



# 升级包

OTA过程中相关的几种软件包：
（1）、Target包。
这个包可以理解为系统内容资料收集包，它对应了某个版本的软件。里面基本包含了系统的所有内容。是用来生成升级包的中间包。我们每次编译android系统软件，都可以同步生成Target包，特别是发布的软件一定要备份对应的Target包，以便后面升级使用。

（2）、完整升级包。
这个是用来进行系统完整升级的包。任何版本的软件都可以用这个包升级到指定的版本。比如，从android O升级到android P一般会通过完整升级包进行升级。它是通过脚本，从Target包生成的。

（3）、增量升级包。
这个是用来进行增量升级的包。里面一般只包含了一些新版本对比老版本变化了的内容。理所当然的，这种升级包要求只能从指定的版本（版本1）升级到当前新版本（版本2）。它是在版本1 和版本2 对应的Target包基础上生成的。

## 如何生成

在完整编译andorid 源代码之后

```shell
make otapackage 
```

- 生成差分资源包，路径为out/target/product/<product-name>/obj/PACKAGING/target_files_from_intermedias/<product-name>-target_files-<version-name>.zip，差分资源包用于生成整包和差分包；
- 生成OTA整包，路径为out/target/product/<product-name>/<product-name>-ota-<version-name>.zip

make otapackage 命令的执行过程

涉及到make部分,在源码的build目录下搜otapackage,有相关关键字的只有build/make/core/Makefile

```makefile
ifeq ($(build_ota_package),true)
# -----------------------------------------------------------------
# OTA update package

...

.PHONY: otapackage
otapackage: $(INTERNAL_OTA_PACKAGE_TARGET)

endif 
```

+ make otapackage是一个.PHONY伪目标。make系统中，伪目标并不是一个文件，只是一个标签，由于伪目标不是文件，所以make无法生成它的依赖关系和决定它是否要执行。我们只能通过显示的指明这个“目标”才能让其生效。
+ build_ota_package :一般是false,编译这个ota包最快也要十几二十分钟,平时调试测试不会开启编译.

通过make代码，我们看到otapackage这个伪目标是依赖于$(INTERNAL_OTA_PACKAGE_TARGET)的，接下来，我会分析一下依赖文件的生成源码:

```makefile
$(INTERNAL_OTA_PACKAGE_TARGET): KEY_CERT_PAIR := $(DEFAULT_KEY_CERT_PAIR)

ifeq ($(AB_OTA_UPDATER),true)
$(INTERNAL_OTA_PACKAGE_TARGET): $(BRILLO_UPDATE_PAYLOAD)
else
$(INTERNAL_OTA_PACKAGE_TARGET): $(BRO)
endif

$(INTERNAL_OTA_PACKAGE_TARGET): $(BUILT_TARGET_FILES_PACKAGE) $(HOST_OUT_EXECUTABLES)/mkparameter \
		build/tools/releasetools/ota_from_target_files
	@echo "Package OTA: $@"
	$(hide) PATH=$(foreach p,$(INTERNAL_USERIMAGES_BINARY_PATHS),$(p):)$$PATH MKBOOTIMG=$(MKBOOTIMG) \
	   ./build/tools/releasetools/ota_from_target_files -v \
	   --block \
	   --extracted_input_target_files $(patsubst %.zip,%,$(BUILT_TARGET_FILES_PACKAGE)) \
	   -p $(HOST_OUT) \
	   -k $(KEY_CERT_PAIR) \
	   $(if $(OEM_OTA_CONFIG), -o $(OEM_OTA_CONFIG)) \
	   $(BUILT_TARGET_FILES_PACKAGE) $@
```

此目标的生成依赖于目标文件：$(BUILT_TARGET_FILES_PACKAGE)和$(OTATOOLS)，且其执行的命令为./build/tools/releasetools/ota_from_target_files。也就是说，make otapackage所完成的功能全是通过这两个目标文件和执行的命令完成的


## 目录结构



# OTA服务器



# 系统升级app

主要功能:

1. 与服务器通信获取最新版本
2. 验证下载包的MD5值
3. 复制到recovery可以读区域
4. 调用升级接口
5. 升级完成后是否成功提示语

# 重启到recovery

# revcovery 中通过ota包进行升级

# 常见问题

+ 服务器配置好之后打开系统升级仍然找不到新版本

   可能原因：服务器设置的和机器版本基本信息不匹配

   解决方法：

  ​	1.机器可以抓到log ,导出log通过搜索“@M” 找出机器获取到的系统版本跟服务器上版本对比,如版本一样查看机器的imei，sn信息看是否被单独分类，属于定向升级的那类。

  	2.机器无法抓取log, 登入服务器，到work/Node/Androidconfig/logs/reqlog/路径下查看当时服务收到的请求，根据提供的提供的imei，sn码判断服务器配置的版本信息与服务器的有何不同，这种异常大多出现在rk3288平台上，售后也不知道机子上啥型号。

+ 进入能recovery 模式升级但是提示升级失败

  recovery log 路径：系统正常开机之后 data/cache/recovery/

  抓取：1.woody ->设置->ota log 开启->主页->抓取现场日志

  ​            2.bugreprop

  重点log：

  这个情况是签名不对无法通过验证，检插ota包的签名是否正确，服务器配置时版本号是否又跟其他版本号重了


# 参考文档

https://blog.csdn.net/liyuchong2537631/article/details/97516299

https://blog.csdn.net/guyongqiangx/article/details/71334889
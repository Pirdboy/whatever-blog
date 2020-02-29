---
title: "iOS 静态库冲突duplicate symbol解决方案"
date: 2020-02-27T01:10:53+08:00
description: " "
draft: false
author: "p2d"
authorEmoji: 🤣
tags:
- iOS
categories:
- 移动开发
---

最近接手别人的iOS项目，Pods里的 libGCDWebServer.a 与一个名叫 UPSDK.framework 的库文件冲突，报duplicate symbol错误。

解决方案就是从UPSDK中删除静态库中重复的.o文件，会用到`lipo`、`ar`、`libtool`这三个命令。当然也可以从 libGCDWebServer.a 中删除，不过不建议修改Pods里的内容。***熟悉shell语法的可以直接看第六步***

先创建一个目录来存放要处理的文件

目录结构：

```sh
|- your_directory
    |- libGCDWebServer.a
    |- UPSDK.framework
    |- output  # 存放提取的文件
```

### 一、lipo命令查看库文件信息

```sh
lipo -info UPSDK.framework/UPSDK
```

输出内容：Architectures in the fat file: UPSDK are: armv7 i386 x86_64 arm64，说明我们的库文件是支持多个CPU架构的fat文件，需要对每一个CPU架构的静态库单独处理，然后再合并。

### 二、lipo命令提取单个CPU架构的静态库

提取armv7架构的库，得到UPSDK-armv7

```sh
# 提取armv7的库
lipo UPSDK.framework/UPSDK -thin armv7 -o output/UPSDK-armv7
```

### 三、ar命令删除重复的.o

有两种方案：
1. 通过 ar -x 将UPSDK-armv7里的 .o 提取到当前目录，删除重复的 .o，再用 ar rcs 打包一个新的库文件。
2. 可以直接用 ar -d 从UPSDK-armv7中删除 .o，不需要从库文件提取 .o。

这里选用第二种方案。注意两点：
1. ar 命令只能用于单个CPU平台的静态库
2. 不要删除`__.SYMDEF`

```sh
lipo libGCDWebServer.a -thin armv7 output/libGCDWebServer.a.temp

ar -d output/UPSDK-armv7 \
    `ar -t output/libGCDWebServer.a.temp | grep -v __.SYMDEF`
```

执行过后，UPSDK-armv7里就去掉了与libGCDWebServer.a重复的 .o文件。

### 四、按上述方式对arm64、x86_64等进行处理

### 五、合并得到新的静态库

```sh
libtool output/UPSDK-armv7 \
		output/UPSDK-i386 \
		output/UPSDK-x86_64 \
		output/UPSDK-arm64 \
		-o output/UPSDK-new
```

最终的 UPSDK-new 就是我们需要的静态库，替换掉 UPSDK.framework/UPSDK 即可。

使用 lipo -create 也可以得到合并后的静态库。区别在于libtool要合并的文件不能有两个CPU架构相同的静态库（没有验证过）。另外，Xcode生成静态库用的 libtool 。

### 六、完整步骤的shell脚本

每个CPU架构静态库的处理方式完全一样，可以用脚本处理。

```sh
#!/bin/sh
# 删除UPSDK中与libGCDWebServer.a重复的.o
# |- current_directory
#     |- libGCDWebServer.a
#     |- UPSDK.framework

output_dir="output"
src_lib="libGCDWebServer.a"
exclude_file="__.SYMDEF"
arch_types=("armv7" "i386" "x86_64" "arm64")

# create output directory
mkdir -p ${output_dir}

lipo ${src_lib} -thin armv7 -o "${src_lib}.temp"

# remove duplicate .o files for each thin file
for arch in ${arch_types[@]};do
	lipo UPSDK.framework/UPSDK -thin "${arch}" -o "${output_dir}/UPSDK-${arch}"
	ar -d "${output_dir}/UPSDK-${arch}" `ar -t "${src_lib}.temp" | grep -v ${exclude_file}`
done

# generate new fat file
libtool ${output_dir}/UPSDK-armv7 \
		${output_dir}/UPSDK-i386 \
		${output_dir}/UPSDK-x86_64 \
		${output_dir}/UPSDK-arm64 \
		-o ${output_dir}/UPSDK-new

rm -f "${src_lib}.temp"
```
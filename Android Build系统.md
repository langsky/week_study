Android编译系统是基于GNU Make和Shell的一套编译环境。Android编译系统包括三大部分，build/core下的文件，device和vendor下的文件，以及每个模块的Android.mk文件。由于Android.mk文件将编译配置与GNU Make分离，使得只要理解了Android提供的编译环境变量和函数，就能方便地配置编译参数。


---

## Android build核心

Android Build系统的核心位于build/core，这个目录包含了部分mk文件以及一个shell和perl脚本。

Android的编译可以使用三条指令完成：

```
source build/envsetup.sh
lunch
make
```

## 编译环境的建立

**envsetup.sh的作用**

source build/envsetup.sh 这个指令运行了envsetup.sh脚本，这个脚本会建立Android的编译环境。同时建立了许多shell指令，lunch这个指令就是在envsetup.sh脚本中初始化的，如果没有初始化，lunch指令将失去作用。

envsetup.sh脚本除了建立shell命令外，就只有执行add_lunch_combo指令而已。

add_lunch_combo的内容来自两个部分，一部分是envsetup.sh脚本自身定义的，另一部分来自vendor目录下的vendorsetup.sh文件。

add_lunch_combo的定义如下所示：

```
unset LUNCH_MENU_CHOICES
function add_lunch_combo()
{
    local new_combo=$1
    local c
    for c in ${LUNCH_MENU_CHOLCES[@]};do
       if[ "$new_combo" = "$c"];then
            return
       fi
   done
   LUNCH_MENU_CHOICES=(${LUNCH_MENU_CHOICES[@]} $new_combo)    
}
```

这个指令的功能是将调用的参数存放在一个全局变量数组LUNCH_MENU_CHOICES中，并在lunch命令时将它打印出来。

此外envsetup.sh提供了一些有用的shell指令，只介绍几个简单的指令。


|命令|说明|
|---|
|lunch|	制定当前编译的产品|
|m|	编译整个源码，但不用将当前目录切换到源码的根目录|
|mm|	编译当前目录下的所有模块，但不编译它们的依赖模块|
|mmm|	编译指定目录下的所有模块，但不编译它们的依赖模块|
|mma|	编译当前目录下的所有模块，同时编译它们的依赖模块，有的时候单独编译一个模块会失败，就是因为没有将与它有关联的模块的一起编译|
|mmma|	编译指定目录下的所有模块，同时编译它们的依赖模块|
|godir|	根据godir后面的参数文件名在整个源码目录下查找，然后切换到该目录|
|jgrep|	对所有java文件执行grep指令 ，此外还有cgrep，ggrep，resgrep和sgrep等，功能类似|


**lunch命令的功能**

```
lunch <product_name>-<build_variant>
```

lunch命令可以带上面形式的参数，第一个必须是系统已经定义的产品名称，第二个参数必须是user，userdebug，eng三者之一。

下面来分析lunch函数的工作：

设置一个本地变量answer和selection，如果lunch后接参数，则将参数赋值给answer，如果lunch不带参数，则打印所有的产品列表，并默认置selection为“aosp_arm-eng”

将answer赋值给selection，并将selection按照“-”拆分为两部分分别存储在product和variant两个变量中。

将product的值赋给环境变量TARGET_PRODUCT，将variant的值赋给环境变量TARGET_BUILD_VARIANT。

**Build相关的环境变量**

lunch指令完成后，会出现一个编译环境变量列表，如下表所示：


|环境变量|	说明|	备注|
|---|
|PLATFORM_VERSION_CODENAME PLATFORM_VERSION|	前者表示平台版本的名称，通常为AOSP	后者表示Android平台的版本号，比如7.1.1||
|TARGET_PRODUCT|	所编译的产品名称|	通常是是产品的代号，|
|TARGET_BUILD_VARIANT TARGET_BUILD_TYPE TARGET_BUILD_APPS|	分别表示产品类型，编译类型和单个模块路径|	产品类型为user，eng，userdebug三者之一，编译类型为release和debug之一，单个模块路径只有在编译单个模块时会显示该模块路径，整体编译为NULL|
|TARGET_ARCH TARGET_ARCH_VARIANT TARGET_CPU_VARIANT|	分别表示编译目标的CPU架构，CPU架构版本以及CPU代号	比如arm64 armv8-a generic||
|TARGET_2ND_ARCH TARGET_2ND_ARCH_VARIANT TARGET_2ND_CPU_VARIANT|	分别表示编译目标的第二CPU架构，第二CPU架构版本以及第二CPU代号	||
|HOST_ARCH HOST_OS HOST_OS_EXTRA|	分别表示编译平台的架构，编译平台使用的操作系统和编译平台操作系统的额外信息	||
|BUILD_ID|	这个值会出现编译版本信息中，用这个环境变量来定义公司的特有标示	||
|OUT_DIR|	指定编译结果的输出目录	||

> 注意，TARGET_2ND_ARCH TARGET_2ND_ARCH_VARIANT TARGET_2ND_CPU_VARIANT是在Android 5.0后加入的，如果是64位编译，必须考虑到支持的32位应用，因此存在两套CPU架构方案。


当然这些环境变量在lunch执行后几乎是默认值，可以在接下来的make指令中对其中的某些环境变量进行再次指定。

```
make BUILD_ID="Android L"
```

## Build系统的层次关系

通过build系统，可以得到若干img文件，这些文件就是刷机所需要的文件，img文件又由若干个小文件组成。这些小文件需要build系统生成，某些文件只需要简单的复制。所以说Android的Build系统最主要的干了三件事，分别是收集并编译模块，复制二进制文件，产生配置文件。

make命令会调用build/core/main.mk脚本，此时将通过include命令将所有的需要的.mk文件包含进来。最终在内存中形成一个包含所有编译脚本的集合，相当于一个巨大的Makefile文件。

Build系统中的编译脚本有很多，这里只选择几个比较有代表性的脚本进行解释：

**main.mk**

main.mk文件主要功能是包含需要被编译的mk文件。

main.mk包括的mk文件有很多，简单介绍几个常用的mk文件：


main.mk 主控文件，主要作用有三点，首先包含其它mk文件，其次定义几个重要的编译目标，比如droid，sdk，ndk等，最后检查编译工具版本，如make，gcc，javac等

help.mk 帮助文件，打印build系统帮助说明

config.mk 编译系统配置文件，有三点作用，首先使用不同常量来负责不同模块的编译，其次定义编译器参数并引入BoardConfig.mk配置产品参数，最后，定义了一些编译工具的路径

version_default.mk 定义系统版本相关的变量

product_config.mk 定义product_config中使用的各种函数

**config.mk**

config.mk相当于Build系统的配置文件。

**product_config.mk**

相当于对Product的配置文件。

**Android产品的配置文件**

Android的产品配置文件位于device下，但是产品的配置文件也可以放在vendor下，两者从Build系统的角度看没有多大区别，不同通常产品的配置文件会放在device下，而vendor目录下主要放置一些硬件的HAL库。

**配置文件Device目录**

common 用来存储各个产品的通用的配置脚本，文件等。

sample 一个产品的配置例子。

google 几个简单的模块，用途不详

generic 存放的是用于模拟器的产品，包括x86，arm，mips架构

asus lge samsung 分别代表asus lg samsung三家公司，对应公司的产品在文件夹中

如果需要新的产品，可以在device目录下新建一个目录。

**vendorsetup.sh**

产品编译相关，定义新的编译选项。

```
AndroidProduct.mk
BoardConfig.mk
device.mk
```

编译类型

**产品中的image文件**

Android编译完成后会产生几个Image文件，包括boot.img system.img recovery.img userdata.img。

boot.img
boot.img是Android的自定义文件格式。给格式包含了一个2*1024大小的文件头，文件头后面是用gzip压缩过的kernel映像，再后面是一个ramdisk映像，最后是一个可选的载入器程序。

boot.img里的各个部分大小都是page的整数倍，这个page大小在BoardConfig.mk中通过编译变量BOARD_KERNEL_PAGESIZE定义，通常的值是2048。ramdisk是一个小型的文件系统，它包括了初始化Linux系统所需要的全部核心文件。

recovery.img
recovery.img相当于一个小型的文本Linux系统，它有自己的内核和文件系统。它的作用是恢复或升级系统，因此在sbin下有一个recovery程序。

system.img
system.img就是设备中的system目录的镜像，里面包含了Android系统的主要目录和文件。主要的目录简介如下：

app 存放一般的app目录

bin 存放Linux的工具，大部分是toolbox的链接

etc 存放系统的配置文件

fonts 存放系统的字体文件

framework 存放系统平台所有的jar包和资源文件包

lib 存放系统的共享库

media 存放系统的多媒体资源，主要是铃声

priv-app 存放系统核心的apk文件

tts 存放系统语音合成文件

usr 存放键盘布局时间区域文件

vendor 存放一些第三方厂商的配置文件，firmware库记忆动态库

xbin 存放系统管理工具

build.prop 系统属性的定义文件

userdata.img
data目录的镜像，起初一般不包含任何文件。



## 编译Android的模块

每个编译模块下都能找到一个Android.mk文件，里面包含了模块代码的位置，模块的名称，需要链接的动态库等一系列的定义。

下面以packages/apps/Settings下的Android.mk为例讲解Android.mk的使用方法。

```
# 设置LOCAL_PATH为当前目录
LOCAL_PATH:=$(call my-dir)
# 清空LOCAL_PATH外所有的"LOCAL_"变量
include $(CLEAR_VARS)
# 指定依赖的共享Java类库
LOCAL_JAVA_LIBRARIES:=bouncycastle conscript telephony-common
# 指定依赖的静态Java类库
LOCAL_STATIC_JAVA_LIBRARIES:=android-support-v4 android-support-v13 jsr305
# 定义模块的标签
LOCAL_MODULE_TAGS:=optional
# 定义源文件列表
LOCAL_SRC_FILES:= \$(call all-java-files-under, src) \
                    src/com/android/settings/EventLogTags.logtags
# 指定模块名称
LOCAL_PACKAGE_NAME:=Settings
# 指定模块签名
LOCAL_CERTIFICATE:=platform
# 指定APP是否安装在priv-app目录下
LOCAL_PRIVILEGED_MODULE:=true
# 指定混淆标志
LOCAL_PROGUARD_FLAG_FILES:=proguard.flags
# 指定AAPT属性
LOCAL_AAPT_FLAGS+= -c zz_ZZ
# 指定编译模块的类型
include $(BUILD_PACKAGE)
# 将源码目录下其余的Android.mk都包含进来
include $(call all-makefiles-under, $(LOCAL_PATH))
```

对于一个mk文件，有几样是必须的：
开头的几行几乎是固定写法，设置LOCAL_PATH为当前目录以及清空除了LOCAL_PATH外所有的LOCAL_*变量。

结尾处的指定编译模块类型语句，不同的编译模块有不同的编译类型。

**模块编译变量**

```
Android.mk之所以能够编译出不同的模块，由结尾处的include $(BUILD_*)来实现，这个include是Android.mk包含了对应的模块编译文件。以下是模块编译变量的一个列表：

BUILD_HOST_STATIC_LIBRARY BUILD_HOST_SHARED_LIBRARY 分别对应host_static_library.mk和host_shared_library.mk。用来产生编译平台使用的本地静态库和共享库。

BUILD_STATIC_LIBRARY BUILD_SHARED_LIBRARY 分别对应static_library.mk和shared_library.mk。用来产生目标系统使用的本地静态库和共享库。

BUILD_HOST_EXECUTABLE 对应host_executable.mk，用来产生编译平台下的使用的Linux可执行程序。

BUILD_EXECUTABLE 对应executable.mk，用来产生目标系统下的使用的Linux可执行程序。

BUILD_HOST_PREBUILT 对应host_prebuilt.mk，用来定义编译平台下的预编译模块目标。

BUILD_PREBUILT 对应prebuilt.mk，用来定义预编译模块目标，这些模块将被引入体系。

BUILD_PACKAGE 对应package.mk，用来产生apk文件。

BUILD_PHONY_PACKAGE 对应phony_package.mk。

BUILD_MULTI_PREBUILT 对应multi_built.mk，定义多个预编译目标。

BUILD_JAVA_LIBRARY 对应java_library.mk，用来产生java共享库。

BUILD_STATIC_JAVA_LIBRARY 对应static_java_library.mk，用来产生java静态库。

BUILD_COPY_HEADERS 对应copy_headers.mk，将LOCAL_COPY_HEADERS定义的文件复制到LOCAL_COPY_HEADERS_TO定义的路径中。
```

## 常用模块编译实例

**apk文件**

```
LOCAL_PATH:=$(call my-dir)
include $(CLEAR_VARS)
# 指定依赖的共享库和静态库
LOCAL_JAVA_LIBRARIES:= 
LOCAL_STATIC_JAVA_LIBRARIES:=
# 指定编译资源，使用系统提供的函数获得src下的文件形成列表
LOCAL_SRC_FILES:=$(call all-java-files-under, src)
# 指定模块的标签
LOCAL_MODULE_TAGS:=optional
# 指定模块的签名方式
LOCAL_CERTIFICATE:=shared
#  指定模块的名称
LOCAL_PACKAGE_NAME:=testapk
include $(BUILD_PACKAGE)
```

**Java共享库**

```
LOCAL_PATH:=$(call my-dir)
include $(CLEAR_VARS)
# 指定编译资源，使用系统提供的函数获得src下的文件形成列表
LOCAL_SRC_FILES:=$(call all-java-files-under, src)
# 指定模块的标签
LOCAL_MODULE_TAGS:=optional
#  指定模块
LOCAL_MODULE:=javadynamiclib
include $(BUILD_JAVA_LIBRARY)
```

**Java静态库**

```
LOCAL_PATH:=$(call my-dir)
include $(CLEAR_VARS)
# 指定编译资源，使用系统提供的函数获得src下的文件形成列表
LOCAL_SRC_FILES:=$(call all-java-files-under, src)
# 指定模块的标签
LOCAL_MODULE_TAGS:=optional
#  指定模块
LOCAL_MODULE:=javastaticlib
include $(BUILD_STATIC_JAVA_LIBRARY)
```

**Java资源包 是没有代码的apk文件**

```
LOCAL_PATH:=$(call my-dir)
include $(CLEAR_VARS)

# 声明没有静态java库依赖
LOCAL_NO_STANDARD_LIBRARIES:=true

# 指定模块的标签
LOCAL_MODULE_TAGS:=optional
# 指定模块的签名方式
LOCAL_CERTIFICATE:=shared
#  指定模块的名称
LOCAL_PACKAGE_NAME:=resapk
# 指定安装路径
LOCAL_MODULE_PATH:=$(TARGET_OUT_JAVA_LIBRARIES)
# 指定AAPT工具参数
LOCAL_AAPT_FLAGS:=-x
# 指定为true其它apk可以引用本apk资源
LOCAL_EXPORT_PACKAGE_RESOURCES:=true

include $(BUILD_JAVA_LIBRARY)
```

**可执行文件**

```
LOCAL_PATH:=$(call my-dir)
include $(CLEAR_VARS)

LOCAL_SRC_FILES:=service.cpp
LOCAL_SGARED_LIBRARIES:=libutils libbinder
# 定义编译标志
ifeq($(TARGET_OS), linux) 
    LOCAL_CFLAGS+=-DPX_UNIX
endif
LOCAL_MODULE:=service
include $(BUILD_EXECUTABLE)
```

**native共享库**

```
LOCAL_PATH:=$(call my-dir)
include $(CLEAR_VARS)

LOCAL_MODULE_TAGS:=optional
LOCAL_MODULE:=libnativedynamic
LOCAL_SRC_FILES:=\nativedynamic.cpp
LOCAL_SHARED_LIBRARIES:=\libcutils \libutils
LOCAL_STATIC_LIBRARIWS:=libnativestatic

LOCAL_C_INCLUDES+=\$(JNI_H_INCLUDE) \$(LOCAL_PATH)/../include

LOCAL_CFLAGS+=-O

include $(BUILD_SHARED_LIBRARY)
```

**native静态库**

```
LOCAL_PATH:=$(call my-dir)
include $(CLEAR_VARS)

LOCAL_MODULE_TAGS:=optional
LOCAL_MODULE:=libnativestatic
LOCAL_SRC_FILES:=\nativestatic.cpp
LOCAL_SHARED_LIBRARIES:=\libcutils \libutils
LOCAL_STATIC_LIBRARIWS:=libnativestatic

LOCAL_C_INCLUDES+=\$(JNI_H_INCLUDE) \$(LOCAL_PATH)/../include

LOCAL_CFLAGS+=-O

include $(BUILD_SHARED_LIBRARY)
```

## 预编译模块的自定义

预编译模块就是事先已经编译好的模块，在Android整体编译时，直接将模块文件拷贝到目标文件夹中，这时使用PRODUCT_COPY_FILES实现，但有的时候，如果apk或者jar需要系统签名，或者有其它的依赖关系，单纯的复制无法解决问题，此时需要对预编译模块进行定义。

对预编译模块的定义和编译模块的定义几乎相同，不同之处在于LOCAL_SRC_FILES的内容，前者指定的是需要预编译的二进制文件的路径，同时需要使用LOCAL_MODULE_CLASS来指定模块的类型。最后使用BUILD的指令为BUILD_PREBUILT。

**定义apk**

```
include $(CLEAR_VARS)
LOCAL_MODULE:=ThemeManager.apk
LCOAL_SRC_FILES:=app/$(LOCAL_MODULE)
LOCAL_MODULE_TAGS:=optional
LOCAL_MODULE_CLASS:=APPS
LOCAL_CERTIFICATE:=platform
include $(BUILD_PREBUILT)
```

**定义静态jar包目标**

```
include $(CLEAR_VARS)
LOCAL_MODULE:=libfirewall.jar
LCOAL_SRC_FILES:=app/$(LOCAL_MODULE)
LOCAL_MODULE_TAGS:=optional
LOCAL_MODULE_CLASS:=JAVA_LIBRRIES
LOCAL_CERTIFICATE:=platform
include $(BUILD_PREBUILT)
```

**定义动态库文件目标**

```
include $(CLEAR_VARS)
LOCAL_MODULE:=libglobaltheme_jni.so
LOCAL_MODULE_OWNER:=
LCOAL_SRC_FILES:=lib/$(LOCAL_MODULE)
LOCAL_MODULE_TAGS:=optional
LOCAL_MODULE_CLASS:=SHARED_LIBRARIES
include $(BUILD_PREBUILT)
```

**定义可执行文件目标**

```
include $(CLEAR_VARS)
LOCAL_MODULE:=bootanimation
LOCAL_MODULE_OWNER:=
LCOAL_SRC_FILES:=bin/$(LOCAL_MODULE)
LOCAL_MODULE_TAGS:=optional
LOCAL_MODULE_CLASS:=EXECUTABLES
LOCAL_MODULE_PATH:=$(TARGET_OUT)/bin
include $(BUILD_PREBUILT)
```

**定义xml文件目标**

```
include $(CLEAR_VARS)
LOCAL_MODULE:=app-conf-cu.xml
LOCAL_MODULE_OWNER:=
LCOAL_SRC_FILES:=etc/$(LOCAL_MODULE)
LOCAL_MODULE_TAGS:=optional
LOCAL_MODULE_CLASS:=ETC
include $(BUILD_PREBUILT)
```

**常用的LOCAL_变量**

```
LOCAL_ASSET_FILES 编译apk时指定资源列表，通常写成LOCAL_ASSET_FILES+=$(call find-subdir-assets)

LOCAL_CC 自定义C编译器来替代缺省的编译器

LOCAL_CXX 自定义C++编译器来替代缺省的编译器

LOCAL_CFLAGS 定义额外的C/C++编译器参数

LOCAL_CPPFLAGS 仅定义额外的C++编译器参数

LOCAL_CPP_EXTENSION 定义C++文件的后缀，比如LOCAL_CPP_EXTENSION:=.cc，目前不支持混合后缀

LOCAL_C_INCLUDES 指定头文件的搜索路径

LOCAL_FORCE_STATIC_EXECUTABLE 优先执行静态链接库

LOCAL_GENERATED_SOURCES 指定由系统自动生成的源文件列表

LOCAL_MODULE_TAGS 定义模块标签

LOCAL_REQUIRED_MODULES 指定依赖的模块，一旦模块被安装，依赖的模块也会被安装

LOCAL_JAVAFLAGS 定义额外的javac编译器参数

LOCAL_LDFLAGS 定义连接器Ld的参数

LOCAL_JAVA_LIBARIES 指定模块依赖的Java共享库

LOCAL_NO_MANIFEST 表示一个apk文件是否有manifest文件

LOCAL_PACKAGE_NAME 指定App应用的名称

LOCAL_PATH 指定Android.mk所在的目录

LOCAL_SHARED_LIBRARIES 指定模块依赖的C/C++共享库列表

LOCAL_SRC_FILES 指定源文件列表

LOCAL_STATIC_LIBRARIES 指定依赖的C/C++静态库列表

LOCAL_MODULE 除了apk使用LOCAL_PACKAGE_NAME命名模块名以外，其它模块都用此命名模块名。

LOCAL_MODULE_PATH 指定模块在目标系统的安装路径

LOCAL_MODULE_CLASS 定义模块的分类，根据分类会将模块放在目标系统的不同位置。它比LOCAL_MODULE_PATH的优先级低

LCOAL_MODULE_NAME 指定模块的名称
```

## Android应用签名

**生成签名文件**

```
keytool -genkey -v -keystore tom.keystore -alias tom_key -keyalg RSA -validity 1000

-keystore tom.keystore 证书文件名
-alias tom_key 别名
-keyalg RSA 加密方式
-validity 1000 有效期1000天
```
**为Apk签名**

```
jarsigner -verbose -keystore tom.keystore -signedjar test_signed.apk test.apk tom_key

-verbose 打印签名过程
-keystore tom.keystore 指定签名文件
-signedjar test_signed.apk 签名后的文件名
```

**处理异常**

```
jarsigner: unable to sign jar: java.util.zip.ZipException: invalid entry compressed size
```

删除META-INF文件夹再打包，原因是使用新的签名时必须删除旧的签名文件。

**校验签名**

```
jarsigner -verify -verbose -certs test_signed.apk
```

**4K对齐**

```
zipalign -v 4 test_signed.apk test_align.apk
```

## Android系统签名

在目录`build/product/security`目录下存在四组后缀为.x509.pem（公钥）和.pk8（私钥）文件。分别代表四种系统签名。

```
testkey 普通apk
platform 系统核心apk
shared 启动器，联系人等apk
media 多媒体和下载类apk
```

**生成系统签名文件**

```
development/tools

make_key testkey '/C=CN/ST=BJ/L=BJ/O=Google/OU=Android/CN=Tom/emailAddress=tom@1.com'

testkey 是生成签名文件的类型，后续的参数是签名的信息
```

**指定签名文件路径**

可以直接将签名文件覆盖掉`build/product/security`，或者新建一个目录，并指定该目录的属性，比如在`device/lge/hammerhead/security`目录下，在device.mk中添加如下属性：

```
PRODUCT_DEFAULT_DEV_CERTIFICATE := device/lge/hammerhead/security
```

**签名apk**

在随源码编译的Apk，不需要手动去签名。

如果想手动签名的话，使用如下指令：

```
java -jar signapk.jar platform.x509.pem platform.pk8 test.apk test_signed.apk
```

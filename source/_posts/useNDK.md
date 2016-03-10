---
title: 使用NDK
date: 2016-03-10 09:08:10
tags: NDK
categories: NDK
---

# 使用NDK
Android是基于Linux平台开发的开源手机操作系统,自然要对C、C++提供原生的支持。通过Google发布的Android手机NDK(Native Development Kit),应用程序可以非常方便地实现Java与C/C++代码的相互沟通。合理的使用NDK,可以提高应用程序的执行效率。

---
## NDK开发环境

Android C/C++开发环境主要由一下几部分构成：

- Android软件开发包(Software Development Kit, SDK)
- Android原生开发包(Native Development Kit, NDK)
- Eclipse上的Android开发工具(Android Development Tools, ADT)插件
- Java开发包(Java Development Kit, JDK)
- Apache ANT构建系统
- GNU Make构建系统
- Eclipse IDE  
*将相应的目录添加到环境变量中去,能够让系统执行相应的命令*
*NDK的一些组件是shell脚本，这些脚本不能直接在Windows系统上执行。Windows平台上需要下载安装Cygwin*

---
<!--more-->
## NDK提供的组件

NDK不是一个单独的工具；它是一个包含API、交叉编译器、连接程序、调试器、构建工具、文档和示例应用程序的综合工具集。以下是一些主要组件：  

- ARM、x86和MIPS交叉编译器
- 构建系统
- Java原生接口头文件
- C库
- Math库
- POSIX线程
- 最小C++库
- ZLib压缩库
- 动态连接库
- Android日志库
- Android像素缓冲区库
- Android原生应用APIs
- OpenGL ES 3D图形库
- OpenSL ES 原生音频库
- OpenMAX AL 最小支持

---
## NDK构建项目
用原生组件构建Android项目需要两步：

- 第一步构建原生组件(ndk-build:调用Android构建系统的辅助脚本,生成so文件)
- 第二步构建Java应用程序并将Java应用程序与其原生组件打包。
生成Apache ANT构建文件：android update project -p . -n hello-jni -t android-14—subprojects 然后通过执行 ant debug命令构建项目将构建Java文件并将该Java文件与原生组件打成一个可安装的Android包，即APK文件。

### NDK构建系统
Android NDK的构建系统是基于GUN Make的。该构建系统的主要目的是使开发人员能够用很短的构建文档来描述原生的Android应用程序。Android NDK构建系统是由多种GUN Makefile片段构成的。build/core子目录中包括不同类型NDK项目所需要的必要片段。除了这些片段，构建系统还要依赖另外两个文件：Android.mk和Application.mk，这两个文件作为NDK项目的一部分由开发人员提供。

**Android.mk**

	opyright (C) 2009 The Android Open Source Project
	#
	# Licensed under the Apache License, Version 2.0 (the "License");
	# you may not use this file except in compliance with the License.
	# You may obtain a copy of the License at
	#
	#      http://www.apache.org/licenses/LICENSE-2.0
	#
	# Unless required by applicable law or agreed to in writing, software
	# distributed under the License is distributed on an "AS IS" BASIS,
	# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
	# See the License for the specific language governing permissions and
	# limitations under the License.
	#
	LOCAL_PATH := $(call my-dir)

	include $(CLEAR_VARS)

	LOCAL_MODULE    := hello-jni
	LOCAL_SRC_FILES := hello-jni.c

	include $(BUILD_SHARED_LIBRARY)
	
逐行分析

	LOCAL_PATH := $(call my-dir) 
每个Android.mk文件必须以定义LOCAL_PATH为开始。它用于在开发tree中查找源文件。宏my-dir 则由Build System提供。返回包含Android.mk的目录路径。  

	include $(CLEAR_VARS)

CLEAR_VARS变量由Build System提供。并指向一个指定的GNU Makefile，由它负责清理很多LOCAL_xxx.例如：LOCAL_MODULE, LOCAL_SRC_FILES, LOCAL_STATIC_LIBRARIES等等。但不清理LOCAL_PATH.这个清理动作是必须的，因为所有的编译控制文件由同一个GNU Make解析和执行，其变量是全局的。所以清理后才能避免相互影响。

	LOCAL_MODULE    := hello-jni
	
LOCAL_MODULE模块必须定义，以表示Android.mk中的每一个模块。名字必须唯一且不包含空格。Build System会自动添加适当的前缀和后缀。例如，foo，要产生动态库，则生成libfoo.so. 但请注意：如果模块名被定为：libfoo.则生成libfoo.so. 不再加前缀。

	LOCAL_SRC_FILES := hello-jni.c
	
LOCAL_SRC_FILES变量必须包含将要打包如模块的C/C++ 源码。不必列出头文件，build System 会自动帮我们找出依赖文件。缺省的C++源码的扩展名为.cpp. 也可以修改，通过LOCAL_CPP_EXTENSION。

	include $(BUILD_SHARED_LIBRARY)
	
BUILD_SHARED_LIBRARY：是Build System提供的一个变量，指向一个GNU Makefile Script。它负责收集自从上次调用 include $(CLEAR_VARS) 后的所有LOCAL_XXX信息。并决定编译为什么。

#### 构建库语句

- CLEAR_VARS
指向一个编译脚本。必须在新模块前包含之。

	```
	include $(CLEAR_VARS)
	```
	
- BUILD_SHARED_LIBRARY：
指向一个编译脚本，它收集自从上次调用 include $(CLEAR_VARS) 后的所有LOCAL_XXX信息。并决定如何将你列出的Source编译成一个动态库。 注意，在包含此文件前，至少应该包含：LOCAL_MODULE and LOCAL_SRC_FILES 例如：
include $(BUILD_SHARED_LIBRARY)
BUILD_STATIC_LIBRARY：
与前面类似，它也指向一个编译脚本，收集自从上次调用 include $(CLEAR_VARS) 后的所有LOCAL_XXX信息。并决定如何将你列出的Source编译成一个静态库。静态库不能够加入到Project或者APK中。但它可以用来生成动态库。LOCAL_STATIC_LIBRARIES and LOCAL_WHOLE_STATIC_LIBRARIES将描述之。

	```
	include $(BUILD_STATIC_LIBRARY)
	```
	
- BUILD_EXECUTABLE:
与前面类似，它也指向一个编译脚本，收集自从上次调用 include $(CLEAR_VARS) 后的所有LOCAL_XXX信息。并决定如何将你列出的Source编译成一个可执行Native程序。

	```
	include $(BUILD_EXECUTABLE)
	```
	
- PREBUILT_SHARED_LIBRARY：
把这个共享库声明为”一个”独立的模块。指向一个build脚本，用来指定一个预先编译好多动态库。与BUILD_SHARED_LIBRARY and BUILD_STATIC_LIBRARY不同，此时模块的LOCAL_SRC_FILES应该被指定为一个预先编译好的动态库，而非source file. LOCAL_PATH := $(call my-dir)

	```
	include $(CLEAR_VARS)	
	LOCAL_MODULE := foo-prebuilt     # 模块名	
	LOCAL_SRC_FILES := libfoo.so     # 模块的文件路径（相对于 LOCAL_PATH）
	include $(PREBUILT_SHARED_LIBRARY) # 注意这里不是 BUILD_SHARED_LIBRARY
	```
    
这个共享库将被拷贝到 $PROJECT/obj/local 和 $PROJECT/libs/ (stripped) 主要是用在将已经编译好的第三方库使用在本Android Project中。

#### 其他构建系统变量
Android NDK 构建系统还支持其他变量,构建系统定义的变量有：

- TARGET_ARCH：目标CPU 体系结构的名称，例如arm
- TARGET_PLATFORM：目标Android 平台的名称，例如：android-3
- TARGET_ARCH_ABI：目标CPU 体系结构和ABI 的名称，例如：armeabi-v7a
- TARGET_ABI：目标平台和ABI 的串联，例如：android-3-armeabi-v7a可被定义为模块说明部分的变量有：
- LOCALMODULE_FILENAME：可选变量，用来重新定义生成的输出文件名称。默认情况下，构建系统使用LOCAL_MODULE 的值作为生成的输出文件名称，但变量LOCAL_MODULE FILENAME 可以覆盖LOCAL_MODULE 的值。
- LOCAL_CPP_EXTENSION：C++源文件的默认扩展名是.cpp。这个变量可以用来为C++源代码指定一个或多个文件扩展名。
  
    ```
	LOCAL_CPP_ EXTENSION :=.cpp .cxx
	```
	
- LOCAL_CPP_FEATURES：可选变量，用来指明模块所依赖的具体C++特性，如RTTI、exceptions 等。

	```
	LOCAL_CPP_FEATURES :=rtti
	```
- LOCAL_C_INCLUDES：可选目录列表，NDK 安装目录的相对路径，用来搜索头文件。

	```
	LOCAL_C_INCLUDES :=sources/shared-module
	LOCAL_C_INCLUDES :=$(LOCAL_PATH)/include
	```
- LOCAL_CFLAGS：一组可选的编译器标志，在编译C 和C++源文件的时候会被传送给编译器。

	```
	LOCAL_CFLAGS :=-DNDEBUG -DPORT=1234
	```
- LOCAL_CPP_FLAGS：一组可选的编译标志，在只编译C++源文件时被传送给编译器。
- LOCAL_WHOLE_STATIC_LIBRARIES：LOCAL_STATIC_LIBRARIES 的变体，用来指明应该被包含在生成的共享库中的所有静态库内容。小贴士：当几个静态库之间有循环依赖时，LOCAL_WHOLE_STATIC_LIBRARIES很有用
- LOCAL_LDLIBS：链接标志的可选列表，当对目标文件进行链接以生成输出文件时该标志将被传送给链接器。它主要用于传送要进行动态链接的系统库列表。例如：要与Android NDK 日志库链接，使用以下代码：

	```
	LOCAL_LDFLAGS :=−llog
	```
- LOCAL_ALLOW_UNDEFINED_SYMBOLS：可选参数，它禁止在生成的文件中进行缺失符号检查。若没有定义，链接器会在符号缺失时生成错误信息。
- LOCAL_ARM_MODE：可选参数，ARM 机器体系结构特有变量，用于指定要生成的ARM 二进制类型。默认情况下，构建系统在拇指模式下用16 位指令生成，但该变量可以被设置为arm 来指定使用32 位指令。

	```
	LOCAL_ARM_MODE :=arm
	```
该变量改变了整个模块的构建系统行为；可以用.arm 扩展名指定只在arm 模式下构建特定文件。

	```
	LOCAL_SRC_FILES :=file1.c file2.c.arm
	```
- LOCAL_ARM_NEON：可选参数，ARM 机器体系结构特有变量，用来指定在源文件中应该使用的ARM 高级单指令流多数据流(Single Instruction Multiple Data，SIMD)(a.k.a. NEON)内联函数。
 
	```
	LOCAL_ARM_NEON :=true
	```
该变量改变了整个模块的构建系统行为；可以用.neon 扩展名指定只构建带有NEON 内联函数的特定文件。

	```
	LOCAL_SRC_FILES ：＝file1.c file2.c.neon
	```
- LOCAL_DISABLE_NO_EXECUTE：可选变量，用来禁用NX Bit 安全特性。NXBit 代表Never Execute(永不执行)，它是在CPU 中使用的一项技术，用来隔离代码区和存储区。这样可以防止恶意软件通过将它的代码插入应用程序的存储区来控制应用程序。
 
	```
	LOCAL_DISABLE_NO_EXECUTE :=true
	```
- LOCAL_EXPORT_CFLAGS：该变量记录一组编译器标志，这些编译器标志会被添加到通过变量LOCAL_STATIC_LIBRARIES 或LOCAL_SHARED_LIBRARIES使用本模块的其他模块的LOCAL_CFLAGS 定义中。
 
	```
	LOCAL_MODULE := avilib
	LOCAL_EXPORT_CFLAGS := − DENABLE_AUDIO
	LOCAL_MODULE := module1
	LOCAL_CFLAGS := − DDEBUG
	LOCAL_SHARED_LIBRARIES := avilib
	```
编译器在构建module1 时会以-DENABLE_AUDIO –DDEBUG 标志执行。

- LOCAL_EXPORT_CPPFLAGS：和LOCAL_EXPORT_CLAGS 一样，但是它是C++特定代码编译器标志。
- LOCAL_EXPORT_LDFLAGS：和LOCAL_EXPORT_CFLAGS 一样，但用作链接器标志。
- LOCAL_EXPORT_C_INCLUDES：该变量允许记录路径集，这些路径会被添加到通过变量LOCAL_STATIC_LIBRARIES 或LOCAL_SHARED_LIBRARIES 使用该模块的LOCAL_C_INCLUDES 定义中。
- LOCAL_SHORT_COMMANDS：对于有大量资源或独立的静态/共享库的模块，该变量应该被设置为true。诸如Windows 之类的操作系统只允许命令行最多输入8 191 个字符；该变量通过分解构建命令使其长度小于8 191 个字符。在较小的模块中不推荐使用该方法，因为使用它会让构建过程变慢。
- LOCAL_FILTER_ASM：该变量定义了用于过滤来自LOCAL_SRC_FILES 变量的装配文件的应用程序。

#### 其他的构建系统函数宏
Android NDK 构建系统支持的其他函数宏。

- all-subdir-makefiles：返回当前目录的所有子目录下的Android.mk 构建文件列表。例如，调用以下命令可以将子目录下的所有Android.mk 文件包含在构建过程中：

	```
	include $(call all-subdir-makefiles)
	```
- this-makefile：返回当前Android.mk 构建文件的路径。
- parent-makefile：返回包含当前构建文件的父Android.mk 构建文件的路径。
- grand-parent-makefile：和parent-makefile 一样但用于祖父目录。

#### 定义新变量
开发人员可以定义其他变量来简化他们的构建文件。以LOCAL和NDK前缀开头的名称预留给Android NDK 构建系统使用。建议开发人员定义的变量以MY_开头，如程序

```
MY_SRC_FILES := avilib.c platform_posix.c
LOCAL_SRC_FILES := $(addprefix avilib/, $(MY_SRC_FILES))
```
#### 条件操作
Android.mk 构建文件也可以包含关于这些变量的条件操作，例如：在每个体系结构中包含一个不同的源文件集，如程序

```
ifeq ($(TARGET_ARCH),arm)
    LOCAL_SRC_FILES + = armonly.c
else
    LOCAL_SRC_FILES + = generic.c
endif
```

#### Application.mk
Application.mk 是Android NDK 构建系统使用的一个可选构建文件。和Android.mk 文件一样，它也被放在jni 目录下。Application.mk 也是一个GUN Makefile 片段。它的目的是描述应用程序需要哪些模块；它也定义所有模块的通用变量。以下是Application.mk 构建文件支持的变量：

- APP_MODULES：默认情况下，Android NDK 构建系统构建Android.mk 文件声明的所有模块。该变量可以覆盖上述行为并提供一个用空格分开的、需要被构建的模块列表。
- APP_OPTIM：该变量可以被设置为release 或debug 以改变生成的二进制文件的优化级别。默认情况下使用的是release 模式，并且此时生成的二进制文件被高度优化。该变量可以被设置为debug 模式以生成更容易调试的未优化二进制文件。
- APP_CLAGS：该变量列出了一些编译器标志，在编译任何模块的C 和C++源文件时这些标志都会被传给编译器。
- APP_CPPFLAGS：该变量列出了一些编译器标志，在编译任何模块的C++源文件时这些标志都会被传给编译器。
- APP_BUILD_SCRIPT：默认情况下，Android NDK 构建系统在项目的jni 子目录
下查找Android.mk 构建文件。可以用该变量改变上述行为，并使用不同的生成
文件。
- APP_ABI：默认情况下，Android NDK 构建系统为armeabi ABI 生成二进制文件。可以用该变量改变上述行为，并为其他ABI 生成二进制文件，例如：
`APP_ABI := mips`
另外，可以设置多个ABI
`APP_ABI := armeabi mips`
为所有支持的ABI 生成二进制文件
`APP_ABI := all`
- APP_STL：默认情况下，Android NDK 构建系统使用最小STL 运行库，也被称为system 库。可以用该变量选择不同的STL 实现。
`APP_STL :=stlport_shared`
- APP_GNUSTL_FORCE_CPP_FEATURES：与LOCAL_CPP_EXTENSIONS 变量相似，该变量表明所有模块都依赖于具体的C++特性，如RTTI、exceptions 等。
- APP_SHORT_COMMANDS：与LOCAL_SHORT_COMMANDS 变量相似，该变量使得构建系统在有大量源文件的情况下可以在项目中使用更短的命令。

### 使用NDK-Build 脚本
如前所述，可以通过执行ndk-build 脚本启动Android NDK 构建系统。该脚本用一组参数使维护和控制构建过程更容易。

- 默认情况下，ndk-build 脚本应该在主项目目录中执行。-C 参数可以用于指定命令
行中NDK 项目的位置，这样一来ndk-build 脚本可以从任意的位置开始。

	```
	ndk-build –C /path/to/the/project
	```
- 如果源文件没被修改，Android NDK 构建系统不会重构建目标。可以用-B 执行ndk-build 脚本来强制重构建所有源代码。
 
	```
	ndk-build –B
	```
- 为了清理生成的二进制文件和目标文件，可以在命令行执行ndk-build clean 命令。Android NDK 构建系统会删除生成的二进制文件。
 
	```
	ndk-build clean
	```
- Android NDK 构建系统依赖于GNU Make 工具对模块进行构建。默认情况下，GNUMake 工具一次执行一句构建命令，等这一句完成以后再执行下一句。如果在命令行使用-j 参数，GNU Make 就可以并行执行构建命令。另外，也可以通过指定该参数之后的数字来指定并行执行的命令总数。
 
	```
	ndk-build –j 4
	```
	
---
以上
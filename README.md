# Using-C-and-Assembly-Language-in-Android-Studio-3.0
在Android Studio 3.0中使用C语言以及汇编语言

<br />

从Android Studio 2.2起，我们可以直接通过CMake在Android Studio中写C源代码以及汇编代码，而不需要通过NDK编译工具链生成好.so文件后再导入到工程中。
而到了Android 3.0，使用C代码就更方便了，我们通过工程向导设置使用C语言之后，向导会自动建立一个完整的利用C++语言JNI的工程，我们只要把默认的那个恶心的cpp源文件修改为C源文件即可。下面我将详细列出直接通过Android Studio 3.0来编写C代码和汇编代码的步骤。对于其中的细节，如果各位有条件的话可以参考Android开发者官网中关于NDK的信息：
这条是关于CMake的：[CMake](https://developer.android.google.cn/ndk/guides/cmake)
这条是关于在项目中使用C++代码的：[向您的项目添加 C 和 C++ 代码](https://developer.android.google.cn/studio/projects/add-native-code.html)

<br />

1、下载Android Studio 3.0，然后第一次打开Android Studio之后会弹出一个欢迎界面。我们点击右下角的Configure按钮，如下图所示。

![1.jpg](https://github.com/zenny-chen/Using-C-and-Assembly-Language-in-Android-Studio-3.0/blob/master/1.jpg)

<br />

2、然后会弹出一个列表框，我们选择第一个：SDK Manager，然后会弹出如下图所示的对话框。我们在中间栏点击“SDK Tools”，这里需要至少增加三个选项——“CMake”，“LLDB”以及“NDK”。

![2.jpg](https://github.com/zenny-chen/Using-C-and-Assembly-Language-in-Android-Studio-3.0/blob/master/2.jpg)

<br />

3、选完了之后，我们开始安装。安装完之后，我们开始新建一个全新的工程。我们在配置这个工程的时候可以根据自己的喜好进行设置，当然大部分用默认的配置即可，不过我们这里要增加对C语言支持的选项，如下图所示。

![3.jpg](https://github.com/zenny-chen/Using-C-and-Assembly-Language-in-Android-Studio-3.0/blob/master/3.jpg)

<br />

4、这里勾选之后，后面的配置可以不用管。点击Finish之后需要等待一些时间让IDE做项目配置以及初始编译。最后我们进入到了主工程中。

这里需要提醒各位的是，在Android Studio 3.3之后，我们必须在第一个界面“Choose your project”中选择“ **Native C++** ”才能看见后面对C++配置的向导界面，否则是没有的。选用“Native C++”这个项目配置其实就相当于用空活动配置加上对JNI的使用来创建一个项目。

<br />

到了主工程中，我们先别着急去写C代码，因为此时还没有C源文件，我们可以先配置build.gradle(Module:app)配置文件，这里列出android配置部分：

```gradle
android {
    compileSdkVersion 26
    defaultConfig {
        applicationId "com.greengames.zennychen.ctest"
        minSdkVersion 23
        targetSdkVersion 26
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
        externalNativeBuild {
            cmake {
                // 使用NEON技术
                arguments "-DANDROID_ARM_NEON=TRUE"

                // C语言使用GNU17标准
                cFlags "-std=gnu17 -Os"
            }
        }
        ndk {
            // Specifies the ABI configurations of your native
            // libraries Gradle should build and package with your APK.
            abiFilters 'x86_64', 'armeabi-v7a', 'arm64-v8a'
        }
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
    externalNativeBuild {
        cmake {
            path "CMakeLists.txt"
        }
    }
}
```

这里各位请注意内部的cmake部分，这里使用了构建参数以及全局编译选项。然后再注意一下对ndk的配置，这里加入了在本项目工程中所支持的ABI架构。ABI架构可以根据自己的需求进行配置，不过笔者所列出的这几个已经能涵盖当前活跃的绝大部分的安卓设备了。

<br />

接下去，我们可以把IDE自动生成的native-lib.cpp文件名修改为native-lib.c，然后编辑成以下C代码：

```cpp
//
// Created by Zenny Chen on 2017/10/27.
//

#include <jni.h>
#include <stdio.h>
#include <string.h>

#ifndef __x86_64__

/**
 * 测试内联汇编，分别根据AArch32架构以及AArch64架构来实现一个简单的减法计算
 * @param a 被减数
 * @param b 减数
 * @return 减法得到的差值
 */
static int __attribute__((naked, pure)) MyASMTest(int a, int b)
{
#ifdef __arm__

    asm(".thumb");
    asm(".syntax unified");

    asm("sub r0, r0, r1");
    asm("add r0, r0, #1");  // 为了区分当前用的是AArch32还是AArch64，这里对于AArch32情况下再加1
    asm("bx lr");

#else

    asm("sub w0, w0, w1");
    asm("ret");

#endif
}

#else

extern int MyASMTest(int a, int b);

#endif

JNICALL jstring Java_com_greengames_zennychen_ctest_MainActivity_stringFromJNI
        (JNIEnv *env, jobject this)
{
    char strBuf[128];

    sprintf(strBuf, "Hello from C! ASM test result: %d", MyASMTest(6, 4));

    return (*env)->NewStringUTF(env, strBuf);
}
```

上述C代码对AArch32与AArch64架构下的内联汇编以及对x86_64架构下待会儿要使用的独立的YASM汇编器编译的汇编代码进行了引用。下面我们可以在默认的cpp文件夹下再新增一个名为test.asm的汇编文件，输入以下x86_64的汇编代码：
```nasm
; 这是一个汇编文件
; YASM的注释风格使用分号形式

global MyASMTest

section .text

MyASMTest:

    sub     edi, esi
    mov     eax, edi
    ret

```

<br />

最后，我们对CMakeLists.txt进行编辑，这里面需要引入YASM汇编器。此外，我们还要把所编辑的C源文件与汇编源文件加入到所要生成的动态库中。

```cmake
# For more information about using CMake with Android Studio, read the
# documentation: https://d.android.com/studio/projects/add-native-code.html

# Sets the minimum version of CMake required to build the native library.

cmake_minimum_required(VERSION 3.6.0)

enable_language(ASM_NASM)

# setup common C flags
set(CMAKE_C_FLAGS "-std=gnu17 -Os ${CMAKE_C_FLAGS}")

if(${ANDROID_ABI} STREQUAL "x86_64")
    set(asm_SRCS src/main/cpp/test.asm)
endif()

# Creates and names a library, sets it as either STATIC
# or SHARED, and provides the relative paths to its source code.
# You can define multiple libraries, and CMake builds them for you.
# Gradle automatically packages shared libraries with your APK.

add_library( # Sets the name of the library.
             native-lib

             # Sets the library as a shared library.
             SHARED

             # Provides a relative path to your source file(s).
             src/main/cpp/native-lib.c ${asm_SRCS})

# Searches for a specified prebuilt library and stores the path as a
# variable. Because CMake includes system libraries in the search path by
# default, you only need to specify the name of the public NDK library
# you want to add. CMake verifies that the library exists before
# completing its build.

find_library( # Sets the name of the path variable.
              log-lib

              # Specifies the name of the NDK library that
              # you want CMake to locate.
              log )

# Specifies libraries CMake should link to your target library. You
# can link multiple libraries, such as libraries you define in this
# build script, prebuilt third-party libraries, or system libraries.

target_link_libraries( # Specifies the target library.
                       native-lib

                       # Links the target library to the log library
                       # included in the NDK.
                       ${log-lib} )
```

我们这里使用了条件判断，只有在当前所编译的架构为x86_64的情况下才把test.asm给加入到所编译的库中。

<br />

上述文件都编辑好之后，我们就可以开测了。如果各位没有x86的安卓设备，那么我们可以新建一个基于x86-64架构的模拟器，x86-64架构的模拟器运行速度还是挺快的，而对于ARM架构的处理器，我们最好用真机来测。

如果我们想在CMake中引入在NDK中常用的库，比如cpufeature，那么我们在`cmake_minimum_required`中添加`include(AndroidNdkModules)`，然后再加上`android_ndk_import_module_cpufeatures()`，最后在`target_link_libraries`中的
***\#Specifies the target library***这一注释下面添加`cpufeatures`。这里面所添加的库的名称可以用空格分隔，也可以用换行符来分隔。下面给出完整的CMakeLists.txt：
```cmake
# For more information about using CMake with Android Studio, read the
# documentation: https://d.android.com/studio/projects/add-native-code.html

# Sets the minimum version of CMake required to build the native library.

cmake_minimum_required(VERSION 3.4.1)

enable_language(ASM_NASM)

include(AndroidNdkModules)
android_ndk_import_module_cpufeatures()

# setup common C flags
set(CMAKE_C_FLAGS "-std=gnu17 -Os ${CMAKE_C_FLAGS}")

if(${ANDROID_ABI} STREQUAL "x86_64")
    set(asm_SRCS src/main/cpp/test.asm)
endif()

# Creates and names a library, sets it as either STATIC
# or SHARED, and provides the relative paths to its source code.
# You can define multiple libraries, and CMake builds them for you.
# Gradle automatically packages shared libraries with your APK.

add_library( # Sets the name of the library.
        native-lib

        # Sets the library as a shared library.
        SHARED

        # Provides a relative path to your source file(s).
        src/main/cpp/native-lib.c ${asm_SRCS})

# Searches for a specified prebuilt library and stores the path as a
# variable. Because CMake includes system libraries in the search path by
# default, you only need to specify the name of the public NDK library
# you want to add. CMake verifies that the library exists before
# completing its build.

find_library( # Sets the name of the path variable.
        log-lib

        # Specifies the name of the NDK library that
        # you want CMake to locate.
        log android)

# Specifies libraries CMake should link to your target library. You
# can link multiple libraries, such as libraries you define in this
# build script, prebuilt third-party libraries, or system libraries.

target_link_libraries( # Specifies the target library.
        native-lib cpufeatures

        # Links the target library to the log library
        # included in the NDK.
        ${log-lib})
```

<br />

在很多时候，我们在Android Studio中写C和汇编时可能还需要引入第三方的库，而第三方的库可能是静态库也可能是动态库。如果我们要把这些库引入到自己的项目中的话则需要添加对这些库的引用。下面，假定我们用独立的Android NDK在自己当前的项目中构建了一个名为“innerC”的动态库，如果我们想在Android Studio的C语言项目中引用它的话则需要先对`build.gradle (Module:app)`文件进行修改，在`buildTypes { }`语句块下面添加：
```gradle
    sourceSets {
        main {
            jniLibs.srcDirs = ['libs']
        }
    }
```

然后再对CMakeLists文件做以下修改：
首先，添加以下内容：
```cmake
add_library(innerC
        SHARED
        IMPORTED )

set_target_properties( # Specifies the target library.
        innerC

        # Specifies the parameter you want to define.
        PROPERTIES IMPORTED_LOCATION

        # Provides the path to the library you want to import.
        ${PROJECT_SOURCE_DIR}/libs/${ANDROID_ABI}/libinnerC.so )
```
然后，在`target_link_libraries`中添加：`innerC `。

<br />

如果我们所引用的innerC模块是静态库（.a）的形式，那么将上面两行CMake命令改为以下形式即可：
```cmake
add_library(innerC
        STATIC
        IMPORTED )

set_target_properties( # Specifies the target library.
        innerC

        # Specifies the parameter you want to define.
        PROPERTIES IMPORTED_LOCATION

        # Provides the path to the library you want to import.
        ${PROJECT_SOURCE_DIR}/obj/local/${ANDROID_ABI}/libinnerC.a)
```

顺便一提，如果我们想在独立的NDK项目中最终生成静态库文件，那么只需要将原来的`include $(BUILD_SHARED_LIBRARY)`指示语句改写为`include $(BUILD_STATIC_LIBRARY)`即可。如果我们想在Android Studio项目的C语言中包含第三方库的头文件，也可以在CMakeLists文件中使用`include_directories(imported-lib/include/ )`命令即可。

**这里还需要额外提供一些注意事项！** 在Android Studio 3.3之后，Google这个沙雕特么把`${PROJECT_SOURCE_DIR}`这个目录的位置换掉了，从以前的`<项目名>/app/`变为了：`<项目名>/app/src/main/cpp/`。所以各位用新的Android Studio从自带NDK的cmake做外部库连接的时候需要特别注意，可以将`/app/libs`这一文件夹拷贝到`/app/src/main/cpp/`目录下；也可以直接通过`../`的方式去找。笔者更偏向于第一种方案，第二种太过丑陋～

<br />

最后，笔者提供一下各位可用于参考的Google官方给的使用Android Studio来写JNI的例子：[https://github.com/googlesamples/android-ndk](https://github.com/googlesamples/android-ndk)。

这篇Wiki文章则比较详细地介绍了NASM的汇编基本用法：[https://en.wikipedia.org/wiki/Netwide_Assembler](https://en.wikipedia.org/wiki/Netwide_Assembler)。

由于YASM衍生自NASM，因此在语法和用法上都差不多，并且YASM使用了更自由的BSD License，因此各位可以毫无顾虑地使用～


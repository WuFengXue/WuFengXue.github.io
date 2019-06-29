---
layout: post
title: '微信语音编解码实现（四）—— 整合 so 库'
subtitle: '直接使用 JNI 调用实现微信语音编解码'
date: 2019-06-29
categories: 技术
cover: 'https://cn.bing.com/th?id=OHR.PantheraLeoDad_ZH-CN9580668524_1920x1080.jpg&rf=LaDigue_1920x1080.jpg&pid=hp'
tags: android JNI
---

## 微信语音编解码实现
* [（一）—— silk 移植](https://wufengxue.github.io/2019/03/12/wechat-voice-codec-silk.html)
* [（二）—— 支持微信语音](https://wufengxue.github.io/2019/04/17/wechat-voice-codec-amr.html)
* [（三）—— lame 移植](https://wufengxue.github.io/2019/05/25/wechat-voice-codec-lame.html)
* （四）—— 整合 so 库

## 配套源码
[WeChatVoiceCodec](https://github.com/WuFengXue/WeChatVoiceCodec)

## 1.文前
&#160; &#160; &#160; &#160;截至目前，我们已经可以实现微信语音在 android 平台的格式转换，如下：
```doc
wechat-voice <--silk--> pcm <--lame--> mp3
```

&#160; &#160; &#160; &#160;不足之处在于，当前只能通过 shell 命令进行调用，而且如果要实现微信语音和 mp3 格式互转，必须执行两次不同的命令。

&#160; &#160; &#160; &#160;基于此，本篇文章将在之前的基础上，直接将 silk 和 lame 库整合成一个 so 库，实现直接使用 JNI 调用完成微信语音的编解码。在实现过程，力争实现以下目标：

* 对 silk 和 lame 库的源码不做修改或少修改
* 易于调整参数

## 2.整合 so 库

### 2.1 新建 libwcvcodec (LibWeChatVoiceCodec) 库模块
* 将 silk 和 lame 应用模块的 cpp 部分拷贝到新的库模块下
* 修改模块的 build.gradle 文件，在 defaultConfig 项下添加如下内容：

```Gradle
android {
    ……
    defaultConfig {
        ……
        externalNativeBuild {
            cmake {
                cppFlags "-std=c++11 -fexceptions -pthread"
                // 要支持 'armeabi-v7a' 需开启 NO_ASM 宏
                cFlags "-DSTDC_HEADERS -DHAVE_LIMITS_H -DHAVE_MPGLIB -DNO_ASM"
            }
            ndk {
                abiFilters 'armeabi-v7a', 'arm64-v8a'
            }
        }
    }
}
```

* 修改模块的 build.gradle 文件，在 android 项下添加如下内容：

```Gradle
android {
    ……
    externalNativeBuild {
        cmake {
            path "CMakeLists.txt"
        }
    }
}
```

* 在模块顶级文件夹下新建 CMakeLists.txt 文件，内容如下：

```Cmake
# For more information about using CMake with Android Studio, read the
# documentation: https://d.android.com/studio/projects/add-native-code.html

# Sets the minimum version of CMake required to build the native library.

cmake_minimum_required(VERSION 3.6)
project(wcvcodec)

# silk
include_directories(src/main/cpp/silk/interface)
include_directories(src/main/cpp/silk/src)

aux_source_directory(src/main/cpp/silk/src SILK_SRC)


# lame
include_directories(src/main/cpp/lame/frontend)
include_directories(src/main/cpp/lame/include)
include_directories(src/main/cpp/lame/libmp3lame)
include_directories(src/main/cpp/lame/mpglib)

aux_source_directory(src/main/cpp/lame/frontend LAME_FRONTEND_SRC)
aux_source_directory(src/main/cpp/lame/libmp3lame LAME_SRC)
aux_source_directory(src/main/cpp/lame/mpglib LAME_MPGLIB_SRC)


# wcv codec (WeChat Voice Codec)
include_directories(src/main/cpp)
aux_source_directory(src/main/cpp/silk SILK_CODEC_SRC)
aux_source_directory(src/main/cpp/lame LAME_CODEC_SRC)
add_library(wcvcodec SHARED src/main/cpp/WcvCodec.c ${SILK_SRC} ${SILK_CODEC_SRC}
        ${LAME_FRONTEND_SRC} ${LAME_SRC} ${LAME_MPGLIB_SRC} ${LAME_CODEC_SRC})
find_library(android-log log)
target_link_libraries(wcvcodec ${android-log})
```

### 2.2 扩展 silk 编解码接口

* 在 xxx/libwcvcodec/src/main/cpp/silk 文件夹下新建 SilkCodec.h 文件，内容如下：

```C
/**
 * Silk decoder and encoder
 *
 * @author Reinhard（李剑波）
 * @date 2019/6/15
 */

#ifndef SILK_CODEC_H
#define SILK_CODEC_H

int silk_decoder_main(int argc, char *argv[]);

int silk_encoder_main(int argc, char *argv[]);

#endif //SILK_CODEC_H
```

* 在 xxx/libwcvcodec/src/main/cpp/silk 文件夹下新建 SilkCodec.c 文件
* 将 xxx/libwcvcodec/src/main/cpp/silk/test/Decoder.c 的全部内容拷贝到 SilkCodec.c
* 将方法 print_usage 重名为 print_decoder_usage
* 将方法 main 重名为 silk_decoder_main
* 将宏 MAX_BYTES_PER_FRAME 重命名为 MAX_BYTES_PER_FRAME_DECODER

* 将 xxx/libwcvcodec/src/main/cpp/silk/test/Encoder.c 的全部内容拷贝到 SilkCodec.c 的尾部
* 将方法 print_usage 重名为 print_encoder_usage
* 将方法 main 重名为 silk_encoder_main
* 将宏 MAX_BYTES_PER_FRAME 重命名为 MAX_BYTES_PER_FRAME_ENCODER
* 将宏 MAX_BYTES_PER_FRAME_ENCODER 移到 宏 MAX_BYTES_PER_FRAME_DECODER 的下方
* 移除方法 print_encoder_usage 之前引入的头文件和相关变量定义（与 Decoder.c 的重复）
* 在 cpp 文件下新建 android_log.h 文件，内容如下：

```C
/**
 * Android log macro definition
 *
 * @author Reinhard（李剑波）
 * @date 2019/6/15
 */

#ifndef ANDROID_LOG_H
#define ANDROID_LOG_H

#include <android/log.h>

#define LOG_TAG "WcvCodec"
#define LOGD(...) __android_log_print(ANDROID_LOG_DEBUG,LOG_TAG,__VA_ARGS__)
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO,LOG_TAG,__VA_ARGS__)
#define LOGW(...) __android_log_print(ANDROID_LOG_WARN,LOG_TAG,__VA_ARGS__)
#define LOGE(...) __android_log_print(ANDROID_LOG_ERROR,LOG_TAG,__VA_ARGS__)

#endif //ANDROID_LOG_H
```
* 在 SilkCodec.c 中引入 android_log.h，并将全部的 printf 调用替换成 LOGD 调用
* 在 SilkCodec.c 中引入 jni.h，并将全部的 exit( 0 ); 语句替换成 return JNI_ERR;

### 2.3 扩展 lame 编解码接口

* 在 xxx/libwcvcodec/src/main/cpp/lame 文件夹下新建 LameCodec.h 文件，内容如下：

```C
/**
 * Lame decoder and encoder
 *
 * @author Reinhard（李剑波）
 * @date 2019/6/22
 */

#ifndef LAME_CODEC_H
#define LAME_CODEC_H

int lame_codec_main(int argc, char *argv[]);

#endif //LAME_CODEC_H

```

* 在 xxx/libwcvcodec/src/main/cpp/lame 文件夹下新建 LameCodec.c 文件
* 将 xxx/libwcvcodec/src/main/cpp/lame/frontend/main.c 中的 c_main 方法拷贝到 Lamecodec.c
* 在 LameCodec.c 中新建 lame_codec_main 方法，内容如下：

```C
int lame_codec_main(int argc, char *argv[]) {
    return c_main(argc, argv);
}
```

* 在 LameCodec.c 中引入 android_log.h，并将 error_printf 调用替换成 LOGE 调用

### 2.4 创建 JNI 接口

* 在 xxx/libwcvcodec/src/main/java/com/reinhard/wcvcodec 文件夹下新建 WcvCodec.java 文件，内容如下：

```Java
package com.reinhard.wcvcodec;

/**
 * WeChat voice decoder and encoder
 *
 * @author Reinhard（李剑波）
 * @date 2019-06-22
 */
public class WcvCodec {
    static {
        System.loadLibrary("wcvcodec");
    }

    /**
     * decode amr to mp3 (1. amr -> pcm  2. pcm -> mp3)
     *
     * @param amrPath amr file path
     * @param pcmPath pcm file path
     * @param mp3Path mp3 file path
     * @return 0 if success, otherwise -1
     */
    public static native int decode(String amrPath, String pcmPath, String mp3Path);

    /**
     * encode pcm to amr
     *
     * @param pcmPath pcm file path
     * @param amrPath amr file path
     * @return 0 if success, otherwise -1
     */
    public static native int encode(String pcmPath, String amrPath);

    /**
     * encode mp3 to amr (1. mp3 -> pcm  2. pcm -> amr)
     *
     * @param mp3Path mp3 file path
     * @param pcmPath pcm file path
     * @param amrPath amr file path
     * @return 0 if success, otherwise -1
     */
    public static native int encode2(String mp3Path, String pcmPath, String amrPath);
}
```

* 在 AndroidStudio 中选中 libwcvcodec 模块，然后点击菜单 Build -> Make Module 'libwcvcodec' 编译模块
* 命令行切换到 xxx/libwcvcodec/build/intermediates/javac/debug/compileDebugJavaWithJavac/classes 文件夹
* 执行生成 JNI 头文件的命令，如下：

```Shell
javah com.reinhard.wcvcodec.WcvCodec
```

* 将 xxx/libwcvcodec/build/intermediates/javac/debug/compileDebugJavaWithJavac/classes 文件下自动生成的 com_reinhard_wcvcodec_WcvCodec.h 头文件拷贝到 xxx/libwcvcodec/src/main/cpp 文件夹，并重命名为 WcvCodec.h
* 在 xxx/libwcvcodec/src/main/cpp 文件夹下新建 WcvCodec.c 文件，内容如下：

```C
/**
 * WeChat voice decoder and encoder
 *
 * @author Reinhard（李剑波）
 * @date 2019/6/22
 */

#include <silk/SilkCodec.h>
#include <lame/LameCodec.h>
#include "WcvCodec.h"
#include "android_log.h"

/*
 * Class:     com_reinhard_wcvcodec_WcvCodec
 * Method:    decode
 * Signature: (Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;)I
 */
JNIEXPORT jint JNICALL Java_com_reinhard_wcvcodec_WcvCodec_decode
        (JNIEnv *env, jclass clazz, jstring amrPath, jstring pcmPath, jstring mp3Path) {
    const char *amr = (*env)->GetStringUTFChars(env, amrPath, JNI_FALSE);
    const char *pcm = (*env)->GetStringUTFChars(env, pcmPath, JNI_FALSE);
    const char *mp3 = (*env)->GetStringUTFChars(env, mp3Path, JNI_FALSE);
    int argc = 5;
    const char *argv[] = {"./Decoder", amr, pcm, "-stx_header", "-quiet"};
    if (silk_decoder_main(argc, (char **) argv) == JNI_OK) {
        int argc2 = 14;
        const char *argv2[] = {"./lame", "-q", "5", "-b", "128", "-m", "m", "-r",
                               "-s", "24000", "--resample", "24000", pcm, mp3};
        return lame_codec_main(argc2, (char **) argv2);
    } else {
        LOGE("silk_decoder_main failed!");
        return JNI_ERR;
    }
}

/*
 * Class:     com_reinhard_wcvcodec_WcvCodec
 * Method:    encode
 * Signature: (Ljava/lang/String;Ljava/lang/String;)I
 */
JNIEXPORT jint JNICALL Java_com_reinhard_wcvcodec_WcvCodec_encode
        (JNIEnv *env, jclass clazz, jstring pcmPath, jstring amrPath) {
    const char *pcm = (*env)->GetStringUTFChars(env, pcmPath, JNI_FALSE);
    const char *amr = (*env)->GetStringUTFChars(env, amrPath, JNI_FALSE);
    int argc = 7;
    const char *argv[] = {"./Encoder", pcm, amr, "-rate", "24000", "-stx_header", "-quiet"};
    return silk_encoder_main(argc, (char **) argv);
}

/*
 * Class:     com_reinhard_wcvcodec_WcvCodec
 * Method:    encode2
 * Signature: (Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;)I
 */
JNIEXPORT jint JNICALL Java_com_reinhard_wcvcodec_WcvCodec_encode2
        (JNIEnv *env, jclass clazz, jstring mp3Path, jstring pcmPath, jstring amrPath) {
    const char *mp3 = (*env)->GetStringUTFChars(env, mp3Path, JNI_FALSE);
    const char *pcm = (*env)->GetStringUTFChars(env, pcmPath, JNI_FALSE);
    const char *amr = (*env)->GetStringUTFChars(env, amrPath, JNI_FALSE);
    int argc = 5;
    const char *argv[] = {"./lame", "--decode", "-t", mp3, pcm};
    if (lame_codec_main(argc, (char **) argv) == JNI_OK) {
        int argc2 = 7;
        const char *argv2[] = {"./Encoder", pcm, amr, "-rate", "24000", "-stx_header", "-quiet"};
        return silk_encoder_main(argc2, (char **) argv2);
    } else {
        LOGE("lame_codec_main failed!");
        return JNI_ERR;
    }
}
```


## 4.测试（待实现）

### 4.1 在手机上创建测试文件夹

```shell
adb shell "mkdir -p /data/local/tmp/lame/"
```

### 4.2 将生成的可执行文件推到手机

```shell
cd xxx/WeChatVoiceCodec/lame/build/intermediates/cmake/release/obj/arm64-v8a/
adb push lame /data/local/tmp/lame/
```

### 4.3 将项目工程中测试用的 pcm 文件推到手机（由 [微信语音编解码实现（二）—— 支持微信语音](https://wufengxue.github.io/2019/04/17/wechat-voice-codec-amr.html) 生成）

### 4.6 测试编码

将生成的 tmp.mp3 拉到电脑播放，验证 OK。

### 4.7 测试解码

```shell
./lame --decode -t tmp.mp3 out.pcm
```

执行上述命令后，会在当前路径生成 out.pcm，并输出以下内容：

```shell
lavender:/data/local/tmp/lame # ./lame --decode -t  tmp.mp3 out.pcm
input:  tmp.mp3  (24 kHz, 1 channel, MPEG-2 Layer III)
output: out.pcm  (16 bit, Microsoft WAVE)
skipping initial 1105 samples (encoder+decoder delay)
skipping final 47 samples (encoder padding-decoder delay)
Frame#    72/72     128 kbps     
```

## 5. 参考

* [开源项目 - Silk_v3_decoder](https://github.com/fishCoder/Silk_v3_decoder)

* [CMakeLists File](https://www.jetbrains.com/help/clion/cmakelists-txt-file.html)


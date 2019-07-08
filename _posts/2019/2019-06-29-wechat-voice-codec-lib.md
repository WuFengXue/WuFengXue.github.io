---
layout: post
title: '微信语音编解码实现（四）—— 整合 so 库'
subtitle: '直接使用 JNI 调用实现微信语音编解码'
date: 2019-06-29
categories: 技术
cover: '../../../assets/img/post_bg/20190629.jpg'
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


## 3.测试

### 3.1 新建一个 app 模块

&#160; &#160; &#160; &#160;AndroidStudio->File->New->New Module...->Phone And Tablet Module->创建带一个空界面的应用

### 3.2 添加 libwcvcodec 库模块依赖

&#160; &#160; &#160; &#160;修改 app 模块的 build.gradle 文件，在 dependencies 项下添加如下内容：

```Gradle
dependencies {
    ……
    implementation project(path: ':libwcvcodec')
}
```

### 3.3 添加 sd 卡访问权限

&#160; &#160; &#160; &#160;修改 app 模块的清单文件 AndroidManifest.xml，添加如下内容：

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.reinhard.wechat.voicecodec">

    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />

    <application ...>
        ...
    </application>

</manifest>
```

### 3.4 添加测试按钮

&#160; &#160; &#160; &#160;修改 app 模块的布局文件 activity_main.xml，内容如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <Button
        android:id="@+id/btn_amr_to_mp3"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:onClick="onClick"
        android:text="amr_to_mp3"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintHorizontal_bias="0.5"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <Button
        android:id="@+id/btn_pcm_to_amr"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="32dp"
        android:onClick="onClick"
        android:text="pcm_to_amr"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintHorizontal_bias="0.5"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toBottomOf="@id/btn_amr_to_mp3" />

    <Button
        android:id="@+id/btn_mp3_to_amr"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="32dp"
        android:onClick="onClick"
        android:text="mp3_to_amr"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintHorizontal_bias="0.5"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toBottomOf="@id/btn_pcm_to_amr" />

</android.support.constraint.ConstraintLayout>
```

### 3.5 编写测试代码

&#160; &#160; &#160; &#160;修改 app 模块的 MainActivity，内容如下：

```java
package com.reinhard.wechat.voicecodec;

import android.app.Activity;
import android.os.Bundle;
import android.util.Log;
import android.view.View;
import android.widget.Toast;

import com.reinhard.wcvcodec.WcvCodec;

public class MainActivity extends Activity {
    private static final String TAG = "WcvCodec";
    private static final String TEST_DIR = "/sdcard/reinhard/";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }

    public void onClick(View view) {
        int id = view.getId();
        switch (id) {
            case R.id.btn_amr_to_mp3:
                testAmrToMp3();
                break;
            case R.id.btn_pcm_to_amr:
                testPcmToAmr();
                break;
            case R.id.btn_mp3_to_amr:
                testMp3ToAmr();
                break;
            default:
                break;
        }
    }

    private void testAmrToMp3() {
        Log.d(TAG, "testAmrToMp3");
        String amrPath = TEST_DIR + "in.amr";
        String pcmPath = TEST_DIR + "out.pcm";
        String mp3Path = TEST_DIR + "out.mp3";
        if (WcvCodec.decode(amrPath, pcmPath, mp3Path) == 0) {
            Toast.makeText(this, "testAmrToMp3 success", Toast.LENGTH_SHORT)
                    .show();
        }
    }

    private void testPcmToAmr() {
        Log.d(TAG, "testPcmToAmr");
        String pcmPath = TEST_DIR + "in.pcm";
        String amrPath = TEST_DIR + "out.amr";
        if (WcvCodec.encode(pcmPath, amrPath) == 0) {
            Toast.makeText(this, "testPcmToAmr success", Toast.LENGTH_SHORT)
                    .show();
        }
    }

    private void testMp3ToAmr() {
        Log.d(TAG, "testMp3ToAmr");
        String mp3Path = TEST_DIR + "in.mp3";
        String pcmPath = TEST_DIR + "out.pcm";
        String amrPath = TEST_DIR + "out.amr";
        if (WcvCodec.encode2(mp3Path, pcmPath, amrPath) == 0) {
            Toast.makeText(this, "testMp3ToAmr success", Toast.LENGTH_SHORT)
                    .show();
        }
    }
}
```

&#160; &#160; &#160; &#160;___注意：demo 只做简单测试，所以直接将操作放在 UI 线程，实际使用时，应放到其他线程。___

### 3.6 将测试用的音频文件推送到手机

```shell
cd xxx/WeChatVoiceCodec/libwcvcodec/test_vectors/
adb shell mkdir -p /sdcard/reinhard/
adb push wechat_voice.amr /sdcard/reinhard/in.amr
adb push wechat_voice.mp3 /sdcard/reinhard/in.mp3
adb push wechat_voice.pcm /sdcard/reinhard/in.pcm
```

### 3.7 将编译生成的 apk 安装到手机，并给予 sd 卡访问权限
&#160; &#160; &#160; &#160;_说明：因为只是 demo，没有引入运行时权限申请，必要时可能需要手动到设置中配置。_

### 3.8 测试编解码

&#160; &#160; &#160; &#160;打开应用后，点击按钮进行测试，成功时会弹出吐司提示，并在 /sdcard/reinhard/ 文件夹生成对应的文件。按钮与测试功能对应关系如下：

* amr_to_mp3 按钮：测试 amr->pcm->mp3
* pcm_to_amr 按钮：测试 pcm->amr
* mp3_to_amr 按钮：测试 mp3->pcm->amr

&#160; &#160; &#160; &#160;可以将生成的文件拉取到电脑进行验证：

* mp3 可以直接用电脑进行播放
* amr 因为是微信语音文件，需要专门的方法进行验证，这里就不提供了

## 4. 总结
* 在某些场景下，通过扩展一个新的入口（类似代理），可以实现不修改或少修改第三方库的源码结构，便于后续同步库的更新内容
* 将二进制程序移植为接口调用时，通过间接调用 main 方法（传入完整 cmd），可以大大提高灵活性

## 5. 回顾与后续

&#160; &#160; &#160; &#160;到此，本系列的文章基本完成了。

&#160; &#160; &#160; &#160;在这些文章中，我尝试着去解答了为什么移植和怎么移植 silk 和 lame 库的问题，但还是存在一些疑惑和可以改进的点：

* 疑惑：在调用 silk 和 lame 时，传入的参数配置是怎么确定的？（大部分直接来自参考资源）
* 待改进：将库编译后上传到 maven 之类的托管网站，实现直接使用 gradle 依赖导入

&#160; &#160; &#160; &#160;对于疑惑点，可能需要进一步学习语音编解码相关的知识才能解答；至于待改进的点，就留待以后有时间了再实现吧。

## 6. 参考

* [开源项目 - Silk_v3_decoder](https://github.com/fishCoder/Silk_v3_decoder)

* [CMakeLists File](https://www.jetbrains.com/help/clion/cmakelists-txt-file.html)


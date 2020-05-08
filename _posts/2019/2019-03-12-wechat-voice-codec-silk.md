---
layout: post
title: '微信语音编解码实现（一）—— silk 移植'
subtitle: 'AndroidStudio 编译 silk 生成可在手机上运行的二进制文件'
date: 2019-03-12
categories: 技术
cover: '../../../assets/img/post_bg/20190312.jpg'
tags: android wechat silk
---

## 微信语音编解码实现
* （一）—— silk 移植
* [（二）—— 支持微信语音](https://wufengxue.github.io/2019/04/17/wechat-voice-codec-amr.html)
* [（三）—— lame 移植](https://wufengxue.github.io/2019/05/25/wechat-voice-codec-lame.html)
* [（四）—— 整合 so 库](https://wufengxue.github.io/2019/06/29/wechat-voice-codec-lib.html)

## 配套源码
[WeChatVoiceCodec](https://github.com/WuFengXue/WeChatVoiceCodec)

## 关于 silk
&#160; &#160; &#160; &#160;[silk](https://en.wikipedia.org/wiki/SILK) 是 [skype](https://baike.baidu.com/item/skype/202823?fr=aladdin) 开源的一款语音编解码器，被微信的语音文件所采用。

## 移植步骤

### 1、下载 silk 源码
&#160; &#160; &#160; &#160;silk 的官方网站 [http://developer.skype.com/silk](http://developer.skype.com/silk) 现在已经无法访问，但是 github 上可以找到别人备份的版本，[源码下载通道](https://github.com/zly394/SilkDecoder/blob/master/SILK_SDK_SRC_v1.0.9.zip)。

### 2、拷贝源码到工程

&#160; &#160; &#160; &#160;下载完成后，解压，将 SILK_SDK_SRC_ARM_v1.0.9 中的 interface、src 和 test 文件夹拷贝到项目模块 xxx/src/main/cpp/silk/ 文件夹下

### 3、模块 build.gradle 文件 android 项下的 defaultConfig 项添加以下内容

```gradle
android {
...
    defaultConfig {
    ...
        externalNativeBuild {
            cmake {
                cppFlags "-std=c++11 -frtti -fexceptions"
                // 要支持 'armeabi-v7a' 需开启 NO_ASM 宏
                cFlags "-DNO_ASM"
            }
        }
    }
...
}
```

### 4、模块 build.gradle 文件的 android 项添加以下内容

```gradle
android {
...
    externalNativeBuild {
        cmake {
            path "CMakeLists.txt"
        }
    }
}
```

### 5、模块顶级文件夹下新增 CMakeLists.txt 文件，内容如下:

```cmake
# For more information about using CMake with Android Studio, read the
# documentation: https://d.android.com/studio/projects/add-native-code.html

# Sets the minimum version of CMake required to build the native library.

cmake_minimum_required(VERSION 3.4.1)

include_directories(src/main/cpp/silk/interface)
include_directories(src/main/cpp/silk/src)

aux_source_directory(src/main/cpp/silk/src SILK_SRC)

add_executable(decoder src/main/cpp/silk/test/Decoder.c ${SILK_SRC})
add_executable(encoder src/main/cpp/silk/test/Encoder.c ${SILK_SRC})
add_executable(signalcompare src/main/cpp/silk/test/signalCompare.c ${SILK_SRC})
```

### 6、编译模块

&#160; &#160; &#160; &#160;编译成功后，将在 xxx/build/intermediates/cmake/debug/obj/arm64-v8a 路径下生成 3 个文件：decoder、encoder 和 signalcompare

## 测试

### 1、在手机上创建测试文件夹

```linux
adb shell "mkdir /data/local/tmp/silk/"
```

### 2、将生成的可执行文件推到手机

```linux
cd xxx/build/intermediates/cmake/debug/obj/arm64-v8a/
adb push decoder /data/local/tmp/silk/
adb push encoder /data/local/tmp/silk/
adb push signalcompare /data/local/tmp/silk/
```

### 3、将 silk 源码自带的测试文件夹推到手机

```linux
cd xxx/SILK_SDK_SRC_ARM_v1.0.9/
adb push test_vectors /data/local/tmp/silk/
```

### 4、添加执行权限

```linux
adb shell "chmod 777 -R /data/local/tmp/silk/"
```

### 5、执行解码测试脚本

```linux
adb shell
cd /data/local/tmp/silk/test_vectors/
bash test_decoder.sh
```

&#160; &#160; &#160; &#160;执行完会在当前路径生成 test_decoder_report.txt 文件，其内容如下：

```linux
Reference:  ./test_vectors/output/testvector_output_8_kHz_60_ms_8_kbps.pcm
Signals are bit-exact          PASS

Reference:  ./test_vectors/output/testvector_output_8_kHz_60_ms_8_kbps_8_kHz_out.pcm
Signals are bit-exact          PASS

Reference:  ./test_vectors/output/testvector_output_8_kHz_60_ms_8_kbps_12_kHz_out.pcm
Signals are bit-exact          PASS

Reference:  ./test_vectors/output/testvector_output_8_kHz_60_ms_8_kbps_16_kHz_out.pcm
Signals are bit-exact          PASS

Reference:  ./test_vectors/output/testvector_output_8_kHz_40_ms_12_kbps.pcm
Signals are bit-exact          PASS

Reference:  ./test_vectors/output/testvector_output_8_kHz_20_ms_20_kbps_10_loss_FEC.pcm
Signals are bit-exact          PASS

Reference:  ./test_vectors/output/testvector_output_12_kHz_60_ms_10_kbps.pcm
Signals are bit-exact          PASS

Reference:  ./test_vectors/output/testvector_output_12_kHz_60_ms_10_kbps_12_kHz_out.pcm
Signals are bit-exact          PASS

Reference:  ./test_vectors/output/testvector_output_12_kHz_60_ms_10_kbps_16_kHz_out.pcm
Signals are bit-exact          PASS

Reference:  ./test_vectors/output/testvector_output_12_kHz_60_ms_10_kbps_32_kHz_out.pcm
Signals are bit-exact          PASS

Reference:  ./test_vectors/output/testvector_output_12_kHz_60_ms_10_kbps_44100_Hz_out.pcm
Signals are bit-exact          PASS

Reference:  ./test_vectors/output/testvector_output_12_kHz_60_ms_10_kbps_48_kHz_out.pcm
Signals are bit-exact          PASS

Reference:  ./test_vectors/output/testvector_output_12_kHz_40_ms_16_kbps.pcm
Signals are bit-exact          PASS

Reference:  ./test_vectors/output/testvector_output_12_kHz_20_ms_24_kbps_10_loss_FEC.pcm
Signals are bit-exact          PASS

Reference:  ./test_vectors/output/testvector_output_16_kHz_60_ms_12_kbps.pcm
Signals are bit-exact          PASS

Reference:  ./test_vectors/output/testvector_output_16_kHz_60_ms_12_kbps_16_kHz_out.pcm
Signals are bit-exact          PASS

Reference:  ./test_vectors/output/testvector_output_16_kHz_40_ms_20_kbps.pcm
Signals are bit-exact          PASS

Reference:  ./test_vectors/output/testvector_output_16_kHz_20_ms_32_kbps_10_loss_FEC.pcm
Signals are bit-exact          PASS

Reference:  ./test_vectors/output/testvector_output_24_kHz_60_ms_16_kbps.pcm
Signals are bit-exact          PASS

Reference:  ./test_vectors/output/testvector_output_24_kHz_40_ms_24_kbps.pcm
Signals are bit-exact          PASS

Reference:  ./test_vectors/output/testvector_output_24_kHz_20_ms_40_kbps_10_loss_FEC.pcm
Signals are bit-exact          PASS

Reference:  ./test_vectors/output/testvector_output_32_kHz_max_8_kHz_20_ms_8_kbps.pcm
Signals are bit-exact          PASS

Reference:  ./test_vectors/output/testvector_output_32_kHz_max_8_kHz_20_ms_8_kbps_32_kHz_out.pcm
Signals are bit-exact          PASS

Reference:  ./test_vectors/output/testvector_output_32_kHz_max_8_kHz_20_ms_8_kbps_44100_Hz_out.pcm
Signals are bit-exact          PASS

Reference:  ./test_vectors/output/testvector_output_32_kHz_max_8_kHz_20_ms_8_kbps_48_kHz_out.pcm
Signals are bit-exact          PASS

Reference:  ./test_vectors/output/testvector_output_44100_Hz_20_ms_7_kbps.pcm
Signals are bit-exact          PASS

```

### 6、执行编码测试脚本

```linux
adb shell
cd /data/local/tmp/silk/test_vectors/
bash test_encoder.sh
```

&#160; &#160; &#160; &#160;执行完会在当前路径生成 test_encoder_report.txt 文件，其内容如下：

```linux
Reference:  ./test_vectors/bitstream/payload_8_kHz_60_ms_8_kbps.bit
Signals are bit-exact          PASS

Reference:  ./test_vectors/bitstream/payload_8_kHz_40_ms_12_kbps.bit
Signals are bit-exact          PASS

Reference:  ./test_vectors/bitstream/payload_8_kHz_20_ms_20_kbps_10_loss_FEC.bit
Signals are bit-exact          PASS

Reference:  ./test_vectors/bitstream/payload_12_kHz_60_ms_10_kbps.bit
Signals are bit-exact          PASS

Reference:  ./test_vectors/bitstream/payload_12_kHz_40_ms_16_kbps.bit
Signals are bit-exact          PASS

Reference:  ./test_vectors/bitstream/payload_12_kHz_20_ms_24_kbps_10_loss_FEC.bit
Signals are bit-exact          PASS

Reference:  ./test_vectors/bitstream/payload_16_kHz_60_ms_12_kbps.bit
Signals are bit-exact          PASS

Reference:  ./test_vectors/bitstream/payload_16_kHz_40_ms_20_kbps.bit
Signals are bit-exact          PASS

Reference:  ./test_vectors/bitstream/payload_16_kHz_20_ms_32_kbps_10_loss_FEC.bit
Signals are bit-exact          PASS

Reference:  ./test_vectors/bitstream/payload_24_kHz_60_ms_16_kbps.bit
Signals are bit-exact          PASS

Reference:  ./test_vectors/bitstream/payload_24_kHz_40_ms_24_kbps.bit
Signals are bit-exact          PASS

Reference:  ./test_vectors/bitstream/payload_24_kHz_20_ms_40_kbps_10_loss_FEC.bit
Signals are bit-exact          PASS

Reference:  ./test_vectors/bitstream/payload_32_kHz_max_8_kHz_20_ms_8_kbps.bit
Signals are bit-exact          PASS

Reference:  ./test_vectors/bitstream/payload_44100_Hz_20_ms_7_kbps.bit
Signals are bit-exact          PASS

```

## 参考文章

[Silk编解码在android实现](https://blog.csdn.net/xyz_lmn/article/details/8015619)

[CMakeLists File](https://www.jetbrains.com/help/clion/cmakelists-txt-file.html)




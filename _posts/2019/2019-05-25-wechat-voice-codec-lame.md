---
layout: post
title: '微信语音编解码实现（三）—— lame 移植'
subtitle: '使用 lame 实现 pcm 和 mp3 格式的相互转换'
date: 2019-05-25
categories: 技术
cover: '../../../assets/img/post_bg/20190525.jpg'
tags: android lame
---

## 微信语音编解码实现
* [（一）—— silk 移植](https://wufengxue.github.io/2019/03/12/wechat-voice-codec-silk.html)
* [（二）—— 支持微信语音](https://wufengxue.github.io/2019/04/17/wechat-voice-codec-amr.html)
* （三）—— lame 移植
* [（四）—— 整合 so 库](https://wufengxue.github.io/2019/06/29/wechat-voice-codec-lib.html)
* [（五）——工作线程编解码](https://wufengxue.github.io/2020/06/22/wechat-voice-codec-SEGV_MAPERR.html)

## 配套源码
[WeChatVoiceCodec](https://github.com/WuFengXue/WeChatVoiceCodec)

## 1.关于 lame
&#160; &#160; &#160; &#160;[lame](https://baike.baidu.com/item/LAME/1022693?fr=aladdin) 是一款开源的高品质 mp3 编码器，最新的版本是 v3.100。

## 2.移植步骤

### 2.1 下载和拷贝
* [下载](http://lame.sourceforge.net/download.php) lame 源码
* 将 lame 源码中的 frontend、include、libmp3lame 和 mpglib 文件夹拷贝到模块的 src/main/cpp/lame/ 文件夹下
* 删除 frontend 文件夹下 mp3rtp 相关的源文件：mp3rtp.c rtp.c
* 删除 frontend 文件夹下 mp3x 相关的源文件：mp3x.c gtkanal.c gpkplotting.c

### 2.2 修改和添加

* 修改 src/main/cpp/lame/frontend/main.h 文件，添加 ieee754_float32_t 的宏定义，内容如下：

```c
/**
 * Added by Reinhard in 20190523--start
 */
#ifndef HAVE_IEEE754_FLOAT32_T
    typedef float ieee754_float32_t;
#endif
/**
 * Added by Reinhard in 20190523--end
 */
```

*  修改 src/main/cpp/lame/libmp3lame/util.h 文件，添加 ieee754_float32_t 的宏定义，内容如下：

```c
/**
 * Added by Reinhard in 20190523--start
 */
#ifndef HAVE_IEEE754_FLOAT32_T
    typedef float ieee754_float32_t;
#endif
/**
 * Added by Reinhard in 20190523--end
 */
```

* 修改模块的 build.gradle 文件，在 defaultConfig 项下添加如下内容：

```gradle
android {
    ……
    defaultConfig {
        ……
        externalNativeBuild {
            cmake {
                cppFlags "-std=c++11 -fexceptions -pthread"
                cFlags "-DSTDC_HEADERS -DHAVE_LIMITS_H -DHAVE_MPGLIB"
            }
            ndk {
                abiFilters 'armeabi-v7a', 'arm64-v8a'
            }
        }
    }
}
```

* 修改模块的 build.gradle 文件，在 android 项下添加如下内容：

```gradle
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

```cmake
cmake_minimum_required(VERSION 3.6)
project(lame)

include_directories(src/main/cpp/lame/frontend)
include_directories(src/main/cpp/lame/include)
include_directories(src/main/cpp/lame/libmp3lame)
include_directories(src/main/cpp/lame/mpglib)

aux_source_directory(src/main/cpp/lame/frontend LAME_FRONTEND_SRC)
aux_source_directory(src/main/cpp/lame/libmp3lame LAME_SRC)
aux_source_directory(src/main/cpp/lame/mpglib LAME_MPGLIB_SRC)
add_executable(lame ${LAME_FRONTEND_SRC} ${LAME_SRC} ${LAME_MPGLIB_SRC})
```

## 3.编译错误及解决方案
### 3.1 fatal error: 'gtk/gtk.h' file not found
* [gtk](https://www.gtk.org) 是一款跨平台的图形工具包（[百度百科](https://baike.baidu.com/item/gtk/3138659?fr=aladdin)）
* MP3X (the LAME frame analyzer)  依赖了 gtk （见 lame 源码/README.WINGTK）
* 手机端无需该图形工具，因此可以直接移除依赖 gtk 的源码
* 解决方案：
	* 删除 frontend 文件夹下 mp3rtp 相关的源文件：mp3rtp.c rtp.c
	* 删除 frontend 文件夹下 mp3x 相关的源文件：mp3x.c gtkanal.c gpkplotting.c
	* 源文件的选取见 lame 源码/frontend/Makefile.in 文件 mp3rtp_SOURCES 和 mp3x_SOURCES 字段的值

### 3.2 error: unknown type name 'ieee754_float32_t'

* 修改 src/main/cpp/lame/frontend/main.h 文件，添加 ieee754_float32_t 的宏定义
*  修改 src/main/cpp/lame/libmp3lame/util.h 文件，添加 ieee754_float32_t 的宏定义
*  ieee754_float32_t 的定义见 lame 源码/configure.in 文件，如下：

```c
AC_CHECK_TYPES([ieee754_float64_t, ieee754_float32_t])

AH_VERBATIM([HAVE_IEEE754_FLOAT64_T],
[/* add ieee754_float64_t type */
#undef HAVE_IEEE754_FLOAT64_T
#ifndef HAVE_IEEE754_FLOAT64_T
	typedef double ieee754_float64_t;
#endif])

AH_VERBATIM([HAVE_IEEE754_FLOAT32_T],
[/* add ieee754_float32_t type */
#undef HAVE_IEEE754_FLOAT32_T
#ifndef HAVE_IEEE754_FLOAT32_T
	typedef float ieee754_float32_t;
#endif])
```

### 3.3 undefined reference to 'bcopy'

* cFlags 添加 -DSTDC_HEADERS

### 3.4 error: use of undeclared identifier 'LONG_MAX'

* cFlags 添加 -DHAVE_LIMITS_H

### 3.5 执行解码操作（./lame --decode xxx）时报错 Error: libmp3lame not compiled with mpg123 *decoding* support
* 添加 lame 源码的 mpglib 文件夹
* cFlags 添加 -DHAVE_MPGLIB


## 4.测试

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

```shell
cd xxx/WeChatVoiceCodec/lame/test_vectors/
adb push wechat_voice.pcm /data/local/tmp/lame/in.pcm
```

### 4.4 添加执行权限

```shell
adb shell "chmod -R 777 /data/local/tmp/lame/"
```

### 4.5 查看使用说明

```shell
adb shell
cd /data/local/tmp/lame/
./lame --help
```
执行完上述命令，可以看到如下输出：

```shell
1|lavender:/data/local/tmp/lame # ./lame --help                                
LAME 64bits version 3.100 (http://lame.sf.net)

usage: ./lame [options] <infile> [outfile]

    <infile> and/or <outfile> can be "-", which means stdin/stdout.

RECOMMENDED:
    lame -V2 input.wav output.mp3

OPTIONS:
    -b bitrate      set the bitrate, default 128 kbps
    -h              higher quality, but a little slower.
    -f              fast mode (lower quality)
    -V n            quality setting for VBR.  default n=4
                    0=high quality,bigger files. 9.999=smaller files
    --preset type   type must be "medium", "standard", "extreme", "insane",
                    or a value for an average desired bitrate and depending
                    on the value specified, appropriate quality settings will
                    be used.
                    "--preset help" gives more info on these

    --help id3      ID3 tagging related options

    --longhelp      full list of options

    --license       print License information

```

执行查看完整帮助的命令

```shell
./lame --longhelp
```

可以看到如下输出（其中包含解码的选项 --decode）：

```shell
lavender:/data/local/tmp/lame # ./lame --longhelp                              
LAME 64bits version 3.100 (http://lame.sf.net)

usage: ./lame [options] <infile> [outfile]

    <infile> and/or <outfile> can be "-", which means stdin/stdout.

RECOMMENDED:
    lame -V2 input.wav output.mp3

OPTIONS:
  Input options:
    --scale <arg>   scale input (multiply PCM data) by <arg>
    --scale-l <arg> scale channel 0 (left) input (multiply PCM data) by <arg>
    --scale-r <arg> scale channel 1 (right) input (multiply PCM data) by <arg>
    --swap-channel  swap L/R channels
    --ignorelength  ignore file length in WAV header
    --gain <arg>    apply Gain adjustment in decibels, range -20.0 to +12.0
    --mp1input      input file is a MPEG Layer I   file
    --mp2input      input file is a MPEG Layer II  file
    --mp3input      input file is a MPEG Layer III file
    --nogap <file1> <file2> <...>
                    gapless encoding for a set of contiguous files
    --nogapout <dir>
                    output dir for gapless encoding (must precede --nogap)
    --nogaptags     allow the use of VBR tags in gapless encoding
    --out-dir <dir> output dir, must exist

  Input options for RAW PCM:
    -r              input is raw pcm
    -s sfreq        sampling frequency of input file (kHz) - default 44.1 kHz
    --signed        input is signed (default)
    --unsigned      input is unsigned
    --bitwidth w    input bit width is w (default 16)
    -x              force byte-swapping of input
    --little-endian input is little-endian (default)
    --big-endian    input is big-endian
    -a              downmix from stereo to mono file for mono encoding


  Operational options:
    -m <mode>       (j)oint, (s)imple, (f)orce, (d)ual-mono, (m)ono (l)eft (r)ight
                    default is (j)
                    joint  = Uses the best possible of MS and LR stereo
                    simple = force LR stereo on all frames
                    force  = force MS stereo on all frames.
    --preset type   type must be "medium", "standard", "extreme", "insane",
                    or a value for an average desired bitrate and depending
                    on the value specified, appropriate quality settings will
                    be used.
                    "--preset help" gives more info on these
    --comp  <arg>   choose bitrate to achieve a compression ratio of <arg>
    --replaygain-fast   compute RG fast but slightly inaccurately (default)
    --noreplaygain  disable ReplayGain analysis
    --flush         flush output stream as soon as possible
    --freeformat    produce a free format bitstream
    --decode        input=mp3 file, output=wav
    -t              disable writing wav header when using --decode


  Verbosity:
    --disptime <arg>print progress report every arg seconds
    -S              don't print progress report, VBR histograms
    --nohist        disable VBR histogram display
    --quiet         don't print anything on screen
    --silent        don't print anything on screen, but fatal errors
    --brief         print more useful information
    --verbose       print a lot of useful information

  Noise shaping & psycho acoustic algorithms:
    -q <arg>        <arg> = 0...9.  Default  -q 3 
                    -q 0:  Highest quality, very slow 
                    -q 9:  Poor quality, but fast 
    -h              Same as -q 2.   
    -f              Same as -q 7.   Fast, ok quality


  CBR (constant bitrate, the default) options:
    -b <bitrate>    set the bitrate in kbps, default 128 kbps
    --cbr           enforce use of constant bitrate

  ABR options:
    --abr <bitrate> specify average bitrate desired (instead of quality)

  VBR options:
    -V n            quality setting for VBR.  default n=4
                    0=high quality,bigger files. 9=smaller files
    -v              the same as -V 4
    --vbr-old       use old variable bitrate (VBR) routine
    --vbr-new       use new variable bitrate (VBR) routine (default)
    -Y              lets LAME ignore noise in sfb21, like in CBR
                    (Default for V3 to V9.999)
    -b <bitrate>    specify minimum allowed bitrate, default  32 kbps
    -B <bitrate>    specify maximum allowed bitrate, default 320 kbps
    -F              strictly enforce the -b option, for use with players that
                    do not support low bitrate mp3
    -t              disable writing LAME Tag
    -T              enable and force writing LAME Tag

  MP3 header/stream options:
    -e <emp>        de-emphasis n/5/c  (obsolete)
    -c              mark as copyright
    -o              mark as non-original
    -p              error protection.  adds 16 bit checksum to every frame
                    (the checksum is computed correctly)
    --nores         disable the bit reservoir
    --strictly-enforce-ISO   comply as much as possible to ISO MPEG spec
    --buffer-constraint <constraint> available values for constraint:
                                     default, strict, maximum

  Filter options:
  --lowpass <freq>        frequency(kHz), lowpass filter cutoff above freq
  --lowpass-width <freq>  frequency(kHz) - default 15% of lowpass freq
  --highpass <freq>       frequency(kHz), highpass filter cutoff below freq
  --highpass-width <freq> frequency(kHz) - default 15% of highpass freq
  --resample <sfreq>  sampling frequency of output file(kHz)- default=automatic


  ID3 tag options:
    --tt <title>    audio/song title (max 30 chars for version 1 tag)
    --ta <artist>   audio/song artist (max 30 chars for version 1 tag)
    --tl <album>    audio/song album (max 30 chars for version 1 tag)
    --ty <year>     audio/song year of issue (1 to 9999)
    --tc <comment>  user-defined text (max 30 chars for v1 tag, 28 for v1.1)
    --tn <track[/total]>   audio/song track number and (optionally) the total
                           number of tracks on the original recording. (track
                           and total each 1 to 255. just the track number
                           creates v1.1 tag, providing a total forces v2.0).
    --tg <genre>    audio/song genre (name or number in list)
    --ti <file>     audio/song albumArt (jpeg/png/gif file, v2.3 tag)
    --tv <id=value> user-defined frame specified by id and value (v2.3 tag)
                    syntax: --tv "TXXX=description=content"
    --add-id3v2     force addition of version 2 tag
    --id3v1-only    add only a version 1 tag
    --id3v2-only    add only a version 2 tag
    --space-id3v1   pad version 1 tag with spaces instead of nulls
    --pad-id3v2     same as '--pad-id3v2-size 128'
    --pad-id3v2-size <value> adds version 2 tag, pad with extra <value> bytes
    --genre-list    print alphabetically sorted ID3 genre list and exit
    --ignore-tag-errors  ignore errors in values passed for tags

    Note: A version 2 tag will NOT be added unless one of the input fields
    won't fit in a version 1 tag (e.g. the title string is longer than 30
    characters), or the '--add-id3v2' or '--id3v2-only' options are used,
    or output is redirected to stdout.

Misc:
    --license       print License information


MPEG-1   layer III sample frequencies (kHz):  32  48  44.1
bitrates (kbps): 32 40 48 56 64 80 96 112 128 160 192 224 256 320

MPEG-2   layer III sample frequencies (kHz):  16  24  22.05
bitrates (kbps):  8 16 24 32 40 48 56 64 80 96 112 128 144 160

MPEG-2.5 layer III sample frequencies (kHz):   8  12  11.025
bitrates (kbps):  8 16 24 32 40 48 56 64

```

### 4.6 测试编码

```shell
./lame -q 5 -b 128 -m m -r -s 24000 --resample 24000 in.pcm tmp.mp3
```
执行上述命令后，会在当前路径生成 tmp.mp3，并输出以下内容：

```shell
lavender:/data/local/tmp/lame #                                                 /lame -q 5 -b 128 -m m -r -s 24000 --resample 24000  in.pcm tmp.mp3           < Assuming raw pcm input file                
LAME 3.100 64bits (http://lame.sf.net)
polyphase lowpass filter disabled
Encoding in.pcm to tmp.mp3
Encoding as 24 kHz single-ch MPEG-2 Layer III (3x) 128 kbps qval=5
    Frame          |  CPU time/estim | REAL time/estim | play/CPU |    ETA 
    72/72    (100%)|    0:00/    0:00|    0:00/    0:00|   51.625x|    0:00 
-------------------------------------------------------------------------------
   kbps       mono %     long switch short %                                   
  128.0      100.0        94.4   2.8   2.8                                     
Writing LAME Tag...done
ReplayGain: +2.2dB
```

执行拉取命令

```shell
adb pull /data/local/tmp/lame/tmp.mp3 ./
```

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

* [Cmake知识----编写CMakeLists.txt文件编译C/C++程序](https://blog.csdn.net/hebbely/article/details/79169965)

* [ndk编译android的lame库](https://www.cnblogs.com/yuanxiaoping_21cn_com/p/5634051.html)

* [Android NDK for x86_64 has no reference for bcopy and index](https://stackoverflow.com/questions/27893149/android-ndk-for-x86-64-has-no-reference-for-bcopy-and-index)

* [Lame MP3 Encoder compile for Android](https://stackoverflow.com/questions/8632835/lame-mp3-encoder-compile-for-android/8632948#8632948)

* [How to add Lame 3.99.5 to Android Studio using NDK? 
[closed]](https://stackoverflow.com/questions/34812738/how-to-add-lame-3-99-5-to-android-studio-using-ndk/34813030#34813030)

* [How to decode mp3 into wav using lame in C/C++?](https://stackoverflow.com/questions/7339408/how-to-decode-mp3-into-wav-using-lame-in-c-c)

* [AndroidStudio用Cmake方式编译NDK代码](https://blog.csdn.net/joe544351900/article/details/53637549)



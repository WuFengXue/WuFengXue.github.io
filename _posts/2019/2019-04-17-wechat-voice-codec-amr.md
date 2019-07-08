---
layout: post
title: '微信语音编解码实现（二）—— 支持微信语音'
subtitle: '使用 silk 编解码微信语音文件'
date: 2019-04-17
categories: 技术
cover: '../../../assets/img/post_bg/20190417.jpg'
tags: android wechat silk
---

## 微信语音编解码实现
* [（一）—— silk 移植](https://wufengxue.github.io/2019/03/12/wechat-voice-codec-silk.html)
* （二）—— 支持微信语音

## 配套源码
[WeChatVoiceCodec](https://github.com/WuFengXue/WeChatVoiceCodec)

## 1. silk 文件格式
&#160; &#160; &#160; &#160;silk 的文件格式如下：
```
   +------------------+
   | Header           |
   +-----------+------+
   | block 1   |
   +-----------+--+
   | block 2      |
   +--------------+--+
   : ...             :
   +--------------+--+
   | block n         |
   +-----------------+
   | Footer          |
   +-----------------+
```

### 1.1 头部（Header）

&#160; &#160; &#160; &#160;头部由以下 ASCII 字符串组成：
```
   "#!SILK_V3" （十六进制：0x23 0x21 0x53 0x49 0x4C 0x4B 0x5F 0x56 0x33）
```

### 1.2 数据块（block）

```
0                   16                                  16+8*nBytes
+-------------------+---------------------------------------------- 
|   字节数 (nBytes)  |                音频数据 
+-------------------+----------------------------------------------
```
&#160; &#160; &#160; &#160;数据块的前 2 个字节为 nBytes（short 类型），用于存储音频数据的字节数，后面紧跟 nBytes 个字节的音频数据。

### 1.3 尾部（Footer）

```
0                   15
+-------------------+
|   F   F   F   F   |
+-------------------+
```

&#160; &#160; &#160; &#160;尾部固定为 2 个字节，内容为 0xFFFF（short 类型的 -1）。__尾部实际上是一个特殊的数据块，因为字节数为 -1，所以没有音频数据部分__。

## 2. 微信语音文件格式
&#160; &#160; &#160; &#160;微信语音的文件格式如下：
```
   +------------------+
   | Header           |
   +-----------+------+
   | block 1   |
   +-----------+--+
   | block 2      |
   +--------------+--+
   : ...             :
   +--------------+--+
   | block n         |
   +-----------------+
```

### 2.1 头部（Header）

&#160; &#160; &#160; &#160;微信语音的头部与 silk 相比，在起始位置多了一个值为 0x02 的字节，对应 ASCII 的控制字符 “STX (start of text)”，意义是“本文开始”，如下：
```
   ".#!SILK_V3" （十六进制：0x02 0x23 0x21 0x53 0x49 0x4C 0x4B 0x5F 0x56 0x33）
```

### 2.2 数据块（block）
&#160; &#160; &#160; &#160;跟 silk 一样。

### 2.3 尾部（Footer）
&#160; &#160; &#160; &#160;微信语音没有尾部。__添加尾部且时长超过 5 秒，在 Windows 电脑上播放将会提示文件已损坏，在 Mac 电脑上播放后将无法停止。__（在手机上播放正常）

## 3. 代码实现

### 3.1 微信语音差异整理
1. __头部的起始位置多了一个值为 0x02 的 ASCII STX 控制字符__
2. __移除了值为 0xFFFF（字节数 -1）的尾部__

### 3.2 编码部分

修改文件：WeChatVoiceCodec/silk/src/main/cpp/silk/test/Encoder.c

#### 3.2.1 添加一个选项，名称为 -wechat

* 添加使用说明

```c
static void print_usage( char* argv[] ) {
	……
    printf( "\n-wechat          : Add ASCII STX code 0x02 to header and remove 0xFFFF footer (for wechat)" );
	……
}
```

* main 函数添加局部变量用于存储传入的 -wechat 选项的值

```c
int main( int argc, char* argv[] )
{
	……
    SKP_int32 DTX_enabled = 0, INBandFEC_enabled = 0, quiet = 0, wechat = 0;
    ……
}
```

* 解析传入的 -wechat 选项的值

```c
int main( int argc, char* argv[] )
{
	……
    while( args < argc ) {
        if( SKP_STR_CASEINSENSITIVE_COMPARE( argv[ args ], "-Fs_API" ) == 0 ) {
            ……
        } else if( SKP_STR_CASEINSENSITIVE_COMPARE( argv[ args ], "-wechat" ) == 0 ) {
            wechat = 1;
            args++;
        } else {
            ……
        }
    }
    ……
}
```

* 添加配置打印

```c
int main( int argc, char* argv[] )
{
	……
    /* Print options */
    if( !quiet ) {
        printf("********** Silk Encoder (Fixed Point) v %s ********************\n", SKP_Silk_SDK_get_version());
        ……
        printf( "Wechat used:                    %d\n",     wechat );
    }
    ……
}
```

#### 3.2.2 开启 -wechat 选项时，添加 STX 头部并跳过尾部写入

* 添加 STX 头部

```c
int main( int argc, char* argv[] )
{
	……
    /* Add Silk header to stream */
    {
        /* Add ASCII STX code 0x02 to header (for wechat) */
        if (wechat) {
            fputc(0x02, bitOutFile);
        }
        static const char Silk_header[] = "#!SILK_V3";
        fwrite( Silk_header, sizeof( char ), strlen( Silk_header ), bitOutFile );
    }
    ……
}
```

* 跳过尾部写入

```c
int main( int argc, char* argv[] )
{
	……
    /* Wechat does not support */
    if (!wechat) {
        /* Write dummy because it can not end with 0 bytes */
        nBytes = -1;

        /* Write payload size */
        fwrite( &nBytes, sizeof( SKP_int16 ), 1, bitOutFile );
    }
    ……
}
```

### 3.3 解码部分

修改文件：WeChatVoiceCodec/silk/src/main/cpp/silk/test/Decoder.c

#### 3.3.1 添加一个选项，名称为 -wechat

* 添加使用说明

```c
static void print_usage(char* argv[]) {
	……
    printf( "\n-wechat  : Skip ASCII STX code of header (for wechat)" );
	……
}
```

* main 函数添加局部变量用于存储传入的 -wechat 选项的值

```c
int main( int argc, char* argv[] )
{
	……
    SKP_int32 frames, lost, quiet, wechat = 0;
    ……
}
```

* 解析传入的 -wechat 选项的值

```c
int main( int argc, char* argv[] )
{
	……
    while( args < argc ) {
        if( SKP_STR_CASEINSENSITIVE_COMPARE( argv[ args ], "-loss" ) == 0 ) {
            ……
        } else if( SKP_STR_CASEINSENSITIVE_COMPARE( argv[ args ], "-wechat" ) == 0 ) {
            wechat = 1;
            args++;
        } else {
            ……
        }
    }
    ……
}
```

* 添加配置打印

```c
int main( int argc, char* argv[] )
{
	……
    if( !quiet ) {
        printf("********** Silk Decoder (Fixed Point) v %s ********************\n", SKP_Silk_SDK_get_version());
        ……
        printf( "Wechat used:                  %d\n", wechat );
    }
    ……
}
```

#### 3.3.2 开启 -wechat 选项时，跳过 STX 头部

```c
int main( int argc, char* argv[] )
{
	……
    /* Check Silk header */
    {
        /* Skip ASCII STX code of header (for wechat) */
        if (wechat) {
            fseek(bitInFile, 1, 0);
        }
        char header_buf[ 50 ];
        counter = fread( header_buf, sizeof( char ), strlen( "#!SILK_V3" ), bitInFile );
        ……
    }
    ……
}
```

## 4. 测试

### 4.1 在手机上创建测试文件夹

```linux
adb shell "mkdir /data/local/tmp/silk/"
```

### 4.2 将生成的可执行文件推到手机

```linux
cd xxx/WeChatVoiceCodec/silk/build/intermediates/cmake/debug/obj/arm64-v8a/
adb push decoder /data/local/tmp/silk/
adb push encoder /data/local/tmp/silk/
adb push signalcompare /data/local/tmp/silk/
```

### 4.3 将项目工程中测试用的微信语音文件推到手机

```linux
cd xxx/WeChatVoiceCodec/silk/test_vectors/
adb push wechat_voice.amr /data/local/tmp/silk/
```

### 4.4 添加执行权限

```linux
adb shell "chmod 777 -R /data/local/tmp/silk/"
```

### 4.5 测试解码

* 查看使用说明

```linux
adb shell
cd /data/local/tmp/silk/
./decoder
```
&#160; &#160; &#160; &#160;执行上述命令后会输出使用说明，可以看到 -wechat 选项已经添加成功，如下：

```linux
usage: ./decoder in.bit out.pcm [settings]

in.bit       : Bitstream input to decoder
out.pcm      : Speech output from decoder
   settings:
-Fs_API <Hz> : Sampling rate of output signal in Hz; default: 24000
-loss <perc> : Simulated packet loss percentage (0-100); default: 0
-quiet       : Print out just some basic values
-wechat      : Skip ASCII STX code of header (for wechat)
```

* 测试解码微信语音

```linux
adb shell
cd /data/local/tmp/silk/
./decoder wechat_voice.amr tmp.pcm -wechat
```

&#160; &#160; &#160; &#160;执行上述命令后会在当前路径生成 tmp.pcm，并输出以下内容：
```linux
********** Silk Decoder (Fixed Point) v 1.0.9 ********************
********** Compiled for 64 bit cpu *******************************
Input:                       in.amr
Output:                      tmp.pcm
Wechat used:                  1
Packets decoded:              86
Decoding Finished 

File length:                 1.720 s
Time for decoding:           0.078 s (4.514% of realtime)
```

### 4.6 测试编码

* 查看使用说明

```linux
adb shell
cd /data/local/tmp/silk/
./encoder
```

&#160; &#160; &#160; &#160;执行上述命令后会输出使用说明，可以看到 -wechat 选项已经添加成功，如下：

```linux
usage: ./encoder in.pcm out.bit [settings]

in.pcm               : Speech input to encoder
out.bit              : Bitstream output from encoder
   settings:
-Fs_API <Hz>         : API sampling rate in Hz, default: 24000
-Fs_maxInternal <Hz> : Maximum internal sampling rate in Hz, default: 24000
-packetlength <ms>   : Packet interval in ms, default: 20
-rate <bps>          : Target bitrate; default: 25000
-loss <perc>         : Uplink loss estimate, in percent (0-100); default: 0
-inbandFEC <flag>    : Enable inband FEC usage (0/1); default: 0
-complexity <comp>   : Set complexity, 0: low, 1: medium, 2: high; default: 2
-DTX <flag>          : Enable DTX (0/1); default: 0
-quiet               : Print only some basic values
-wechat              : Add ASCII STX code 0x02 to header and remove 0xFFFF footer (for wechat)
```

* 测试编码微信语音

```linux
adb shell
cd /data/local/tmp/silk/
./encoder tmp.pcm tmp.amr -rate 24000 -wechat
```

&#160; &#160; &#160; &#160;执行上述命令后会在当前路径生成 tmp.amr，并输出以下内容：

```linux
********** Silk Encoder (Fixed Point) v 1.0.9 ********************
********** Compiled for 64 bit cpu ******************************* 
Input:                          tmp.pcm
Output:                         tmp.amr
API sampling rate:              24000 Hz
Maximum internal sampling rate: 24000 Hz
Packet interval:                20 ms
Inband FEC used:                0
DTX used:                       0
Complexity:                     2
Target bitrate:                 24000 bps
Wechat used:                    1
Packets encoded:                84
File length:                    1.680 s
Time for encoding:              0.321 s (19.099% of realtime)
Average bitrate:                20.957 kbps
Active bitrate:                 21.813 kbps
```

&#160; &#160; &#160; &#160;将生成的 tmp.amr 拉取到电脑，用二进制编辑器打开，头部确实添加了 STX 控制字符且文件末尾确实不存在 0xFFFF 尾部。


## 5. 参考

[Silk_v3_decoder](https://github.com/fishCoder/Silk_v3_decoder)

SILK_SDK_SRC_ARM_v1.0.9/doc/SILK_RTP_PayloadFormat.pdf 第5节



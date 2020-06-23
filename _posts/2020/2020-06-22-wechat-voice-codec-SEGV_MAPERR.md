---
layout: post
title: '记录一次 SEGV_MAPERR 问题的分析、解决过程'
subtitle: '日志分析 -> 动态调试 -> 推断验证'
date: 2020-06-22
categories: 技术
cover: '../../../assets/img/post_bg/20200622.jpeg'
tags: android wechat NDK
---


## 1. 前言

&#160; &#160; &#160; &#160;前段时间有人向我的微信语音编解码库 [WeChatVoiceCodec](https://github.com/WuFengXue/WeChatVoiceCodec) 提了一个 [issue](https://github.com/WuFengXue/WeChatVoiceCodec/issues/2) ，反馈无法在工作线程编解码。解决该问题时颇费了一点功夫，便决定将过程整理、记录一下，于是便有了本文。

## 2. 环境

* OSX 10.14.6
* Android Studio 4.0
* 魅蓝 Note5 [Android 6.0]
* Nexus 5 [Android 4.4] [__Root__]


## 3. 复现问题

* git 切换到 tag 1.0.1 所在提交
* 将 app 模块的编解码测试代码改到工作线程执行

```java
diff --git a/app/src/main/java/com/reinhard/wechat/voicecodec/MainActivity.java b/app/src/main/java/com/reinhard/wechat/voicecodec/MainActivity.java
index d4938ce..52b59b5 100644
--- a/app/src/main/java/com/reinhard/wechat/voicecodec/MainActivity.java
+++ b/app/src/main/java/com/reinhard/wechat/voicecodec/MainActivity.java
@@ -59,33 +59,66 @@ public class MainActivity extends Activity {
 
     private void testAmrToMp3() {
         Log.d(TAG, "testAmrToMp3");
-        String amrPath = TEST_DIR + "in.amr";
-        String pcmPath = TEST_DIR + "out.pcm";
-        String mp3Path = TEST_DIR + "out.mp3";
-        if (WcvCodec.decode(amrPath, pcmPath, mp3Path) == 0) {
-            Toast.makeText(this, "testAmrToMp3 success", Toast.LENGTH_SHORT)
-                    .show();
-        }
+        final String amrPath = TEST_DIR + "in.amr";
+        final String pcmPath = TEST_DIR + "out.pcm";
+        final String mp3Path = TEST_DIR + "out.mp3";
+        // demo 展示用，实际项目中请不要这样使用线程
+        new Thread(new Runnable() {
+            @Override
+            public void run() {
+                if (WcvCodec.decode(amrPath, pcmPath, mp3Path) == 0) {
+                    runOnUiThread(new Runnable() {
+                        @Override
+                        public void run() {
+                            Toast.makeText(MainActivity.this, "testAmrToMp3 success", Toast.LENGTH_SHORT)
+                                    .show();
+                        }
+                    });
+                }
+            }
+        }).start();
     }
 
     private void testPcmToAmr() {
         Log.d(TAG, "testPcmToAmr");
-        String pcmPath = TEST_DIR + "in.pcm";
-        String amrPath = TEST_DIR + "out.amr";
-        if (WcvCodec.encode(pcmPath, amrPath) == 0) {
-            Toast.makeText(this, "testPcmToAmr success", Toast.LENGTH_SHORT)
-                    .show();
-        }
+        final String pcmPath = TEST_DIR + "in.pcm";
+        final String amrPath = TEST_DIR + "out.amr";
+        // demo 展示用，实际项目中请不要这样使用线程
+        new Thread(new Runnable() {
+            @Override
+            public void run() {
+                if (WcvCodec.encode(pcmPath, amrPath) == 0) {
+                    runOnUiThread(new Runnable() {
+                        @Override
+                        public void run() {
+                            Toast.makeText(MainActivity.this, "testPcmToAmr success", Toast.LENGTH_SHORT)
+                                    .show();
+                        }
+                    });
+                }
+            }
+        }).start();
     }
 
     private void testMp3ToAmr() {
         Log.d(TAG, "testMp3ToAmr");
-        String mp3Path = TEST_DIR + "in.mp3";
-        String pcmPath = TEST_DIR + "out.pcm";
-        String amrPath = TEST_DIR + "out.amr";
-        if (WcvCodec.encode2(mp3Path, pcmPath, amrPath) == 0) {
-            Toast.makeText(this, "testMp3ToAmr success", Toast.LENGTH_SHORT)
-                    .show();
-        }
+        final String mp3Path = TEST_DIR + "in.mp3";
+        final String pcmPath = TEST_DIR + "out.pcm";
+        final String amrPath = TEST_DIR + "out.amr";
+        // demo 展示用，实际项目中请不要这样使用线程
+        new Thread(new Runnable() {
+            @Override
+            public void run() {
+                if (WcvCodec.encode2(mp3Path, pcmPath, amrPath) == 0) {
+                    runOnUiThread(new Runnable() {
+                        @Override
+                        public void run() {
+                            Toast.makeText(MainActivity.this, "testMp3ToAmr success", Toast.LENGTH_SHORT)
+                                    .show();
+                        }
+                    });
+                }
+            }
+        }).start();
     }
 }
```

* 安装到手机后，依次测试各个功能

![](../../../assets/img/2020/06/SEGV_MAPERR/demo.gif)

* 测试结果汇总
	* __amr to mp3：闪退__
	* pcm to amr：成功
	* __mp3 to amr：闪退__

## 4. 问题定位

### 4.1 抓日志
* 只打印包所在进程的日志
	* 先打开应用，设置好 AndroidStudio 的日志过滤条件
	* 清空日志，然后点击手机测试页面的【amr to mp3】按钮

![](../../../assets/img/2020/06/SEGV_MAPERR/logcat_pkg_only.png)

```shell
    --------- beginning of crash
06-17 17:18:42.382 13997-14061/com.reinhard.wechat.voicecodec A/libc: Fatal signal 11 (SIGSEGV), code 1, fault addr 0x7f5f0fbf20 in tid 14061 (Thread-396)

```

&#160; &#160; &#160; &#160;结果显示只有几行新增信息。

* 打印全部进程的日志
	* 移除 AndroidStudio 的日志过滤条件
	* 清空日志，然后点击手机测试页面的【amr to mp3】按钮

![](../../../assets/img/2020/06/SEGV_MAPERR/logcat_all.png)

&#160; &#160; &#160; &#160;打印全部进程时，日志会不断的打印，不利于查找和查看日志。这里有一个小技巧，可以先打印全部进程的日志，在找到目标日志后，截取其关键字再次打印日志。__过滤关键字支持多个，用竖线【\|】进行分隔。__

* 打印包含指定关键字的日志
	* 设置 AndroidStudio 的日志过滤关键字
	* 清空日志，然后点击手机测试页面的【amr to mp3】按钮

![](../../../assets/img/2020/06/SEGV_MAPERR/logcat_keywords.png)

```shell
06-18 16:42:52.528 307-1600/? D/BufferQueueProducer: [com.reinhard.wechat.voicecodec/com.reinhard.wechat.voicecodec.MainActivity](this:0x7f833b4000,id:390,api:1,p:17816,c:307) queueBuffer: fps=0.83 dur=3629.54 max=3602.89 min=10.43
06-18 16:42:52.555 17816-17816/com.reinhard.wechat.voicecodec D/WcvCodec: testAmrToMp3
06-18 16:42:52.608 17816-17875/com.reinhard.wechat.voicecodec D/WcvCodec: 0.038 2.187 86
    
    
    --------- beginning of crash
06-18 16:42:52.609 17816-17875/com.reinhard.wechat.voicecodec A/libc: Fatal signal 11 (SIGSEGV), code 2, fault addr 0x7f5d16ef20 in tid 17875 (Thread-456)
06-18 16:42:52.610 17876-17876/? I/AEE/AED: handle_request(12)
06-18 16:42:52.610 17876-17876/? I/AEE/AED: check process 17816 name:chat.voicecodec
06-18 16:42:52.610 17876-17876/? I/AEE/AED: tid 17875 abort msg address is:0x0000000000000000 si_code is:2 (request from 17816:10100)
06-18 16:42:52.610 17876-17876/? I/AEE/AED: BOOM: pid=17816 uid=10100 gid=10100 tid=17875
06-18 16:42:52.611 17876-17876/? I/AEE/AED: [OnPurpose Redunant in void preset_info(aed_report_record*, int, int)] pid: 17816, tid: 17875, name: Thread-456  >>> com.reinhard.wechat.voicecodec <<<
06-18 16:42:52.663 17876-17876/? I/AEE/AED: *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
06-18 16:42:52.663 17876-17876/? I/AEE/AED: Build fingerprint: 'Meizu/M1621/M1621:6.0/MRA58K/1495809194:user/release-keys'
06-18 16:42:52.663 17876-17876/? I/AEE/AED: Revision: '0'
06-18 16:42:52.663 17876-17876/? I/AEE/AED: ABI: 'arm64'
06-18 16:42:52.664 17876-17876/? I/AEE/AED: pid: 17816, tid: 17875, name: Thread-456  >>> com.reinhard.wechat.voicecodec <<<
06-18 16:42:52.664 17876-17876/? I/AEE/AED: signal 11 (SIGSEGV), code 2 (SEGV_ACCERR), fault addr 0x7f5d16ef20
06-18 16:42:52.675 17876-17876/? I/AEE/AED:     x0   0000007f76c88f80  x1   000000000000000e  x2   0000007f5d302f10  x3   0000000000000048
06-18 16:42:52.676 17876-17876/? I/AEE/AED:     x4   0000000000000000  x5   0000000000000001  x6   0000000000000000  x7   0000000000000315
06-18 16:42:52.676 17876-17876/? I/AEE/AED:     x8   0000007f5d300e20  x9   0000007f5d16ef78  x10  28fbe9c0d7667f76  x11  0000000000000316
06-18 16:42:52.676 17876-17876/? I/AEE/AED:     x12  0000000000279000  x13  0000000000000004  x14  0000000000000005  x15  0000007f76e82640
06-18 16:42:52.676 17876-17876/? I/AEE/AED:     x16  0000007f5d1cfee8  x17  0000007f5d152b3c  x18  0000007f76e827e0  x19  0000000012ea7340
06-18 16:42:52.676 17876-17876/? I/AEE/AED:     x20  0000007f766f4520  x21  0000007f756ace00  x22  00000000710e9110  x23  0000007f5d3031f8
06-18 16:42:52.676 17876-17876/? I/AEE/AED:     x24  0000007f5d3032a8  x25  0000007f5d3032c8  x26  00000000715b852a  x27  00000000710e9110
06-18 16:42:52.676 17876-17876/? I/AEE/AED:     x28  0000007f756a8700  x29  0000007f5d302e40  x30  0000007f5d1aa014
06-18 16:42:52.676 17876-17876/? I/AEE/AED:     sp   0000007f5d16eea0  pc   0000007f5d152b68  pstate 0000000080000000
06-18 16:42:52.680 17876-17876/? I/AEE/AED: backtrace:
06-18 16:42:52.680 17876-17876/? I/AEE/AED:     #00 pc 000000000004db68  /data/app/com.reinhard.wechat.voicecodec-1/lib/arm64/libwcvcodec.so (lame_main+44)
06-18 16:42:52.680 17876-17876/? I/AEE/AED:     #01 pc 00000000000a5010  /data/app/com.reinhard.wechat.voicecodec-1/lib/arm64/libwcvcodec.so
06-18 16:42:52.680 17876-17876/? I/AEE/AED:     #02 pc 00000000000a4f94  /data/app/com.reinhard.wechat.voicecodec-1/lib/arm64/libwcvcodec.so (lame_codec_main+28)
06-18 16:42:52.680 17876-17876/? I/AEE/AED:     #03 pc 0000000000014adc  /data/app/com.reinhard.wechat.voicecodec-1/lib/arm64/libwcvcodec.so (Java_com_reinhard_wcvcodec_WcvCodec_decode+400)
06-18 16:42:52.680 17876-17876/? I/AEE/AED:     #04 pc 00000000000c9a5c  /data/app/com.reinhard.wechat.voicecodec-1/oat/arm64/base.odex (offset 0x5e000) (int com.reinhard.wcvcodec.WcvCodec.decode(java.lang.String, java.lang.String, java.lang.String)+208)
06-18 16:42:52.680 17876-17876/? I/AEE/AED:     #05 pc 00000000000ca99c  /data/app/com.reinhard.wechat.voicecodec-1/oat/arm64/base.odex (offset 0x5e000) (void com.reinhard.wechat.voicecodec.MainActivity$1.run()+160)
06-18 16:42:52.681 17876-17876/? I/AEE/AED:     #06 pc 00000000026983c4  /system/framework/arm64/boot.oat (offset 0x266d000)
06-18 16:42:52.879 17876-17876/? I/AEE/AED: Tombstone written to: /data/tombstones/tombstone_08
06-18 16:42:52.942 1122-1208/? W/InputDispatcher: channel 'c65bc2e com.reinhard.wechat.voicecodec/com.reinhard.wechat.voicecodec.MainActivity (server)' ~ Consumer closed input channel or an error occurred.  events=0x9
06-18 16:42:52.942 1122-1208/? E/InputDispatcher: channel 'c65bc2e com.reinhard.wechat.voicecodec/com.reinhard.wechat.voicecodec.MainActivity (server)' ~ Channel is unrecoverably broken and will be disposed!
06-18 16:42:52.950 1122-2011/? I/WindowState: WIN DEATH: Window{c65bc2e u0 com.reinhard.wechat.voicecodec/com.reinhard.wechat.voicecodec.MainActivity}
06-18 16:42:52.951 1122-2011/? W/InputDispatcher: Attempted to unregister already unregistered input channel 'c65bc2e com.reinhard.wechat.voicecodec/com.reinhard.wechat.voicecodec.MainActivity (server)'
06-18 16:42:53.470 307-2984/? D/SurfaceFlinger:     remove: com.reinhard.wechat.voicecodec/com.reinhard.wechat.voicecodec.MainActivity
06-18 16:42:53.483 307-307/? I/BufferQueue: [com.reinhard.wechat.voicecodec/com.reinhard.wechat.voicecodec.MainActivity](this:0x7f833b4000,id:390,api:1,p:-1,c:-1) ~BufferQueueCore
```


&#160; &#160; &#160; &#160;抓取到的日志虽然给出了出错的内存地址，但未给出错误地址映射的区域。

### 4.2 获取墓碑（tombstone）文件

```shell
06-18 16:42:52.879 17876-17876/? I/AEE/AED: Tombstone written to: /data/tombstones/tombstone_08
```

&#160; &#160; &#160; &#160;当原生程序出现崩溃的时候，除了会输出日志，还会将一份更详细的墓碑文件写入手机的 /data/tombstones/ 文件夹（[官网 - 崩溃转储](https://source.android.com/devices/tech/debug/index.html#debuggerd)（需翻墙））

* 不过，需要具备系统权限才可直接读取墓碑文件

```shell
root@hammerhead:/ # ls -l /data | grep tombstone
drwx------ system   system            2017-10-31 18:24 tombstones
root@hammerhead:/ # ls -l /data/tombstones                                     
-rw------- system   system      13760 2020-06-18 10:12 tombstone_00
-rw------- system   system      13760 2020-06-17 10:32 tombstone_01
-rw------- system   system      13760 2020-06-17 10:37 tombstone_02
-rw------- system   system      13760 2020-06-17 10:42 tombstone_03
-rw------- system   system      13758 2020-06-18 12:15 tombstone_04
-rw------- system   system      13760 2020-06-17 10:27 tombstone_05
-rw------- system   system      13760 2020-06-17 10:37 tombstone_06
-rw------- system   system      13760 2020-06-17 10:47 tombstone_07
-rw------- system   system      13760 2020-06-17 10:47 tombstone_08
-rw------- system   system      13760 2020-06-17 13:01 tombstone_09
```

* 如果手机有 Root，那么可以直接用 adb pull 到电脑上进行查看
* 如果手机没有 Root，那么需要借助 adb bugreport 命令进行抓取（[官网 - 使用 adb 获取错误报告（需翻墙）](https://developer.android.com/studio/debug/bug-report#bugreportadb)）
	* 如果是安卓7.0以下，那么不支持将其打包成zip

	```shell
	$ adb bugreport ./
Failed to get bugreportz version: 'bugreportz -v' returned '/system/bin/sh: bugreportz: not found' (code 0).
If the device does not run Android 7.0 or above, try 'adb bugreport' instead.
	```
	
	* 直接运行 adb bugreport 会将全部报告输出到控制台（内容很多，不便查看）
	* 可以将其输出到指定文件中，方便查阅（__报告内容比较多，可能要等一会才执行完__）

	```shell
	$ adb bugreport > bugreport.txt
Failed to get bugreportz version, which is only available on devices running Android 7.0 or later.
Trying a plain-text bug report instead.
	$ 
	```
	
### 4.4 查阅墓碑（tombstone）文件

* 7.0 及以上版本，直接打开 zip 压缩包解压后的 tombstones 文件夹中的对应文件
* 7.0 以下的版本，用文本编辑器打开 adb bugreport 输出的文件

	```shell
06-18 16:42:52.879 17876-17876/? I/AEE/AED: Tombstone written to: /data/tombstones/tombstone_08
```

	* 搜索关键字 tombstone_XX（下方的墓碑号是 9 / 在写本文时，操作了很多次，偷个懒，就不更新前面的内容了）

	```shell
	------ TOMBSTONE (/data/tombstones/tombstone_09: 2020-06-19 10:38:16) ------
*** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
Build fingerprint: 'Meizu/M1621/M1621:6.0/MRA58K/1495809194:user/release-keys'
Revision: '0'
ABI: 'arm64'
pid: 5733, tid: 6455, name: Thread-328  >>> com.reinhard.wechat.voicecodec <<<
signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0x7f67b2f000
    x0   0000007f67b0b648  x1   0000000000000000  x2   00000000000a46d0  x3   00000000000c8090
    ……
    x28  0000007f660cce40  x29  0000007f67c9ee40  x30  0000007f5ab44dc0
    sp   0000007f67b0aea0  pc   0000007f832278e8  pstate 0000000020000000
    v0   2e2e2e2e2e2e2e2e2e2e2e2e2e2e2e2e  v1   000000000000000000000000000009c0
    ……
    v30  000000000000000000000000439b8000  v31  00000000000000000000000042f10000
    fpsr 00000010  fpcr 00000000
--
backtrace:
    #00 pc 000000000001c8e8  /system/lib64/libc.so (memset+360)
    #01 pc 000000000004ddbc  /data/app/com.reinhard.wechat.voicecodec-1/lib/arm64/libwcvcodec.so (lame_main+640)
    #02 pc 00000000000a5010  /data/app/com.reinhard.wechat.voicecodec-1/lib/arm64/libwcvcodec.so
    #03 pc 00000000000a4f94  /data/app/com.reinhard.wechat.voicecodec-1/lib/arm64/libwcvcodec.so (lame_codec_main+28)
    #04 pc 0000000000014adc  /data/app/com.reinhard.wechat.voicecodec-1/lib/arm64/libwcvcodec.so (Java_com_reinhard_wcvcodec_WcvCodec_decode+400)
    #05 pc 00000000000c9a5c  /data/app/com.reinhard.wechat.voicecodec-1/oat/arm64/base.odex (offset 0x5e000) (int com.reinhard.wcvcodec.WcvCodec.decode(java.lang.String, java.lang.String, java.lang.String)+208)
    #06 pc 00000000000ca99c  /data/app/com.reinhard.wechat.voicecodec-1/oat/arm64/base.odex (offset 0x5e000) (void com.reinhard.wechat.voicecodec.MainActivity$1.run()+160)
    #07 pc 00000000026983c4  /system/framework/arm64/boot.oat (offset 0x266d000)
--
stack:
         ........  ........
         ........  ........
    #02  0000007f67c9ee50  0000007f7f644d30  [anon:libc_malloc]
         0000007f67c9ee58  0000000000000000
         0000007f67c9ee60  0000000000000000
         0000007f67c9ee68  0000007f68426c00  [anon:libc_malloc]
         0000007f67c9ee70  0000007f67c9ef10
         0000007f67c9ee78  0000000e00000000
         0000007f67c9ee80  0000007f67c9eea0
         0000007f67c9ee88  0000007f5ab9bf98  /data/app/com.reinhard.wechat.voicecodec-1/lib/arm64/libwcvcodec.so (lame_codec_main+32)
    ……
--
memory map: (fault address prefixed with --->)
    ……
    0000007f'66140000-0000007f'6617ffff rw-         0     40000  [anon:libc_malloc]
    0000007f'661b8000-0000007f'661c7fff rw-  7f661b8000     10000  /dev/mali0
    0000007f'661c8000-0000007f'66208fff ---  7f661c8000     41000  /dev/mali0
    0000007f'66219000-0000007f'66228fff rw-  7f66219000     10000  /dev/mali0
    0000007f'6627b000-0000007f'662bafff rw-  7f6627b000     40000  /dev/mali0
    0000007f'66326000-0000007f'66b1efff rw-  7f66326000    7f9000  /dev/mali0
    0000007f'66b1f000-0000007f'67316fff rw-  7f66b1f000    7f8000  /dev/mali0
    0000007f'67327000-0000007f'67336fff rw-  7f67327000     10000  /dev/mali0
    0000007f'67337000-0000007f'67b2efff rw-         0    7f8000  anon_inode:dmabuf
--->Fault address falls at 0000007f'67b2f000 between mapped regions
    0000007f'67b9b000-0000007f'67b9bfff ---         0      1000
    0000007f'67b9c000-0000007f'67b9cfff ---         0      1000
    0000007f'67b9d000-0000007f'67c9ffff rw-         0    103000
	```
	
	* 可以看到报错的地址所在区间没有映射到任何区域（以 --->Fault address 开头）

	```shell
    0000007f'67337000-0000007f'67b2efff rw-         0    7f8000  anon_inode:dmabuf
--->Fault address falls at 0000007f'67b2f000 between mapped regions
    0000007f'67b9b000-0000007f'67b9bfff ---         0      1000
	```
	
	* __说明：如果输出的 bugreport.txt 文件中找不到对应的墓碑文件，那么可以重启下手机，然后重新操作使其闪退后，再重新执行 adb bugreport 命令__

### 4.5 题外（ndk-stack 工具介绍）
&#160; &#160; &#160; &#160;如果崩溃日志中无法打印出错误所在的文件及行号（正式版的 so 库文件可能会抹除一些信息），此时可以借助 ndk-stack 工具。该工具会将共享库内的任何地址替换为源代码中对应的 \<source-file>:\<line-number>，从而简化调试流程。（[官网 - ndk-stack（需翻墙）](https://developer.android.com/ndk/guides/ndk-stack)）

## 5. 分析问题

### 5.1 SEGV_MAPERR 错误的定义

&#160; &#160; &#160; &#160;linux man 手册中 [SEGV_MAPERR](https://man7.org/linux/man-pages/man0/signal.h.0p.html) 的定义如下：

```doc
┌───────┬─────────────┬──────────────────────────────────────────────────────────────────┐
│Signal │    Code     │                             Reason                               │
├───────┼─────────────┼──────────────────────────────────────────────────────────────────┤
│
├───────┼─────────────┼──────────────────────────────────────────────────────────────────┤
│SIGSEGV│SEGV_MAPERR  │Address not mapped to object. （地址未映射）                       │
│       │SEGV_ACCERR  │Invalid permissions for mapped object.（无权操作）                 │
├───────┼─────────────┼──────────────────────────────────────────────────────────────────┤
│
```

### 5.2 问题所在代码块

* 出错时的调用堆栈

```shell
06-19 14:23:56.695 17270-17270/? I/AEE/AED: backtrace:
06-19 14:23:56.695 17270-17270/? I/AEE/AED:     #00 pc 000000000001c8e8  /system/lib64/libc.so (memset+360)（系统函数）
06-19 14:23:56.695 17270-17270/? I/AEE/AED:     #01 pc 000000000004ddbc  /data/app/com.reinhard.wechat.voicecodec-1/lib/arm64/libwcvcodec.so (lame_main+640)（项目最后一个函数调用）
06-19 14:23:56.695 17270-17270/? I/AEE/AED:     #02 pc 00000000000a5010  /data/app/com.reinhard.wechat.voicecodec-1/lib/arm64/libwcvcodec.so
06-19 14:23:56.695 17270-17270/? I/AEE/AED:     #03 pc 00000000000a4f94  /data/app/com.reinhard.wechat.voicecodec-1/lib/arm64/libwcvcodec.so (lame_codec_main+28)
06-19 14:23:56.695 17270-17270/? I/AEE/AED:     #04 pc 0000000000014adc  /data/app/com.reinhard.wechat.voicecodec-1/lib/arm64/libwcvcodec.so (Java_com_reinhard_wcvcodec_WcvCodec_decode+400)
06-19 14:23:56.695 17270-17270/? I/AEE/AED:     #05 pc 00000000000c9a5c  /data/app/com.reinhard.wechat.voicecodec-1/oat/arm64/base.odex (offset 0x5e000) (int com.reinhard.wcvcodec.WcvCodec.decode(java.lang.String, java.lang.String, java.lang.String)+208)
06-19 14:23:56.695 17270-17270/? I/AEE/AED:     #06 pc 00000000000ca99c  /data/app/com.reinhard.wechat.voicecodec-1/oat/arm64/base.odex (offset 0x5e000) (void com.reinhard.wechat.voicecodec.MainActivity$1.run()+160)
06-19 14:23:56.695 17270-17270/? I/AEE/AED:     #07 pc 00000000026983c4  /system/framework/arm64/boot.oat (offset 0x266d000)
```

* 在项目中搜索 lame_main 找到目标函数，然后再通过 memset 找到出错的行


```c
// libwcvcodec/src/main/cpp/lame/frontend/lame_main.c
int
lame_main(lame_t gf, int argc, char **argv)
{
    char    inPath[PATH_MAX + 1];
    char    outPath[PATH_MAX + 1];
    char    nogapdir[PATH_MAX + 1];
    /* support for "nogap" encoding of up to 200 .wav files */
#define MAX_NOGAP 200
    int     nogapout = 0;
    int     max_nogap = MAX_NOGAP;
    char    nogap_inPath_[MAX_NOGAP][PATH_MAX + 1];
    char   *nogap_inPath[MAX_NOGAP];
    char    nogap_outPath_[MAX_NOGAP][PATH_MAX + 1];
    char   *nogap_outPath[MAX_NOGAP];

    int     ret;
    int     i;
    FILE   *outf = NULL;

    lame_set_msgf(gf, &frontend_msgf);
    lame_set_errorf(gf, &frontend_errorf);
    lame_set_debugf(gf, &frontend_debugf);
    if (argc <= 1) {
        usage(stderr, argv[0]); /* no command-line args, print usage, exit  */
        return 1;
    }

    // 可疑点1
    memset(inPath, 0, sizeof(inPath));
    // 可疑点2
    memset(nogap_inPath_, 0, sizeof(nogap_inPath_));
    for (i = 0; i < MAX_NOGAP; ++i) {
        nogap_inPath[i] = &nogap_inPath_[i][0];
    }
    // 可疑点3
    memset(nogap_outPath_, 0, sizeof(nogap_outPath_));
    for (i = 0; i < MAX_NOGAP; ++i) {
        nogap_outPath[i] = &nogap_outPath_[i][0];
    }

    ……
    return ret;
}

```


### 5.3 断点调试

* 分别在3个可疑点下断点，然后附加到目标进程进行调试
* 实际调试的时候，memset 所在行的断点没有进去（不知道为什么），而是断在可疑点2和可疑点3之间的 for 语句里面
* 此时点 Rusume Program 继续执行，会发现断点跑到 string.h 文件了

![](../../../assets/img/2020/06/SEGV_MAPERR/debug_string_h.png)

* 点击查看调用堆栈里面 lame_main lame_main.c:562 对应的内容，出错行为可疑点3

![](../../../assets/img/2020/06/SEGV_MAPERR/debug_memset.png)

### 5.4 错误分析

* [memset](https://man7.org/linux/man-pages/man0/signal.h.0p.html) 的主要功能是向一个内存块写入某个常量

```doc
MEMSET(3)                 Linux Programmer's Manual                MEMSET(3)
NAME         top
       memset - fill memory with a constant byte
SYNOPSIS         top
       #include <string.h>

       void *memset(void *s, int c, size_t n);
DESCRIPTION         top
       The memset() function fills the first n bytes of the memory area
       pointed to by s with the constant byte c.
RETURN VALUE         top
       The memset() function returns a pointer to the memory area s.
ATTRIBUTES         top
       For an explanation of the terms used in this section, see
       attributes(7).

       ┌──────────┬───────────────┬─────────┐
       │Interface │ Attribute     │ Value   │
       ├──────────┼───────────────┼─────────┤
       │memset()  │ Thread safety │ MT-Safe │
       └──────────┴───────────────┴─────────┘
```

* 查看 nogap_inPath、nogap_inPath_、nogap_outPath 和 nogap_outPath_ 的声明

```c
// lame_main.c
int
lame_main(lame_t gf, int argc, char **argv)
{
    char    inPath[PATH_MAX + 1];
    char    outPath[PATH_MAX + 1];
    char    nogapdir[PATH_MAX + 1];
    /* support for "nogap" encoding of up to 200 .wav files */
#define MAX_NOGAP 200
    int     nogapout = 0;
    int     max_nogap = MAX_NOGAP;
    // 局部变量声明1
    char    nogap_inPath_[MAX_NOGAP][PATH_MAX + 1];
    // 局部变量声明2
    char   *nogap_inPath[MAX_NOGAP];
    // 局部变量声明3
    char    nogap_outPath_[MAX_NOGAP][PATH_MAX + 1];
    // 局部变量声明3
    char   *nogap_outPath[MAX_NOGAP];

    ……
}
```

* PATH_MAX 的定义在 limits.h

```c
// limits.h
……
#define PATH_MAX 4096
……
```

* 这些数组都是直接通过声明创建的，照理说不应该有问题。算了下，nogap_inPath_ 和 nogap_outPath_ 大致各占 800K（200 * 4096 bytes = 800k），合起来就是1.6M左右
* 联想到主线程是正常的，推断子线程的栈空间大小可能比较小，将 MAX_NOGAP 的宏定义减小到 20（原来的十分之一），测试通过了，程序不再崩溃

### 5.5 验证

* 在搜索引擎上搜索【Android 子线程栈内存大小】，最终搜索到了 [《[bionic源码解读]Android线程栈大小》](https://zhuanlan.zhihu.com/p/33562383)，结论抽取如下：

```doc
安卓主线程栈大小默认为8M，子线程栈大小稍微小于1M，具体如下。

              子线程栈大小           mmap大小
Android 6/7   1016KB（包含GUARD）    1020KB
Android 8     1008KB（包含GUARD）    1012KB
Android > 8   1008KB（不包含GUARD）  1016KB

```

* 在6.0源码中子线程的默认栈大小为1016K（1024 - 8），如下（[PTHREAD_STACK_SIZE_DEFAULT 传送门](http://androidxref.com/6.0.1_r10/xref/bionic/libc/bionic/pthread_internal.h#131)）([SIGSTKSZ 传送门](http://androidxref.com/6.0.1_r10/xref/bionic/libc/kernel/uapi/asm-arm/asm/signal.h#87))：

```c
// bionic/libc/bionic/pthread_internal.h
#define PTHREAD_STACK_SIZE_DEFAULT ((1 * 1024 * 1024) - SIGSTKSZ)
```

```c
// bionic/libc/kernel/uapi/asm-arm/asm/signal.h
#define SIGSTKSZ 8192
```
	
*  参考 [《线程堆栈大小的使用介绍》](https://blog.csdn.net/u011784994/article/details/55669961) 的方法，添加测试方法

```c
$ git diff WcvCodec.c
diff --git a/libwcvcodec/src/main/cpp/WcvCodec.c b/libwcvcodec/src/main/cpp/WcvCodec.c
index c35d23f..dbea322 100644
--- a/libwcvcodec/src/main/cpp/WcvCodec.c
+++ b/libwcvcodec/src/main/cpp/WcvCodec.c
@@ -7,9 +7,29 @@
 
 #include <silk/SilkCodec.h>
 #include <lame/LameCodec.h>
+#include <pthread.h>
 #include "WcvCodec.h"
 #include "android_log.h"
 
+// test code
+int print_stack_size() {
+    pthread_attr_t attr;
+    size_t stack_size = 0;
+
+    int ret = pthread_attr_init(&attr);
+    if (ret != 0) {
+        LOGE("pthread_attr_init FAILED");
+        return JNI_ERR;
+    }
+
+    ret = pthread_attr_getstacksize(&attr, &stack_size);
+    if (ret != 0) {
+        LOGE("pthread_attr_getstacksize FAILED");
+    } else {
+        LOGE("stack_size = %d Byte, %d k\n", stack_size, stack_size / 1024);
+    }
+}
+
 /*
  * Class:     com_reinhard_wcvcodec_WcvCodec
  * Method:    decode
@@ -22,6 +42,10 @@ JNIEXPORT jint JNICALL Java_com_reinhard_wcvcodec_WcvCodec_decode
     const char *mp3 = (*env)->GetStringUTFChars(env, mp3Path, JNI_FALSE);
     int argc = 5;
     const char *argv[] = {"./Decoder", amr, pcm, "-wechat", "-quiet"};
+
+    // test code
+    print_stack_size();
+
     if (silk_decoder_main(argc, (char **) argv) == JNI_OK) {
         int argc2 = 14;
         const char *argv2[] = {"./lame", "-q", "5", "-b", "128", "-m", "m", "-r",
```

* 打印出来的结果刚好为1016K，如下：

```c
06-19 17:29:22.322 21673-21746/com.reinhard.wechat.voicecodec E/WcvCodec: stack_size = 1040384 Byte, 1016 k
```

* 2 * 800k = 1600 k > 1016k，会出现 SIGSEGV 也就可以理解了
* 经测试在魅蓝Note5上 MAX_NOGAP 的临界值为 89，超过该值就会闪退，2 * 89 * 4k = 712k，因为线程栈还有其他地方会用到，所以无法达到 1016 k 的上限是可以理解的
* 修改后，【mp3 to amr】也正常了（mp3 转 pcm 也是用的 lame，都是走的 lame_main）

## 6. 修改
* 简单看了下代码，MAX_NOGAP 应该是用来控制音频分段（切片）的，语音场景语音文件应该不会特别大，这样修改应该不会导致副作用
* 但是 lame 毕竟为第三方的库，直接在上面修改不是很友好，比较合理的修改方式是通过配置去调整，最终的方案如下：

```c
$ git show 3f09f873
commit 3f09f873736e7cef04c720e5413e6dc7f3ccb4fc
Author: 李剑波 <jianbo.li01@ucarinc.com>
Date:   Tue Jun 9 15:56:07 2020 +0800

    fix(libwcvcodec): lame decode and encode failed (SIGSEGV) in worker thread

diff --git a/libwcvcodec/build.gradle b/libwcvcodec/build.gradle
index 1731be3..87a0eb9 100644
--- a/libwcvcodec/build.gradle
+++ b/libwcvcodec/build.gradle
@@ -19,7 +19,11 @@ android {
             cmake {
                 cppFlags "-std=c++11 -fexceptions -pthread"
                 // 要支持 'armeabi-v7a' 需开启 NO_ASM 宏
-                cFlags "-DSTDC_HEADERS -DHAVE_LIMITS_H -DHAVE_MPGLIB -DNO_ASM"
+                // lame 编解码库的 lame_main.c 在方法 lame_main 中声明的
+                // 局部变量占用内存较大，会导致申请的内存超过子线程栈的内存上限从而
+                // 导致 SIGSEGV 错误，所以需要减小 MAX_NOGAP 的大小，具体见：
+                // [issue2](https://github.com/WuFengXue/WeChatVoiceCodec/issues/2)
+                cFlags "-DSTDC_HEADERS -DHAVE_LIMITS_H -DHAVE_MPGLIB -DNO_ASM -DMAX_NOGAP=50"
             }
             ndk {
                 abiFilters 'armeabi-v7a', 'arm64-v8a'
diff --git a/libwcvcodec/src/main/cpp/lame/frontend/lame_main.c b/libwcvcodec/src/main/cpp/lame/frontend/lame_main.c
index bb8fb48..36901ba 100644
--- a/libwcvcodec/src/main/cpp/lame/frontend/lame_main.c
+++ b/libwcvcodec/src/main/cpp/lame/frontend/lame_main.c
@@ -534,7 +534,9 @@ lame_main(lame_t gf, int argc, char **argv)
     char    outPath[PATH_MAX + 1];
     char    nogapdir[PATH_MAX + 1];
     /* support for "nogap" encoding of up to 200 .wav files */
-#define MAX_NOGAP 200
+#ifndef MAX_NOGAP
+    #define MAX_NOGAP 200
+#endif
     int     nogapout = 0;
     int     max_nogap = MAX_NOGAP;
     char    nogap_inPath_[MAX_NOGAP][PATH_MAX + 1];
xinyoudeMacBook-Pro:armeabi-v7a newgame$ 

```

* 题外：[《线程堆栈大小的使用介绍》](https://blog.csdn.net/u011784994/article/details/55669961) 介绍了 linux 下修改线程栈大小的方法，__经测试，该方案在 android 下无效（可能是处于安全的考虑）__（即使有效也不建议）

```C
diff --git a/libwcvcodec/src/main/cpp/WcvCodec.c b/libwcvcodec/src/main/cpp/WcvCodec.c
index c35d23f..975c4c8 100644
--- a/libwcvcodec/src/main/cpp/WcvCodec.c
+++ b/libwcvcodec/src/main/cpp/WcvCodec.c
@@ -7,9 +7,53 @@
 
 #include <silk/SilkCodec.h>
 #include <lame/LameCodec.h>
+#include <pthread.h>
 #include "WcvCodec.h"
 #include "android_log.h"
 
+// test code
+int print_stack_size() {
+    pthread_attr_t attr;
+    size_t stack_size = 0;
+
+    int ret = pthread_attr_init(&attr);
+    if (ret != 0) {
+        LOGE("pthread_attr_init FAILED");
+        return JNI_ERR;
+    }
+
+    ret = pthread_attr_getstacksize(&attr, &stack_size);
+    if (ret != 0) {
+        LOGE("pthread_attr_getstacksize FAILED");
+    } else {
+        LOGE("stack_size = %d Byte, %d k\n", stack_size, stack_size / 1024);
+    }
+}
+
+
+// test code
+int set_stack_size() {
+    size_t stack_size = 0;
+    pthread_attr_t attr;
+
+    int ret = pthread_attr_init(&attr);
+    if(ret != 0)
+    {
+        LOGE("pthread_attr_init FAILED");
+        return JNI_ERR;
+    }
+
+    stack_size = 1048*1024;
+    ret = pthread_attr_setstacksize(&attr, stack_size);
+    if(ret != 0)
+    {
+        LOGE("pthread_attr_setstacksize FAILED");
+        return JNI_ERR;
+    }
+
+    LOGE("set_stack_size SUCCESS");
+}
+
 /*
  * Class:     com_reinhard_wcvcodec_WcvCodec
  * Method:    decode
@@ -22,6 +66,14 @@ JNIEXPORT jint JNICALL Java_com_reinhard_wcvcodec_WcvCodec_decode
     const char *mp3 = (*env)->GetStringUTFChars(env, mp3Path, JNI_FALSE);
     int argc = 5;
     const char *argv[] = {"./Decoder", amr, pcm, "-wechat", "-quiet"};
+
+    // test code
+    print_stack_size();
+    // test code
+    set_stack_size();
+    // test code
+    print_stack_size();
+
     if (silk_decoder_main(argc, (char **) argv) == JNI_OK) {
         int argc2 = 14;
         const char *argv2[] = {"./lame", "-q", "5", "-b", "128", "-m", "m", "-r",
```

![](../../../assets/img/2020/06/SEGV_MAPERR/logcat_set_stack_size.png)

## 7.最后总结

&#160; &#160; &#160; &#160;本文主要记录了一次 NDK 崩溃的解决过程：通过查询日志找到问题所在函数，然后通过动态调试定位到出错所在行，最后再结合推断和验证找到解决方案。

&#160; &#160; &#160; &#160;知识点汇总：
* AS 日志过滤支持同时过滤多个关键字，用竖线【\|】进行分隔
* 在分析 NDK 崩溃问题时可以通过查阅墓碑（tombstone）文件获取更详细的信息
	* Root 设备可以直接用 adb pull 拉取到电脑进行查阅
	* 非 Root 设备可以通过 adb bugreport 命令获取
* android 主线程栈默认大小可以通过输入 adb shell "ulimit -s" 查看（6.0为8M）
* android 子线程栈默认大小为1M左右（不同版本不一样，略小于1M）


## 附：微信语音编解码实现

* [（一）—— silk 移植](https://wufengxue.github.io/2019/03/12/wechat-voice-codec-silk.html)
* [（二）—— 支持微信语音](https://wufengxue.github.io/2019/04/17/wechat-voice-codec-amr.html)
* [（三）—— lame 移植](https://wufengxue.github.io/2019/05/25/wechat-voice-codec-lame.html)
* [（四）—— 整合 so 库](https://wufengxue.github.io/2019/06/29/wechat-voice-codec-lib.html)
* （五）——工作线程编解码
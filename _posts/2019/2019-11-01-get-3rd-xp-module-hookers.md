---
layout: post
title: '一种获取第三方 Xposed 模块全部钩子的方法'
subtitle: '通过 hook Xposed Framework API 实现'
date: 2019-11-01
categories: 技术
cover: '../../../assets/img/post_bg/20191101.jpeg'
tags: android Xposed
---


## 1. 前言

&#160; &#160; &#160; &#160;前段时间在分析一款 Xposed 模块时，发现其加固了。使用 [FDex2](https://bbs.pediy.com/thread-224105.htm) 成功脱出 dex 后，发现它的混淆做得极好，甚至字符串也做了加密，加之它的功能比较强大，钩子数量很多，静态分析基本行不通。

&#160; &#160; &#160; &#160;后来想起以前在[吾爱破解](https://www.52pojie.cn)看过一篇文章，作者也遇到了类似的问题，后面通过 hook Xposed 框架的 API 打印出了全部的钩子。因为时间比较久远，无法找到那篇文章，而且作者只提了思路，并没有给出具体实现，所以就自己实现了一下。


## 2. 常规方法

&#160; &#160; &#160; &#160;先复习一下查看 Xposed 模块钩子的常规方法：

* 1、查看 apk 的 assets/xposed_init 文件，获取模块入口类
* 2、使用反编译工具（如 [jeb](https://www.pnfsoftware.com)、[jadx-gui](https://github.com/skylot/jadx)）打开入口类
* 3、重点阅读 [handleLoadPackage](https://api.xposed.info/reference/de/robv/android/xposed/IXposedHookLoadPackage.html)、[initZygote](https://api.xposed.info/reference/de/robv/android/xposed/IXposedHookZygoteInit.html) 等 Xposed 模块入口方法
* 4、也可通过全局搜索 [XposedHelpers](https://api.xposed.info/reference/de/robv/android/xposed/XposedHelpers.html) 和 [XposedBridge](https://api.xposed.info/reference/de/robv/android/xposed/XposedBridge.html) 关键字进行定位


## 3. 新方法

### 3.1 思路

&#160; &#160; &#160; &#160;在编写 Xposed 模块的钩子时，都是通过以下接口实现：

&#160; &#160; &#160; &#160;[XposedHelpers](https://api.xposed.info/reference/de/robv/android/xposed/XposedHelpers.html)

```java
* static XC_MethodHook.Unhook	findAndHookConstructor(Class<?> clazz, Object... parameterTypesAndCallback)
	* Look up a constructor and hook it.
* static XC_MethodHook.Unhook	findAndHookConstructor(String className, ClassLoader classLoader, Object... parameterTypesAndCallback)
	* Look up a constructor and hook it.
* static XC_MethodHook.Unhook	findAndHookMethod(Class<?> clazz, String methodName, Object... parameterTypesAndCallback)
	* Look up a method and hook it.
* static XC_MethodHook.Unhook	findAndHookMethod(String className, ClassLoader classLoader, String methodName, Object... parameterTypesAndCallback)
	* Look up a method and hook it.
```

&#160; &#160; &#160; &#160;[XposedBridge](https://api.xposed.info/reference/de/robv/android/xposed/XposedBridge.html)

```java
* static Set<XC_MethodHook.Unhook>	hookAllConstructors(Class<?> hookClass, XC_MethodHook callback)
	* Hook all constructors of the specified class.
* static Set<XC_MethodHook.Unhook>	hookAllMethods(Class<?> hookClass, String methodName, XC_MethodHook callback)
	* Hooks all methods with a certain name that were declared in the specified class.
```

&#160; &#160; &#160; &#160;***因此，我们只需 hook Xposed Framework API 中的这些方法，将传入参数打印出来，就可以知道模块使用了哪些钩子。***


### 3.2 实现

&#160; &#160; &#160; &#160;移除存在间接调用的接口后，最终实现如下：

```java
    @Override
    public void initZygote(StartupParam startupParam) throws Throwable {
        hookXposedFrameworkApi(startupParam);
    }

    /**
     * 用于分析第三方 XP 模块采用了哪些钩子，主要用于模块包体比较大，
     * 做了加固或者混淆做得比较彻底的场景
     * <p>
     * 钩子的目标类和方法名，可在 Xposed Installer 的日志中查看
     */
    private void hookXposedFrameworkApi(StartupParam param) {
        hookXpFindAndHookMethod();
        hookXpFindAndHookConstructor();
        hookXpHookAllMethods();
        hookXpHookAllConstructors();
    }

    private void hookXpFindAndHookMethod() {
        try {
            Class cls = XposedHelpers.findClass(
                    "de.robv.android.xposed.XposedHelpers",
                    HookUtils.class.getClassLoader());
            XposedHelpers.findAndHookMethod(cls, "findAndHookMethod",
                    Class.class, String.class, Object[].class, new XC_MethodHook() {
                        @Override
                        protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                            super.beforeHookedMethod(param);
                            XposedBridge.log("findAndHookMethod: cls = " + param.args[0]
                                    + ", method = " + param.args[1]);
                        }
                    });
        } catch (Exception e) {
            XposedBridge.log(e);
        }
    }

    private void hookXpFindAndHookConstructor() {
        try {
            Class cls = XposedHelpers.findClass(
                    "de.robv.android.xposed.XposedHelpers",
                    HookUtils.class.getClassLoader());
            XposedHelpers.findAndHookMethod(cls, "findAndHookConstructor",
                    Class.class, Object[].class, new XC_MethodHook() {
                        @Override
                        protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                            super.beforeHookedMethod(param);
                            XposedBridge.log("findAndHookConstructor: cls = " + param.args[0]);
                        }
                    });
        } catch (Exception e) {
            XposedBridge.log(e);
        }
    }

    private void hookXpHookAllMethods() {
        try {
            Class cls = XposedHelpers.findClass(
                    "de.robv.android.xposed.XposedBridge",
                    HookUtils.class.getClassLoader());
            Class clsXC_MethodHook = XposedHelpers.findClass(
                    "de.robv.android.xposed.XC_MethodHook", HookUtils.class.getClassLoader()
            );
            XposedHelpers.findAndHookMethod(cls, "hookAllMethods",
                    Class.class, String.class, clsXC_MethodHook, new XC_MethodHook() {
                        @Override
                        protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                            super.beforeHookedMethod(param);
                            XposedBridge.log("hookAllMethods: cls = " + param.args[0]
                                    + ", method = " + param.args[1]);
                        }
                    });
        } catch (Exception e) {
            XposedBridge.log(e);
        }
    }

    private void hookXpHookAllConstructors() {
        try {
            Class cls = XposedHelpers.findClass(
                    "de.robv.android.xposed.XposedBridge",
                    HookUtils.class.getClassLoader());
            Class clsXC_MethodHook = XposedHelpers.findClass(
                    "de.robv.android.xposed.XC_MethodHook", HookUtils.class.getClassLoader()
            );
            XposedHelpers.findAndHookMethod(cls, "hookAllConstructors",
                    Class.class, clsXC_MethodHook, new XC_MethodHook() {
                        @Override
                        protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                            super.beforeHookedMethod(param);
                            XposedBridge.log("hookAllConstructors: cls = " + param.args[0]);
                        }
                    });
        } catch (Exception e) {
            XposedBridge.log(e);
        }
    }
```

### 3.3 说明

* 1、实际验证时，发现对 Xposed Framework API 的 hook 操作，只能放在 [initZygote](https://api.xposed.info/reference/de/robv/android/xposed/IXposedHookZygoteInit.html) 中执行，放在 [handleLoadPackage](https://api.xposed.info/reference/de/robv/android/xposed/IXposedHookLoadPackage.html) 中不会触发（具体原因留待后续分析）
* 2、在使用时，***记得只保留目标模块和实现了上述钩子的模块，以排除其他模块的干扰***
* 3、触发目标模块后，可以在 [Xposed Installer](https://github.com/rovo89/XposedInstaller) 的日志中查看目标模块触发的全部钩子（也可先导出到 SD 卡，再拉到电脑后查看）
* 4、代码也可见我个人的[测试小模块](https://github.com/WuFengXue/MyXposed/blob/master/app/src/main/java/com/newgame/reinhard/myxposed/HookUtils.java)


### 3.4 优缺点

*优点*

* 1、无需反编译
* 2、即使模块使用了加固和字符串加密，也可以获取到钩子

*缺点*

* 1、只能获取到钩子，钩子具体做了什么，仍然只能分析模块的代码实现

## 4. 总结
&#160; &#160; &#160; &#160;你无法确定现在学到的知识是否会在将来的某一天派上用场，所以努力扩充你的知识，善待周边的人吧！


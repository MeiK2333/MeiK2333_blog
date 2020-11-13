---
title: "Android Hook 技术"
date: 2020-11-11T10:53:37+08:00
---

浅谈 Android Hook 技术。

<!--more-->

## 什么是 Hook

Hook 直译为中文就是“钩子”，一般指将自定义的逻辑“钩”在要 Hook 的事件上。在 Android 平台上，Hook 的目标事件的粒度一般为函数级别，即在指定函数被调用时执行我们 Hook 的代码，根据我们的逻辑拦截修改原有的函数响应。

## Hook 可以做什么

- 对于 APP 开发者来说，可以通过 Hook 实现 AOP 编程、插桩性能监控、热补丁、在线修复等
- 对于用户来说，可以使用 Hook 来管理应用权限，保护隐私等；也可以通过 Hook 来跳过 APP 开屏广告、自动抢红包等，获取更好的使用体验

<div style="text-align:center">
    <img src="/images/android-hook/关闭广告.jpg" />
    <figcaption style="font-size: 14px; padding-bottom: 4px;">永远点不到的关闭广告按钮</figcaption>
</div>

## 常见的 Hook 方案

安卓下 Hook 的方案五花八门，有基于 Linux 调试的、基于文件替换的、基于动态库加载的、基于 JVM 虚拟机的等等，可谓是八仙过海，各显神通。我们将其分为以下几大类，分别介绍一下。

### root 环境下全局 Hook

功能最强大，也是技术要求最高的一种方案，需要用户有一定的动手能力来刷机等。借助 root 后的 sudo 权限，Hook 基本可以做到为所欲为。

<div style="text-align:center">
    <img src="/images/android-hook/root.jpg" />
    <figcaption style="font-size: 14px; padding-bottom: 4px;">通过刷机进行 root</figcaption>
</div>

这种方案下，用户能够获取手机的所有权限，不止能修改 Java 方法，向 APP 注入代码，甚至可以修改系统本身。用户在 root 后安装的插件可以做到最大程度的优化用户体验，但也可以做到获取用户隐私数据、拦截短信电话等恶意操作。

事实上，已经出现过很多次因用户 root 后安装第三方插件而导致的安全问题了。因此，这种方案虽然功能最强大，但同时也是最危险的。

### 非 root 环境下 Hook 自身

应用最广泛的方案，每个安卓用户都应该有意无意的使用过这种 Hook 技术。为了更好的体验（插桩、AOP、热修复等），很多开发者都会在 APP 中搭载这些功能，尤其是大厂的应用，无一例外都使用了这个机制。

<div style="text-align:center">
    <img src="/images/android-hook/hotfix.gif" />
    <figcaption style="font-size: 14px; padding-bottom: 4px;">修复线上 BUG</figcaption>
</div>

因为只能由 APP 自己来 Hook 自己，因此这种方案几乎没有危害，也不会有什么安全隐患（也不是完全没有，APP 可以通过动态下载代码执行来绕过商店审核和安全扫描，不过使用大厂的 APP 基本不会有这种问题）。

### 非 root 环境下 Hook 其他 APP

root 后 Hook 虽然功能强大，但很危险，非 root 环境下 Hook 自身对用户来说又作用不大，那么，有没有两全其美的方案呢？

<div style="text-align:center">
    <img src="/images/android-hook/allbuy.jpg" />
    <figcaption style="font-size: 14px; padding-bottom: 4px;">我全都要</figcaption>
</div>

答案是肯定的，为了实现这个目标，很多项目使用了不同的方案，从不同的角度切入。有通过虚拟化空间来运行 APP 来实现的，有通过重新打包 APP 来实现的，还有使用类似 Docker 的容器方案来实现的。令人惊艳的套路层出不穷。

## 不同 Hook 方案的原理与实现

<div style="text-align:center">
    <img src="/images/android-hook/hook.jpg" />
    <figcaption style="font-size: 14px; padding-bottom: 4px;">不同的 Hook 技术，图转自知乎@wawa yang</figcaption>
</div>

### [原版 Xposed](https://repo.xposed.info/module/de.robv.android.xposed.installer)

Xposed 不知道是不是出现最早的 Android Hook 方案，但绝对是影响力最大的方案。后面出现的 Android Hook 框架几乎都使用了 Xposed 的 API 设计，也因此，大部分框架的模块都可以复用。

对安卓开发有过一定了解的同学应该都知道，Android 上所有 APP 进程都是一个叫 `Zygote` 进程的子进程，这个进程是由 `/system/bin/app_process` 这个可执行文件执行得到的。在安卓的安全机制里，父进程是可以对自身的子进程进程调试注入的，因此，替换这个统一的入口，即可做到 Hook 任意 APP。

Xposed 具体的实现为：修改 `ART/Davilk` 虚拟机，将要 Hook 的函数注册为 `Native` 函数，虚拟机在调用同名函数时，会优先选择 `Native` 方法执行，从而达到 Hook 指定函数的目的。

在最初的时期，Xposed 可谓是一统 Hook 江湖，叱咤风云。然而由于不同版本的 Android 内核都不同，再加上很多厂商会自己对 AOSP 做很多魔改，导致 Xposed 需要做大量的适配工作。同时替换系统文件的方式也容易造成系统不稳定，导致系统崩溃等。近几年原版的 Xposed 也慢慢停止了开发，最终版本止步 Android 8。

### [EdXposed](https://github.com/ElderDrivers/EdXposed)

Xposed 的精神继承者，官方推荐的继承者。API 设计与原版 Xposed 相同，因此双方的插件都是可以复用的。

EdXposed 使用 [Riru](https://github.com/RikkaApps/Riru) 来注入系统，Riru 选定了一个系统自带的动态链接库 `libmemtrack.so` 进行修改，这个库非常简单，只有 10 个导出函数。

Riru 会在自己的代码里实现这 10 个导出函数，然后在额外附带自己的 Hook 代码，编译为 `so` 文件后替换掉原始的 `libmemtrack.so`，当 `Zygote` 进程启动时会链接这个库，执行内部的代码，从而实现注入。在这个基础上，EdXposed 封装出与 Xposed 相同的 API。

因为几乎没有手机厂商会修改这个简单的动态链接库，因此 Riru 不需要频繁的进行适配更新，EdXposed 也比原版的 Xposed 更加稳定。

在最近的更新中，Riru 使用了新方案进行注入，不再替换系统原有文件，基本杜绝了因为替换系统文件而导致的不稳定与崩溃。不过截至本文发布时，EdXposed 还没有合并这个更新。

因为需要修改系统文件，Xposed 和 EdXposed 都需要手机系统 root 之后才能使用。

### 太极（Tai Chi）

与上面两个框架不同，太极无需 root 就可以对 APP 进行 Hook。因为太极并没有开源，因此对于它的具体实现我们也无从了解，只能从开发者的一些言论中猜测，太极通过“解包 APP -> 添加 Hook 代码 -> 重新打包”的方式实现无 root Hook。

由于太极并未开源，内部实现完全是黑盒；同时太极内置了对不同 APP 的特殊逻辑，比如不能对支付宝进行 Hook，这些信息用户是完全无法感知的，因此太极的安全性存疑。

Magisk 框架（强大的 root 解决方案，Xposed、EdXposed 以及几乎所有 root 类型框架的基础）的作者曾公开发表言论认为太极会进行恶意活动，太极的开发者对此进行了否认，但作为用户，我们完全不知道太极框架都做了哪些操作。安装太极之后，太极可以获取我们 Hook 的 APP 的完全控制权，一个不知道它做了什么的高权限的第三方应用十分危险，我个人不推荐使用（仅在安全性上，无需 root 就可以 Hook 其他 APP，太极确实是个很优秀的框架）。

### [epic](https://github.com/tiann/epic)

并不是所有的 Hook 都是为了对 APP 进行魔改，也有一些 Hook 是出于优化体验的，比如 epic。

epic 本身有一篇讲解非常详细的[文章](http://weishu.me/2017/11/23/dexposed-on-art/)，这里只大概介绍一下原理：基于 `dynamic dispatch` 的 `dynamic callee-side rewriting`，一头雾水对吧，其实我也一知半解，大概描述一下的话，就是 epic 会去修改函数的内容，将函数的前两条指令修改为跳转到自己自定义的逻辑，从而实现对任意方法的 Hook。

因为 Android 系统的限制，对函数体的注入，只能在同一个 Uid 的进程之间进行。Android 会给每个 APP 分配一个独立的 `uid`，因此，epic 只能对自己进行 Hook，epic 主要用于优化体验，比如热修复、AOP、插桩等。

### [VirtualApp](https://github.com/asLody/VirtualApp)

商业软件，提供了类似虚拟机的功能，可以在 VirtualApp 内部启动第三方 APP，VA 对内部的 APP 拥有完全的控制权。

4 年前 VirtualApp 开源版本停止更新，我们无法获知新版本的 VA 原理。就 4 年前的版本来分析，VA 会代理所有的系统服务，包括 `ActivityManager` 和 `PackageManager`，通过被代理的 `ActivityManager` 启动的 APP 会被 VA 归到自己麾下（同一个 `uid`），从而实现后续的代理。

VirtualApp 轻量、强大、无需 root，因此很受青睐，很多公司都购买了 VirtualApp 的商业授权。要注意的是，即使是四年前的开源版本，没有购买商业授权的话也是不允许用于商业活动的。

### [VirtualXposed](https://github.com/android-hacker/VirtualXposed)

epic 可以实现任意方法 Hook，但只能对同一个 `uid` 的进程生效，VA 可以运行任意 APP，使其与自己使用同一个 `uid`。结合上面两种技术，有没有想到什么？

是的，这就是 VirtualXposed，结合了 epic 与 VA 的终极框架。无需 root，类 Xposed 的 API，可以说是一个相当完善的方案了。

可惜，由于 VirtualApp 的许可证要求，我们不能使用 VA 及其衍生项目进行商业活动，只能个人使用。

### 各类 XX OS

独辟蹊径的想法：Android 系统也是基于 Linux 的，那我们能不能像 Linux 上运行 Docker 一样，在 Android 上再运行一个 Android 呢？

答案是可以的，这就是近几年层出不穷的各种 XX OS。如果说 VirtualApp 通过代理提供了类似虚拟机的功能，那这些 OS 就是真的提供了一个虚拟机。这些 OS 在 Android 宿主机上运行一个新的 Android 虚拟机，宿主机拥有对虚拟机的完全控制权限，因此可以使用各种 Hook 技术。

然而，基于虚拟机的 Android，虽然易于控制，但性能也令人堪忧。据我亲身体验，只是打开这些 OS 就会占用手机不少资源，如果在这些 OS 里面玩游戏，那么手机就会秒变暖手宝……

## Android Hook 难点

从上面可以看出来，Android 的 Hook 技术可谓是百花齐放，但大体来说，都有下面两个痛点。

- 版本兼容：每个大版本，Google 都会毫不留情的推翻重构 Android 的大量特性，也导致 Hook 框架需要对不同的版本进行兼容
- 厂商魔改：Android 大版本有 4 - 11 共 8 个，开发者还可以跟得上兼容。但加上厂商魔改，这个数字直接涨了一个数量级。要想跟上所有厂商的魔改可不是件容易的事。大的项目靠大量的贡献者贡献代码，小的项目跟不上厂商魔改的速度，就只能慢慢没落了。

## Hook 防范

用户可以对 Hook 后的 APP 进行各种修改，跳过广告、自动抢红包、自动刷单等等，这无疑会损害商家的利益。因此，就像用户想尽方法 Hook APP 一样，APP 也会想尽方法来防止自己被 Hook。

常见的反 Hook 的手段有以下几种。

### Xposed 检测

最知名、应用最广泛的 Hook 技术，肯定也是要第一个被检测的。通过 `PackageManager` 或者 `ps` 等方法获取手机内已安装和正在运行的应用，如果检测到了 Xposed 就可以停止服务来防止受到损失。

### 自造异常读取栈

绝大部分 Hook 方案都采用 Xposed API，也就是在 Hook 的函数前后添加自定义的逻辑。这时候我们就可以通过触发异常来检查异常栈，查看是否有 Xposed 参与其中。

```java
try {
    throw new Exception("blah");
} catch(Exception e) {
    for (StackTraceElement stackTraceElement: e.getStackTrace()) {
        // stackTraceElement.getClassName() stackTraceElement.getMethodName() 是否存 在Xposed
    }
}
```
```java
E/GEnvironment: no such table: preference (code 1): while compiling: SELECT keyguard_show_livewallpaper FROM preference
...
at com.meituan.test.extpackage.ExtPackageManager.checkUpdate(ExtPackageManager.java:127)
at com.meituan.test.MiFGService$1.run(MiFGService.java:41)
at android.os.Looper.loop(Looper.java:136)
at android.app.ActivityThread.main(ActivityThread.java:5072)
at java.lang.reflect.Method.invokeNative(Native Method)
at java.lang.reflect.Method.invoke(Method.java:515)
...
at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:793)
at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:609)
at de.robv.android.xposed.XposedBridge.main(XposedBridge.java:132) //发现Xposed模块
at dalvik.system.NativeStart.main(Native Method)
```

<div style="text-align:center">
    <figcaption style="font-size: 14px; padding-bottom: 4px;">代码来自《Android Hook技术防范漫谈》</figcaption>
</div>

### 检查方法类型

有些框架的 Hook 方法是将要注入的代码注册为 `Native` 方法，因此我们可以通过检测我们调用的方法有没有异常的变为 `Native` 方法，如果有的话就需要注意了。

### 动态链接库检查

通过读取 `/proc/self/maps` 文件来检查 APP 本身链接的动态链接库，看有没有异常的库。EdXposed 和 Riru 会通过随机名来绕过这类检查，其他 Hook 方法又很少会修改动态链接库，因此这种方法并不是很可靠，聊胜于无。

### ptrace 调试检查

一些基于 native 层注入的框架是通过 `ptrace` 来进行注入的，因为一切都发生在系统层面，在 Java 层很难检测。

不过 `ptrace` 有个特性：被 `ptrace` 的目标进程不能再次执行 `ptrace`，我们可以利用这个特性，在自己的代码里执行 `ptrace`，如果失败了，则说明我们很可能被人利用 `ptrace` 进行注入了。

## 参考

- [《VirtualApp 框架浅析》](https://www.jianshu.com/p/ce2f26f4b03b)
- [《VirtualApp沙盒基本原理》](http://rk700.github.io/2017/03/15/virtualapp-basic/)
- [《盘点Android常用Hook技术》](https://zhuanlan.zhihu.com/p/109157321)
- [《Android Hook技术防范漫谈》](https://tech.meituan.com/2018/02/02/android-anti-hooking.html)
- [《理解 Android Hook 技术以及简单实战》](https://www.jianshu.com/p/4f6d20076922)
- [《我为Dexposed续一秒——论ART上运行时 Method AOP实现》](http://weishu.me/2017/11/23/dexposed-on-art/)

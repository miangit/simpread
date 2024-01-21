> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-274137-1.htm)

> Android 不信任证书导致无法抓包的解决办法

Android 不信任证书导致无法抓包的解决办法

2022-8-23 14:18 6277

[举报](javascript:void(0);)

### Android 不信任证书导致无法抓包的解决办法

 [![](http://passport.kanxue.com/upload/avatar/709/887709.png?1659609858)](user-home-887709.htm) [Ankeys](user-home-887709.htm) ![](https://bbs.kanxue.com/view/img/rank/0.png)  ![](http://passport.kanxue.com/pc/view/img/star_0.gif) [ 举报](javascript:void(0);) 2022-8-23 14:18  6277

前言
==

众所周知, Android7.0 以后系统不在信任用户的证书. 这一改动使得我们在抓包的时候产生了诸多不便。在 Android 端很多人的做法是先刷入 Magsik 在通过 Magsik 模块的方式来将抓包证书修改成系统证书。  
比较出名的就是大名鼎鼎的 HttpCanary(小黄鸟), 只是它虽然抓包方便，调试起来却没有 Fiddler Charles 等抓包工具方便。  
接下来，我将介绍几种方法能让你像装了 Magsik 模块的小黄鸟一样方便的抓包，并且方便的调试，希望能对你有所帮助。

1. 懒人专用法
--------

Android 系统对证书的不信任是从高版本开始的, 那 == 直接用低版本的 Android 系统 == 即可完美解决这一问题。如果你手头没有低版本的安卓真机进行调试, 可以尝试用 PC 端的 Android 模拟器来解决。诸如夜神 (Nox)，MuMu 等等，绝大多数的模拟器都提供了 Android5.0 版本等低 Android 版本的系统。  
在使用这些低版本的系统进行抓包时, 除了 APP 使用上可能不如高版本流畅, 在抓包这一方面效率绝对是相当的不错。  
_注意 --> 某些 APP 由于不支持 32 位可能无法运行。_

2. 逆向破解法
--------

此法虽然名字高大上，实际上比较针对于自己写 APP 进行调试时使用。不过对于一些冷门的小软件或者是没那么注重安全的开发者所作出的产品来说，可能会有奇效。

 

⑴. 让 apk 的 targetSDKVersion <= 23 即可解决. 碰到做了反编译等保护的很难实现  
⑵. 在 res/xml 目录下添加 network_security_config.xml 文件, 并在 application 下设置好如下属性：  
`android:networkSecurityConfig="@xml/network_security_config"`  
_xml 文件模板:_

3. 移动证书法
--------

本文的重点方法，通过将证书移动至系统证书目录, 来解决不信任证书造成的无法抓包问题。

### (1). 导出证书

这里根据你的抓包工具来自行导出, 但是你需要注意的是，现阶段的 Android 有效时间超过两年的证书。所以你导出的来的证书最好有效期不要超过这个时长，以避免不必要的麻烦。  
此处以 Fiddler 为例，我们使用 Fiddler 的证书[制作工具](https://telerik-fiddler.s3.amazonaws.com/fiddler/addons/fiddlercertmaker.exe)来导出一个证书。

 

![](https://bbs.kanxue.com/upload/attach/202208/887709_NGXPQAVKBYEYMKZ.jpg)

### (2). 转换证书格式

使用 [OpenSSL](http://slproweb.com/products/Win32OpenSSL.html) 对 cer 证书进行格式转换, 变为 pem 格式

```
openssl x509 -inform DER -in FiddlerRoot.cer -out FiddlerRoot.pem
```

_注意自己替换证书路径_

 

在转换成 pem 格式后查看证书的 hash 值

```
openssl x509 -inform PEM -subject_hash_old -in FiddlerRoot.pem
```

如图所示:  
![](https://bbs.kanxue.com/upload/attach/202208/887709_7WAVMNWS6VENAPJ.jpg)  
记住图中圈出来的 hash 值，这个 hash 值就是最后转换出来的文件的文件名。  
此时我们直接将转换好的 pem 证书进行重命名即可, 如果你想用命令的话, Windows 下可以用 ren 命令

```
ren FiddlerRoot.pem e5c3944b.0
```

如果你在 Linux 操作或者 Android 中使用终端模拟器的话就用 mv 命令  
最后你将获得一个 e5c3944b.0 文件

### (3). 推送证书

将转换好的证书推送到 Android 的 **_/system/etc/security_** 目录下并赋予文件可读权限即可，这个目录就是系统证书的目录。

#### **推送方法 1**: 使用 adb 进行推送

首先 Android 端打开 adb 调试, 这里我是用的网络调试，也可以自己接线，如果是模拟器的话，也可以用模拟器的办法连接模拟器的 adb。  
![](https://bbs.kanxue.com/upload/attach/202208/887709_EE6PZJK5ANM6C85.jpg)

 

PC 端使用 adb 命令连接设备

> 首先与设备进行配对  
> **adb pair 192.168.2.43:41053**  
> 随后连接设备  
> **adb connect 192.168.2.43:37877**  
> 最后进入 shell 环境  
> **adb shell**

 

![](https://bbs.kanxue.com/upload/attach/202208/887709_THB9FCT2QT8PCE4.jpg)  
_注意上面连接设备时输入的 ip 和端口，是与设备截图中一一对应的。_

 

连接好设备后，来到目标目录

```
cd /system/etc/security
```

注意这里的目录权限，是`755`  
![](https://bbs.kanxue.com/upload/attach/202208/887709_WQWWCRAQQ423GPM.jpg)  
而我们想要往目录中存放文件需要写的权限，此处采用 adb 临时赋予`777`权限

```
chmod 777 cacerts
```

这里可能会失败，由于我是真机且已经获取了 root，所以直接`su`获取权限给目录加上了 777  
如果你是带 root 的模拟器，一般不需要专门 su 获取权限  
![](https://bbs.kanxue.com/upload/attach/202208/887709_QDGAW95YXEMNP65.jpg)  
如果碰到 == 提示 Read-only file system==，可以使用`mount -o remount,rw /system`, 将系统文件夹挂载为可读写。然后再用 chmod 赋予 777 权限。  
![](https://bbs.kanxue.com/upload/attach/202208/887709_GU3Z4Y3YYFMBY89.jpg)  
修改好权限后，输入`exit`退出 shell(如果你 su 了需要两次 exit 才能退出)。再用 adb 命令来推送证书文件

```
adb push E:\Desktop\e5c3944b.0 /system/etc/security/cacerts
```

_这里的证书文件目录记得自己替换_

* * *

 

如果你是模拟器的话，经过上边的操作应该已经推送成功了，不过我这里是真机所以无法直接推送到 / system 下，所以我这里稍微绕一下，先推送到手机内存中，然后再进入 shell 转移到 cacerts 目录下

```
adb push E:\Desktop\e5c3944b.0 /sdcard/
su
mv /sdcard/e5c3944b.0 /system/etc/security/cacerts
```

![](https://bbs.kanxue.com/upload/attach/202208/887709_3WXGE2A73Q3AXW7.jpg)  
这里还要检查下你把文件推送过去，一定要给读的权限，不然在已信任的证书中是看不到这个证书的

```
chmod 666 e5c3944b.0
```

![](https://bbs.kanxue.com/upload/attach/202208/887709_GVW8VNACVGEXX2C.jpg)

 

同样是真机的话记得推送完文件以后把目录的权限改回 755  
![](https://bbs.kanxue.com/upload/attach/202208/887709_4UEEW23UVAE6DM6.jpg)

#### **推送方法 2:Root 权限直接转移**

既然你已经有了 root 权限，完全可以通过软件直接在 Android 端直接进行转移  
![](https://bbs.kanxue.com/upload/attach/202208/887709_K6DBHMYHDZMQCWP.jpg)  
图示为使用 MT 管理器直接将证书移动到系统证书目录中

#### **总结:**

**_不管你用什么办法，只要把证书文件移动到`/system/etc/security/cacerts`这个目录下, 并赋予证书文件可读的权限即可._**

4. 全局代 {过}{滤} 理法
----------------

这个办法我并没有试验过，不过据说雷电模拟器是可行的

```
adb shell settings put global http_proxy <代{过}{滤}理ip>:<代{过}{滤}理端口>
```

  

[[CTF 入门培训] 顶尖高校博士及硕士团队亲授《30 小时教你玩转 CTF》，视频 + 靶场 + 题目！助力进入 CTF 世界](http://www.kanxue.com/book-brief-170.htm#h3a6WRhDT9Q_3D)

最后于 2022-8-23 14:25 被 Ankeys 编辑 ，原因： 修复图片缺失 [#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm) [#系统相关](forum-161-1-126.htm) [#其他](forum-161-1-129.htm)

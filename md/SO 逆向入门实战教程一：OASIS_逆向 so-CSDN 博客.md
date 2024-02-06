> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/qq_38851536/article/details/117418582)

#### 文章目录

*   *   *   [一、前言](#_3)
        *   [二、准备](#_13)
        *   [三、Unidbg 模拟执行](#Unidbg_46)
        *   [四、ExAndroidNativeEmu 模拟执行](#ExAndroidNativeEmu__217)
        *   [五、算法分析](#_267)
        *   [六、尾声](#_331)

#### 一、前言

这是 SO 逆向入门实战教程的第一篇，总共会有十三篇，十三个实战。有以下几个注意点：

*   主打**入门级**的实战，适合有一定基础但缺少实战的朋友（了解 JNI，也上过一些 Native 层逆向的课，但感觉实战匮乏，想要壮壮胆，入入门）。
*   侧重**新工具、新思路、新方法**的使用，算法分析的常见路子是 Frida Hook + IDA ，在本系列中，会淡化 Frida 的作用，采用 Unidbg Hook + IDA 的路线。
*   主打入门，但**并不限于入门**，你会在样本里看到有浅有深的魔改加密算法、以及 OLLVM、SO 对抗等内容。
*   细，非常的细，奶妈级教学。
*   一共十三篇，1-2 天更新一篇。每篇的资料放在文末的百度网盘中。另外，我创建了一个 Unidbg 学习交流群，欢迎私聊我入群玩耍。

#### 二、准备

![](https://img-blog.csdnimg.cn/20210531155613610.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
s 方法就是我们的分析目标，它接收两个参数。参数 1 是字节数组，参数二是布尔值，为 false。伪代码如下：

```
import java.nio.charset.StandardCharsets;

public class oasis {

    public static void main(String[] args) {
        String input1 = "aid=01A-khBWIm48A079Pz_DMW6PyZR8" +
                "uyTumcCNm4e8awxyC2ANU.&cfrom=28B529501" +
                "0&cuid=5999578300&noncestr=46274W9279Hr1" +
                "X49A5X058z7ZVz024&platform=ANDROID×tamp" +
                "=1621437643609&ua=Xiaomi-MIX2S__oasis__3.5.8_" +
                "_Android__Android10&version=3.5.8&vid=10190135" +
                "94003&wm=20004_90024";

        Boolean input2 = false;

        oasis test = new oasis();
        String sign = test.s(input1.getBytes(StandardCharsets.UTF_8), input2);
        System.out.println(sign);

    }

    public String s(byte[] barr, boolean z){
        return "Sign";
    };
}
```

返回值是 32 位的 Sign，比如伪代码中的输入，返回 **3882b522d0c62171d51094914032d5ea** ，且输入不变的情况下，输出也固定不变。

#### 三、Unidbg 模拟执行

目标函数的实现在 liboasiscore.so 中，在 IDA 中看一下  
![](https://img-blog.csdnimg.cn/20210531155654545.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
似乎并不是静态绑定，接下来去 JNI OnLoad 查看一下动态绑定的位置。

![](https://img-blog.csdnimg.cn/20210531155709508.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

很不幸的是，这个样本经过了 OLLVM 混淆，很难直接找到动态绑定的地址。

在以 Frida 为主的路子里，我们可以使用 [hook_RegisterNatives](https://github.com/lasting-yang/frida_hook_libart/blob/master/hook_RegisterNatives.js) 脚本得到动态绑定的地址，但这里我们采用 Unidbg 的方案。

下载 Unidbg 最新版，在 IDEA 中打开，跑通如图例子，和图示一致即可。

![](https://img-blog.csdnimg.cn/2021053115572159.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

首先搭一个架子，新建包和类，并把 APK 和 SO 资源放在目录下

![](https://img-blog.csdnimg.cn/20210531155731105.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

首先导入通用且标准的类库，然后一步步往下，如下写了注释

```
package com.lession1;

// 导入通用且标准的类库
import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.android.dvm.array.ByteArray;
import com.github.unidbg.memory.Memory;

import java.io.File;

// 继承AbstractJni类
public class oasis extends AbstractJni{
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;

    oasis() {
        // 创建模拟器实例,进程名建议依照实际进程名填写，可以规避针对进程名的校验
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.sina.oasis").build();
        // 获取模拟器的内存操作接口
        final Memory memory = emulator.getMemory();
        // 设置系统类库解析
        memory.setLibraryResolver(new AndroidResolver(23));
        // 创建Android虚拟机,传入APK，Unidbg可以替我们做部分签名校验的工作
        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\lession1\\lvzhou.apk"));
        // 加载目标SO
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\lession1\\liboasiscore.so"), true); // 加载so到虚拟内存
        //获取本SO模块的句柄,后续需要用它
        module = dm.getModule();
        vm.setJni(this); // 设置JNI
        vm.setVerbose(true); // 打印日志

        dm.callJNI_OnLoad(emulator); // 调用JNI OnLoad
    };

    public static void main(String[] args) {
        oasis test = new oasis();
    }
}
```

样本的 init 相关函数和 JNI OnLoad 函数已经运行过了，接下来 Run  
![](https://img-blog.csdnimg.cn/20210531155819986.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
日志中可以发现，JNI OnLoad 中主要做了两件事

*   签名校验
*   动态绑定

值得一提的是，如果创建 Android 虚拟机时，选择不传入 APK，填入 null，那么样本在 JNI OnLoad 中所做的签名校验，就需要我们手动补环境校验了。

![](https://img-blog.csdnimg.cn/2021053115583524.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
接下来就是如何执行我们目标函数的问题了，这并不是一个小问题。Unidbg 封装了相关方法执行 JNI 函数以及有符号函数等，但需要区分类方法和实例方法，我觉得有些别扭，在 Frida 的使用过程中，通过地址直接 Hook 和 Call 是一种非常美妙的体验，所以这里我们只介绍通过地址模拟执行这个更一般和通用的法子，如果对前一种方法感兴趣，可以看 Unidbg 提供的相关测试代码。

字节数组需要裹上 unidbg 的包装类，并加到本地变量里，两件事缺一不可。

```
package com.lession1;

// 导入通用且标准的类库
import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.android.dvm.array.ByteArray;
import com.github.unidbg.memory.Memory;

import java.io.File;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;

// 继承AbstractJni类
public class oasis extends AbstractJni{
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;

    oasis() {
        // 创建模拟器实例,进程名建议依照实际进程名填写，可以规避针对进程名的校验
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.sina.oasis").build();
        // 获取模拟器的内存操作接口
        final Memory memory = emulator.getMemory();
        // 设置系统类库解析
        memory.setLibraryResolver(new AndroidResolver(23));
        // 创建Android虚拟机,传入APK，Unidbg可以替我们做部分签名校验的工作
        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\lession1\\lvzhou.apk"));
        //
//        vm = emulator.createDalvikVM(null);

        // 加载目标SO
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\lession1\\liboasiscore.so"), true); // 加载so到虚拟内存
        //获取本SO模块的句柄,后续需要用它
        module = dm.getModule();
        vm.setJni(this); // 设置JNI
        vm.setVerbose(true); // 打印日志

        dm.callJNI_OnLoad(emulator); // 调用JNI OnLoad
    };

    public static void main(String[] args) {
        oasis test = new oasis();
        System.out.println(test.getS());
    }

    public String getS(){
        // args list
        List<Object> list = new ArrayList<>(10);
        // arg1 env
        list.add(vm.getJNIEnv());
        // arg2 jobject/jclazz 一般用不到，直接填0
        list.add(0);
        // arg3 bytes
        String input = "aid=01A-khBWIm48A079Pz_DMW6PyZR8" +
                "uyTumcCNm4e8awxyC2ANU.&cfrom=28B529501" +
                "0&cuid=5999578300&noncestr=46274W9279Hr1" +
                "X49A5X058z7ZVz024&platform=ANDROID×tamp" +
                "=1621437643609&ua=Xiaomi-MIX2S__oasis__3.5.8_" +
                "_Android__Android10&version=3.5.8&vid=10190135" +
                "94003&wm=20004_90024";
        byte[] inputByte = input.getBytes(StandardCharsets.UTF_8);
        ByteArray inputByteArray = new ByteArray(vm,inputByte);
        list.add(vm.addLocalObject(inputByteArray));
        // arg4 ,boolean false 填入0
        list.add(0);
        // 参数准备完成
        // call function
        Number number = module.callFunction(emulator, 0xC365, list.toArray())[0];
        String result = vm.getObject(number.intValue()).getValue().toString();
        return result;
    }
}
```

可以发现，通过地址方式调用，似乎有一点点麻烦，但这个麻烦其实并不大，我们常常需要对未导出函数进行分析，与其一会儿用符号名调用，一会儿用地址，不如统一用地址嘛。

运行测试  
![](https://img-blog.csdnimg.cn/20210531155907571.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
结果与 Hook 得到的一致，即模拟执行顺利完成。

#### 四、ExAndroidNativeEmu 模拟执行

这个样本非常简单，我们用 [ExAndroidNativeEmu](https://github.com/maiyao1988/ExAndroidNativeEmu) 来梅开二度。如果说 Unidbg 是小号游轮，那么 ExAndroidNativeEmu 就是皮划艇。在此处皮划艇可有可无，我们只是做个演示，但有时非皮划艇不可，遇到了我们再说。

```
import posixpath
from androidemu.emulator import Emulator, logger
from androidemu.java.classes.string import String

# Initialize emulator
emulator = Emulator(
    vfp_inst_set=True,
    vfs_root=posixpath.join(posixpath.dirname(__file__), "vfs")
)

# 加载SO
lib_module = emulator.load_library("tests/bin/liboasiscore.so")

# find My module
for module in emulator.modules:
    if "liboasiscore" in module.filename:
        base_address = module.base
        logger.info("base_address=> 0x%08x - %s" % (module.base, module.filename))
        break

# run jni onload
emulator.call_symbol(lib_module, 'JNI_OnLoad', emulator.java_vm.address_ptr, 0x00)
# 准备参数
a1 = "aid=01A-khBWIm48A079Pz_DMW6PyZR8uyTumcCNm4e8awxyC2ANU.&cfrom=28B5295010&cuid=5999578300&noncestr=46274W9279Hr1X49A5X058z7ZVz024&platform=ANDROID×tamp=1621437643609&ua=Xiaomi-MIX2S__oasis__3.5.8__Android__Android10&version=3.5.8&vid=1019013594003&wm=20004_90024"
# 通过地址直接调用
result = emulator.call_native(module.base + 0xC364 + 1, emulator.java_vm.jni_env.address_ptr, 0x00, String(a1).getBytes(emulator, String("utf-8")), 0)
# 打印结果
print("result:"+ result._String__str)
```

ExAndroidNativeEmu 同样提供了对 JNI 函数的调用封装，但我们这边依然用地址方式调用，就是不用，就是玩儿。

![](https://img-blog.csdnimg.cn/20210531155954393.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

可以发现结果也很顺利，在这个样本上，Unidbg 和 ExAndroidNativeEmu 都能很轻松的处理，其中 ExAndroidNativeEmu 的代码量甚至更少一些，这得益于样本中和 JAVA 层的交互极少，一旦涉及到 JNI 交互，皮划艇就让人难受了，在 Python 中补 JAVA 的逻辑，简直不是人该受的委屈。

但 ExAndroidNativeEmu 也有它的用武之地

*   代码量较少，适合学习和分析，可以方便的结合自己的知识和业务增删功能。
*   在样本比较简单的情况下（即与 JAVA 交互少，系统调用少，一切都少，只是纯粹的 Native 运算）甚至比 Unidbg 更好用。
*   ExAndroidNativeemu 的 code trace 做的比 Unidbg 好很多，在指令的 trace 上做了非常多的优化。

#### 五、算法分析

因为这是个简单的样本，输出又是 32 位，很容易就让人联想到哈希算法，掏出 [FindHash](https://github.com/Pr0214/findhash) 跑一下。

![](https://img-blog.csdnimg.cn/20210531160035569.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
运行 FindHash 提示的脚本，根据输出找到对应的函数并分析，很快就定位到 0x8AB2 这个函数，并且它是 MD5_Update 函数。如果对各类算法的原理缺少了解，可以看一下 R0ysue 的 SO 基础课哟，手算 MD5/SHA1/DES/AES + 工程实践 + 逆向分析，就等你来。

```
function hook_md5_update(){
    var targetSo = Module.findBaseAddress("liboasiscore.so");
    let relativePtr = 0x8AB2 + 1;

    console.log("Enter");
    let funcPtr = targetSo.add(relativePtr);
    Interceptor.attach(funcPtr,{
        onEnter:function (args) {
            console.log(args[2]);
            console.log(hexdump(args[1],{length:args[2].toInt32()}));

        },onLeave:function (retval){
        }
    })
}
```

Hook 结果

```
0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
b2fa5800  59 50 31 56 74 79 26 24 58 6d 2a 6b 4a 6b 6f 52  YP1Vty&$Xm*kJkoR
b2fa5810  2c 4f 70 6b 26 61 69 64 3d 30 31 41 2d 6b 68 42  ,Opk&aid=01A-khB
b2fa5820  57 49 6d 34 38 41 30 37 39 50 7a 5f 44 4d 57 36  WIm48A079Pz_DMW6
b2fa5830  50 79 5a 52 38 75 79 54 75 6d 63 43 4e 6d 34 65  PyZR8uyTumcCNm4e
b2fa5840  38 61 77 78 79 43 32 41 4e 55 2e 26 63 66 72 6f  8awxyC2ANU.&cfro
b2fa5850  6d 3d 32 38 42 35 32 39 35 30 31 30 26 63 75 69  m=28B5295010&cui
b2fa5860  64 3d 35 39 39 39 35 37 38 33 30 30 26 6e 6f 6e  d=5999578300&non
b2fa5870  63 65 73 74 72 3d 4a 32 33 33 39 67 41 43 79 30  cestr=J2339gACy0
b2fa5880  44 35 6b 33 32 39 35 33 71 30 31 67 74 66 36 78  D5k32953q01gtf6x
b2fa5890  30 38 31 39 26 70 6c 61 74 66 6f 72 6d 3d 41 4e  0819&platform=AN
b2fa58a0  44 52 4f 49 44 26 74 69 6d 65 73 74 61 6d 70 3d  DROID×tamp=
b2fa58b0  31 36 32 31 35 32 36 32 39 38 31 32 37 26 75 61  1621526298127&ua
b2fa58c0  3d 58 69 61 6f 6d 69 2d 4d 49 58 32 53 5f 5f 6f  =Xiaomi-MIX2S__o
b2fa58d0  61 73 69 73 5f 5f 33 2e 35 2e 38 5f 5f 41 6e 64  asis__3.5.8__And
b2fa58e0  72 6f 69 64 5f 5f 41 6e 64 72 6f 69 64 31 30 26  roid__Android10&
b2fa58f0  76 65 72 73 69 6f 6e 3d 33 2e 35 2e 38 26 76 69  version=3.5.8&vi
b2fa5900  64 3d 31 30 31 39 30 31 33 35 39 34 30 30 33 26  d=1019013594003&
b2fa5910  77 6d 3d 32 30 30 30 34 5f 39 30 30 32 34        wm=20004_90024
0x1a
           0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
bbe3c322  80 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
bbe3c332  00 00 00 00 00 00 00 00 00 00                    ..........
0x8
           0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
bae15ed8  f0 08 00 00 00 00 00 00
```

MD5 Update 一共被调用了三次，需要注意的是，MD5 的 Update 的后两次调用，都是数据的填充，属于算法内部细节，所以我们只用关注第一次的输出。

我们的明文是从 aid 开始的，前面多了一块，这一块每次运行都不变，所以猜测它是盐，使用逆向之友 [Cyberchef](https://gchq.github.io/CyberChef/#recipe=MD5%28%29&input=WVAxVnR5JiRYbSprSmtvUixPcGsmYWlkPTAxQS1raEJXSW00OEEwNzlQel9ETVc2UHlaUjh1eVR1bWNDTm00ZThhd3h5QzJBTlUuJmNmcm9tPTI4QjUyOTUwMTAmY3VpZD01OTk5NTc4MzAwJm5vbmNlc3RyPTQ2Mjc0VzkyNzlIcjFYNDlBNVgwNTh6N1pWejAyNCZwbGF0Zm9ybT1BTkRST0lEJnRpbWVzdGFtcD0xNjIxNDM3NjQzNjA5JnVhPVhpYW9taS1NSVgyU19fb2FzaXNfXzMuNS44X19BbmRyb2lkX19BbmRyb2lkMTAmdmVyc2lvbj0zLjUuOCZ2aWQ9MTAxOTAxMzU5NDAwMyZ3bT0yMDAwNF85MDAyNA) 测试一下:

![](https://img-blog.csdnimg.cn/20210531160124720.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

大功告成！

#### 六、尾声

这是一个非常简单的样本，用于熟悉 Unidbg 的简单操作。下一讲会复杂一点点的，熟悉 Unidbg 的更多基础操作。

样本https://github.com/miangit/simpread/releases/download/backup/unidbg1.7z

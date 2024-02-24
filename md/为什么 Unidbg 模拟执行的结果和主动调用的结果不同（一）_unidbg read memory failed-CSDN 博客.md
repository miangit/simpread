> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/qq_38851536/article/details/118334978)

#### 一、前言

前言打个广告，Unidbg 有很多实操问题，文章说起来不直观而且费劲，最典型的就是怎么补 JNI 环境，很多朋友都补的又慢又折腾，欢迎大家私信了解 Unidbg 课程。  
除此之外，建了一个 Unidbg 学习交流群，纯粹的技术交流，欢迎各位大佬来玩，私聊我拉进群。

接下来进入正题，看文章标题大家也能猜到，这一篇的内容是一个新系列的开始，这并不意味着《逆向十三篇》结束了，而是 **Unidbg 模拟执行的结果和主动调用的结果不同**这个问题太过重要了，单独归为一档比较合适。

#### 二、准备

![](https://img-blog.csdnimg.cn/20210629144142595.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

如图红框是我们的目标函数，使用 Frida 主动调用测试一下吧！

```
function callRs(){
    Java.perform(function() {
        var clazz = Java.use('com.xunmeng.pinduoduo.secure.SecureNative');
        var rsresulr = clazz.rs();
        console.log(rsresulr);
    })
}
```

![](https://img-blog.csdnimg.cn/20210629144156278.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
在我的测试机上结果为 1，需要注意的是，你的测试机可能不是这个值，先别急，继续看下去。

#### 三、Unidbg 模拟执行

```
package com.findDiff1;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.debugger.Debugger;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.linux.android.dvm.DalvikModule;
import com.github.unidbg.linux.android.dvm.VM;
import com.github.unidbg.memory.Memory;

import java.io.File;
import java.io.FileNotFoundException;
import java.util.ArrayList;
import java.util.List;

public class Rs extends AbstractJni {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;
    public final Memory memory;

    Rs(){
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.rs").build();

        memory = emulator.getMemory();
        memory.setLibraryResolver(new AndroidResolver(23)); // 设置系统类库解析

        vm = emulator.createDalvikVM(null); // 创建Android虚拟机
        vm.setVerbose(true); // 设置是否打印Jni调用细节
        DalvikModule dm = vm.loadLibrary(new File("libpdd_secure.so"), true);

        module = dm.getModule(); //
        vm.setJni(this);
        dm.callJNI_OnLoad(emulator);
    }

    public void call(){
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv());
        list.add(0);
        Number number = module.callFunction(emulator, 0x10321, list.toArray())[0];
        System.out.println(number.intValue());
    };

    public static void main(String[] args) {
        Rs test = new Rs();
        System.out.println("call rs");
        test.call();
    }

}
```

运行看一下，常规问题，JAVA 环境嘛，补一下  
![](https://img-blog.csdnimg.cn/20210629144840385.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

```
@Override
    public void callStaticVoidMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
        switch (signature) {
            case "com/tencent/mars/xlog/PLog->i(Ljava/lang/String;Ljava/lang/String;)V": {
                System.out.println("init PLog->i");
                return;
            }
        }
        super.callStaticVoidMethodV(vm, dvmClass, signature, vaList);
    }
```

![](https://img-blog.csdnimg.cn/20210629144216916.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
出结果了，-1，这是为什么呢？报错中有 memory failed 字眼，但具体是什么地方出了什么问题呢？把常见的日志开关都打开，再测试一次，报错发生在 libc 的 strlen 函数中，调用源来自样本 SO  
![](https://img-blog.csdnimg.cn/20210629144854297.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

顺道 IDA 中看看  
![](https://img-blog.csdnimg.cn/2021062914491369.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
strlen(v41) 出错了，v41 来自 getenv 函数。  
![](https://img-blog.csdnimg.cn/20210629144923846.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
那我们就得好好说 getenv 函数了

> getenv() 用来取得参数 envvar 环境变量的内容。参数为环境变量的名称，如果该变量存在则会返回指向该内容的指针，如果不存在则返回 null。

此处就是 getenv 没取到值，但样本的程序里没有考虑取不到值的情况，所以 strlen(null) 报错。

为什么会我们取不到值呢？Android 存在一些默认的系统环境变量，除此之外我们还可以自己增加环境变量。但是 Unidbg 没有这些环境变量，这就导致得不到结果啦。我们可以通过 adb 查看自己机子的环境变量

![](https://img-blog.csdnimg.cn/20210629144945155.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
我们也可以查看这些环境变量具体的值

```
C:\Users\pr0214>adb shell
bullhead:/ $ export
ANDROID_ASSETS
ANDROID_BOOTLOGO
ANDROID_DATA
ANDROID_ROOT
ANDROID_SOCKET_adbd
ANDROID_STORAGE
ASEC_MOUNTPOINT
BOOTCLASSPATH
DOWNLOAD_CACHE
EXTERNAL_STORAGE
HOME
HOSTNAME
LOGNAME
PATH
SHELL
SYSTEMSERVERCLASSPATH
TERM
TMPDIR
USER
bullhead:/ $ echo $HOME
/
bullhead:/ $ echo $ANDROID_DATA
/data
bullhead:/ $ echo $SYSTEMSERVERCLASSPATH
/system/framework/services.jar:/system/framework/ethernet-service.jar:/system/framework/wifi-service.jar:/system/framework/com.android.location.provider.jar
bullhead:/ $ echo $PATH
/sbin:/system/sbin:/system/bin:/system/xbin:/vendor/bin:/vendor/xbin
bullhead:/ $
```

接下来让我们通过 Unidbg 灵敏的 console debugger 来确认一下（之前已经讲过了 console debugger）——样本中 getenv 传入了某个环境变量  
![](https://img-blog.csdnimg.cn/20210629145013375.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
再在返回处下个断点  
![](https://img-blog.csdnimg.cn/20210629145024316.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
现在已经确认问题出在哪里了，接下来试图解决问题。Android 系统预设了一些环境变量，可以通过 getenv 去取出来，Unidbg 没有预设环境变量，导致返回空，进而导致后面 strlen 报错。那么应该有两条基本的修复路径，1 是能不能填上这些环境变量？

getenv 用于取环境变量，setenv 用于设置环境变量，我们主动调用 setenv，是不是就可以添加上环境变量了呢？

> NAME  
> setenv - change or add an environmentvariable // 改变或添加一个环境变量  
> SYNOPSIS  
> int setenv(const char _name, const char_value, int overwrite);
> 
> overwrite 参数：  
> 非 0 表示覆盖原有环境变量，0 表示不覆盖。

```
package com.findDiff1;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.Symbol;
import com.github.unidbg.debugger.Debugger;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.memory.Memory;
import com.github.unidbg.memory.MemoryBlock;
import org.apache.log4j.Level;
import org.apache.log4j.Logger;

import java.io.File;
import java.io.FileNotFoundException;
import java.util.ArrayList;
import java.util.List;

public class Rs extends AbstractJni {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;
    public final Memory memory;

    Rs(){
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.rs").build();

        memory = emulator.getMemory();
        memory.setLibraryResolver(new AndroidResolver(23)); // 设置系统类库解析

        vm = emulator.createDalvikVM(null); // 创建Android虚拟机
        vm.setVerbose(true); // 设置是否打印Jni调用细节
        DalvikModule dm = vm.loadLibrary(new File("libpdd_secure.so"), true);

        module = dm.getModule(); //
        vm.setJni(this);
        dm.callJNI_OnLoad(emulator);

        Debugger debugger = emulator.attach();
        debugger.addBreakPoint(module.base + 0x3436E + 1);
    }

    public void call(){
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv());
        list.add(0);
        Number number = module.callFunction(emulator, 0x10321, list.toArray())[0];
        System.out.println(number.intValue());
    };

    @Override
    public void callStaticVoidMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
        switch (signature) {
            case "com/tencent/mars/xlog/PLog->i(Ljava/lang/String;Ljava/lang/String;)V": {
                System.out.println("init PLog->i");
                return;
            }
        }
        super.callStaticVoidMethodV(vm, dvmClass, signature, vaList);
    }

    public void setEnv(){
        Symbol setenv = module.findSymbolByName("setenv", true);
        setenv.call(emulator, "PATH", "/sbin:/system/sbin:/system/bin:/system/xbin:/vendor/bin:/vendor/xbin", 0);
    };

    public static void main(String[] args) {
        Logger.getLogger("com.github.unidbg.linux.ARM32SyscallHandler").setLevel(Level.DEBUG);
        Logger.getLogger("com.github.unidbg.unix.UnixSyscallHandler").setLevel(Level.DEBUG);
        Logger.getLogger("com.github.unidbg.AbstractEmulator").setLevel(Level.DEBUG);
        Logger.getLogger("com.github.unidbg.linux.android.dvm.DalvikVM").setLevel(Level.DEBUG);
        Logger.getLogger("com.github.unidbg.linux.android.dvm.BaseVM").setLevel(Level.DEBUG);
        Logger.getLogger("com.github.unidbg.linux.android.dvm").setLevel(Level.DEBUG);
        Rs test = new Rs();
        test.setEnv();
        System.out.println("call rs");
        test.call();
    }

}
```

运行测试，发现确实过了这个坑，getenv 取到了我们 set 的值。

![](https://img-blog.csdnimg.cn/20210629145053522.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
但考虑一个问题，如果没有 setenv 函数怎么办，毕竟不是所有的 get 都有对应的 set。所以方法二就是 Hook 了。即 Hook getEnv 函数，将返回值改成正确的。

```
package com.findDiff1;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Emulator;
import com.github.unidbg.Module;
import com.github.unidbg.Symbol;
import com.github.unidbg.arm.context.EditableArm32RegisterContext;
import com.github.unidbg.debugger.Debugger;
import com.github.unidbg.hook.hookzz.HookEntryInfo;
import com.github.unidbg.hook.hookzz.HookZz;
import com.github.unidbg.hook.hookzz.IHookZz;
import com.github.unidbg.hook.hookzz.WrapCallback;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.memory.Memory;
import com.github.unidbg.memory.MemoryBlock;
import com.github.unidbg.pointer.UnidbgPointer;
import com.sun.jna.Pointer;
import org.apache.log4j.Level;
import org.apache.log4j.Logger;

import java.io.File;
import java.io.FileNotFoundException;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;

public class Rs extends AbstractJni {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;
    public final Memory memory;

    Rs(){
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.rs").build();

        memory = emulator.getMemory();
        memory.setLibraryResolver(new AndroidResolver(23)); // 设置系统类库解析

        vm = emulator.createDalvikVM(null); // 创建Android虚拟机
        vm.setVerbose(true); // 设置是否打印Jni调用细节
        DalvikModule dm = vm.loadLibrary(new File("libpdd_secure.so"), true);

        module = dm.getModule(); //
        vm.setJni(this);
        dm.callJNI_OnLoad(emulator);

        Debugger debugger = emulator.attach();
        debugger.addBreakPoint(module.base + 0x3436E + 1);
    }

    public void call(){
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv());
        list.add(0);
        Number number = module.callFunction(emulator, 0x10321, list.toArray())[0];
        System.out.println(number.intValue());
    };

    @Override
    public void callStaticVoidMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
        switch (signature) {
            case "com/tencent/mars/xlog/PLog->i(Ljava/lang/String;Ljava/lang/String;)V": {
                System.out.println("init PLog->i");
                return;
            }
        }
        super.callStaticVoidMethodV(vm, dvmClass, signature, vaList);
    }

    public void setEnv(){
        Symbol setenv = module.findSymbolByName("setenv", true);
        setenv.call(emulator, "PATH", "/sbin:/system/sbin:/system/bin:/system/xbin:/vendor/bin:/vendor/xbin", 0);
    };

    public void hookgetEnv(){
        IHookZz hookZz = HookZz.getInstance(emulator);

        hookZz.wrap(module.findSymbolByName("getenv"), new WrapCallback<EditableArm32RegisterContext>() {
            String name;
            @Override
            public void preCall(Emulator<?> emulator, EditableArm32RegisterContext ctx, HookEntryInfo info) {
                name = ctx.getPointerArg(0).getString(0);
            }
            @Override
            public void postCall(Emulator<?> emulator, EditableArm32RegisterContext ctx, HookEntryInfo info) {
                switch (name){
                    case "PATH":{
                        Pointer result = ctx.getPointerArg(0);
                        result.setString(0, "/sbin:/system/sbin:/system/bin:/system/xbin:/vendor/bin:/vendor/xbin");
                    }
                }

            }
        });
    };

    public static void main(String[] args) {
        Logger.getLogger("com.github.unidbg.linux.ARM32SyscallHandler").setLevel(Level.DEBUG);
        Logger.getLogger("com.github.unidbg.unix.UnixSyscallHandler").setLevel(Level.DEBUG);
        Logger.getLogger("com.github.unidbg.AbstractEmulator").setLevel(Level.DEBUG);
        Logger.getLogger("com.github.unidbg.linux.android.dvm.DalvikVM").setLevel(Level.DEBUG);
        Logger.getLogger("com.github.unidbg.linux.android.dvm.BaseVM").setLevel(Level.DEBUG);
        Logger.getLogger("com.github.unidbg.linux.android.dvm").setLevel(Level.DEBUG);
        Rs test = new Rs();
//        test.setEnv();
        test.hookgetEnv();
        System.out.println("call rs");
        test.call();
    }

}
```

我们的代码看起来很合理，运行时却出错了  
![](https://img-blog.csdnimg.cn/20210629145112220.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
正常情况下，getenv 返回指向环境变量值的指针，但因为找不到环境变量，所以直接返回了 0 而不是指针，从而导致代码出了问题。

那该怎么写呢？看一下吧

```
public void hookgetEnv(){
        IHookZz hookZz = HookZz.getInstance(emulator);

        hookZz.wrap(module.findSymbolByName("getenv"), new WrapCallback<EditableArm32RegisterContext>() {
            String name;
            @Override
            public void preCall(Emulator<?> emulator, EditableArm32RegisterContext ctx, HookEntryInfo info) {
                name = ctx.getPointerArg(0).getString(0);
            }
            @Override
            public void postCall(Emulator<?> emulator, EditableArm32RegisterContext ctx, HookEntryInfo info) {
                switch (name){
                    case "PATH":{
                        MemoryBlock replaceBlock = memory.malloc(0x100, true);
                        UnidbgPointer replacePtr = replaceBlock.getPointer();
                        String pathValue = "/sbin:/system/sbin:/system/bin:/system/xbin:/vendor/bin:/vendor/xbin";
                        replacePtr.write(0, pathValue.getBytes(StandardCharsets.UTF_8), 0, pathValue.length());
                        ctx.setR0(replacePtr.toIntPeer());
                    }
                }

            }
        });
    };
```

两种方法全部演示完毕，我们接着往下看

![](https://img-blog.csdnimg.cn/20210629145127743.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
可以发现绕过这个坑后，我们直接得到了结果，0。可是我们主动调用时，结果明明是 1 呢？看一下日志  
![](https://img-blog.csdnimg.cn/20210629145143480.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
原来样本与文件系统有大量的交互，所以需要继续做处理，将其涉及到的文件，一律拷贝到之前讲过的 rootfs 对应目录下。才补充了第一个文件，结果就成 1 了。  
![](https://img-blog.csdnimg.cn/20210629145210127.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

![](https://img-blog.csdnimg.cn/20210629145216460.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

如果想完完全全一模一样，可以接着继续补。

#### 四、尾声

在我的另外一台日用机上，RS 的结果为 5，你知道怎么补到 5 吗？欢迎来到《Unidbg 应用与实战》课堂上寻找答案哟。  
链接：https://pan.baidu.com/s/1QeXcCBFj_JqBoKr5LBIgwQ  
提取码：sc4j
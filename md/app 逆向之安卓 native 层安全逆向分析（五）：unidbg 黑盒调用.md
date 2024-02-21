> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/Eeyhan/p/17356251.html)

前言
--

继续跟着龙哥的 unidbg 学习：[SO 逆向入门实战教程五：qxs_白龙~ 的博客 - CSDN 博客](https://blog.csdn.net/qq_38851536/article/details/117828334)

还是那句，我会借鉴龙哥的文章，以一个初学者的角度，加上自己的理解，把内容丰富一下，尽量做到不在龙哥的基础上画蛇添足，哈哈。感谢观看的朋友

分析
--

首先，安装 app，发现龙哥给的 apk 包已经安装不上，网上重新找了个最新版，安装后，正常打开，然后，抓个包：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230426153119722-813467439.png)

ok，这个 sfsecurity 参数就是今天的重点了。

这个 app 有 ajm 的壳，没事，脱壳手段上，结果搞半天没脱出来，卧槽了，而且还有 frida 反调试

脱壳机也不在身边，那没法，直接黑盒调用，unidbg 就是干这个的。

调试
--

### 1. 搭架子

运行没问题

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230426160412648-1745435186.png)

不过这里，unidbg 并没有打印出来我们要的方法和地址 

### 2.ida 找地址

这时候就需要用 ida 找地址了，结果发现，这个 so 文件有点奇怪，搜不到，但是在导出表里看得到，这就很奇怪了，根据分析，猜测是有一定的保护，也没办法通过 F5 反编译 

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230426160559210-803794212.png)

很快就找到这里了，【0xAB2C】 ，注意，这里我用的是最新版，所以地址跟龙哥博客的地址不一样

### 3. 黑盒调用 & 补环境

这里，我选用的是 32 位，所以得加 1，因为我把 apk 解包，从 lib\armeabi-v7a 里找的这个 so 文件，所以他一定是 32 位的（根据安卓开发规范）

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230426160745263-1972094406.png)

ok，报了个环境的错，补一补：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230426160922491-938914907.png)

 ok，又有新的报错，继续补：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230426161004600-1369158626.png)

ok，搞定

这里，由于我是根据龙哥的博客，知道是哪个 so 方法，所以直接黑盒调用，也没有用 objection hook，它有 frida 反调试（objection 基于 frida），那就尴尬了，所以这里没法验证 hook 值和主动调用值是否一致了。那就暂时这样，反正值是有了

完整代码：

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

```
package com.qxs;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.android.dvm.api.Binder;
import com.github.unidbg.linux.android.dvm.api.ServiceManager;
import com.github.unidbg.memory.Memory;

import java.io.File;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.security.cert.CertificateException;
import java.security.cert.CertificateFactory;
import java.util.ArrayList;
import java.util.List;
import java.util.Locale;
import java.util.UUID;

public class qxs extends AbstractJni {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;

    qxs() {
        // 创建模拟器实例,进程名建议依照实际进程名填写，可以规避针对进程名的校验
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.sina.oasis").build();
        // 获取模拟器的内存操作接口
        final Memory memory = emulator.getMemory();
        // 设置系统类库解析
        memory.setLibraryResolver(new AndroidResolver(23));
        // 创建Android虚拟机,传入APK，Unidbg可以替我们做部分签名校验的工作
        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\qxs\\com.sfacg.apk"));
        // 加载目标SO
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\qxs\\libsfdata.so"), true); // 加载so到虚拟内存
        //获取本SO模块的句柄,后续需要用它
        module = dm.getModule();
        vm.setJni(this); // 设置JNI
        vm.setVerbose(true); // 打印日志

        dm.callJNI_OnLoad(emulator); // 调用JNI OnLoad
    }
    public static void main(String[] args) {
        qxs test = new qxs();
        System.out.println(test.sfsecurity());
    }
    public String sfsecurity(){
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv());
        list.add(0);
        DvmObject<?> context = vm.resolveClass("android/content/Context").newObject(null);
        list.add(vm.addGlobalObject(context));
        list.add(vm.addLocalObject(new StringObject(vm,"test")));
        Number number =  module.callFunction(emulator,0xAB2C+1,list.toArray());
        String result = vm.getObject(number.intValue()).getValue().toString();
        return result;
    }
    @Override
    public DvmObject<?> callStaticObjectMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
        switch (signature) {
            case "java/util/UUID->randomUUID()Ljava/util/UUID;":
                return dvmClass.newObject(UUID.randomUUID());
        }
        return super.callStaticObjectMethodV(vm, dvmClass, signature, vaList);
    }

    @Override
    public DvmObject<?> callObjectMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {
        switch (signature) {
            case "java/util/UUID->toString()Ljava/lang/String;":
                String uuid = dvmObject.getValue().toString();
                return new StringObject(vm, uuid);
        }
        return super.callObjectMethodV(vm, dvmObject, signature, vaList);
    }
}
```

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

知识点总结
-----

本篇和上一篇感觉的挺简单的，所以没有啥总结的
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/Eeyhan/p/17353245.html)

前言
--

继续跟着龙哥的 unidbg 学习：[SO 逆向入门实战教程四：mfw_so 层逆向_白龙~ 的博客 - CSDN 博客](https://blog.csdn.net/qq_38851536/article/details/117550316)

还是那句，我会借鉴龙哥的文章，以一个初学者的角度，加上自己的理解，把内容丰富一下，尽量做到不在龙哥的基础上画蛇添足，哈哈。感谢观看的朋友

分析
--

首先打开目标 app，然后随便搜个东西，抓个包看看：  

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230425160756788-1854271395.png)

其中，上面的这个 zzzghostsigh 参数就是我们要逆向的加密参数了。

### 1. 找加密位置

jadx 直接搜：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230425160925871-384214542.png)

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230425160933309-1781897363.png)

然后再搜这个【HTTP_BASE_PARAM_GHOSTSIGH】

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230425161004641-1679887456.png)

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230425161114153-1212811668.png)

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230425161147373-1484198487.png)

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230425161204876-842121520.png)

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230425161236708-1735568513.png)

 ok，根据刚才的逻辑跟栈，就是这个 xPreAuthencode 方法了

objection hook 确认：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230425163639752-210110757.png)

好的，就是这里了，看到第一个参数是一个 context，第二个是哥字符串，感觉像是把 url 啥的全拼到一起了，第三个参数就是这个 app 的包名

### 2.ida 分析 so

把目标 so 文件拖进去，卧槽，一团乱，也是动态注册的，卧槽

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230425161520524-1740403567.png)

多说无益，直接用 unidbg 吧

unidbg 调试
---------

### 1. 搭架子

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

```
package com.mfw;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.linux.android.dvm.DalvikModule;
import com.github.unidbg.linux.android.dvm.VM;
import com.github.unidbg.memory.Memory;

import java.io.File;

public class roadbook extends AbstractJni {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;

    roadbook() {
        // 创建模拟器实例,进程名建议依照实际进程名填写，可以规避针对进程名的校验
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.sina.oasis").build();
        // 获取模拟器的内存操作接口
        final Memory memory = emulator.getMemory();
        // 设置系统类库解析
        memory.setLibraryResolver(new AndroidResolver(23));
        // 创建Android虚拟机,传入APK，Unidbg可以替我们做部分签名校验的工作
        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\mfw\\mafengwo.apk"));
        // 加载目标SO
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\mfw\\libmfw.so"), true); // 加载so到虚拟内存
        //获取本SO模块的句柄,后续需要用它
        module = dm.getModule();
        vm.setJni(this); // 设置JNI
        vm.setVerbose(true); // 打印日志
        dm.callJNI_OnLoad(emulator); // 调用JNI OnLoad
    }


    public static void main(String[] args) {
        roadbook test = new roadbook();
    }
}
```

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

运行一下，好像没有啥问题，而且还把地址打印出来了：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230425161923389-1284460136.png)

### 2. 主动调用

直接干吧，没啥好说的，把 hook 的参数复制过去

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

```
package com.mfw;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.memory.Memory;

import java.io.File;
import java.util.ArrayList;
import java.util.List;

public class roadbook extends AbstractJni {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;

    roadbook() {
        // 创建模拟器实例,进程名建议依照实际进程名填写，可以规避针对进程名的校验
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.sina.oasis").build();
        // 获取模拟器的内存操作接口
        final Memory memory = emulator.getMemory();
        // 设置系统类库解析
        memory.setLibraryResolver(new AndroidResolver(23));
        // 创建Android虚拟机,传入APK，Unidbg可以替我们做部分签名校验的工作
        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\mfw\\mafengwo.apk"));
        // 加载目标SO
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\mfw\\libmfw.so"), true); // 加载so到虚拟内存
        //获取本SO模块的句柄,后续需要用它
        module = dm.getModule();
        vm.setJni(this); // 设置JNI
        vm.setVerbose(true); // 打印日志
        dm.callJNI_OnLoad(emulator); // 调用JNI OnLoad
    }


    public static void main(String[] args) {
        roadbook test = new roadbook();
        System.out.println(test.xPreAuthencode());
    }
    public String xPreAuthencode(){
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv());
        list.add(0);
        DvmObject<?> context = vm.resolveClass("android/content/Context").newObject(null);
        list.add(vm.addGlobalObject(context));
        list.add(vm.addLocalObject(new StringObject(vm,"GET&https%3A%2F%2Fmapi.mafengwo.cn%2Frest%2Fapp%2Fv2%2Fmdd%2Fposition%2F104.068237_30.537864&app_code%3Dcom.mfw.roadbook%26app_ver%3D9.3.7%26app_version_code%3D734%26brand%3DXiaomi%26channel_id%3DMFW%26dev_ver%3DD1907.0%26device_id%3D4C%253A49%253AE3%253ACA%253A75%253A35%26device_type%3Dandroid%26hardware_model%3DMI%25206%26has_notch%3D0%26mfwsdk_ver%3D20140507%26o_lat%3D30.537864%26o_lng%3D104.068237%26oauth_consumer_key%3D5%26oauth_nonce%3D0b4bc80a-6496-4e96-82f9-b67d289759d1%26oauth_signature_method%3DHMAC-SHA1%26oauth_timestamp%3D1682412429%26oauth_token%3D0_0969044fd4edf59957f4a39bce9200c6%26oauth_version%3D1.0%26open_udid%3D4C%253A49%253AE3%253ACA%253A75%253A35%26patch_ver%3D3.0%26screen_height%3D1920%26screen_scale%3D2.88%26screen_width%3D1080%26sys_ver%3D9%26time_offset%3D480%26x_auth_mode%3Dclient_auth")));
        list.add(vm.addLocalObject(new StringObject(vm,"com.mfw.roadbook")));
        Number number =  module.callFunction(emulator,0x2e301,list.toArray());
        String result = vm.getObject(number.intValue()).getValue().toString();
        return result;
    }
}
```

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

然后执行，结果一致

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230425170316739-381164663.png)

 就很轻松加愉快

知识点总结
-----

第四部分没啥技术知识点，都是老套路
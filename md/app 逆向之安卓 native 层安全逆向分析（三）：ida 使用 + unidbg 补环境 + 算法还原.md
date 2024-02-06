> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/Eeyhan/p/17346150.html)

前言
--

继续跟着龙哥的 unidbg 学习：[SO 逆向入门实战教程三：V2-Sign_unidbg context_白龙~ 的博客 - CSDN 博客](https://blog.csdn.net/qq_38851536/article/details/117533396)

还是那句，我会借鉴龙哥的文章，以一个初学者的角度，加上自己的理解，把内容丰富一下，尽量做到不在龙哥的基础上画蛇添足，哈哈。感谢观看的朋友

分析
--

打开 app，抓包，发现有个 sign

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230423104018304-1050013649.png)

这个 sign 就是今天的重点了，jadx 打开 apk，可以，没有加壳，一搜，发现很快就搜到这些了，而且也不多

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230423104144682-1802543435.png)

问题不大，用 objcetion 把这几个都 hook 了，看看是走的哪里，没搞多久，就看到这里，入参和返回值，感觉就是这里了

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230423110024441-743506321.png)

因为这个返回值的格式，根抓包看到的格式基本一致

还有一个，我们看看请求的加密和解密，objection:

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230423110248324-1257423327.png)

感觉 objection 不是太好看这个结果，那么这里改用 frida 看看吧，虽然 objection 可以改源码，然后让结果可读，但是这里感觉不是有太大的必要，直接用 frida 就行了。

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230423115942290-1475802824.png)

就是这里了，直接就解密了。

ida 查看
------

打开 ida 看看这个 so:

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230424170212722-695003839.png)

卧槽，这个 so 文件有点硬啊，看看，jni_onload:

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230424170303587-1925403009.png)

 卧槽，确实硬啊，这就尴尬了。那直接上 unidbg 吧

unidbg 调试
---------

### 1. 搭架子

先把 main 跑起来

```
package com.xiaochuankeji;
 
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.linux.android.dvm.DalvikModule;
import com.github.unidbg.linux.android.dvm.VM;
import com.github.unidbg.memory.Memory;
import com.weibo.international;
 
import java.io.File;
 
public class tieba extends AbstractJni {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;
    tieba() {
        // 创建模拟器实例,进程名建议依照实际进程名填写，可以规避针对进程名的校验
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("cn.xiaochuankeji.tieba").build();
        // 获取模拟器的内存操作接口
        final Memory memory = emulator.getMemory();
        // 设置系统类库解析
        memory.setLibraryResolver(new AndroidResolver(23));
        // 创建Android虚拟机,传入APK，Unidbg可以替我们做部分签名校验的工作
        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\cn\\xiaochuankeji\\right573.apk"));
        // 加载目标SO
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\xiaochuankeji\\libnet_crypto.so"), true); // 加载so到虚拟内存
        //获取本SO模块的句柄,后续需要用它
        module = dm.getModule();
        vm.setJni(this); // 设置JNI
        vm.setVerbose(true); // 打印日志
        dm.callJNI_OnLoad(emulator); // 调用JNI OnLoad
    }
 
    public static void main(String[] args) {
        tieba test = new tieba();
        System.out.println(test);
    }
}
```

运行，看起来没啥问题

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230424170830545-1904618265.png)

根据龙哥的文章，说如果这里设置为 false:

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230424171111746-486849271.png)

结果就会乱码。原因是这个 so 样本，做了【字符串的混淆或加密】 ，以此来对抗分析人员，但字符串总是要解密的

这个解密一般发生在 Init array 节或者 JNI OnLoad 中，又或者是该字符串使用前的任何一个时机，而本例呢，就发生在 Init array 节中，Shift+F7 快捷键查看节区验证这一点

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230424171333105-188556603.png)

Init array 节内有大量函数，解密就发生在其中。当我们使用 Unidbg 模拟执行时，如果加载 SO 时配置为不执行 Init 相关函数，这导致整个 SO 中的字符串都没有被解密，自然输出就是一团” 乱码 “。

由此还可以衍生出一个小话题——如果样本中的字符串被加密了，如何还原？使得分析者可以愉快的用 IDA 静态分析？

*   从内存中 Dump 出解密后的 SO 或者字符串（可以用 Frida/IDA 脚本 / adb 等），将结果回填或者说修复本身 SO。
*   使用 Unicorn 或基于 Unicorn 的模拟执行工具（Unidbg、ExAndroidNativeemu 等）运行 SO，dump 解密后的虚拟内存，回填修复 SO

### 2. 执行 native_init 

为啥要执行 native_init，看 jadx，他注册 so 的时候，同时执行了这个方法，那么我们要完全的模拟安卓环境，那就也要先执行这个方法，再执行核心的方法

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230424170945724-793671421.png)

再看看，刚才的 main 函数执行打印的日志，刚好，已经有了 native 的方法地址 

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230424171601726-12592275.png)

用 ida 看看是 thumb 还是 arm，看这个应该是 thumb

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230424171911342-183057529.png)

执行下这个方法：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230424171949417-1629085472.png)

卧槽，这个啥情况，意思是没有这个方法，那不急，不加 1 看看

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230424172130869-832250918.png)

现在好像可以了，这个就尴尬了，跟咱们之前说的规则对不上啊，这就奇怪了。

**根据大佬的解答，我们这里拿的地址是 unidbg 打印的地址，而不是像之前在 ida 看到的地址，unidbg 的地址是真实的地址，如果是 arm 的话，就会自动 + 1**

再仔细看看这个，好像是报错：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230424172259965-342619018.png)

它提示，意思是不支持这个方法，可以的，终于到了补环境的环节

### 3.unidbg 补环境

点进第一个报错的地方看看：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230424172437849-118242545.png)

 确实有补了很多 java 环境，但是差我们这个，那我们就自己补了，怎么补，关于补法，有两种实践方法都很有道理

*   全部在用户类中补，防止项目迁移或者 Unidbg 更新带来什么问题，这样做代码的移植性比较好。
*   样本的自定义 JAVA 方法在用户类中补，通用的方法在 AbstractJNI 中补，这样做的好处是，之后运行的项目如果调用通用方法，就不用做重复的修补工作

那么这里我就直接在自己定义的地方补了，防止对 unidbg 源码修改过多，没啥意义。再看，这个报错，它要的是一个 context，那么给他一个 context 就行。context 怎么来，在第二篇有讲过了：

> ```
> package com.xiaochuankeji;
>  
> import com.github.unidbg.AndroidEmulator;
> import com.github.unidbg.Module;
> import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
> import com.github.unidbg.linux.android.AndroidResolver;
> import com.github.unidbg.linux.android.dvm.*;
> import com.github.unidbg.linux.android.dvm.api.AssetManager;
> import com.github.unidbg.linux.android.dvm.array.ByteArray;
> import com.github.unidbg.memory.Memory;
> import com.weibo.international;
>  
> import java.io.File;
> import java.nio.charset.StandardCharsets;
> import java.util.ArrayList;
> import java.util.List;
>  
> public class tieba extends AbstractJni {
>     private final AndroidEmulator emulator;
>     private final VM vm;
>     private final Module module;
>  
>     tieba() {
>         // 创建模拟器实例,进程名建议依照实际进程名填写，可以规避针对进程名的校验
>         emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("cn.xiaochuankeji.tieba").build();
>         // 获取模拟器的内存操作接口
>         final Memory memory = emulator.getMemory();
>         emulator.getSyscallHandler().setEnableThreadDispatcher(true);
>         // 设置系统类库解析
>         memory.setLibraryResolver(new AndroidResolver(23));
>         // 创建Android虚拟机,传入APK，Unidbg可以替我们做部分签名校验的工作
>         vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\xiaochuankeji\\right573.apk"));
>         // 加载目标SO
>         DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\xiaochuankeji\\libnet_crypto.so"), true); // 加载so到虚拟内存
>         //获取本SO模块的句柄,后续需要用它
>         module = dm.getModule();
>         vm.setJni(this); // 设置JNI
>         vm.setVerbose(true); // 打印日志
>         dm.callJNI_OnLoad(emulator); // 调用JNI OnLoad
>     }
>  
>     public static void main(String[] args) {
>         tieba test = new tieba();
> //        System.out.println(test);
>         test.native_init();
>         test.sign();
>     }
>  
>     public void native_init() {
>         List<Object> list = new ArrayList<>(10);
>         list.add(vm.getJNIEnv()); // arg1,env
>         list.add(0); // arg2,jobject
>         module.callFunction(emulator, 0x4a069, list.toArray());
>     }
>  
>     @Override
>     public DvmObject<?> callStaticObjectMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
>         switch (signature) {
>             case "com/izuiyou/common/base/BaseApplication->getAppContext()Landroid/content/Context;":
>                 return vm.resolveClass("android/content/Context").newObject(null);
>         }
>         return super.callStaticObjectMethodV(vm, dvmClass, signature, vaList);
>     }
>  
>     @Override
>     public DvmObject<?> callObjectMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {
>         switch (signature) {
>             case "android/content/Context->getClass()Ljava/lang/Class;":
>                 return dvmObject.getObjectType();
>             case "java/lang/Class->getSimpleName()Ljava/lang/String;":
>                 return new StringObject(vm, "AppController");
>             case "android/content/Context->getFilesDir()Ljava/io/File;":
>             case "java/lang/String->getAbsolutePath()Ljava/lang/String;":
>                 return new StringObject(vm, "/data/user/0/cn.xiaochuankeji.tieba/files");
>         }
>         return super.callObjectMethodV(vm, dvmObject, signature, vaList);
>     }
>  
>     @Override
>     public boolean callStaticBooleanMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
>         switch (signature) {
>             case "android/os/Debug->isDebuggerConnected()Z":
>                 return false;
>         }
>         return super.callStaticBooleanMethodV(vm, dvmClass, signature, vaList);
>     }
>     @Override
>     public int callStaticIntMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
>         switch (signature) {
>             case "android/os/Process->myPid()I":
>                 return emulator.getPid();
>         }
>         return super.callStaticIntMethodV(vm, dvmClass, signature, vaList);
>     }
>  
>     public String sign() {
>         List<Object> list = new ArrayList<>(10);
>         list.add(vm.getJNIEnv()); // arg1,env
>         list.add(0); // arg2,jobject
>         list.add(vm.addLocalObject(new StringObject(vm, "https://api.izuiyou.com/index/recommend"))); // arg3
>         byte[] ss = new byte[]{-61,8,81,112,-113,62,93,124,-101,59,-104,72,103,-122,-91,69,-101,86,18,29,118,-53,-110,-34,-35,100,9,114,-24,-56,84,20,-62,117,-95,76,90,-53,66,85,-128,-104,126,114,-91,13,111,11,121,12,77,68,101,52,-60,25,61,-30,38,-59,118,31,110,9,109,-87,119,83,-12,48,-111,41,116,-114,103,98,43,-12,-84,122,-9,-91,28,-26,-79,-45,-74,-52,6,56,63,-32,108,106,3,-23,-67,56,35,-100,91,-42,-22,-68,-112,-83,-79,39,49,92,112,-111,100,103,-127,16,-112,-10,-12,83,46,-108,-121,-54,-16,26,82,-102,83,97,-115,-2,118,-103,-52,-46,-118,-120,9,63,59,77,31,9,-14,68,69,-125,-16,105,-88,103,-80,61,78,115,112,-4,10,101,-21,53,39,-78,57,92,56,-77,20,-17,79,-76,-82,-12,17,-4,2,84,115,-71,20,115,-58,-105,116,-102,-103,-43,-105,96,95,-35,-53,-35,-56,-7,104,7,72,20,-113,-109,37,117,107,64,37,41,-107,70,10,-12,-127,-37,-24,74,21,66,-82,57,-7,76,97,-37,-19,-18,4,60,-49,-15,120,-24,12,-23,3,-63,-55,-12,-70,65,-16,117,9,-90,35,-3,110,-86,49,-97,24,85,60,38,16,-36,-64,24,117,-55,106,9,67,-5,-115,-74,20,-83,32,5,18,54,17,93,83,-18,-19,56,78,87,80,57,46,-105,10,49,-109,97,4,-104,48,69,73,119,45,9,-57,74,-44,3,-97,-101,57,124,44,-17,-54,66,-94,32,-64,-46,-30,87,-104,17,-68,-85,-69,36,46,-42,23,16,0,-117,-72,117,-20,-96,3,-115,104,-11,-26,-35,-32,127,24,37,21,77,16,66,-75,124,101,42,117,118,89,26,-86,62,45,-106,-111,6,-74,-83,13,0,32,-120,-94,-56,44,0,-12,-51,58,105,1,-74,8,105,65,22,-98,112,-90,-66,-13,-126,-96,-122,-113,-97,-85,-100,-126,123,47,-89,109,14,93,15,35,-61,46,76,40,63,119,52,87,-19,-28,41,-11,68,17,69,-101,21,-60,-73,2,-72,-97,-95,-98,-20,-97,65,-8,-29,-4,78,-118,25,-31,-16,-105,90,91,38,-100,77,24,34,-28,-23,81,74,44,-78,68,-62,115,125,-109,-76,-97,88,51,-13,100,-38,110,110,-51,17,51,5,91,-24,10,-102,40,60,-27,106,71,97,-125,37,-75,40,40,-43,101,-8,41,-124,-8,93,-18,-128,8,-50,-128,-128,-99,-23,97,-25,-118,-25,-23,24,-125,-47,-91,108,-66,4,73,19,31,-34,62,-78,122,-81,5,41,11,66,-26,-116,81,107,-14,-106,-110,35,78,-9,-88,20,-41,-111,-37,39,-104,82,71,-36,-116,-47,29,24,-107,24,32,-7,-90,-52,124,-94,-107,57,-7,117,38,-99,-37,50,-126,-68,45,-32,-25,-118,-69,20,55,-24,40,17,-116,111,-36,-15,109,102,37,-7,-118,63,88,52,46,58,-91,111,33,-23,102,116,-77,-100,102,-124,8,7,92,-84,-33,-26,54,-4,75,56,100,-5,-59,-66,51,127,-126,-31,39,-20,-81,72,106,64,-71,8,88,9,-18,-50,-118,-109,31,-55,11,89,0,-49,-69,-54,-118,14,-78,109,36,95,-6,68,21,58,-86,-103,55,94,4,-23,-75,106,-60,-20,72,3,-66,54,78,12,-9,-70,-13,106,-112,-51,100,62,-101,28,-63,8,-120,-67,24,-10,50,41,114,-41,42,-108,-19,-116,-62,-45,17,112,81,91,18,-108,-113,108,108,-88,-18,57,-40,117,-64,38,65,36,-24,-90,53,-61,-37,-45,-70,-14,-41,109,3,-59,-7,-101,-87,84,-39,25,85,-91,-111,17,35,77,-57,75,-105,-97,65,-37,-103,-112,-62,51,-42,40,59,70,79,50,-83,63,-23,-6,58,29,-5,-3,-33,-29,-91,-59,-36,55,34,24,74,38,-72,99,125,84,-124,60,97,-111,47,40,-29,-85,-76,-119,-89,106,-1,38,17,-17,-123,94,-104,-85,-10,13,53,110,103,100,123,-21,-14,107,11,45,-47,-22,-40,7,-102,26,59,98,120,-28,-27,123,55,97,18,23,-111,109,-127,-14,27,88,-19,-93,59,-118,-118,114,-17,-55,-99,86,-87,-73,-80,17,71,-32,-37,-50,53,29,-112,-81,-124,89,11,83,41,-41,24,-75,6,27,54,-121,12,-19,110,-73,35,-111,51,-118,115,30,106,-56,32,99,39,-75,67,-104,-77,-124,71,7,-36,-38,75,1,-8,-96,-44,99,115,-7,28,-128,38,39,-74,-43,-24,-35};
>         ByteArray inbarr = new ByteArray(vm, ss);
>         list.add(vm.addGlobalObject(inbarr)); //arg4
>         Number number = module.callFunction(emulator, 0x4a28d, list.toArray());
>         String result = vm.getObject(number.intValue()).getValue().toString();
>         return result;
>     }
> }
> ```

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230424172850994-1722394776.png)

再运行看看，卧槽，内存不够了，这么猛的吗

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230424173032221-201898625.png)

没事，查阅资料：

> [unidbg/SignalTest.java at 56c1397c639716817896766df9bbce9c3a0a15b9 · zhkl0228/unidbg (github.com)](https://github.com/zhkl0228/unidbg/blob/56c1397c639716817896766df9bbce9c3a0a15b9/unidbg-android/src/test/java/com/github/unidbg/android/SignalTest.java#L53)

加一行这个就行：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230424173305945-469339294.png)

再次运行，现在好像没报错了：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230424173332723-1953130741.png)

### 4. 执行 sign 方法

解析来开始执行加密方法 sign 了：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230424174058998-209348126.png)

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230424174048350-1320818593.png)

点进去，

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230424174531969-825138422.png)

照着补了

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230424174729693-2082772732.png)

运行，又有新的环境缺失：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230424174926865-1845391926.png)

捋一下完整的流程，在 com/izuiyou/common/base/BaseApplication 中调用 getAppContext 方法，获得一个 Context 上下文，然后 getClass 获取它的类，最后查看它的类名。类名就是这一系列操作的最终目的，我们前面几步都只浅浅的补了一下，只能说类型给对了，别的都没给。但只要最后的类名给它返回正确的字符串，就没问题。

用 objection  的 wallerbreaker 看看：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230424175438970-785668448.png)

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230424175710975-1293750102.png)

看这个，说明 class 就是这个 AppController 了，补完接着跑：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230424175842683-135858836.png)

它要一个文件的地址，补完接着跑，

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230424180157019-1139262251.png)

它要绝对路径，其实还是上面那个，接着补，新的报错：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230424180316222-1677732211.png)

查看这个是否在调试，这里，按正常的逻辑，肯定是给 false

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230424180634676-2021988202.png)

又有新的报错，继续补：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230424180812982-1607545769.png)

卧槽，好像终于出结果了！！！！，不容易啊。

为了保险起见，还是验证下，跟 hook 到的值，是否能保持一致

### 5. 验证值

先用 frida 重新 hook 下这个方法：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230424183801198-614132043.png)

本来想把这个 bytearray 转成字符串看看的，试了以下几种方法，都不行：

> console.log('sign arg2===》', ByteString.of(arg2).utf8())  
> console.log('sign arg2===》', ByteString.of(arg2).toString())  
> console.log('sign arg2===》', hex2str(ByteString.of(arg2).hex()))  
> console.log('sign arg2===》', jbhexdump(arg2))  
> console.log('sign arg2===》', Java.use("java.lang.String").$new(Java.array('byte', arg2)))  
> console.log(Java.use("java.lang.String").$new(arg2))

那没法了，直接复制这个 byte array 吧

unidbg 里稍微调一下：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230425102941657-209399970.png)

 牛逼，结果对上了。舒服

完整代码：

```
# coding:utf-8
 
import os
import subprocess
import jpype
import time
 
 
def query_by_java_jar(jar_path, param=None):
    if param:
        execute = "java -jar {} '{}'".format(jar_path, param)
    else:
        execute = "java -jar {}".format(jar_path)
    print(execute)
    output = subprocess.Popen(execute, shell=True, stdout=subprocess.PIPE, stderr=subprocess.STDOUT)
    res = output.stdout.readlines()
    print('result', res)
    return res
 
 
if __name__ == '__main__':
    query_by_java_jar(r"E:\work\myproject\unidbg\out\artifacts\unidbg_jar\unidbg.jar")
```

python 调用
---------

有没有想过，把这个项目，打包成 jar 后，给 py 调用呢，这样不就可以直接模拟调用了？有点意思对吧，上一章是直接给 java 调用，好像还没有给 python 调用过

先改一下这两个文件的路径，然后打包

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230425104409818-1812514007.png)

打包流程就省略了，前面有

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230425105047585-1614658122.png)

用 python 调用，记得把 apk 和 so 也放到这个 py 所在的路径下，因为打包的时候写的相对路径，谁调用，谁就是主程序，就要把这两个文件放到主程序文件所在同级目录：  

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230425112534675-2045467629.png)

这里这个 test.py 就是我的文件

```
import binascii
 
SV = [0xd76aa478, 0xe8c7b756, 0x242070db, 0xc1bdceee, 0xf57c0faf,
      0x4787c62a, 0xa8304613, 0xfd469501, 0x698098d8, 0x8b44f7af,
      0xffff5bb1, 0x895cd7be, 0x6b901122, 0xfd987193, 0xa679438e,
      0x49b40821, 0xf61e2562, 0xc040b340, 0x265e5a51, 0xe9b6c7aa,
      0xd62f105d, 0x2441453, 0xd8a1e681, 0xe7d3fbc8, 0x21e1cde6,
      0xc33707d6, 0xf4d50d87, 0x455a14ed, 0xa9e3e905, 0xfcefa3f8,
      0x676f02d9, 0x8d2a4c8a, 0xfffa3942, 0x8771f681, 0x6d9d6122,
      0xfde5380c, 0xa4beea44, 0x4bdecfa9, 0xf6bb4b60, 0xbebfbc70,
      0x289b7ec6, 0xeaa127fa, 0xd4ef3085, 0x4881d05, 0xd9d4d039,
      0xe6db99e5, 0x1fa27cf8, 0xc4ac5665, 0xf4292244, 0x432aff97,
      0xab9423a7, 0xfc93a039, 0x655b59c3, 0x8f0ccc92, 0xffeff47d,
      0x85845dd1, 0x6fa87e4f, 0xfe2ce6e0, 0xa3014314, 0x4e0811a1,
      0xf7537e82, 0xbd3af235, 0x2ad7d2bb, 0xeb86d391]
 
 
# 根据ascil编码把字符转成对应的二进制
def binvalue(val, bitsize):
    binval = bin(val)[2:] if isinstance(val, int) else bin(ord(val))[2:]
    if len(binval) > bitsize:
        raise ("binary value larger than the expected size")
    while len(binval) < bitsize:
        binval = "0" + binval
    return binval
 
 
def string_to_bit_array(text):
    array = list()
    for char in text:
        binval = binvalue(char, 8)
        array.extend([int(x) for x in list(binval)])
    return array
 
 
# 循环左移
def leftCircularShift(k, bits):
    bits = bits % 32
    k = k % (2 ** 32)
    upper = (k << bits) % (2 ** 32)
    result = upper | (k >> (32 - (bits)))
    return (result)
 
 
# 分块
def blockDivide(block, chunks):
    result = []
    size = len(block) // chunks
    for i in range(0, chunks):
        result.append(int.from_bytes(block[i * size:(i + 1) * size], byteorder="little"))
    return result
 
 
# F函数作用于“比特位”上
# if x then y else z
def F(X, Y, Z):
    compute = ((X & Y) | ((~X) & Z))
    return compute
 
 
# if z then x else y
def G(X, Y, Z):
    return ((X & Z) | (Y & (~Z)))
 
 
# if X = Y then Z else ~Z
def H(X, Y, Z):
    return (X ^ Y ^ Z)
 
 
def I(X, Y, Z):
    return (Y ^ (X | (~Z)))
 
 
# 四个F函数
def FF(a, b, c, d, M, s, t):
    result = b + leftCircularShift((a + F(b, c, d) + M + t), s)
    return (result)
 
 
def GG(a, b, c, d, M, s, t):
    result = b + leftCircularShift((a + G(b, c, d) + M + t), s)
    return (result)
 
 
def HH(a, b, c, d, M, s, t):
    result = b + leftCircularShift((a + H(b, c, d) + M + t), s)
    return (result)
 
 
def II(a, b, c, d, M, s, t):
    result = b + leftCircularShift((a + I(b, c, d) + M + t), s)
    return (result)
 
 
# 数据转换
def fmt8(num):
    bighex = "{0:08x}".format(num)
    binver = binascii.unhexlify(bighex)
    result = "{0:08x}".format(int.from_bytes(binver, byteorder='little'))
    return (result)
 
 
# 计算比特长度
def bitlen(bitstring):
    return len(bitstring) * 8
 
 
def md5sum(msg):
    # 计算比特长度，如果内容过长，64个比特放不下。就取低64bit。
    msgLen = bitlen(msg) % (2 ** 64)
    # 先填充一个0x80，其实是先填充一个1，后面跟对应个数的0，因为一个明文的编码至少需要8比特，所以直接填充 0b10000000即0x80
    msg = msg + b'\x80'  # 0x80 = 1000 0000
    # 似乎各种编码，即使是一个字母，都至少得1个字节，即8bit才能表示，所以不会出现原文55bit，pad1就满足的情况？可是不对呀，要是二进制文件呢？
    # 填充0到满足要求为止。
    zeroPad = (448 - (msgLen + 8) % 512) % 512
    zeroPad //= 8
    msg = msg + b'\x00' * zeroPad + msgLen.to_bytes(8, byteorder='little')
    # 计算循环轮数，512个为一轮
    msgLen = bitlen(msg)
    iterations = msgLen // 512
    # 初始化变量
    # 算法魔改的第一个点，也是最明显的点
    A = 0x67452301
    B = 0xefcdab89
    C = 0x98badcfe
    D = 0x10325476
 
    # MD5的主体就是对abcd进行n次的迭代，所以得有个初始值，可以随便选，也可以用默认的魔数，这个改起来毫无风险，所以大家爱魔改它，甚至改这个都不算魔改。
    # main loop
    for i in range(0, iterations):
        a = A
        b = B
        c = C
        d = D
        block = msg[i * 64:(i + 1) * 64]
        # 明文的处理，顺便调整了一下端序
        M = blockDivide(block, 16)
        # Rounds
        a = FF(a, b, c, d, M[0], 7, SV[0])
        d = FF(d, a, b, c, M[1], 12, SV[1])
        c = FF(c, d, a, b, M[2], 17, SV[2])
        b = FF(b, c, d, a, M[3], 22, SV[3])
        a = FF(a, b, c, d, M[4], 7, SV[4])
        d = FF(d, a, b, c, M[5], 12, SV[5])
        c = FF(c, d, a, b, M[6], 17, SV[6])
        b = FF(b, c, d, a, M[7], 22, SV[7])
        a = FF(a, b, c, d, M[8], 7, SV[8])
        d = FF(d, a, b, c, M[9], 12, SV[9])
        c = FF(c, d, a, b, M[10], 17, SV[10])
        b = FF(b, c, d, a, M[11], 22, SV[11])
        a = FF(a, b, c, d, M[12], 7, SV[12])
        d = FF(d, a, b, c, M[13], 12, SV[13])
        c = FF(c, d, a, b, M[14], 17, SV[14])
        b = FF(b, c, d, a, M[15], 22, SV[15])
 
        a = GG(a, b, c, d, M[1], 5, SV[16])
        d = GG(d, a, b, c, M[6], 9, SV[17])
        c = GG(c, d, a, b, M[11], 14, SV[18])
        b = GG(b, c, d, a, M[0], 20, SV[19])
        a = GG(a, b, c, d, M[5], 5, SV[20])
        d = GG(d, a, b, c, M[10], 9, SV[21])
        c = GG(c, d, a, b, M[15], 14, SV[22])
        b = GG(b, c, d, a, M[4], 20, SV[23])
        a = GG(a, b, c, d, M[9], 5, SV[24])
        d = GG(d, a, b, c, M[14], 9, SV[25])
        c = GG(c, d, a, b, M[3], 14, SV[26])
        b = GG(b, c, d, a, M[8], 20, SV[27])
        a = GG(a, b, c, d, M[13], 5, SV[28])
        d = GG(d, a, b, c, M[2], 9, SV[29])
        c = GG(c, d, a, b, M[7], 14, SV[30])
        b = GG(b, c, d, a, M[12], 20, SV[31])
 
        a = HH(a, b, c, d, M[5], 4, SV[32])
        d = HH(d, a, b, c, M[8], 11, SV[33])
        c = HH(c, d, a, b, M[11], 16, SV[34])
        b = HH(b, c, d, a, M[14], 23, SV[35])
        a = HH(a, b, c, d, M[1], 4, SV[36])
        d = HH(d, a, b, c, M[4], 11, SV[37])
        c = HH(c, d, a, b, M[7], 16, SV[38])
        b = HH(b, c, d, a, M[10], 23, SV[39])
        a = HH(a, b, c, d, M[13], 4, SV[40])
        d = HH(d, a, b, c, M[0], 11, SV[41])
        c = HH(c, d, a, b, M[3], 16, SV[42])
        b = HH(b, c, d, a, M[6], 23, SV[43])
        a = HH(a, b, c, d, M[9], 4, SV[44])
        d = HH(d, a, b, c, M[12], 11, SV[45])
        c = HH(c, d, a, b, M[15], 16, SV[46])
        b = HH(b, c, d, a, M[2], 23, SV[47])
 
        a = II(a, b, c, d, M[0], 6, SV[48])
        d = II(d, a, b, c, M[7], 10, SV[49])
        c = II(c, d, a, b, M[14], 15, SV[50])
        b = II(b, c, d, a, M[5], 21, SV[51])
        a = II(a, b, c, d, M[12], 6, SV[52])
        d = II(d, a, b, c, M[3], 10, SV[53])
        c = II(c, d, a, b, M[10], 15, SV[54])
        b = II(b, c, d, a, M[1], 21, SV[55])
        a = II(a, b, c, d, M[8], 6, SV[56])
        d = II(d, a, b, c, M[15], 10, SV[57])
        c = II(c, d, a, b, M[6], 15, SV[58])
        b = II(b, c, d, a, M[13], 21, SV[59])
        a = II(a, b, c, d, M[4], 6, SV[60])
        d = II(d, a, b, c, M[11], 10, SV[61])
        c = II(c, d, a, b, M[2], 15, SV[62])
        b = II(b, c, d, a, M[9], 21, SV[63])
        A = (A + a) % (2 ** 32)
        B = (B + b) % (2 ** 32)
        C = (C + c) % (2 ** 32)
        D = (D + d) % (2 ** 32)
    result = fmt8(A) + fmt8(B) + fmt8(C) + fmt8(D)
    return result
 
 
if __name__ == "__main__":
    data = str("https://api.izuiyou.com/index/recommend").encode("UTF-8")
    print("plainText: ", data)
    print("result: ", md5sum(data))
```

运行结果：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230425112402187-1175061914.png)

我试了下那个 jpype，感觉各种问题，还是这个 subproess 方便点，直接出结果 

算法还原
----

用 findhash，一顿搜索

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230425144029655-509661668.png)

然后找到这个函数

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230425144051245-1826019599.png)

鼠标直接点过去：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230425144121552-344035678.png)

按 tab 键

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230425144150170-380786550.png)

光标放上去，然后键盘按【h】，它就变成这样了

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230425144421785-883619088.png)

把这 4 个都这么操作下：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230425144449562-1156823751.png)

又由，先看看 md5 的源码：

```
import binascii
 
SV = [0xd76aa478, 0xe8c7b756, 0x242070db, 0xc1bdceee, 0xf57c0faf,
      0x4787c62a, 0xa8304613, 0xfd469501, 0x698098d8, 0x8b44f7af,
      0xffff5bb1, 0x895cd7be, 0x6b901122, 0xfd987193, 0xa679438e,
      0x49b40821, 0xf61e2562, 0xc040b340, 0x265e5a51, 0xe9b6c7aa,
      0xd62f105d, 0x2441453, 0xd8a1e681, 0xe7d3fbc8, 0x21e1cde6,
      0xc33707d6, 0xf4d50d87, 0x455a14ed, 0xa9e3e905, 0xfcefa3f8,
      0x676f02d9, 0x8d2a4c8a, 0xfffa3942, 0x8771f681, 0x6d9d6122,
      0xfde5380c, 0xa4beea44, 0x4bdecfa9, 0xf6bb4b60, 0xbebfbc70,
      0x289b7ec6, 0xeaa127fa, 0xd4ef3085, 0x4881d05, 0xd9d4d039,
      0xe6db99e5, 0x1fa27cf8, 0xc4ac5665, 0xf4292244, 0x432aff97,
      0xab9423a7, 0xfc93a039, 0x655b59c3, 0x8f0ccc92, 0xffeff47d,
      0x85845dd1, 0x6fa87e4f, 0xfe2ce6e0, 0xa3014314, 0x4e0811a1,
      0xf7537e82, 0xbd3af235, 0x2ad7d2bb, 0xeb86d391]
 
 
# 根据ascil编码把字符转成对应的二进制
def binvalue(val, bitsize):
    binval = bin(val)[2:] if isinstance(val, int) else bin(ord(val))[2:]
    if len(binval) > bitsize:
        raise ("binary value larger than the expected size")
    while len(binval) < bitsize:
        binval = "0" + binval
    return binval
 
 
def string_to_bit_array(text):
    array = list()
    for char in text:
        binval = binvalue(char, 8)
        array.extend([int(x) for x in list(binval)])
    return array
 
 
# 循环左移
def leftCircularShift(k, bits):
    bits = bits % 32
    k = k % (2 ** 32)
    upper = (k << bits) % (2 ** 32)
    result = upper | (k >> (32 - (bits)))
    return (result)
 
 
# 分块
def blockDivide(block, chunks):
    result = []
    size = len(block) // chunks
    for i in range(0, chunks):
        result.append(int.from_bytes(block[i * size:(i + 1) * size], byteorder="little"))
    return result
 
 
# F函数作用于“比特位”上
# if x then y else z
def F(X, Y, Z):
    compute = ((X & Y) | ((~X) & Z))
    return compute
 
 
# if z then x else y
def G(X, Y, Z):
    return ((X & Z) | (Y & (~Z)))
 
 
# if X = Y then Z else ~Z
def H(X, Y, Z):
    return (X ^ Y ^ Z)
 
 
def I(X, Y, Z):
    return (Y ^ (X | (~Z)))
 
 
# 四个F函数
def FF(a, b, c, d, M, s, t):
    result = b + leftCircularShift((a + F(b, c, d) + M + t), s)
    return (result)
 
 
def GG(a, b, c, d, M, s, t):
    result = b + leftCircularShift((a + G(b, c, d) + M + t), s)
    return (result)
 
 
def HH(a, b, c, d, M, s, t):
    result = b + leftCircularShift((a + H(b, c, d) + M + t), s)
    return (result)
 
 
def II(a, b, c, d, M, s, t):
    result = b + leftCircularShift((a + I(b, c, d) + M + t), s)
    return (result)
 
 
# 数据转换
def fmt8(num):
    bighex = "{0:08x}".format(num)
    binver = binascii.unhexlify(bighex)
    result = "{0:08x}".format(int.from_bytes(binver, byteorder='little'))
    return (result)
 
 
# 计算比特长度
def bitlen(bitstring):
    return len(bitstring) * 8
 
 
def md5sum(msg):
    # 计算比特长度，如果内容过长，64个比特放不下。就取低64bit。
    msgLen = bitlen(msg) % (2 ** 64)
    # 先填充一个0x80，其实是先填充一个1，后面跟对应个数的0，因为一个明文的编码至少需要8比特，所以直接填充 0b10000000即0x80
    msg = msg + b'\x80'  # 0x80 = 1000 0000
    # 似乎各种编码，即使是一个字母，都至少得1个字节，即8bit才能表示，所以不会出现原文55bit，pad1就满足的情况？可是不对呀，要是二进制文件呢？
    # 填充0到满足要求为止。
    zeroPad = (448 - (msgLen + 8) % 512) % 512
    zeroPad //= 8
    msg = msg + b'\x00' * zeroPad + msgLen.to_bytes(8, byteorder='little')
    # 计算循环轮数，512个为一轮
    msgLen = bitlen(msg)
    iterations = msgLen // 512
    # 初始化变量
    # 算法魔改的第一个点，也是最明显的点
    A = 0x67452301
    B = 0xefcdab89
    C = 0x98badcfe
    D = 0x10325476
 
    # MD5的主体就是对abcd进行n次的迭代，所以得有个初始值，可以随便选，也可以用默认的魔数，这个改起来毫无风险，所以大家爱魔改它，甚至改这个都不算魔改。
    # main loop
    for i in range(0, iterations):
        a = A
        b = B
        c = C
        d = D
        block = msg[i * 64:(i + 1) * 64]
        # 明文的处理，顺便调整了一下端序
        M = blockDivide(block, 16)
        # Rounds
        a = FF(a, b, c, d, M[0], 7, SV[0])
        d = FF(d, a, b, c, M[1], 12, SV[1])
        c = FF(c, d, a, b, M[2], 17, SV[2])
        b = FF(b, c, d, a, M[3], 22, SV[3])
        a = FF(a, b, c, d, M[4], 7, SV[4])
        d = FF(d, a, b, c, M[5], 12, SV[5])
        c = FF(c, d, a, b, M[6], 17, SV[6])
        b = FF(b, c, d, a, M[7], 22, SV[7])
        a = FF(a, b, c, d, M[8], 7, SV[8])
        d = FF(d, a, b, c, M[9], 12, SV[9])
        c = FF(c, d, a, b, M[10], 17, SV[10])
        b = FF(b, c, d, a, M[11], 22, SV[11])
        a = FF(a, b, c, d, M[12], 7, SV[12])
        d = FF(d, a, b, c, M[13], 12, SV[13])
        c = FF(c, d, a, b, M[14], 17, SV[14])
        b = FF(b, c, d, a, M[15], 22, SV[15])
 
        a = GG(a, b, c, d, M[1], 5, SV[16])
        d = GG(d, a, b, c, M[6], 9, SV[17])
        c = GG(c, d, a, b, M[11], 14, SV[18])
        b = GG(b, c, d, a, M[0], 20, SV[19])
        a = GG(a, b, c, d, M[5], 5, SV[20])
        d = GG(d, a, b, c, M[10], 9, SV[21])
        c = GG(c, d, a, b, M[15], 14, SV[22])
        b = GG(b, c, d, a, M[4], 20, SV[23])
        a = GG(a, b, c, d, M[9], 5, SV[24])
        d = GG(d, a, b, c, M[14], 9, SV[25])
        c = GG(c, d, a, b, M[3], 14, SV[26])
        b = GG(b, c, d, a, M[8], 20, SV[27])
        a = GG(a, b, c, d, M[13], 5, SV[28])
        d = GG(d, a, b, c, M[2], 9, SV[29])
        c = GG(c, d, a, b, M[7], 14, SV[30])
        b = GG(b, c, d, a, M[12], 20, SV[31])
 
        a = HH(a, b, c, d, M[5], 4, SV[32])
        d = HH(d, a, b, c, M[8], 11, SV[33])
        c = HH(c, d, a, b, M[11], 16, SV[34])
        b = HH(b, c, d, a, M[14], 23, SV[35])
        a = HH(a, b, c, d, M[1], 4, SV[36])
        d = HH(d, a, b, c, M[4], 11, SV[37])
        c = HH(c, d, a, b, M[7], 16, SV[38])
        b = HH(b, c, d, a, M[10], 23, SV[39])
        a = HH(a, b, c, d, M[13], 4, SV[40])
        d = HH(d, a, b, c, M[0], 11, SV[41])
        c = HH(c, d, a, b, M[3], 16, SV[42])
        b = HH(b, c, d, a, M[6], 23, SV[43])
        a = HH(a, b, c, d, M[9], 4, SV[44])
        d = HH(d, a, b, c, M[12], 11, SV[45])
        c = HH(c, d, a, b, M[15], 16, SV[46])
        b = HH(b, c, d, a, M[2], 23, SV[47])
 
        a = II(a, b, c, d, M[0], 6, SV[48])
        d = II(d, a, b, c, M[7], 10, SV[49])
        c = II(c, d, a, b, M[14], 15, SV[50])
        b = II(b, c, d, a, M[5], 21, SV[51])
        a = II(a, b, c, d, M[12], 6, SV[52])
        d = II(d, a, b, c, M[3], 10, SV[53])
        c = II(c, d, a, b, M[10], 15, SV[54])
        b = II(b, c, d, a, M[1], 21, SV[55])
        a = II(a, b, c, d, M[8], 6, SV[56])
        d = II(d, a, b, c, M[15], 10, SV[57])
        c = II(c, d, a, b, M[6], 15, SV[58])
        b = II(b, c, d, a, M[13], 21, SV[59])
        a = II(a, b, c, d, M[4], 6, SV[60])
        d = II(d, a, b, c, M[11], 10, SV[61])
        c = II(c, d, a, b, M[2], 15, SV[62])
        b = II(b, c, d, a, M[9], 21, SV[63])
        A = (A + a) % (2 ** 32)
        B = (B + b) % (2 ** 32)
        C = (C + c) % (2 ** 32)
        D = (D + d) % (2 ** 32)
    result = fmt8(A) + fmt8(B) + fmt8(C) + fmt8(D)
    return result
 
 
if __name__ == "__main__":
    data = str("https://api.izuiyou.com/index/recommend").encode("UTF-8")
    print("plainText: ", data)
    print("result: ", md5sum(data))
```

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230425144901497-1386613346.png)

对比在线 cyberchef:

> https://ctf.mzy0.com/CyberChef3

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230425144734957-790392598.png)

对比下那四个 iv;

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230425145016351-2125140317.png)

看样子，就是改了 4 个 iv：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230425145056059-531282207.png)

ok 的，这个 md5 的算法，搞定

知识点总结
-----

1. 看 so 的调用逻辑，如果有 loadlibrary 的同时有调用某个方法，unidbg 模拟执行的时候也要先调用这个方法

2. 如果加载 so 文件的时候，给定的第二个参数是 false，加上 so 文件有字符串加密和混淆的话就会乱码，所以这里最好给为 true，ida 里，shift+F7，可以看

3. 补环境的时候，要根据报错，最核心的部分进行补环境，同时也要结合实际的执行逻辑

4. 构造 context，用 vm.resolveClass("android/content/Context").newObject(null) 

5.dvmObject.getObjectType() 可以获取当前类的 class

6.emulator.getPid()，获取 pid

7.6 种 frida 里打印 byte[] 的方法

> console.log('sign arg2===》', ByteString.of(arg2).utf8())  
> console.log('sign arg2===》', ByteString.of(arg2).toString())  
> console.log('sign arg2===》', hex2str(ByteString.of(arg2).hex()))  
> console.log('sign arg2===》', jbhexdump(arg2))  
> console.log('sign arg2===》', Java.use("java.lang.String").$new(Java.array('byte', arg2)))  
> console.log(Java.use("java.lang.String").$new(arg2))

8.ida，光标放到上面，按【h】，可以转为 16 进制

9.**unidbg 打印的地址是真实的地址，如果是 arm 的话，就会自动 + 1**
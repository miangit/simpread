> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/Eeyhan/p/17337827.html)

前言
--

继续跟着龙哥的 unidbg 学习：[SO 逆向入门实战教程二：calculateS_so 逆向_白龙~ 的博客 - CSDN 博客](https://blog.csdn.net/qq_38851536/article/details/117419041)

还是那句，我会借鉴龙哥的文章，以一个初学者的角度，加上自己的理解，把内容丰富一下，尽量做到不在龙哥的基础上画蛇添足，哈哈。感谢观看的朋友

分析
--

首先抓包分析：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420174821419-349308679.png)

其中，里面的 s 就是今天的需要逆向的加密参数了。

调试
--

老样子，打开 jadx，发现没壳，可以的，直接看吧，拿着这几个参数一顿搜，直接搜【p】

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420175041315-925473731.png)

感觉有两个地方很可疑，进去一看：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420175300410-894826470.png)

跟下调用栈，很快就找到这里：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420175328330-1543850930.png)

ok，用 objection hook 下，发现确实调用了这里

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420175354022-1666335075.png)

再仔细看看这里，明显这里很奇怪了

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420175640473-1628856121.png)

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420175703860-656027974.png)

ok，终于到这里了，这里就跟龙哥给的位置一致了

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420175714646-264991838.png)

先不急着用 unidbg，先调试下，hook 下这个方法，哎哟，我擦，这直接就对上了

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420180004544-1619326855.png)

看看这三个参数 ，一个是 context，上下文，又叫寄存器，第二个是账号密码加起来，第三个就是一个特殊的值，大概率是加的盐，刺激。

unidbg 调试
---------

### 1.ida 分析

先用 ida 打开看看：发现是静态注册的，可以的

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420180524860-2050585914.png)

选中 a1

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420180601145-718118654.png)

然后按键盘【y】 ，把第一个参数的类型改成 JNIEnv *，这样就可以更好的反编译 c 代码：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420180621559-115472694.png)

点 ok，瞬间这段代码的可读性就更强了

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420180713697-1237531922.png)

大概看了一眼，反正前面有个 if 判断，然后就进入主逻辑，然后返回

选到这个函数

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420181452261-479027209.png)

然后按【tab】 键：这个 0x1E7C 就是这个函数的地址了。记一下，后面会用到

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420181512024-1258465964.png)

### 2. 搭架子

开始搭建 unidbg 的架子，新建一个文件，然后把 apk 和目标 so 文件放进去

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420180148962-58160482.png)

先把架子搭起来，这里我们直接复制前面 oasis 的：

```
package com.weibo;
 
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.memory.Memory;
import com.github.unidbg.pointer.UnidbgPointer;
import com.github.unidbg.utils.Inspector;
import com.sun.jna.Pointer;
import keystone.Keystone;
import keystone.KeystoneArchitecture;
import keystone.KeystoneEncoded;
import keystone.KeystoneMode;
 
import java.io.File;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
 
public class international extends AbstractJni {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;
 
    international() {
        // 创建模拟器实例,进程名建议依照实际进程名填写，可以规避针对进程名的校验
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.weibo.international").build();
        // 获取模拟器的内存操作接口
        final Memory memory = emulator.getMemory();
        // 设置系统类库解析
        memory.setLibraryResolver(new AndroidResolver(23));
        // 创建Android虚拟机,传入APK，Unidbg可以替我们做部分签名校验的工作
        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\weibo\\sinaInternational.apk"));
        // 加载目标SO
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\weibo\\libutility.so"), true); // 加载so到虚拟内存
        //获取本SO模块的句柄,后续需要用它
        module = dm.getModule();
        vm.setJni(this); // 设置JNI
        vm.setVerbose(true); // 打印日志
//        dm.callJNI_OnLoad(emulator); // 调用JNI OnLoad
    }
 
    public static void main(String[] args) {
        international test = new international();
        System.out.println(test);
    }
     
 
}
```

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420180318316-886156176.png)

运行下，没啥问题

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420180442542-434556376.png)

### 2. 地址调用

方法先写好，参数还是用 list，然后用 vm.addLocalObject 包装一下添加进去，第一个参数是 context，直接给个空就行，剩下的参数直接复制 hook 到的放进去

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420181225207-490452805.png)

好的，现在运行一下

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420181615475-1521294361.png)

### 3. 找异常执行原因——签名校验，以及绕过 

 可以的，跟龙哥的博客一样，报错了，然后找找日志，看看上面的一个

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420181857465-856884872.png)

复制这个地址，ida 里，按键盘【g】输入地址跳转过去

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420181912974-1144953877.png)

跳转到这里：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420181935828-1834780171.png)

 改下 JNIEnv

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420181945793-794984807.png)

 好的，根据龙哥说的，有 packagemanagger 之类的，大概率是签名检测

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420182000597-988127920.png)

选中这个方法，按【x】找交叉引用

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420182042127-1200682768.png)

很有缘分的又看到了这个 sub_1c60

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420182114491-149563085.png)

进去看：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420182149959-750169002.png)

再回到这里，应该就是这个判断了

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420182220025-28072870.png)

按下 tab 键，记住地址，就是这个 0xFFF7EBFE 了

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420182629094-961879817.png)

要能返回是 true 才给过，ok，直接把这里 hook 下，也就是直接这么改下，

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420182310325-1822684310.png)

在 java 层里，我们直接 hook 这个方法修改返回值为 1 就行，但是在这里，是在 so 里面，怎么搞呢？根据龙哥说的，用 arm 指令修改就行

> [Online ARM to HEX Converter (armconverter.com)](https://armconverter.com/?code=mov%20R0,1)

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420182536366-520745762.png)

用上面的地址，写个过验证的：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420183234154-1316674436.png)

龙哥还给了另一种 patch 的方法：

```
public void patchverfify(){
        int patchCode = 0x4FF00100;
        emulator.getMemory().pointer(module.base+0x1E86).setInt(0,patchCode);
    }
 
    public void patchVerify1(){
        Pointer pointer = UnidbgPointer.pointer(emulator, module.base + 0x1E86);
        assert pointer != null;
        byte[] code = pointer.getByteArray(0, 4);
        if (!Arrays.equals(code, new byte[]{ (byte)0xFF, (byte) 0xF7, (byte) 0xEB, (byte) 0xFE })) { // BL sub_1C60
            throw new IllegalStateException(Inspector.inspectString(code, "patch32 code=" + Arrays.toString(code)));
        }
        try (Keystone keystone = new Keystone(KeystoneArchitecture.Arm, KeystoneMode.ArmThumb)) {
            KeystoneEncoded encoded = keystone.assemble("mov r0,1");
            byte[] patch = encoded.getMachineCode();
            if (patch.length != code.length) {
                throw new IllegalStateException(Inspector.inspectString(patch, "patch32 length=" + patch.length));
            }
            pointer.write(0, patch, 0, patch.length);
        }
    }
```

然后执行下，ok，这就出来了，舒服： 

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420183334079-926296609.png)

但是好像跟 hook 到的返回不一样：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420183414804-1719856891.png)

不急，再来看看，擦，复制的时候，激动了，把逗号复制进去了

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420183722327-1038134558.png)

ok，这下对上了，跟 hook 的结果一致

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420183752294-213034800.png)

代码

```
package com.weibo;
 
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.memory.Memory;
import com.github.unidbg.pointer.UnidbgPointer;
import com.github.unidbg.utils.Inspector;
import com.sun.jna.Pointer;
import keystone.Keystone;
import keystone.KeystoneArchitecture;
import keystone.KeystoneEncoded;
import keystone.KeystoneMode;
 
import java.io.File;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
 
public class international extends AbstractJni {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;
 
    international() {
        // 创建模拟器实例,进程名建议依照实际进程名填写，可以规避针对进程名的校验
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.weibo.international").build();
        // 获取模拟器的内存操作接口
        final Memory memory = emulator.getMemory();
        // 设置系统类库解析
        memory.setLibraryResolver(new AndroidResolver(23));
        // 创建Android虚拟机,传入APK，Unidbg可以替我们做部分签名校验的工作
        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\weibo\\sinaInternational.apk"));
        // 加载目标SO
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\weibo\\libutility.so"), true); // 加载so到虚拟内存
        //获取本SO模块的句柄,后续需要用它
        module = dm.getModule();
        vm.setJni(this); // 设置JNI
        vm.setVerbose(true); // 打印日志
//        dm.callJNI_OnLoad(emulator); // 调用JNI OnLoad
    }
 
    public static void main(String[] args) {
        international test = new international();
        test.patchverfify();
        System.out.println(test.calculateS());
    }
    public void patchverfify(){
        int patchCode = 0x4FF00100;
        emulator.getMemory().pointer(module.base+0x1E86).setInt(0,patchCode);
    }
 
    public void patchVerify1(){
        Pointer pointer = UnidbgPointer.pointer(emulator, module.base + 0x1E86);
        assert pointer != null;
        byte[] code = pointer.getByteArray(0, 4);
        if (!Arrays.equals(code, new byte[]{ (byte)0xFF, (byte) 0xF7, (byte) 0xEB, (byte) 0xFE })) { // BL sub_1C60
            throw new IllegalStateException(Inspector.inspectString(code, "patch32 code=" + Arrays.toString(code)));
        }
        try (Keystone keystone = new Keystone(KeystoneArchitecture.Arm, KeystoneMode.ArmThumb)) {
            KeystoneEncoded encoded = keystone.assemble("mov r0,1");
            byte[] patch = encoded.getMachineCode();
            if (patch.length != code.length) {
                throw new IllegalStateException(Inspector.inspectString(patch, "patch32 length=" + patch.length));
            }
            pointer.write(0, patch, 0, patch.length);
        }
    }
    public String calculateS() {
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // arg1,env
        list.add(0); // arg2,jobject
        DvmObject<?> context = vm.resolveClass("android/content/Context").newObject(null);
        list.add(vm.addGlobalObject(context));
        list.add(vm.addLocalObject(new StringObject(vm, "135691695686123456789")));
        list.add(vm.addLocalObject(new StringObject(vm, "CypCHG2kSlRkdvr2RG1QF8b2lCWXl7k7")));
        Number number = module.callFunction(emulator, 0x1E7C + 1, list.toArray());
 
        String result = vm.getObject(number.intValue()).getValue().toString();
        return result;
    }
 
 
}
```

### 4. 符号调用

 前面都是地址调用，而龙哥自己也说过，他喜欢用地址调用。不过这里，总要学习下，怎么符号调用

怎么找符号，首选选中这个方法：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230421095425047-794968925.png)

然后按 tab 键，进入汇编页面：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230421095510689-1768633144.png)

然后再按空格，然后这个 export 就是要用的符号表了，注意了，这是只是刚好都一样，很多时候，这三个，不一样的，找准 export 的才行

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230421095545612-1307586054.png)

 开始调用，就换了下名字，其他都没变，对比两个不同的调用方式，结果一样，没毛病

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230421100619207-1717558393.png)

代码：

```
package com.weibo;
 
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.memory.Memory;
import com.github.unidbg.pointer.UnidbgPointer;
import com.github.unidbg.utils.Inspector;
import com.sun.jna.Pointer;
import keystone.Keystone;
import keystone.KeystoneArchitecture;
import keystone.KeystoneEncoded;
import keystone.KeystoneMode;
 
import java.io.File;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
 
public class international extends AbstractJni {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;
 
    international() {
        // 创建模拟器实例,进程名建议依照实际进程名填写，可以规避针对进程名的校验
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.weibo.international").build();
        // 获取模拟器的内存操作接口
        final Memory memory = emulator.getMemory();
        // 设置系统类库解析
        memory.setLibraryResolver(new AndroidResolver(23));
        // 创建Android虚拟机,传入APK，Unidbg可以替我们做部分签名校验的工作
        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\weibo\\sinaInternational.apk"));
        // 加载目标SO
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\weibo\\libutility.so"), true); // 加载so到虚拟内存
        //获取本SO模块的句柄,后续需要用它
        module = dm.getModule();
        vm.setJni(this); // 设置JNI
        vm.setVerbose(true); // 打印日志
//        dm.callJNI_OnLoad(emulator); // 调用JNI OnLoad
    }
 
    public static void main(String[] args) {
        international test = new international();
        test.patchverfify();
        System.out.println("offset===>:" + test.calculateS());
        System.out.println("symbol===>:" + test.calculateS1());
    }
 
     
     
    public String calculateS1() {
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // arg1,env
        list.add(0); // arg2,jobject
        DvmObject<?> context = vm.resolveClass("android/content/Context").newObject(null);
        list.add(vm.addGlobalObject(context));
        list.add(vm.addLocalObject(new StringObject(vm, "135691695686123456789")));
        list.add(vm.addLocalObject(new StringObject(vm, "CypCHG2kSlRkdvr2RG1QF8b2lCWXl7k7")));
        Number number = module.callFunction(emulator, "Java_com_sina_weibo_security_WeiboSecurityUtils_calculateS", list.toArray());
        String result = vm.getObject(number.intValue()).getValue().toString();
        return result;
    }
 
    public void patchverfify() {
        int patchCode = 0x4FF00100;
        emulator.getMemory().pointer(module.base + 0x1E86).setInt(0, patchCode);
    }
 
    public void patchVerify1() {
        Pointer pointer = UnidbgPointer.pointer(emulator, module.base + 0x1E86);
        assert pointer != null;
        byte[] code = pointer.getByteArray(0, 4);
        if (!Arrays.equals(code, new byte[]{(byte) 0xFF, (byte) 0xF7, (byte) 0xEB, (byte) 0xFE})) { // BL sub_1C60
            throw new IllegalStateException(Inspector.inspectString(code, "patch32 code=" + Arrays.toString(code)));
        }
        try (Keystone keystone = new Keystone(KeystoneArchitecture.Arm, KeystoneMode.ArmThumb)) {
            KeystoneEncoded encoded = keystone.assemble("mov r0,1");
            byte[] patch = encoded.getMachineCode();
            if (patch.length != code.length) {
                throw new IllegalStateException(Inspector.inspectString(patch, "patch32 length=" + patch.length));
            }
            pointer.write(0, patch, 0, patch.length);
        }
    }
 
    public String calculateS() {
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // arg1,env
        list.add(0); // arg2,jobject
        DvmObject<?> context = vm.resolveClass("android/content/Context").newObject(null);
        list.add(vm.addGlobalObject(context));
        list.add(vm.addLocalObject(new StringObject(vm, "135691695686123456789")));
        list.add(vm.addLocalObject(new StringObject(vm, "CypCHG2kSlRkdvr2RG1QF8b2lCWXl7k7")));
        Number number = module.callFunction(emulator, 0x1E7C + 1, list.toArray());
 
        String result = vm.getObject(number.intValue()).getValue().toString();
        return result;
    }
 
 
}
```

部署
--

有没有想过，搞的这个，如果想部署成一个 web 服务，以供后续的业务代码调用呢？不然这么个项目，部署在服务器上，然后 每次启动都调用 unidbg，说实话不是太现实，所以这里就需要把 unidbg 打包成 jar 包：

最实用且简单的方法:

1.

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230421103906617-23714464.png)

 2.

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230421104007498-1138609109.png)

 3.

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230421104049717-1355304256.png)

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230421104102672-609585990.png)

 4.

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230421104125368-1762417356.png)

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230421104140644-2137682133.png)

 5.

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230421104209403-741828984.png)

 6.

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230421104226437-76216322.png)

等待结果：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230421104245822-256376511.png)

然后根目录就会多一个 out 目录：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230421104302285-1014499299.png)

7. 这个文件就是最后能直接通过 java 执行的：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230421104335075-1235650444.png)

但是有个问题，我们写的 apk 和 so 文件，给定的地址是 unidgb-android/src 里的

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230421104443105-1459744943.png)

 要打包的话，就得改下路径，改成相对路径

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230421104502206-1543814429.png)

然后删除 out 文件夹，重新上面的打包操作，然后把 apk 和 so 放到跟 unidbg-jar 同级的目录：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230421104608765-1803468151.png)

这样就好了， 终端执行看看，很好，结果也有的，这样就可以把这个 out 包整个打包 ，然后部署到服务器上就行了。

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230421104657663-204990381.png)

还有更多的打包方式：

> [Unidbg 打 Jar 包方式_unidbg 打包成 jar_Forgo7ten 的博客 - CSDN 博客](https://blog.csdn.net/Palmer9/article/details/125220599)

知识点总结
-----

1.ida，按 y 修改 JNIEnv，IDA 7.5 之前，JNIEnv 需要导入 jni.h，7.5 之后不需要导入 jni.h 文件

2.ida，按 g 输入地址跳转

3.ida，按 x，查看交叉引用

4.ida，optional->generel，把这个改成 4，可以查看架构模式，arm 架构是固定 4 位，thumb 架构是混合的

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230420183924162-1005690660.png)

5. 有签名校验的需要去 patch，patch 的时候不能 + 1，只有运行和 hook 的时候才 + 1

6.setJNIload 方法只有动态注册方法的时候才执行，静态注册的不用执行

7.keystone，是将汇编语言转成地址的。capstone 是将地址转成汇编语言的

8. 在线汇编和地址互转的网站：https://armconverter.com/

9. 参数的基本类型，比如 int，long 等，其他的对象类型一律要手动 addLocalObject，其中 context 对象用：

```
DvmObject<?> context = vm.resolveClass("android/content/Context").newObject(null);// context
list.add(vm.addLocalObject(context))
```

10. 找准一行汇编，Alt+G 快捷键，查看架构类型，值为 1 则是 thumb，为 0 则是 arm，如果 ida 解析有误，可以手动修改这个值

11.RM 调用约定，入参前四个分别通过 R0-R3 调用，返回值通过 R0 返回

12.unidbg 的项目可以打包成 java 包执行

13.Unidbg 内嵌了多种 Hook 工具，目前主要是四种

*   Dobby
*   HookZz
*   xHook
*   Whale

> xHook 是爱奇艺开源的基于 PLT HOOK 的 Hook 框架，它无法 Hook 不在符号表里的函数，也不支持 inline hook，这在我们的逆向分析中是无法忍受的，所以在这里不去理会它。
> 
> hale 在 Unidbg 的测试用例中只有对符号表函数的 Hook，没看到 Inline Hook 或者 非导出函数的 Hook，所以也不去考虑。
> 
> HookZz 是 Dobby 的前身，两者都可以 Hook 非导出表中的函数，即 IDA 中显示为 sub_xxx 的函数，也都可以进行 inline hook，所以二选一就行了，HookZz 针对 32 位比较稳定，Dobby 针对 64 位比较稳定

比如这里 hook  MDStringOld 方法：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230421150434702-788188791.png)

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230421150409520-1300470859.png)

 完整代码

```
package com.weibo;
 
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Emulator;
import com.github.unidbg.Module;
import com.github.unidbg.hook.hookzz.*;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.memory.Memory;
import com.github.unidbg.pointer.UnidbgPointer;
import com.github.unidbg.utils.Inspector;
import com.sun.jna.Pointer;
import keystone.Keystone;
import keystone.KeystoneArchitecture;
import keystone.KeystoneEncoded;
import keystone.KeystoneMode;
 
import java.io.File;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
 
public class international extends AbstractJni {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;
 
    international() {
        // 创建模拟器实例,进程名建议依照实际进程名填写，可以规避针对进程名的校验
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.weibo.international").build();
        // 获取模拟器的内存操作接口
        final Memory memory = emulator.getMemory();
        // 设置系统类库解析
        memory.setLibraryResolver(new AndroidResolver(23));
        // 创建Android虚拟机,传入APK，Unidbg可以替我们做部分签名校验的工作
        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\weibo\\sinaInternational.apk"));
        // 加载目标SO
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\weibo\\libutility.so"), true); // 加载so到虚拟内存
        //获取本SO模块的句柄,后续需要用它
        module = dm.getModule();
        vm.setJni(this); // 设置JNI
        vm.setVerbose(true); // 打印日志
//        dm.callJNI_OnLoad(emulator); // 调用JNI OnLoad
    }
 
    public static void main(String[] args) {
        international test = new international();
        test.patchverfify();
        test.HookMDString();
        System.out.println("offset===>:" + test.calculateS());
    }
 
    public String calculateS() {
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // arg1,env
        list.add(0); // arg2,jobject
        DvmObject<?> context = vm.resolveClass("android/content/Context").newObject(null);
        list.add(vm.addGlobalObject(context));
        list.add(vm.addLocalObject(new StringObject(vm, "135691695686123456789")));
        list.add(vm.addLocalObject(new StringObject(vm, "CypCHG2kSlRkdvr2RG1QF8b2lCWXl7k7")));
        Number number = module.callFunction(emulator, 0x1E7C + 1, list.toArray());
        String result = vm.getObject(number.intValue()).getValue().toString();
        return result;
    }
 
    public void HookMDString(){
        IHookZz hookZz = HookZz.getInstance(emulator);
        hookZz.wrap(module.base+0x1BD0+1, new WrapCallback<HookZzArm32RegisterContext>() {
            @Override
            public void preCall(Emulator<?> emulator, HookZzArm32RegisterContext ctx, HookEntryInfo info) {
                Pointer input = ctx.getPointerArg(0);
                System.out.println("input:"+input.getString(0));
            }
            @Override
            public void postCall(Emulator<?> emulator, HookZzArm32RegisterContext ctx, HookEntryInfo info) {
                Pointer result = ctx.getPointerArg(0);
                System.out.println("result:"+result.getString(0));
            }
        });
    }
 
 
 
 
    public String calculateS1() {
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // arg1,env
        list.add(0); // arg2,jobject
        DvmObject<?> context = vm.resolveClass("android/content/Context").newObject(null);
        list.add(vm.addGlobalObject(context));
        list.add(vm.addLocalObject(new StringObject(vm, "135691695686123456789")));
        list.add(vm.addLocalObject(new StringObject(vm, "CypCHG2kSlRkdvr2RG1QF8b2lCWXl7k7")));
        Number number = module.callFunction(emulator, "Java_com_sina_weibo_security_WeiboSecurityUtils_calculateS", list.toArray());
        String result = vm.getObject(number.intValue()).getValue().toString();
        return result;
    }
 
    public void patchverfify() {
        int patchCode = 0x4FF00100;
        emulator.getMemory().pointer(module.base + 0x1E86).setInt(0, patchCode);
    }
 
    public void patchVerify1() {
        Pointer pointer = UnidbgPointer.pointer(emulator, module.base + 0x1E86);
        assert pointer != null;
        byte[] code = pointer.getByteArray(0, 4);
        if (!Arrays.equals(code, new byte[]{(byte) 0xFF, (byte) 0xF7, (byte) 0xEB, (byte) 0xFE})) { // BL sub_1C60
            throw new IllegalStateException(Inspector.inspectString(code, "patch32 code=" + Arrays.toString(code)));
        }
        try (Keystone keystone = new Keystone(KeystoneArchitecture.Arm, KeystoneMode.ArmThumb)) {
            KeystoneEncoded encoded = keystone.assemble("mov r0,1");
            byte[] patch = encoded.getMachineCode();
            if (patch.length != code.length) {
                throw new IllegalStateException(Inspector.inspectString(patch, "patch32 length=" + patch.length));
            }
            pointer.write(0, patch, 0, patch.length);
        }
    }
 
}
```

结语
--

说实话，目前来说，不难，得后续的大厂项目才会很难
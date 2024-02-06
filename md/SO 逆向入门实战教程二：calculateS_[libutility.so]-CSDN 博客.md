> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/qq_38851536/article/details/117419041)

#### 文章目录

*   *   *   [一、前言](#_3)
        *   [二、准备](#_13)
        *   [三、Unidbg 模拟执行](#Unidbg_18)
        *   [四、算法分析](#_359)
        *   [五、尾声](#_562)

#### 一、前言

这是 SO 逆向入门实战教程的第二篇，总共会有十三篇，十三个实战。有以下几个注意点：

*   主打**入门级**的实战，适合有一定基础但缺少实战的朋友（了解 JNI，也上过一些 Native 层逆向的课，但感觉实战匮乏，想要壮壮胆，入入门）。
*   侧重**新工具、新思路、新方法**的使用，算法分析的常见路子是 Frida Hook + IDA ，在本系列中，会淡化 Frida 的作用，采用 Unidbg Hook + IDA 的路线。
*   主打入门，但**并不限于入门**，你会在样本里看到有浅有深的魔改加密算法、以及 OLLVM、SO 对抗等内容。
*   细，非常的细，奶妈级教学。
*   一共十三篇，1-2 天更新一篇。每篇的资料放在文末的百度网盘中。

#### 二、准备

![](https://img-blog.csdnimg.cn/20210531160521274.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
这是我们的分析目标  
参数一是 Context 上下文，参数二是传入的明文，参数三是固定的值，疑似 Key 或者盐。  
返回值是 8 位的 Sign，且输入不变的情况下，输出也固定不变。

#### 三、Unidbg 模拟执行

IDA 中打开 libutility.so，先搜索一下会不会是静态绑定。  
![](https://img-blog.csdnimg.cn/20210531160605815.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
难得遇到静态绑定的 Native 函数，先参数重命名，在笔者的 IDA 7.5 中，JNIEnv 不需要导入 jni.h，设置一下 type 就可以识别 JNI 函数。  
![](https://img-blog.csdnimg.cn/202105311606277.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
![](https://img-blog.csdnimg.cn/20210531160634472.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

```
if ( sub_1C60(a1, context) )
  {
    if ( (*a1)->PushLocalFrame(a1, 16) >= 0 )
    {
      v6 = (*a1)->GetStringUTFChars(a1, inputKey, 0);
      v18 = (char *)(*a1)->GetStringUTFChars(a1, inputBytes, 0);
      v7 = j_strlen(v18);
      v8 = v7 + j_strlen(v6) + 1;
      v9 = j_malloc(v8);
      j_memset(v9, 0, v8);
      j_strcpy((char *)v9, v18);
      j_strcat((char *)v9, v6);
      v10 = (_BYTE *)MDStringOld(v9);
      v11 = (char *)j_malloc(9u);
      *v11 = v10[1];
      v11[1] = v10[5];
      v11[2] = v10[2];
      v11[3] = v10[10];
      v11[4] = v10[17];
      v11[5] = v10[9];
      v11[6] = v10[25];
      v12 = v10[27];
      v11[8] = 0;
      v11[7] = v12;
      v21 = (*a1)->FindClass(a1, "java/lang/String");
      v22 = (*a1)->GetMethodID(a1, v21, "<init>", "([BLjava/lang/String;)V");
      v13 = j_strlen(v11);
      v19 = (*a1)->NewByteArray(a1, v13);
      v14 = j_strlen(v11);
      (*a1)->SetByteArrayRegion(a1, v19, 0, v14, v11);
      v15 = (*a1)->NewStringUTF(a1, "utf-8");
      v16 = (*a1)->NewObject(a1, v21, v22, v19, v15);
      j_free(v11);
      j_free(v9);
      (*a1)->ReleaseStringUTFChars(a1, (jstring)inputBytes, v18);
      inputBytes = (int)(*a1)->PopLocalFrame(a1, v16);
    }
    else
    {
      inputBytes = 0;
    }
  }
  return inputBytes;
```

如果 sub_1C60 函数 False，函数直接返回 0，显然这是一条错误的逻辑，而传入的参数又是 context，这很容易让人想到是一个签名校验函数。先不往下看了，上 Unidbg。

同样先搭一下基础的架子，这个样本连 JNI OnLoad 都没有。

```
package com.lession2;

// 导入通用且标准的类库
import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.android.dvm.array.ByteArray;
import com.github.unidbg.memory.Memory;
import com.lession1.oasis;

import java.io.File;

public class sina extends AbstractJni{
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;

    sina() {
        // 创建模拟器实例,进程名建议依照实际进程名填写，可以规避针对进程名的校验
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.sina.International").build();
        // 获取模拟器的内存操作接口
        final Memory memory = emulator.getMemory();
        // 设置系统类库解析
        memory.setLibraryResolver(new AndroidResolver(23));
        // 创建Android虚拟机,传入APK，Unidbg可以替我们做部分签名校验的工作
        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\lession2\\sinaInternational.apk"));
        //
//        vm = emulator.createDalvikVM(null);

        // 加载目标SO
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\lession2\\libutility.so"), true); // 加载so到虚拟内存
        //获取本SO模块的句柄,后续需要用它
        module = dm.getModule();
        vm.setJni(this); // 设置JNI
        vm.setVerbose(true); // 打印日志
        // 样本连JNI OnLoad都没有
        // dm.callJNI_OnLoad(emulator); // 调用JNI OnLoad
    };

    public static void main(String[] args) {
        sina test = new sina();
    }
}
```

接下来添加一个 calculateS 函数，依然是地址方式调用，ARM32 有 Thumb 和 ARM 两种指令模式，此处是 thumb 模式，所以地址要在 start 基础上 + 1。  
![](https://img-blog.csdnimg.cn/20210531160732536.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
注意看代码，相较于第一讲，这里的入参有一些新情况

*   context 如何构造
*   字符串类型如何构造

除了基本类型，比如 int，long 等，其他的对象类型一律要手动 addLocalObject。

```
public String calculateS(){
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // 第一个参数是env
        list.add(0); // 第二个参数，实例方法是jobject，静态方法是jclazz，直接填0，一般用不到。
        DvmObject<?> context = vm.resolveClass("android/content/Context").newObject(null);// context
        list.add(vm.addLocalObject(context));
        list.add(vm.addLocalObject(new StringObject(vm, "12345")));
        list.add(vm.addLocalObject(new StringObject(vm, "r0ysue")));
		// 因为代码是thumb模式，别忘了+1
        Number number = module.callFunction(emulator, 0x1E7C + 1, list.toArray())[0];
        String result = vm.getObject(number.intValue()).getValue().toString();
        return result;
    };
```

完整代码如下

```
package com.lession2;

// 导入通用且标准的类库
import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.android.dvm.array.ByteArray;
import com.github.unidbg.memory.Memory;
import com.lession1.oasis;

import java.io.File;
import java.util.ArrayList;
import java.util.List;

public class sina extends AbstractJni{
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;

    sina() {
        // 创建模拟器实例,进程名建议依照实际进程名填写，可以规避针对进程名的校验
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.sina.International").build();
        // 获取模拟器的内存操作接口
        final Memory memory = emulator.getMemory();
        // 设置系统类库解析
        memory.setLibraryResolver(new AndroidResolver(23));
        // 创建Android虚拟机,传入APK，Unidbg可以替我们做部分签名校验的工作
        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\lession2\\sinaInternational.apk"));
        //
//        vm = emulator.createDalvikVM(null);

        // 加载目标SO
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\lession2\\libutility.so"), true); // 加载so到虚拟内存
        //获取本SO模块的句柄,后续需要用它
        module = dm.getModule();
        vm.setJni(this); // 设置JNI
        vm.setVerbose(true); // 打印日志
        // 样本连JNI OnLoad都没有
        // dm.callJNI_OnLoad(emulator); // 调用JNI OnLoad
    };

    public String calculateS(){
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // 第一个参数是env
        list.add(0); // 第二个参数，实例方法是jobject，静态方法是jclazz，直接填0，一般用不到。
        DvmObject<?> context = vm.resolveClass("android/content/Context").newObject(null);// context
        list.add(vm.addLocalObject(context));
        list.add(vm.addLocalObject(new StringObject(vm, "12345")));
        list.add(vm.addLocalObject(new StringObject(vm, "r0ysue")));

        Number number = module.callFunction(emulator, 0x1E7C + 1, list.toArray())[0];
        String result = vm.getObject(number.intValue()).getValue().toString();
        return result;
    };

    public static void main(String[] args) {
        sina test = new sina();
        System.out.println(test.calculateS());
    }
}
```

顺嘴一提如何判断是 Thumb 还是 Arm 模式，最粗暴的方式就是试错法，比如此处不加 1，指令肯定就跑偏，会报错非法指令  
![](https://img-blog.csdnimg.cn/20210531160756139.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
这个办法粗鲁且有效，第二个办法是从知识角度出发，ARM 模式指令总是 4 字节长度，Thumb 指令长度多数为 2 字节，少部分指令是 4 字节。

IDA 顶部选项框：Options-General  
![](https://img-blog.csdnimg.cn/20210531160822320.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
查看汇编指令的机器码  
![](https://img-blog.csdnimg.cn/20210531160835672.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)我们发现此处指令大多为两个字节长度，那就是 Thumb。如果你还不放心，找准一行汇编，Alt+G 快捷键。  
![](https://img-blog.csdnimg.cn/20210531160922995.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)Thumb 模式是 1，ARM 模式是 0。除此之外，如果偶尔 IDA 反汇编出了问题，可以考虑它识别错了模式，需要 Alt+G 手动修改，调整模式。

言归正传，运行我们的代码。  
![](https://img-blog.csdnimg.cn/20210531160945216.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
真恼人，竟然报错了，而且没有较为明确的提示

看一下 Warn 一行显示的报错所处地址  
![](https://img-blog.csdnimg.cn/2021053116100276.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
IDA 快捷键 G 跳转到 0x2c8d，看这个架势 a1 是 JNIEnv 指针  
![](https://img-blog.csdnimg.cn/20210531161035900.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
把 a1 转成 JNIEnv  
![](https://img-blog.csdnimg.cn/20210531161049243.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
按 X 查看一下交叉引用，再往上看看，可以发现就是 sub_1C60 函数。从先前的分析可以看出，这个函数会返回一个值，如果为真，就继续执行，为假，就返回 0。再结合此地里面找的这些类，诸如 PackageManager 之流，很难不让人联想到签名校验函数。

可以直接 patch 掉对这个函数的调用，说人话就是把这儿的函数跳转改成不跳转了呗。  
![](https://img-blog.csdnimg.cn/20210531161131505.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
正常执行这个函数的话，如果校验没问题返回真，比如 1，校验失败返回 0。

根据 ARM 调用约定，入参前四个分别通过 R0-R3 调用，返回值通过 R0 返回，所以这儿可以通过 “mov r0,1” 实现我们的目标——不执行这个函数，并给出正确的返回值。除此之外还有一个幸运的地方在于，这个函数并没有产生一些之后需要使用的值或者中间变量，这让我们不需要管别的寄存器。

此处的机器码是 FF F7 EB FE, 查看一下 “mov r0,1” 的机器码，这里我们使用 [ARMConvert](https://armconverter.com/?code=mov%20r0,1) 看一下，除此之外，使用别的工具查看汇编代码也是可以的。

![](https://img-blog.csdnimg.cn/20210531161146588.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
即把 FF F7 EB FE 替换成 4FF00100 即可

这个事儿我们过去用 Keypatch 干，用 Frida 干，用 010Editor 干，现在用 Unidbg 干罢了，新瓶装旧酒！

Unidbg 提供了两种方法打 Patch，简单的需求可以调用 Unicorn 对虚拟内存进行修改，如下

```
public void patchVerify(){
    int patchCode = 0x4FF00100; // 
    emulator.getMemory().pointer(module.base + 0x1E86).setInt(0,patchCode);
}
```

需要注意的是，这儿地址可别 + 1 了，Thumb 的 + 1 只在运行和 Hook 时需要考虑，打 Patch 可别想。

看一下现在的完整代码，以及运行结果。

```
package com.lession2;

// 导入通用且标准的类库
import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.android.dvm.array.ByteArray;
import com.github.unidbg.memory.Memory;
import com.lession1.oasis;
import org.apache.log4j.Level;
import org.apache.log4j.Logger;

import java.io.File;
import java.util.ArrayList;
import java.util.List;

public class sina extends AbstractJni{
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;

    sina() {
        // 创建模拟器实例,进程名建议依照实际进程名填写，可以规避针对进程名的校验
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.sina.International").build();
        // 获取模拟器的内存操作接口
        final Memory memory = emulator.getMemory();
        // 设置系统类库解析
        memory.setLibraryResolver(new AndroidResolver(23));
        // 创建Android虚拟机,传入APK，Unidbg可以替我们做部分签名校验的工作
        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\lession2\\sinaInternational.apk"));
        //
//        vm = emulator.createDalvikVM(null);

        // 加载目标SO
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\lession2\\libutility.so"), true); // 加载so到虚拟内存
        //获取本SO模块的句柄,后续需要用它
        module = dm.getModule();
        vm.setJni(this); // 设置JNI
        vm.setVerbose(true); // 打印日志
        // 样本连JNI OnLoad都没有
        // dm.callJNI_OnLoad(emulator); // 调用JNI OnLoad
    };

    public String calculateS(){
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // 第一个参数是env
        list.add(0); // 第二个参数，实例方法是jobject，静态方法是jclazz，直接填0，一般用不到。
        DvmObject<?> context = vm.resolveClass("android/content/Context").newObject(null);// context
        list.add(vm.addLocalObject(context));
        list.add(vm.addLocalObject(new StringObject(vm, "12345")));
        list.add(vm.addLocalObject(new StringObject(vm, "r0ysue")));

        Number number = module.callFunction(emulator, 0x1E7C + 1, list.toArray())[0];
        String result = vm.getObject(number.intValue()).getValue().toString();
        return result;
    };

    public void patchVerify(){
        int patchCode = 0x4FF00100; //
        emulator.getMemory().pointer(module.base + 0x1E86).setInt(0,patchCode);
    }

    public static void main(String[] args) {
        sina test = new sina();
        test.patchVerify();
        System.out.println(test.calculateS());
    }
}
```

![](https://img-blog.csdnimg.cn/20210531161202483.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
直接出结果了。  
我们的 Patch 效果非常可，帮助我们绕过了签名校验的烦人逻辑。但有些情况下，我们可能要动态打 Patch，或者我们并不想上什么网站，看 MOV R0,1 的机器码是什么，这时候可以使用 Unidbg 给我们封装的 Patch 方法。

```
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
    };
```

逻辑也非常清晰，先确认有没有找对地方，地址上是不是 FF F7 EB FE，再用 Unicorn 的好兄弟 Keystone 把 patch 代码 “mov r0,1" 转成机器码，填进去，校验一下长度是否相等，收工。

#### 四、算法分析

在第一篇中，我们使用 Findhash 对算法做了分析，但是纯粹用 Unidbg 做算法分析一定是件激动人心的事，让我们来试一下吧。  
![](https://img-blog.csdnimg.cn/20210531161250473.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
代码的逻辑非常简单，将 text 和 key 拼接起来，然后放到 MDStringOld 函数中，出来的结果，从中分别抽出第 1 位（从 0 开始），第 5 位，等 8 位，就是结果了。

所以这个时候我们的关注点就是 MDStringOld 函数，首要的就是获取它的参数和返回值。

*   它的参数可以验证我们对 MDStringOld 函数前面的分析有没有出错
    
*   它的返回值可以验证我们对 MDStringOld 函数后面和结果的分析有没有出错
    

这个函数的地址是 0x1BD0+1  
![](https://img-blog.csdnimg.cn/20210531161309414.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
如果是 Frida 动态分析，我们会通过如下方式 Hook

```
function hookMDStringOld() {
    var baseAddr = Module.findBaseAddress("libutility.so")
    var MDStringOld = baseAddr.add(0x1BD0).add(0x1)
    Interceptor.attach(MDStringOld, {
        onEnter: function (args) {
            console.log("input:\n", hexdump(this.arg0))
        },
        onLeave: function (retval) {
            console.log("result:\n", hexdump(retval))
        }
    })
}
```

那么在 Unidbg 中，我们该怎么做呢

Unidbg 内嵌了多种 Hook 工具，目前主要是四种

*   Dobby
*   HookZz
*   xHook
*   Whale

但我们没必要四种都学

xHook 是爱奇艺开源的基于 PLT HOOK 的 Hook 框架，它无法 Hook 不在符号表里的函数，也不支持 inline hook，这在我们的逆向分析中是无法忍受的，所以在这里不去理会它。

Whale 在 Unidbg 的测试用例中只有对符号表函数的 Hook，没看到 Inline Hook 或者 非导出函数的 Hook，所以也不去考虑。

HookZz 是 Dobby 的前身，两者都可以 Hook 非导出表中的函数，即 IDA 中显示为 sub_xxx 的函数，也都可以进行 inline hook，所以二选一就行了。我喜欢 HookZz 这个名字，所以就 HookZz 了。

使用 HookZz hook MDStringOld 函数，MDStringOld 是导出函数，可以传入符号名，解析地址，但管他什么 findsymbol，findExport 呢，我就认准地址，地址，yyds。

看一下完整代码

```
package com.lession2;

// 导入通用且标准的类库
import com.github.unidbg.Emulator;
import com.github.unidbg.arm.context.Arm32RegisterContext;
import com.github.unidbg.hook.hookzz.*;
import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.android.dvm.array.ByteArray;
import com.github.unidbg.memory.Memory;
import com.github.unidbg.pointer.UnidbgPointer;
import com.github.unidbg.utils.Inspector;
import com.lession1.oasis;
import com.sun.jna.Pointer;
import keystone.Keystone;
import keystone.KeystoneArchitecture;
import keystone.KeystoneEncoded;
import keystone.KeystoneMode;
import org.apache.log4j.Level;
import org.apache.log4j.Logger;

import java.io.File;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;

public class sina extends AbstractJni{
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;

    sina() {
        // 创建模拟器实例,进程名建议依照实际进程名填写，可以规避针对进程名的校验
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.sina.International").build();
        // 获取模拟器的内存操作接口
        final Memory memory = emulator.getMemory();
        // 设置系统类库解析
        memory.setLibraryResolver(new AndroidResolver(23));
        // 创建Android虚拟机,传入APK，Unidbg可以替我们做部分签名校验的工作
        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\lession2\\sinaInternational.apk"));
        //
//        vm = emulator.createDalvikVM(null);

        // 加载目标SO
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\lession2\\libutility.so"), true); // 加载so到虚拟内存
        //获取本SO模块的句柄,后续需要用它
        module = dm.getModule();
        vm.setJni(this); // 设置JNI
        vm.setVerbose(true); // 打印日志
        // 样本连JNI OnLoad都没有
        // dm.callJNI_OnLoad(emulator); // 调用JNI OnLoad
    };

    public String calculateS(){
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // 第一个参数是env
        list.add(0); // 第二个参数，实例方法是jobject，静态方法是jclazz，直接填0，一般用不到。
        DvmObject<?> context = vm.resolveClass("android/content/Context").newObject(null);// context
        list.add(vm.addLocalObject(context));
        list.add(vm.addLocalObject(new StringObject(vm, "12345")));
        list.add(vm.addLocalObject(new StringObject(vm, "r0ysue")));

        Number number = module.callFunction(emulator, 0x1E7C + 1, list.toArray())[0];
        String result = vm.getObject(number.intValue()).getValue().toString();
        return result;
    };

    public void patchVerify(){
        int patchCode = 0x4FF00100; //
        emulator.getMemory().pointer(module.base + 0x1E86).setInt(0,patchCode);
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
    };

    public void HookMDStringold(){
        // 加载HookZz
        IHookZz hookZz = HookZz.getInstance(emulator);

        hookZz.wrap(module.base + 0x1BD0 + 1, new WrapCallback<HookZzArm32RegisterContext>() { // inline wrap导出函数
            @Override
            // 类似于 frida onEnter
            public void preCall(Emulator<?> emulator, HookZzArm32RegisterContext ctx, HookEntryInfo info) {
                // 类似于Frida args[0]
                Pointer input = ctx.getPointerArg(0);
                System.out.println("input:" + input.getString(0));
            };

            @Override
            // 类似于 frida onLeave
            public void postCall(Emulator<?> emulator, HookZzArm32RegisterContext ctx, HookEntryInfo info) {
                Pointer result = ctx.getPointerArg(0);
                System.out.println("input:" + result.getString(0));
            }
        });
    }

    public static void main(String[] args) {
        sina test = new sina();
//        test.patchVerify();
        test.patchVerify1();
        test.HookMDStringold();
        System.out.println(test.calculateS());
    }
}
```

运行结果  
![](https://img-blog.csdnimg.cn/20210531161334496.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

可以发现，入参就是 text+key

验证返回值：439a333788b0cecfce1389d4b83ba1cb

![](https://img-blog.csdnimg.cn/20210531161345888.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

```
result = 439a333788b0cecfce1389d4b83ba1cb 

result[1] = 3
result[5] = 3
result[2] = 9
result[10] = b
```

验证发现我们关于结果来源的猜想也完全正确。

那么接下来的焦点就是 MDStringOld 函数了，因为结果是 32 位，我们首先想到 MD5 函数，验证一下。

![](https://img-blog.csdnimg.cn/20210531161412844.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
结果完全一致。

NICE，大功告成。

#### 五、尾声

这个样本十分简单，但让我们更多的理解了 Unidbg 的功能，下一讲中，让我们解锁更难的样本，探索 Unidbg 更多功能吧！

样本https://github.com/miangit/simpread/releases/download/backup/unidbg2.7z

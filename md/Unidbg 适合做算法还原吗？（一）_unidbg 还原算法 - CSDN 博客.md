> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/qq_38851536/article/details/118073818)

**Unidbg 是模拟执行的强大工具，这是毋庸置疑的，可是，它在算法还原上是否依然是得力的助手？或者说，当我们想要对一个样本进行算法还原而非模拟执行呢，需要关注 Unidbg 吗？**

我们看一下 AB 两人的辩论，他俩会进行数篇的辩论，客官也可展开思考，发表和交流看法。

A：Unidbg 在算法模拟执行上很有用，但算法还原上用它纯属画蛇添足。  
B：为什么这么觉得？  
A：先看第一种情况——目标 SO 比较简单，这种情况下，IDA 中静态分析样本，再 Frida Hook 一下样本中主要的函数，就能七七八八明白逻辑了，何苦在 Unidbg 中去补各种环境呢？  
B：你说的有道理，但我要反驳一点，如果 SO 比较简单，那么多半，它的环境也不会太难或者太复杂，对应的，Unidbg 中补环境的工作量也不会太大。  
A：这无法打动我，环境不太难补，那不还是有一定工作量吗？本来我只用 Frida Hook，现在还要补环境，平白无故增加了工作量！  
B：emmm 确实，但是，Unidbg 本就不是设计给简单 SO 的嘛，简单样本直接 Frida hook + IDA 确实是更好的选择。但不简单的样本，Unidbg 就很有必要了。  
A：你认为不简单的样本，Unidbg 会给算法分析带来一定的帮助？  
B：嗯，在 Unidbg 跑通之后，只需要日志全开，我们可以清清楚楚的查看：

*   JNI 调用
*   系统调用
*   标准库函数调用
*   汇编 trace

A：我并不把这些当回事！首先，“Unidbg 帮助我们看清 JNI 调用” 就是一个伪命题！用 JNItrace 不是看得清清楚楚吗？Unidbg 并没有在这件事上帮助我们，甚至相反，我们常常得依靠 JNItrace 的结果以及 Unidbg 的报错来补环境！补 JAVA 环境！  
B：呃，那系统调用呢？Unidbg 不是让把系统调用看得清清楚楚吗？  
A：别提了！trace 系统调用可不是 Unidbg 的特权！我们有大把的工具去做好这件事！  
B：比如呢？  
A: Strace！  
B：举个例子演示一下它吧  
A：在 Native 层，如何我们想获得某个系统属性（system property），有这样几种常用方法

*   通过 JNI 与 JAVA 交互，获取信息
    
    ```
    jclass androidBuildClass = env->FindClass("android/os/Build");
    jfieldID SERIAL = env->GetStaticFieldID(androidBuildClass, "SERIAL", "Ljava/lang/String;");
    jstring serialNum = (jstring) env->GetStaticObjectField(androidBuildClass, SERIAL);
    ```
    
*   通过__system_property_get api 获取信息
    
    ```
    char *key = "ro.build.id";
    char value[PROP_VALUE_MAX] = {0};
    __system_property_get(key, value);
    ```
    
*   通过 popen() 管道从 shell 中获取返回值
    
    ```
    char value[PROP_VALUE_MAX] = {0};
    std::string cmd = "getprop ro.build.id";
    FILE* file = popen(cmd.c_str(), "r");
    fread(value, PROP_VALUE_MAX, 1, file);
    pclose(file);
    ```
    

以这常见的三种方式为例，你说 Unidbg 怎么处理？

B：这非常简单，第一种情况就是补 JAVA 环境

```
@Override
    public DvmObject<?> getStaticObjectField(BaseVM vm, DvmClass dvmClass, String signature) {
        switch (signature) {
            case "android/os/Build->SERIAL:Ljava/lang/String;":
                // serial 的值
                return new StringObject(vm, "xxxx");
        }
        return super.getStaticObjectField(vm, dvmClass, signature);
    }
```

第二种情况还真比较特殊，当样本通过__system_property_get 获取相关属性时，如果我们不做自定义的处理，它会试图从这个文件中读取，如果获取的属性在该文件中找不到，就返回空值。

![](https://img-blog.csdnimg.cn/20210620193225264.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

A：那如果我想控制属性值呢？  
B：处理也很简单，下面是示例代码

```
public class getproperty extends AbstractJni{
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;
    public final Memory memory;

    getproperty(){
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.getproperty").build();
        
        memory = emulator.getMemory(); // 模拟器的内存操作接口
        memory.setLibraryResolver(new AndroidResolver(19)); // 设置系统类库解析

        SystemPropertyHook systemPropertyHook = new SystemPropertyHook(emulator);
        systemPropertyHook.setPropertyProvider(new SystemPropertyProvider() {
            @Override
            public String getProperty(String key) {
                switch (key){
                    case "ro.build.id":
                        return "12345";
                }
                throw new UnsupportedOperationException(key);
            }
        });
        memory.addHookListener(systemPropertyHook);

        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\learnproperty\\getproperty.apk"));

        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\learnproperty\\getproperty.so"), true);
        module = dm.getModule();

        vm.setJni(this);
        vm.setVerbose(true);
        dm.callJNI_OnLoad(emulator);
    }
```

A：那如果是 popen 方式读取系统属性呢，Unidbg 会遇到问题吗？  
B: 你等等。。emmm，“通过 popen() 管道从 shell 中获取返回值” 是什么意思？  
A：在 adb shell 中 我们可以如下方式获取系统属性

![](https://img-blog.csdnimg.cn/20210620193243646.png)  
通过 popen() 管道 就是在代码里开了个 shell ! popen 的实现比较复杂，里面有很多系统调用的参与。我们就用这个例子，测试一下。（apk 见百度网盘）

首先安装一下 strace，[Android 上利用 strace 跟踪系统调用 | m4bln (mabin004.github.io)](https://mabin004.github.io/2019/06/27/Android%E4%B8%8A%E5%88%A9%E7%94%A8Strace%E8%B7%9F%E8%B8%AA%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8/)。使用 strace 产生了数百条日志

![](https://img-blog.csdnimg.cn/20210620193324227.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
在其中可以看到我们需要的内容，Strace 对系统调用的追踪能力是 OK 的，除此之外，Frida 也可以做这件事，比如 [AeonLucid/frida-syscall-interceptor (github.com)](https://github.com/AeonLucid/frida-syscall-interceptor)

B：Strace 输出的干扰也太多了，除此之外，Frida trace syscall 的项目都不太完善，比如只支持 arm64 或者容易崩溃，让你看看 Unidbg 的表现！呃。。。事实上，Unidbg 目前还不支持直接跑 popen，需要手动处理一下，继承和使用自己的 syscallHandler

```
package com.learnproperty;

import com.github.unidbg.Emulator;
import com.github.unidbg.arm.context.EditableArm32RegisterContext;
import com.github.unidbg.linux.file.ByteArrayFileIO;
import com.github.unidbg.linux.file.DumpFileIO;
import com.github.unidbg.memory.SvcMemory;
import com.sun.jna.Pointer;

import java.util.concurrent.ThreadLocalRandom;

public class MyARMSyscallHandler extends com.github.unidbg.linux.ARM32SyscallHandler {
    public MyARMSyscallHandler(SvcMemory svcMemory) {
        super(svcMemory);
    }
    @Override
    protected boolean handleUnknownSyscall(Emulator emulator, int NR) {
        switch (NR) {
            case 190:
                vfork(emulator);
                return true;
            case 359:
                pipe2(emulator);
                return true;
        }

        return super.handleUnknownSyscall(emulator, NR);
    }

    private void vfork(Emulator<?> emulator) {
        EditableArm32RegisterContext context = (EditableArm32RegisterContext) emulator.getContext();
        int childPid = emulator.getPid() + ThreadLocalRandom.current().nextInt(256);
        int r0 = 0;
        r0 = childPid;
        System.out.println("vfork pid=" + r0);
        context.setR0(r0);
    }

    private void pipe2(Emulator<?> emulator) {
        EditableArm32RegisterContext context = (EditableArm32RegisterContext) emulator.getContext();
        Pointer pipefd = context.getPointerArg(0);
        int flags = context.getIntArg(1);
        int write = getMinFd();
        this.fdMap.put(write, new DumpFileIO(write));
        int read = getMinFd();
        String stdout = "myid\n"; // getprop ro.build.id
        this.fdMap.put(read, new ByteArrayFileIO(0, "pipe2_read_side", stdout.getBytes()));
        pipefd.setInt(0, read);
        pipefd.setInt(4, write);
        System.out.println("pipe2 pipefd=" + pipefd + ", flags=0x" + flags + ", read=" + read + ", write=" + write + ", stdout=" + stdout);
        context.setR0(0);
    }
}
```

```
public class getproperty extends AbstractJni{
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;
    public final Memory memory;

    getproperty(){
        AndroidEmulatorBuilder builder = new AndroidEmulatorBuilder(false) {
            @Override
            public AndroidEmulator build() {
                return new AndroidARMEmulator(processName, rootDir, backendFactories) {
                    @Override
                    protected UnixSyscallHandler<AndroidFileIO> createSyscallHandler(SvcMemory svcMemory) {
                        return new MyARMSyscallHandler(svcMemory);
                    }
                };
            }

            ;
        };
        emulator = builder.setProcessName("com.readProperty").build();
        memory = emulator.getMemory(); // 模拟器的内存操作接口
        memory.setLibraryResolver(new AndroidResolver(19)); // 设置系统类库解析

        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\learnproperty\\getproperty.apk"));

        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\learnproperty\\getproperty.so"), true);
        module = dm.getModule();

        vm.setJni(this);
        vm.setVerbose(true);
        dm.callJNI_OnLoad(emulator);
    }
```

A：所以综合来说，Unidbg 在对系统调用这块的处理和表现，并没有表现出让人” 非用不可 “的诱惑。  
B：是的，甚至我得诚实地说，Unidbg 中部分系统调用还未实现，除此之外，还有一些 Bug，某些样本上会跑不通，这都是急需解决的问题。  
A：那你还有什么观点？  
B：Unidbg 在 Hook 上的功能非常强大，让人着迷！  
A：为什么你这么觉得？Frida 不强大吗？打个比方，使用 Frida Hook 一个函数，参数 1 是输入，参数 2 是指针，用于存放返回值，参数 3 是长度，我该怎么打印函数运行完之后，参数 2 中的内存呢？在 frida 中我可以这么做

```
Interceptor.attach(ptr, {
    onEnter: function (args) {
        this.arg0 = args[0];
        this.arg1 = args[1];
        this.arg2 = args[2];
    }, onLeave: function (retval) {
        console.log("onLeave:\r\n", hexdump(this.arg1,  {length: this.arg2}));
    }
});
```

在 Unidbg 中标准流程这么做

```
public void hook(){
        IHookZz hookZz = HookZz.getInstance(emulator); // 加载HookZz
        hookZz.wrap(module.base + 0x95E5C + 1, new WrapCallback<HookZzArm32RegisterContext>() {
            @Override
            // 方法执行前
            public void preCall(Emulator<?> emulator, HookZzArm32RegisterContext ctx, HookEntryInfo info) {
                int length = ctx.getIntArg(1);
                Pointer out = ctx.getPointerArg(2);
                ctx.push(out);
                ctx.push(length);
            };

            @Override
            // 方法执行后
            public void postCall(Emulator<?> emulator, HookZzArm32RegisterContext ctx, HookEntryInfo info) {
                int length = ctx.pop();
                Pointer output = ctx.pop();
                byte[] outputhex = output.getByteArray(0, length);
                Inspector.inspect(outputhex, "SHA1 output");
            }
        });
        hookZz.disable_arm_arm64_b_branch();
    }
```

或者这样

```
public void hook(){
        IHookZz hookZz = HookZz.getInstance(emulator); // 加载HookZz
        hookZz.wrap(module.base + 0x95E5C + 1, new WrapCallback<HookZzArm32RegisterContext>() {
            int length;
            Pointer out;
            
            @Override
            // 方法执行前
            public void preCall(Emulator<?> emulator, HookZzArm32RegisterContext ctx, HookEntryInfo info) {
                length = ctx.getIntArg(1);
                out = ctx.getPointerArg(2);
            };

            @Override
            // 方法执行后
            public void postCall(Emulator<?> emulator, HookZzArm32RegisterContext ctx, HookEntryInfo info) {
                byte[] outputhex = output.getByteArray(0, length);
                Inspector.inspect(outputhex, "SHA1 output");
            }
        });
        hookZz.disable_arm_arm64_b_branch();
    }
```

代码量并没有比 Frida 少吧？甚至还更多！  
B：Unidbg 支持 Xhook、HookZz 等 Hook 工具，以及 IDA server 和 GDB。确实，xHook，HookZz 等并不比 Frida 用起来更简单或者更强大，而 Unidbg 对 IDA server/GDB 的支持仍处于玩具性质。但我说的 Hook 强大，指的是 Unidbg 的 console debugger。它简单、优雅、强大！

A：噢，有用例吗？  
B：下一篇详细介绍！

链接：https://github.com/miangit/simpread/releases/download/backup/getproperty.7z

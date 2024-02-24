> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/qq_38851536/article/details/118122592)

#### 1.Long 参数的传递

假设一个 [native](https://so.csdn.net/so/search?q=native&spm=1001.2101.3001.7020) 函数中参数是 long 类型, 比如这样

![](https://img-blog.csdnimg.cn/20210622235010451.png)

在编译成 arm32 的 SO 时, 一定概率会被转成两个 int.

long a = 0x1000L → int a1 = 0,int a2 = 0x1000

在 Unidbg 主动调用时, 一定要记得处理, 否则会出问题. 这是一个常见问题, JAVA 层传入的[时间戳](https://so.csdn.net/so/search?q=%E6%97%B6%E9%97%B4%E6%88%B3&spm=1001.2101.3001.7020), 常常就是 jlong.

处理办法有 2

1 是按照 SO 的情况, 传给它两个 int

2 是按照传入诸如 `long tm= 1621265630L;`的标准写法, Unidbg 自动帮我们分割成两个

![](https://img-blog.csdnimg.cn/20210622235018559.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

#### 2.jbytearray 怎么查看

更宽泛的问法是, 有个 jobject 对象, 想查看它的内容, 最常见的就是 jbyteArray

Frida 中可以这么操作

```
hexdump(ptr(Java.vm.tryGetEnv().getByteArrayElements(args[0])))
```

Unidbg 中当然也可以, 以 hookZz 中为例

```
public void preCall(Emulator<?> emulator, HookZzArm32RegisterContext ctx, HookEntryInfo info) {
    UnidbgPointer jbytearrayptr = ctx.getPointerArg(2);

    DvmObject<?> dvmbytes = vm.getObject(jbytearrayptr.toIntPeer());
    // 取出byte
    byte[] result = (byte[]) dvmbytes.getValue();
    // 转换成String 或者按需转成其他
    System.out.println(new String(result));
};
```

#### 3.std::string 的读写

```
public String readStdString(Pointer strptr){
    Boolean isTiny = (strptr.getByte(0) & 1) == 0;
    if(isTiny){
        return strptr.getString(1);
    }
    return strptr.getPointer(emulator.getPointerSize()* 2L).getString(0);
}

public void writeStdString(Pointer strptr, String content){
    Boolean isTiny = (strptr.getByte(0) & 1) == 0;
    if(isTiny){
        strptr.write(1, content.getBytes(StandardCharsets.UTF_8), 0, content.length());
    }
    strptr.getPointer(emulator.getPointerSize()* 2L).write(0, content.getBytes(StandardCharsets.UTF_8), 0, content.length());
};
```

#### 4.TraceCode 为什么 trace 不到 module init 中的指令

![](https://img-blog.csdnimg.cn/20210622235039458.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

是因为 traceCode 的执行时机晚了, load library 在它前面… 只需要确认 module base 和 module size 后, 将 emulator.traceCode 写在 loadlibrary 前面即可.

#### 5.HOOK 框架使用问题

Unidbg 支持了数个 Hook 框架, HookZz 和 Dobby 就是其中两个. 有人会困惑, HookZz 不就是 Dobby 前身吗, 两者不是一个东西吗? 为什么要说两个 Hook 框架?

这其实是有原因的, Unidbg 作者在注释中写道: HookZz 在 arm32 位上支持较好, Dobby 在 64 位上支持较好.(因此将两者, 或者说 Dobby 以及其前身 HookZz 作为两个独立 Hook 工具)
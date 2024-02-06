> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/Eeyhan/p/17326769.html)

前言
--

这个专题是根据白龙，龙哥的 unidbg 博客的案例，进行从 0 开始到逆向的流程，核心部分会借鉴龙哥的 unidbg，通过借鉴大佬的思路，完整的分析某个 so 层的加密参数

各位朋友也可以直接读龙哥的博客，我只是用我的角度进一步加工一下

原文地址：[SO 逆向入门实战教程一：OASIS_so 逆向学习路线_白龙~ 的博客 - CSDN 博客](https://blog.csdn.net/qq_38851536/article/details/117418582)

分析
--

首先拿到这个 app，安装啥的就不多说了。

进入到注册界面：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230417174907626-2127778749.png)

点击获取验证码，然后这边抓包工具抓到的包：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230417175000073-938368084.png)

然后，这里面的【sign】就是今天的重点了。

用神秘的工具脱壳完之后，有如下的 dex：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230417175054739-859674407.png)

把这些 dex，全部选中，拖进 jadx 

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230417175125927-1963395186.png)

快速定位
----

接下来开始找 sign 所在的位置，先看看抓包工具这边的参数，根据这些键名，一顿搜，总能找到一些

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230417175213338-490416289.png)

先搜下【sign】，虽然搜出来的结果不多，但是感觉有很多干扰项

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230417175332786-374771655.png)

 进最后那个，到这里，

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230417175407815-528188318.png)

这些参数看着很像，用 frida hook，得知，发现并没有走到这里

搜【ua】

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230417175314211-1205286796.png)

进到这里，发现很可疑，好多参数都对上了

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230417175708904-393212360.png)

啥都不说，先用 objection hook 下，然后 app 端点击【重新发送】看看：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230417175805955-713873928.png)

可以，这不直接就定位逻辑了

但是我们要找【sign】，所以还得再看看，jadx 查看发现，这个方法反编译效果不太好，问题不大，记住这个 dex 名，然后用 GDA 打开这个 dex，这就挺好，基本都反编译出来了

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230417175952230-554176367.png)

反正看着确实可疑，但是这三个方法，还不确定到底是是哪个方法里有【sign】部分，所以这里直接 hook 它这个 a,b，c 三个方法，然后 app 点下重新获取

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230417180125977-336012723.png)

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230417180151270-1128627255.png)

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230417180215370-100943870.png)

ok，发现最后的方法里基于有我们要的 sign，这边再来下，用抓包工具对比下，谨慎一点，别搞半天没搞对位置：

重新 hook 下，然后抓包工具打开看看

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230417180749199-1199424995.png)

对上了，ok，就是这里了【g.a.c.g.c.a】

在 jadx 里，反编译失败：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230417180831270-510264223.png)

问题不大，用 GDA 看：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230417181048756-1914661399.png)

这不就越来越接近了吗，嘻嘻，点进去：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230417181136312-879124513.png)

hook 下这个 c 方法看看，ok，对上了

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230417181335419-1781712006.png)

 接着一顿分析后再进入这里

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230417181426048-430813443.png)

 ok，进到了 native 层，那么核心的逻辑就在这里了。

像这种，常规的方法怎么解决呢？

*   如果你是为了拿数据，赶工期的话，那你可以直接主动调用这个方法
*   如果你是为了纯算还原的话，那就把这个 so 文件拖进 ida 一顿分析了
*   那么假如，这个方法的纯算很复杂，而你又不想主动调用，感觉很 low 的话，那你就可以尝试用 unidbg 了

unidbg，就是本系列文章的重点了。

unidbg 简介
---------

### 什么是 unidbg

unidbg 是一个基于 unicorn 的逆向工具，可以实现黑盒调用安卓和 iOS 中的 so 文件

unidbg 是凯神写的一个开源的 java 项目

### 使用场景

因为现在的大多数 app 把核心的加密算法放到了 so 文件中（比如本例的 app），你要想破解签名算法，必须能够破解 so 文件。但 C++ 的逆向远比 Java 的逆向要难得多，有各种混淆啊，ollvm 啥的，所以好多时候是没法纯算还原破解的

那么你是否有过一个想法，能不能把安卓的环境模拟出来，但是又脱离了安卓真机的环境，就可以直接使用 so 里的方法呢？

unidbg 就是这样一个工具，它模拟好了好几种虚拟环境，他不需要直接运行 app，也无需逆向 so 文件，而是直接找到对应的 JNI 接口，然后用 unicorn 引擎（也不止这一个引擎）直接执行这个 so 文件，所以效率也比较高。

### 配置 unidbg

1. 首先，用 git 把这个项目 clone 下来：

> [zhkl0228/unidbg: Allows you to emulate an Android native library, and an experimental iOS emulation (github.com)](https://github.com/zhkl0228/unidbg)  

2. 再用 idea（写 java 项目那个编辑器）打开，

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230417182843851-717085216.png)

首次打开，右下角会下载很多依赖环境，等待即可 

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230417183147156-751265501.png)

然后随便找一个项目，执行下 main 方法，有正常输出，说明环境配置好了：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230417183319035-1040207653.png)

执行结果：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230417183349642-505614367.png)

ok，这就很 nice。 

frida 调试
--------

接下来，先用 ida 打开那个目标 so 文件

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230418101218956-631778421.png)

等左下角这个数据没有再变的时候，说明加载好了

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230418101157142-125725144.png)

接下来找导出表【export】，或者在左边的栏里搜【java】

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230418101408611-1600073190.png)

发现并没有任何东西，那么这里就是动态注册的 so 方法了。

### 什么是动态注册、静态注册

静态注册（又叫静态绑定），就是，so 的方法名直接 export 导出表里，且命名格式为【java_app 包名_方法名】，比如 java_com_sina_oasxxx_nativeapi_s

动态注册（又叫动态绑定）就是 export 到处表里没有的就是动态注册。

那么这里我们的目标方法就是动态注册的了，那咋办，看看 JNI_load 方法：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230418102021387-1183710472.png)

按下【tab】键，会由上面的汇编代码反编译为 c 代码：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230418102146367-1860821917.png)

再按下【\】反斜杠，可读性更强点：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230418102214914-889791491.png)

但是这里发现，一顿 while 和 if，根据龙哥的博客说的，大概率是 ollvm 混淆。那咋办？

用 yang 神的 hook_native 脚本来 hook 出目标函数的偏移地址 ，然后加上 so 的基址，就可以得到目标函数的地址了（看是 thumb 还是 arm，thumb 要加 1，arm 不用）

> [frida_hook_libart/hook_RegisterNatives.js at master · lasting-yang/frida_hook_libart (github.com)](https://github.com/lasting-yang/frida_hook_libart/blob/master/hook_RegisterNatives.js)

用 frida hook 一下，结果到这就报错退出了

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230418110143053-1275146957.png)

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230418110200251-234932398.png)

就很尴尬，看看，他报的哪个 class 名，用来过滤下试试：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230418110248881-474578226.png)

重新运行 frida 试试，可以，这下直接就定位到我们要的方法的位置了：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230418110333945-2145391750.png)

这里有朋友估计会说，卧槽，这不都出来了吗，这地址，拿着直接用啊，不急，我重启脚本看看：仔细看，fnptr 地址变了，fnoffset 和后面的 jni-load + 的地址没变的

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230418152851459-1469858173.png)

 多次 hook 发现确实如此，因为这里就是上面说的动态注册，所以这个 fnptr，也就是 so 的基址，app 没启动一次就会变，但是目标方法的偏移值是不会变的。ok

用 frida hook so 看看：

相关的 hoo so，有个大佬总结的很好：

> [分类: frida | 凡墙总是门 (kevinspider.github.io)](https://kevinspider.github.io/frida/frida-hook-so/)

ok，这里我们 hook 下，拿下入参和返回值看看：

```
function inline_hook() {
    var so_addr = Module.findBaseAddress("liboasiscore.so");
    console.log("so_addr:", so_addr);
    if (so_addr) {
        var sub = so_addr.add(0x116cc); // 不用加1，是arm架构
        console.log("The addr_0x116cc:", sub);
        Java.perform(function () {
            Interceptor.attach(sub,
                {
                    onEnter: function (args) {
                        console.log("addr_0x116cc OnEnter :", this.context.PC,
                            this.context.x1, this.context.x5,
                            this.context.x10);
                    },
                    onLeave: function (retval) {
                        console.log("retval is :", retval)
                    },
                })
        })
    }
}
 
 
setTimeout(inline_hook, 1000)
```

运行结果：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230418153309044-273571313.png)

 发现并不可读，没事，反正至少是有了

```
function stringToBytes(str) {
  return hexToBytes(stringToHex(str))
}
 
function stringToHex(str) {
  return str
    .split('')
    .map(function (c) {
      return ('0' + c.charCodeAt(0).toString(16)).slice(-2)
    })
    .join('')
}
 
function hexToBytes(hex) {
  for (let bytes = [], c = 0; c < hex.length; c += 2) bytes.push(parseInt(hex.substr(c, 2), 16))
  return bytes
}
 
function hexToString(hexStr) {
  let hex = hexStr.toString()
  let str = ''
  for (let i = 0; i < hex.length; i += 2) str += String.fromCharCode(parseInt(hex.substr(i, 2), 16))
  return str
}<br><br><br>
```

```
function inline_hook() {
    var so_addr = Module.findBaseAddress("liboasiscore.so");
    console.log("so_addr:", so_addr);
    if (so_addr) {
        // var sub = so_addr.add(0x116cc); // 不用加1，是arm架构
        console.log("The addr_0x116cc:", so_addr);
        var ss = "aid=01A8SBOtNRVqsR1ywgkR4tHsZEsgXkGDrgKO2OvFBeThKWZDE.&cfrom=28B5295010&cuid=0&noncestr=L83x8Z40132Wan450y736563n3kmWj&phone=138469655665&platform=ANDROID×tamp=1681790293128&ua=Xiaomi-MI6__oasis__3.5.8__Android__Android9&version=3.5.8&vid=2010511512550&wm=20004_90024";
        var add_addr = so_addr.add(0x116cc); // 32位需要加1
        var add = new NativeFunction(add_addr, 'pointer', ['pointer', 'int']);
        console.log(add)
        var result = add(stringToByte(Memory.allocUtf8String(ss)), false)
        console.log("add2 result is ->" + result.readCString());
    }
 
}
function stringToByte(str) {
    var ch, st, re = [];
    for (var i = 0; i < str.length; i++) {
        ch = str.charCodeAt(i);
        st = [];
        do {
            st.push(ch & 0xFF);
            ch = ch >> 8;
        } while (ch);
        re = re.concat(st.reverse());
    }   // return an array of bytes
    return re;
}
 
setTimeout(inline_hook, 2000)
```

尝试主动调用，调试了很久，就是不行

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230418173951177-1309045497.png)

突然反应过来，这个 so 文件有 ollvm 混淆啊，虽然用 yang 神的代码 hook 到了偏移地址，但是他内部可能并不是这些参数，而我们又没法直接分析，那没法了。其实以上的代码，在其他地方是可以用的，

那接下来咋办？上 unidbg 吧

unidbg 调试
---------

上面用了 frida+ida 调试，发现有的时候没法搞啊，那么这里，终于要用 unidbg 来模拟执行生成上面目标 app 的 sign 了

 先创建一个文件，把该有的都放进去

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230418111130588-512650158.png)

然后再 oasis 里写代码，照着龙哥的搞就完了：注意文件路径，跟你实际的路径保持一致

```
package com.sina;
 
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.linux.android.dvm.DalvikModule;
import com.github.unidbg.linux.android.dvm.VM;
import com.github.unidbg.memory.Memory;
 
import java.io.File;
 
public class oasis extends AbstractJni {
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
        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\sina\\lvzhou.apk"));
        // 加载目标SO
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\sina\\liboasiscore.so"), true); // 加载so到虚拟内存
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

 然后运行一下，没啥问题，说明架子是搭上了

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230418111501239-294090644.png)

仔细看这里，这样也把我们要的方法的地址拿到了。舒服啊

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230418111614868-267125661.png)

用前面 frida 的对比，好像不太一样，问题不大

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230418111721944-122482129.png)

再来看看代码：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230418112339024-1333296878.png)

再来仔细看看他这个 main 干了啥：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230418111947650-183205918.png)

### 调用

架子搭好了，接下来调用，调用有两种方式，一种是符号（symbol）调用，一种是地址调用，符号调用对应静态注册，地址调用对应动态注册。那么根据前面的解析，这里我们只能选用地址调用了

先看看我们要调用的方法：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230418113209094-619972206.png)

有两个参数，一个 byte 数组，一个 boolean，然后 native 层的方法，默认前面会自动加两个参数 ，一个 jni env，一个 jobject

还是上面的 frida 脚本，先 hook 下，看看入参和返回：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230418122024013-1458046140.png)

ok，直接拿着这个 str 参数的值去 unidbg 构造：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230418122500553-195364131.png)

注意，如果你用的旧版，也就是龙哥案例的代码，直接报错：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230418122747391-1066307937.png)

新版得这么用，运行结果：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230418122838759-1917888737.png)

```
package com.sina;
 
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.linux.android.dvm.DalvikModule;
import com.github.unidbg.linux.android.dvm.VM;
import com.github.unidbg.linux.android.dvm.array.ByteArray;
import com.github.unidbg.memory.Memory;
import com.sun.jna.Pointer;
 
import java.io.File;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;
import java.lang.Number;
 
public class oasis extends AbstractJni {
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
        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\sina\\lvzhou.apk"));
        // 加载目标SO
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\sina\\liboasiscore.so"), true); // 加载so到虚拟内存
        //获取本SO模块的句柄,后续需要用它
        module = dm.getModule();
        vm.setJni(this); // 设置JNI
        vm.setVerbose(true); // 打印日志
 
        dm.callJNI_OnLoad(emulator); // 调用JNI OnLoad
    };
 
    public static void main(String[] args) {
        oasis test = new oasis();
        System.out.println(test.getSign());
    }
    public String getSign(){
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // arg1,env
        list.add(0); // arg2,jobject
        String keywords = "aid=01A8SBOtNRVqsR1ywgkR4tHsZEsgXkGDrgKO2OvFBeThKWZDE.&cfrom=28B5295010&cuid=0&noncestr=L83x8Z40132Wan450y736563n3kmWj&phone=138469655665&platform=ANDROID×tamp=1681790293128&ua=Xiaomi-MI6__oasis__3.5.8__Android__Android9&version=3.5.8&vid=2010511512550&wm=20004_90024";
        byte[] keyB = keywords.getBytes(StandardCharsets.UTF_8);
        ByteArray inbarr = new ByteArray(vm,keyB);
        list.add(vm.addGlobalObject(inbarr)); //arg3
        list.add(0); //arg4
//        Number number = module.callFunction(emulator,0xC365,list.toArray())[0];
        Number number = module.callFunction(emulator,0xC365,list.toArray());
        String result = vm.getObject(number.intValue()).getValue().toString();
        return result;
    }
}
```

验证下结果，对上了，舒服：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230418122907475-1248344619.png)

结语
--

unidbg 的实用之处就在这里，相信不用我多说，你已经发现了很多妙用之处
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/Eeyhan/p/17356812.html)

前言
--

继续跟着龙哥的 unidbg 学习：[SO 逆向入门实战教程六：s_白龙~ 的博客 - CSDN 博客](https://blog.csdn.net/qq_38851536/article/details/117923970)

还是那句，我会借鉴龙哥的文章，以一个初学者的角度，加上自己的理解，把内容丰富一下，尽量做到不在龙哥的基础上画蛇添足，哈哈。感谢观看的朋友

分析
--

首先抓个包看看：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230426173234596-1315855206.png)

这里面这个 sign，就是今天的重点了

这个 app 没有壳，直接 jadx 打开，然后搜【sign"】或者【sign=】

很快的，就找到这个地方

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230426173412825-2000401660.png)

目前确实还不确定是不是这里，用 objection hook 下就知道了：

 先 hook 了类，一堆，卧槽

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230426173917477-1299812095.png)

hook tostring 方法：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230426180127904-674320315.png)

app 多划两下：可以看到基本是对的上的

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230426180556706-985554445.png)

然后再找找堆栈调用，很快定位到这里：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230426174816806-2133623017.png)

hook 下这个类，发现有一堆，

慢慢来，首先这个 appkey，就是这里了

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230426182134340-136152087.png)

 根据龙哥说的，就是 libBili.s 方法，但是 hook 了之后就加载就一直卡住。换用 fridahook :

```
function showStacks() {
            console.log(
                Java.use("android.util.Log")
                    .getStackTraceString(
                        Java.use("java.lang.Throwable").$new()
                    )
            );
        }
function main() {
    Java.perform(function () {
        var ClassName = "com.bilibili.nativelibrary.LibBili";
        var Bilibili = Java.use(ClassName);
        Bilibili.s.overload('java.util.SortedMap').implementation = function (arg) {
            console.log(JSON.stringify(arg))
            Java.openClassFile("/data/local/tmp/r0gson.dex").load()
            const gson = Java.use('com.r0ysue.gson.Gson').$new();
            const json_x = gson.toJson(arg)
            console.log("json_x===>", json_x);
            let res = this.s(arg)
            console.log('result===>', res)
 
            showStacks()
            return res;
        }
    });
}
 
setTimeout(main, 2000)
```

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427102257842-1371727425.png)

感觉就是这个 s 了，入参是：

> {"ad_extra":"E0DA3BAB712CAB3902D558CFD4FEB865AB1563B7D61A4A57D02854BB9A4512C5E4737CA15664C505F3E63B954BB68837690B81FE77B2E7EB578F3E3D7C528439725348FE86527D7415BA2267FCCD2C95895AE80E69A960812FE60  
> BC477171DAD34472EC31D010940A2876CADE887EA26FD5FBFB718D09F193D14732E54E42EB329EBF61CA33DE876AF24C03A7B9E89DB9C831AF6D1CB8B3B883AA3856AF08424AD8913BDE1207A5C2E010E23274ECCA7BC44814561E236325F0C4D2FFE069D2F9B  
> F5B40411A3D26D0BB864EC87713391FAE1197ACFD402D6D386B035BF04995C280989B7305964F179E149067FA864F66FA856EFBD3C8A005F92F1D7C93D82BC8C5B4804DC54154F34E5820B2A481C3F33EEDDB4323C142359920F60686D95E715E30894676AEDE  
> 6933A31092E98F979157D267D580D056C3EDDD31934E762D7354A2E1959AAD44D1DDAF094B6E8F38C5450C6A164BD40AF2C3FB2E44BD8E0F914C7BA0FBA4FA553A2A1291BAF9E87FC528C2D53A6B3D3EFEA88F5C359119A31018AD879124BE62B4A4BCFEF98FA  
> 5EE1BB5675D9828D1B246A10841CF1F6D1CC1C13C3C11C7E37BCEF93E933395A1FA4D7AEA4CB79DC3ED1EC162C926CFE1157366100A3AF788CF3262DC381B620FB68913694FD332D7D555CD60E3AC54E2F0A5FC2C90530A27721664A64B008FAB0A6","appkey  
> ":"1d8b6e7d45233436","autoplay_card":"11","build":"6180500","c_locale":"zh_CN","channel":"shenma069","column":"2","device_name":"MI 6","device_type":"0","flush":"8","fnval":"400","fnver":"0","force_host":"  
> 0","fourk":"1","guidance":"0","https_url_req":"0","idx":"1682561936","inline_danmu":"2","inline_sound":"1","login_event":"0","mobi_app":"android","network":"wifi","open_event":"","platform":"android","play  
> er_net":"1","pull":"false","qn":"32","recsys_mode":"0","s_locale":"zh_CN","splash_id":"","statistics":"{\"appId\":1,\"platform\":3,\"version\":\"6.18.0\",\"abtest\":\"\"}"}

我上面的流程只是为了完整的展示，拿到 app，定位加密，然后开始分析的，看过我前面的文章的应该都知道我在干嘛。

调试
--

ok，那就开始 so 层的调试和分析了

### 1. 找 so 文件

apk 我们知道，但是 so 文件，在这里，乍一看啥都没有：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427105240944-1201744532.png)

根据我们之前的定律，加载 so 文件都是 static 方法，然后，system.loadlibrary，但是这里就奇怪了，啥都没有 

只看到这里有点可疑：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427105347288-838859662.png)

 点进去，知道 G 是 bili

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427105354514-664391994.png)

c.c 会不会是 system.loadlibrary 呢？追进去：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427105457722-1440520563.png)

看到使用 str 只有 f，其他的逻辑我们可以不用管

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427105529150-812305510.png)

 f 追进去，我们知道，后面两个参数是 null，因为根本就没传这两个，那就是这个 g 方法

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427105604228-840561127.png)

 g 追进去，然后一下就看到了这个 loadlibrary:

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427105650342-1778586608.png)

再追进去，感觉这个【com.getkeepsafe.relinker】类有点奇怪

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427105810337-1317351535.png)

网上搜一下即可： 

> [KeepSafe/ReLinker：一个强大的 Android 原生库加载器。 (github.com)](https://github.com/KeepSafe/ReLinker)

看到这个介绍，ok，就是这里了。

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427105955094-55312804.png)

所以我们的 so 文件就是这个 bili 了。

### 2. 搭架子

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427110148565-2053679286.png)

又报了这个错：

>  [main]W/libc: pthread_create failed: clone failed: Out of memory

前面我们已经用过了：

在这加一行这个

> ```
> emulator.getSyscallHandler().setEnableThreadDispatcher(true);
> ```

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427110325906-948832774.png)

ok 了

而且，我们的目标函数地址也打印出来了：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427111426468-742427054.png)

### 3. 开始调用 s 方法

但是参数是 TreeMap，根据龙哥的博客说的，需要搞个继承关系，map->abstractMap->TreeMap

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427121601072-1702311287.png)

补一下：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427121630275-141988234.png)

继续补：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427121858212-1637961432.png)

插一句，如果这里，你没给 ts，会报如下错，而且提示都没有，就很骚，我是用 frida hook 到的某一个流程的参数，刚好就没有 ts。但是实际调用的时候是有 ts 的：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427163601233-114967644.png)

 回到刚才的地方，卧槽，他这里要一个 SignedQuery 对象的 r 方法，也就是这个：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427121951887-238073802.png)

新建了一个 util 类，然后把 jadx 的 copy 过来，导入好已有的，然后如下：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427122344994-296242505.png)

把 ContainerUtils 的这两个属性拿过来：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427122424556-1731864279.png)

现在就差 b 方法了

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427122542267-1563306628.png)

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427122610613-991755714.png)

把 a,b,c 都 copy 完，还有这个：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427122820144-482568515.png)

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427122833496-1266989975.png)

 整过来

现在还有 cv.m 了，

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427122959452-1246779782.png)

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427122952520-1126970745.png)

ok，整好了运行，卧槽，还要这个初始化对象：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427151140812-231097194.png)

再看看这个类的构造方法是啥：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427152036630-2036296835.png)

也还好，就两个属性

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427152717353-1500169974.png)

好了终于没有报错了，发现，这个第二个参数就是 sign：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427152811079-2038651479.png)

再看这个返回结果 ，好像有点不对劲，以前的不都是直接把结果字符串打印出来了吗，这里咋没打印，

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427153203638-570595987.png)

 根据前面的跟栈我们知道，他会调用下 toString，那么，这里我们也把 toString 补上去看看：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427153613721-1777943104.png)

牛逼，终于出来了。

完整代码

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

```
package com.danmaku;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.android.dvm.array.ByteArray;
import com.github.unidbg.memory.Memory;

import java.io.File;
import java.io.UnsupportedEncodingException;
import java.util.ArrayList;
import java.util.List;
import java.util.TreeMap;

public class bili extends AbstractJni {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;

    bili() {
        // 创建模拟器实例,进程名建议依照实际进程名填写，可以规避针对进程名的校验
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.bilibili.app").build();
        // 获取模拟器的内存操作接口
        final Memory memory = emulator.getMemory();
        emulator.getSyscallHandler().setEnableThreadDispatcher(true);
        // 设置系统类库解析
        memory.setLibraryResolver(new AndroidResolver(23));
        // 创建Android虚拟机,传入APK，Unidbg可以替我们做部分签名校验的工作
        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\danmaku\\bilibili.apk"));
        // 加载目标SO
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\danmaku\\libbili.so"), true); // 加载so到虚拟内存
        //获取本SO模块的句柄,后续需要用它
        module = dm.getModule();
        vm.setJni(this); // 设置JNI
        vm.setVerbose(true); // 打印日志

        dm.callJNI_OnLoad(emulator); // 调用JNI OnLoad
    };

    public static void main(String[] args) {
        bili test = new bili();
        System.out.println(test.getSign());
    }
    @Override
    public boolean callBooleanMethod(BaseVM vm, DvmObject<?> dvmObject, String signature, VarArg varArg) {
        if ("java/util/Map->isEmpty()Z".equals(signature)) {
            TreeMap<String, String> treeMap = (TreeMap<String, String>)dvmObject.getValue();
            return treeMap.isEmpty();
        }

        return super.callBooleanMethod(vm, dvmObject, signature, varArg);
    }
    @Override
    public DvmObject<?> callObjectMethod(BaseVM vm, DvmObject<?> dvmObject, String signature, VarArg varArg) {
        switch (signature) {
            case "java/util/Map->get(Ljava/lang/Object;)Ljava/lang/Object;":
                StringObject keyobject = varArg.getObjectArg(0);
                String key = keyobject.getValue();
                TreeMap<String, String> treeMap = (TreeMap<String, String>)dvmObject.getValue();
                String value = treeMap.get(key);
                return new StringObject(vm, value);
        }
        return super.callObjectMethod(vm, dvmObject, signature, varArg);
    }
    @Override
    public DvmObject<?> callStaticObjectMethod(BaseVM vm, DvmClass dvmClass, String signature, VarArg varArg) {
        switch (signature){
            case "com/bilibili/nativelibrary/SignedQuery->r(Ljava/util/Map;)Ljava/lang/String;":{
                DvmObject<?> mapObject = varArg.getObjectArg(0);
                TreeMap<String, String> mymap = (TreeMap<String, String>) mapObject.getValue();
                String result = util.r(mymap);
                return new StringObject(vm, result);
            }
        }
        return super.callStaticObjectMethod(vm, dvmClass, signature, varArg);
    }


    @Override
    public DvmObject<?> newObject(BaseVM vm, DvmClass dvmClass, String signature, VarArg varArg) {
        switch (signature) {
            case "com/bilibili/nativelibrary/SignedQuery-><init>(Ljava/lang/String;Ljava/lang/String;)V":
                StringObject stringObject1 = varArg.getObjectArg(0);
                StringObject stringObject2 = varArg.getObjectArg(1);
                String str1 = stringObject1.getValue();
                String str2 = stringObject2.getValue();
                return vm.resolveClass("com/bilibili/nativelibrary/SignedQuery").newObject(new SignedQuery(str1, str2));
        }
        return super.newObject(vm, dvmClass, signature, varArg);
    }

    public class SignedQuery {
        public final String a;
        public final String b;

        public SignedQuery(String str, String str2) {
            this.a = str;
            this.b = str2;
        }
        public String toString() {
            String str = this.a;
            if (str == null) {
                return "";
            }
            if (this.b == null) {
                return str;
            }
            return this.a + "&sign=" + this.b;
        }

    };

    public String getSign(){
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv());
        list.add(0);
        // 这里传入treemap
        TreeMap<String, String> keymap = new TreeMap<String, String>();
        keymap.put("ad_extra", "E0DA3BAB712CAB3902D558CFD4FEB865AB1563B7D61A4A57D02854BB9A4512C5E4737CA15664C505F3E63B954BB68837690B81FE77B2E7EB578F3E3D7C528439725348FE86527D7415BA2267FCCD2C95895AE80E69A960812FE60BC477171DAD34472EC31D010940A2876CADE887EA26FD5FBFB718D09F193D14732E54E42EB329EBF61CA33DE876AF24C03A7B9E89DB9C831AF6D1CB8B3B883AA3856AF08424AD8913BDE1207A5C2E010E23274ECCA7BC44814561E236325F0C4D2FFE069D2F9BF5B40411A3D26D0BB864EC87713391FAE1197ACFD402D6D386B035BF04995C280989B7305964F179E149067FA864F66FA856EFBD3C8A005F92F1D7C93D82BC8C5B4804DC54154F34E5820B2A481C3F33EEDDB4323C142359920F60686D95E715E30894676AEDE6933A31092E98F979157D267D580D056C3EDDD31934E762D7354A2E1959AAD44D1DDAF094B6E8F38C5450C6A164BD40AF2C3FB2E44BD8E0F914C7BA0FBA4FA553A2A1291BAF9E87FC528C2D53A6B3D3EFEA88F5C359119A31018AD879124BE62B4A4BCFEF98FA5EE1BB5675D9828D1B246A10841CF1F6D1CC1C13C3C11C7E37BCEF93E933395A1FA4D7AEA4CB79DC3ED1EC162C926CFE1157366100A3AF788CF3262DC381B620FB68913694FD332D7D555CD60E3AC54E2F0A5FC2C90530A27721664A64B008FAB0A6");
        keymap.put("appkey", "1d8b6e7d45233436");
        keymap.put("autoplay_card", "11");
        keymap.put("build", "6180500");
        keymap.put("c_locale", "zh_CN");
        keymap.put("channel", "shenma069");
        keymap.put("column", "2");
        keymap.put("device_name", "MI 6");
        keymap.put("device_type", "0");
        keymap.put("flush", "8");
        keymap.put("fnval", "400");
        keymap.put("fnver", "0");
        keymap.put("force_host", "0");
        keymap.put("fourk", "1");
        keymap.put("guidance", "0");
        keymap.put("https_url_req", "0");
        keymap.put("idx", "1682561936");
        keymap.put("inline_danmu", "2");
        keymap.put("inline_sound", "1");
        keymap.put("login_event", "0");
        keymap.put("mobi_app", "android");
        keymap.put("network", "wifi");
        keymap.put("open_event", "");
        keymap.put("platform", "android");
        keymap.put("player_net", "1");
        keymap.put("pull", "false");
        keymap.put("qn", "32");
        keymap.put("recsys_mode", "0");
        keymap.put("s_locale", "zh_CN");
        keymap.put("splash_id", "");
        keymap.put("statistics", "{\"appId\":1,\"platform\":3,\"version\":\"6.18.0\",\"abtest\":\"\"}");
        keymap.put("ts", "1612693177");
        DvmClass Map = vm.resolveClass("java/util/Map");
        DvmClass AbstractMap = vm.resolveClass("java/util/AbstractMap",Map);
        DvmObject<?> input_map = vm.resolveClass("java/util/TreeMap", AbstractMap).newObject(keymap);
        list.add(vm.addLocalObject(input_map));
        Number number =  module.callFunction(emulator,0x1c97,list.toArray());
        String result = vm.getObject(number.intValue()).getValue().toString();
        return result;
    }
}
```

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

util:

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

```
package com.danmaku;

import java.io.UnsupportedEncodingException;
import java.util.Map;
import java.util.SortedMap;
import java.util.TreeMap;

public class util {
    private static final char[] f14884c = "0123456789ABCDEF".toCharArray();
    private static boolean a(char c2, String str) {
        return (c2 >= 'A' && c2 <= 'Z') || (c2 >= 'a' && c2 <= 'z') || !((c2 < '0' || c2 > '9') && "-_.~".indexOf(c2) == -1 && (str == null || str.indexOf(c2) == -1));
    }

    static String b(String str) {
        return c(str, null);
    }

    static String c(String str, String str2) {
        StringBuilder sb = null;
        if (str == null) {
            return null;
        }
        int length = str.length();
        int i2 = 0;
        while (i2 < length) {
            int i3 = i2;
            while (i3 < length && a(str.charAt(i3), str2)) {
                i3++;
            }
            if (i3 != length) {
                if (sb == null) {
                    sb = new StringBuilder();
                }
                if (i3 > i2) {
                    sb.append((CharSequence) str, i2, i3);
                }
                i2 = i3 + 1;
                while (i2 < length && !a(str.charAt(i2), str2)) {
                    i2++;
                }
                try {
                    byte[] bytes = str.substring(i3, i2).getBytes("UTF-8");
                    int length2 = bytes.length;
                    for (int i4 = 0; i4 < length2; i4++) {
                        sb.append('%');
                        sb.append(f14884c[(bytes[i4] & 240) >> 4]);
                        sb.append(f14884c[bytes[i4] & 15]);
                    }
                } catch (UnsupportedEncodingException e) {
                    throw new AssertionError(e);
                }
            } else if (i2 == 0) {
                return str;
            } else {
                sb.append((CharSequence) str, i2, length);
                return sb.toString();
            }
        }
        return sb == null ? str : sb.toString();
    }


    static String r(Map<String, String> map) {
        String str;
        if (!(map instanceof SortedMap)) {
            map = new TreeMap<>(map);
        }
        StringBuilder sb = new StringBuilder(256);
        for (Map.Entry<String, String> entry : map.entrySet()) {
            String key = entry.getKey();
            if (!key.isEmpty()) {
                sb.append(b(key));
                sb.append("=");
                String value = entry.getValue();
                if (value == null) {
                    str = "";
                } else {
                    str = b(value);
                }
                sb.append(str);
                sb.append("&");
            }
        }
        int length = sb.length();
        if (length > 0) {
            sb.deleteCharAt(length - 1);
        }
        if (length == 0) {
            return null;
        }
        return sb.toString();
    }

}
```

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

欧克。

### 4. 直接补类

 根据龙哥的博客，说还有另一种补环境的方式，重建类：

 不再继承 abstrctjni 和不再用 setjni 方法，改用 vm.setDvmClassFactory(new ProxyClassFactory());

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427160237123-1445633879.png)

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

```
package com.danmaku;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.android.dvm.jni.ProxyClassFactory;
import com.github.unidbg.memory.Memory;

import java.io.File;
import java.util.ArrayList;
import java.util.List;
import java.util.TreeMap;

public class bili2 {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;

    bili2() {
        // 创建模拟器实例,进程名建议依照实际进程名填写，可以规避针对进程名的校验
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.bilibili.app").build();
        // 获取模拟器的内存操作接口
        final Memory memory = emulator.getMemory();
        emulator.getSyscallHandler().setEnableThreadDispatcher(true);
        // 设置系统类库解析
        memory.setLibraryResolver(new AndroidResolver(23));
        // 创建Android虚拟机,传入APK，Unidbg可以替我们做部分签名校验的工作
        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\danmaku\\bilibili.apk"));
        // 加载目标SO
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\danmaku\\libbili.so"), true); // 加载so到虚拟内存
        //获取本SO模块的句柄,后续需要用它
        module = dm.getModule();
//        vm.setJni(this); // 设置JNI
        vm.setDvmClassFactory(new ProxyClassFactory());
        vm.setVerbose(true); // 打印日志

        dm.callJNI_OnLoad(emulator); // 调用JNI OnLoad
    };

    public static void main(String[] args) {
        bili2 test = new bili2();
        System.out.println(test.getSign());
    }


    public String getSign(){
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv());
        list.add(0);
        // 这里传入treemap
        TreeMap<String, String> keymap = new TreeMap<String, String>();
        keymap.put("ad_extra", "E0DA3BAB712CAB3902D558CFD4FEB865AB1563B7D61A4A57D02854BB9A4512C5E4737CA15664C505F3E63B954BB68837690B81FE77B2E7EB578F3E3D7C528439725348FE86527D7415BA2267FCCD2C95895AE80E69A960812FE60BC477171DAD34472EC31D010940A2876CADE887EA26FD5FBFB718D09F193D14732E54E42EB329EBF61CA33DE876AF24C03A7B9E89DB9C831AF6D1CB8B3B883AA3856AF08424AD8913BDE1207A5C2E010E23274ECCA7BC44814561E236325F0C4D2FFE069D2F9BF5B40411A3D26D0BB864EC87713391FAE1197ACFD402D6D386B035BF04995C280989B7305964F179E149067FA864F66FA856EFBD3C8A005F92F1D7C93D82BC8C5B4804DC54154F34E5820B2A481C3F33EEDDB4323C142359920F60686D95E715E30894676AEDE6933A31092E98F979157D267D580D056C3EDDD31934E762D7354A2E1959AAD44D1DDAF094B6E8F38C5450C6A164BD40AF2C3FB2E44BD8E0F914C7BA0FBA4FA553A2A1291BAF9E87FC528C2D53A6B3D3EFEA88F5C359119A31018AD879124BE62B4A4BCFEF98FA5EE1BB5675D9828D1B246A10841CF1F6D1CC1C13C3C11C7E37BCEF93E933395A1FA4D7AEA4CB79DC3ED1EC162C926CFE1157366100A3AF788CF3262DC381B620FB68913694FD332D7D555CD60E3AC54E2F0A5FC2C90530A27721664A64B008FAB0A6");
        keymap.put("appkey", "1d8b6e7d45233436");
        keymap.put("autoplay_card", "11");
        keymap.put("build", "6180500");
        keymap.put("c_locale", "zh_CN");
        keymap.put("channel", "shenma069");
        keymap.put("column", "2");
        keymap.put("device_name", "MI 6");
        keymap.put("device_type", "0");
        keymap.put("flush", "8");
        keymap.put("fnval", "400");
        keymap.put("fnver", "0");
        keymap.put("force_host", "0");
        keymap.put("fourk", "1");
        keymap.put("guidance", "0");
        keymap.put("https_url_req", "0");
        keymap.put("idx", "1682561936");
        keymap.put("inline_danmu", "2");
        keymap.put("inline_sound", "1");
        keymap.put("login_event", "0");
        keymap.put("mobi_app", "android");
        keymap.put("network", "wifi");
        keymap.put("open_event", "");
        keymap.put("platform", "android");
        keymap.put("player_net", "1");
        keymap.put("pull", "false");
        keymap.put("qn", "32");
        keymap.put("recsys_mode", "0");
        keymap.put("s_locale", "zh_CN");
        keymap.put("splash_id", "");
        keymap.put("statistics", "{\"appId\":1,\"platform\":3,\"version\":\"6.18.0\",\"abtest\":\"\"}");
        keymap.put("ts", "1612693177");
        DvmClass Map = vm.resolveClass("java/util/Map");
        DvmClass AbstractMap = vm.resolveClass("java/util/AbstractMap",Map);
        DvmObject<?> input_map = vm.resolveClass("java/util/TreeMap", AbstractMap).newObject(keymap);
        list.add(vm.addLocalObject(input_map));
        Number number =  module.callFunction(emulator,0x1c97,list.toArray());
        String result = vm.getObject(number.intValue()).getValue().toString();
        return result;
    }
}
```

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

直接运行：

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427160708816-341996469.png)

把需要的类全都放进去，SignedQuery、ContainerUtils、cv

记得把 signedQuery 放到他该有的类，其他就随意了，他会自己在同级目录查找，ok，直接出结果。这里龙哥把 ContainerUtils、cv 类都单独创建了该有的类名前缀，我这里就没有了，一样出结果，感觉还是归类在一起好点

![](https://img2023.cnblogs.com/blog/1249183/202304/1249183-20230427162328704-966976815.png)

这样的效果就是，不需要去手动补需要的环境了（因为需要的环境类整个都有了）

知识点总结
-----

1.treemap 需要注意下，在 unidbg 里需要继承 map，abstractmap

2.varArg.getObjectArg(0) 可以获取该方法的参数，0，指第一个，1 指第二个

> StringObject stringObject1 = varArg.getObjectArg(0);
> 
> StringObject stringObject2 = varArg.getObjectArg(1);
> 
> String str1 = stringObject1.getValue();
> 
> String str2 = stringObject2.getValue();

3.unidbg 有新的写法，不继承 abstractjni，setjni 的地方改成 vm.setDvmClassFactory(new ProxyClassFactory());

这种方法适合 java 环境太多的情况，不用一个一个手动补，直接用 java 用到的类
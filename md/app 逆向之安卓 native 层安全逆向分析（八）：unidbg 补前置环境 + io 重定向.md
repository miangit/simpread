> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/Eeyhan/p/17385949.html)

前言
--

继续跟着龙哥的 unidbg 学习：[SO 逆向入门实战教程七：main_unidbg 重定向_白龙~ 的博客 - CSDN 博客](https://blog.csdn.net/qq_38851536/article/details/118000259?spm=1001.2014.3001.5502)[  
](https://blog.csdn.net/qq_38851536/article/details/117923970)

还是那句，我会借鉴龙哥的文章，以一个初学者的角度，加上自己的理解，把内容丰富一下，尽量做到不在龙哥的基础上画蛇添足。感谢观看的朋友。

分析
--

首先，抓个包

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230509102514774-727983530.png)

里面这个 mtgsig 就是该 app 很经典的加密参数了，siua 参数后续有时间就分析，没有就算了。本篇文章的重点是 mtgsig.

### 1. 静态分析

jadx 分析，一搜：

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230509103139529-1555732834.png)

发现其实并不多。

就拿这几个类进行 hook，看看哪些方法被调用了就行了，没花多久时间，就找到这里：

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230509170940099-1997677747.png)

>  com.xxxxx.plugin.sign.core.CandyPreprocessor.makeHeader(Uri$Builder) String

再跟一下：

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230509175632751-1803699391.png)

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230509175640957-891215101.png)

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230509175654449-798218289.png)

这个 main 就是我们要的部位了。我们知道，第一个 int 参数，其实都写死了，就是 203

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230511172316611-25519376.png)

第二个是一个对象数组

 hook 下这两个方法，main 的地方参数中间有个 byte[] 没有打印出来

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230511174437913-1557698282.png)

### 2.frida 动态调试 

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230511183612629-501375311.png)

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230511183647820-1041627874.png)

所以这个 byte[] 就是个 请求方式  url 的 path  子路径

好了，入参大概都知道了 

ida 调试
------

调试之前，得找找他加载的是哪个 so 文件

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230516122657884-167335402.png)

这静态看代码，看了半天，愣是没看出来哪里用了 system.loadlibrary。

用这个 grep 找到的是这个 so，其实并不是

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230516122747205-2130397674.png)

这里继续用 yang 神的 hook_RegisterNative.js，一顿输出，直接找到这个 so:

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230516122537792-224599591.png)

很 nice。

卧槽，跑完直接把我手机干关机了。有点 6

接下来打开 ida 看看，导出表都不用看了

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230516140904028-998576415.png)

 方法也看不到啊，上面用 hook_RegisterNative 已经 hook 到位置了，直接定位过去就行

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230516160155868-209718271.png)

 结果就是解析异常了。

unidbg 调试
---------

那直接 unidbg 调试吧

### 1. 搭架子

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230516160818955-125717270.png)

跑起来，好像没啥问题 ，main 也有了。

### 2. 调用

那么直接调用吧，首先把之前 hook 到的数据，传递进去

上面需要的都有 hook 了，

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230516163519104-26094827.png)

现在就差这个 key，往上找找：

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230516163625278-574625184.png)

hook 下，等会儿，还 hook 啥呀，前面已经有了，就是这个 9b69xxxx，思路不能乱，慢慢来

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230516180931663-1815713900.png)

开始调用，结果如下：

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230516181855371-468240218.png)

尴尬，报错的意思是这个指针结果是一个空指针。根据龙哥的博客，应该是上下文缺失。jnitrace 也可以发现。

### 3. 解决问题

再来看看，这个 main 的调用实例

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517100908396-2134634676.png)

有这么多

用 fridahook 下这些

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517101818123-1637080514.png)

发现，果然是了，刚开始调用，还是 null，然后再次调用，就有值了。

再来，把入参打印一下：

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517102043010-219988355.png)

看来第一次是 111，也就是代码的这里：

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517102111984-196854784.png)

 这个好像还算简单。

补一下这里的前置条件，然后执行：

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517104040683-1727544056.png)

补下这个环境，但是这个是啥呢，直接用 firda hook 可知如下：

 因为他是 static

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517104355635-1424958916.png)

那么直接调用

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517104019505-904497078.png)

补完执行：

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517104337795-657271532.png)

又来一个新的，同样操作

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517104428680-1820708423.png)

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517104504781-2001880168.png)

还有报错，

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517104608737-763713859.png)

继续

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517104736953-256815685.png)

卧槽，还有报错，而且这个报错，龙哥的博客好像没有这个，我手机是米 10，安卓 12，估计有啥不一样

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517105214450-1719191997.png)

问题不大，他既然要返回一个 i，也就是整形，那直接这么补，然后执行：

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517105445859-1293114381.png)

说白了这个就是要获取当前的进程 apk 的路径，继续补

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517110555929-783204542.png)

没啥可说的，继续补：

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517110541796-843072707.png)

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517110703013-929805863.png)

继续

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517110856993-1072653540.png)

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517110954997-337708174.png)

继续

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517111520347-694419156.png)

好了，终于不报错了，但是结果并没有被打印出来

仔细看这里，根据龙哥的博客，意思是这里做了文件读取操作，但是 - 100，好像读取异常了

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517112309048-450698633.png)

这时候有两种方案解决这个

*   到 unidbg 目录，把 apk 放进去，用这个文件 src/main/java/com/github/unidbg/file/BaseFileSystem.java 获取当前程序路径
*   还可以用代码的方式进行操作，设置 io 重定向，把访问的路径放到我们指定的路径即可

这里我用重定向的方式：

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517114824676-869763544.png)

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517114844106-919496207.png)

然后执行：

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517114904234-1712667963.png)

继续补

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517115144507-2092981894.png)

但这里出现了问题，这里是一个空，尴尬，查看用例：

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517115125531-736737159.png)

就这里做了赋值

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517115225321-1341969022.png)

想 hook 一下，结果 hook 不到，尴尬，但是那边确实看到是空，龙哥的博客是有数据的，不重要，遵从内心，补一个空，执行看看：

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517121050334-634371139.png)

唉，结果有了，好像结果数据有点不对，最后是一个 dvmobject，那就跟前面的文章一样了，转一下，打印下

这两个方式都可以：

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517122815547-1998018610.png)

这个结果能不能用呢？

重新抓个包，然后替换下试试：

![](https://img2023.cnblogs.com/blog/1249183/202305/1249183-20230517140844990-2100931452.png)

ok，果然可以。很 nice

代码

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

```
int input = vaList.getIntArg(0);
return vm.resolveClass("java/lang/Integer").newObject(input);
```

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

知识点总结
-----

1. 找不到该 native 方法引入的哪个 so 文件时，用 hook_RegisterNative 脚本分析

2. 执行结果发现不对返回空指针时，很大可能是逻辑问题，需要结合 jnitrace 分析

3. 报 dirfd=-100，就是读取文件错误，需要指导程序到正确的路径

4. 补文件读取有两个方法

*   到 unidbg 目录，把 apk 放进去，用这个文件 src/main/java/com/github/unidbg/file/BaseFileSystem.java 获取当前程序路径
*   还可以用代码的方式进行操作，设置 io 重定向，把访问的路径放到我们指定的路径即可

5. 补环境的时候 getIntArg 是获取方法的参数

我的代码是：

```
int input = vaList.getInt(0);
return new DvmInteger(vm, input);
```

龙哥的代码：

```
int input = vaList.getInt(0);
return new DvmInteger(vm, input);
```

6. 如果最后的数据是一个数组而不是字符串，有两种方式获取

```
StringObject result = (StringObject) ((DvmObject[])((ArrayObject)vm.getObject(number.intValue())).getValue())[0];
System.out.println(result.getValue());
```

```
DvmObject[] result1 = (DvmObject[]) vm.getObject(number.intValue()).getValue();
String sign = result1[0].toString();
System.out.println(sign);
```
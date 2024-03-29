> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/qq_38851536/article/details/118178411)

#### 一、问题提出

在《SO 逆向入门实战教程七：main》我们讨论了一个样本，一位读者朋友给我提出了问题：有另一个同系的应用，也使用了样本 SO，但是在其初始化函数中，流程略有不同，并会最终导致 Unidbg 在一个函数中死循环。

样本和代码详见百度网盘

#### 二、问题解决

我花了数个小时分析样本，依然找不到问题根源，但在无意的一次测试中发现，当使用 HookZz 对死循环发生的函数中某些位置 inline hook 时（无需修改任何内容），死循环就不复存在并恢复到正常执行流，并能顺利模拟执行。

样本对 Unidbg 的检测吗？我认为概率不大。inline hook 在此事中发挥了怎样的作用？Unidbg 有 bug？我现在也是一头雾水喽。欢迎大家和我讨论交流此样本或工作生活中遇到的各类样本，但请专注研究，不要做坏事噢。

#### 三、尾声

链接：https://github.com/miangit/simpread/releases/download/backup/mtyx.7z

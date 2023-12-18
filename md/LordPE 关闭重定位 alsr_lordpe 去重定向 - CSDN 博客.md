> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/lhk124/article/details/107542647)

前言：alsr 多多少少影响逆向时对程序的分析。关闭它。

两种情况：

1. 操作系统开启了 alsr。关闭方法文章如下：

> win10 参考文章：[https://www.52pojie.cn/thread-1099755-1-1.html](https://www.52pojie.cn/thread-1099755-1-1.html) 
> 
> win7 参考文章：[https://bbs.pediy.com/thread-258653.htm](https://bbs.pediy.com/thread-258653.htm)2

2. 程序保护关闭方法

> 1.  爱盘下载 LordPE
> 2.  ![](https://img-blog.csdnimg.cn/20200723172602152.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xoazEyNA==,size_16,color_FFFFFF,t_70)
> 3.  ![](https://img-blog.csdnimg.cn/2020072317274377.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xoazEyNA==,size_16,color_FFFFFF,t_70)
> 4.  ![](https://img-blog.csdnimg.cn/20200723172909428.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xoazEyNA==,size_16,color_FFFFFF,t_70)
> 5.  ![](https://img-blog.csdnimg.cn/20200723173011704.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xoazEyNA==,size_16,color_FFFFFF,t_70)
> 6.  ![](https://img-blog.csdnimg.cn/20200723173208328.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xoazEyNA==,size_16,color_FFFFFF,t_70)

重新用 OD 载入程序后，可以看见不会再变化了。（程序其他程序也占用的话，不能保存）
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-279440.htm)

> [原创] 某手游完整性校验分析

[原创] 某手游完整性校验分析

2023-11-2 23:16 9313

[举报](javascript:void(0);)

### [原创] 某手游完整性校验分析

 [![](https://bbs.kanxue.com/view/img/avatar.png)](user-home-953698.htm) [wx_嗨](user-home-953698.htm) ![](https://bbs.kanxue.com/view/img/rank/7.png)  ![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) [ 举报](javascript:void(0);) 2023-11-2 23:16  9313

前言
--

只是普通的单机手游，广告比较多，所以分析处理了下，校验流程蛮有意思的，所以就分享出来了

1. 重打包崩溃处理
----------

样本进行了加固，对其 dump 出 dex 后重打包出现崩溃

![](https://bbs.kanxue.com/upload/attach/202311/953698_NKYB9D4PCBB6DUE.png)

ida 分析地址发现为 jni 函数引起

![](https://bbs.kanxue.com/upload/attach/202311/953698_39CAXT3R593WARQ.webp)  
![](https://bbs.kanxue.com/upload/attach/202311/953698_UNCQUWBCRBN9CKP.webp)  
利用 Xposed 直接替换该函数，崩溃问题解决

```
XposedHelpers.findAndHookMethod("com.unity3d.player.UnityPlayerActivity", classLoader, "IsHDR_DisplayBoot", java.lang.String.class, java.lang.String.class, new XC_MethodReplacement() {
    @Override
    protected Object replaceHookedMethod(MethodHookParam methodHookParam) throws Throwable {
        return null;
    }
});
```

2. 卡加载界面处理
----------

但是出现了新的问题，游戏卡在了加载页面

![](https://bbs.kanxue.com/upload/attach/202311/953698_VWGJFF9UQ37PMK3.webp)  
![](https://bbs.kanxue.com/upload/attach/202311/953698_VHKXD3WYQ4JEE9Z.webp)  
对该类的其他 jni 函数进行分析，发现 supportVulkan 很明显进行了签名读取，对其去除签名校验。

此处参考项目：**[ApkSignatureKillerEx](https://github.com/L-JINBIN/ApkSignatureKillerEx)**

3. 地图无法进入问题分析（基于原包）
-------------------

去除完签名校验后，游戏能正常进入主页面，但是点击游戏地图没有任何反应。

在对原包进行多次测试发现，在首次启动游戏时，如果断网也会出现无法正常进入地图的问题，怀疑是游戏首次启动进行了网络请求进行数据校验

经过抓包对比定位到可疑数据包

![](https://bbs.kanxue.com/upload/attach/202311/953698_HEW44MS2T8Z27XR.webp)

利用 Frida 的算法通杀脚本没有定位到相关内容，怀疑是 so 层进行了请求

#### 3.1 hook send 函数进行调用栈分析

![](https://bbs.kanxue.com/upload/attach/202311/953698_C3TRDEBT5TYHUXU.webp)

通过调用栈可以很清晰看到请求是从 unity 引擎相关的 so 中发出的。

#### 3.2 il2cppTrace 对请求调用栈分析

查阅资料，在 unity 中，网络请求主要通过`UnityWebRequest` 类来执行网络请求，利用 [frida-il2cpp-bridge](https://github.com/vfsfitvnm/frida-il2cpp-bridge) 对 UnityWebRequest 进行 trace，打印调用栈得

```
0x0247db9c ┌─UnityEngine.Networking.UnityWebRequest::.ctor(this = UnityEngine.Networking.UnityWebRequest, url = "http://106.54.194.167:8077/CheckUpdate", method = "POST")
0x0247db00 │ ┌─UnityEngine.Networking.UnityWebRequest::set_url(this = UnityEngine.Networking.UnityWebRequest, value = "http://106.54.194.167:8077/CheckUpdate")
0x0247ef38 │ │ ┌─UnityEngine.Networking.UnityWebRequest::InternalSetUrl(this = UnityEngine.Networking.UnityWebRequest, url = "http://106.54.194.167:8077/CheckUpdate")
0x0247ef38 │ │ └─UnityEngine.Networking.UnityWebRequest::InternalSetUrl
0x0247db00 │ └─UnityEngine.Networking.UnityWebRequest::set_url
0x0247dc1c │ ┌─UnityEngine.Networking.UnityWebRequest::set_method(this = UnityEngine.Networking.UnityWebRequest, value = "POST")
0x0247e4e0 │ │ ┌─UnityEngine.Networking.UnityWebRequest::InternalSetMethod(this = UnityEngine.Networking.UnityWebRequest, methodType = Post)
0x0247e4e0 │ │ └─UnityEngine.Networking.UnityWebRequest::InternalSetMethod
0x0247dc1c │ └─UnityEngine.Networking.UnityWebRequest::set_method
0x0247db9c └─UnityEngine.Networking.UnityWebRequest::.ctor
```

在 IDA 中进行交叉引用分析定位到 UnitySDKManager 类

```
追踪调用定位到
class UnitySDK.UnitySDKManager.d__25 : System.Object, System.Collections.Generic.IEnumerator, System.Collections.IEnumerator, System.IDisposable
    private Boolean MoveNext() { } 
```

增加 UnitySDKManager 类重新对其 trace（因为出现报错，把参数输出关了）

```
0x00b15b44 ┌─UnitySDK.UnitySDKManager::ServerVerifyApk
0x00b04b3c │ ┌─UnitySDK.UnitySDKManager::GetFileInfoList
0x00b04b3c │ └─UnitySDK.UnitySDKManager::GetFileInfoList
0x00b06ec0 │ ┌─UnitySDK.UnitySDKManager::GetGamePackageName
0x00b06ec0 │ └─UnitySDK.UnitySDKManager::GetGamePackageName
0x00b06204 │ ┌─UnitySDK.UnitySDKManager::GetSDKVersion
0x00b06204 │ └─UnitySDK.UnitySDKManager::GetSDKVersion
0x00b06278 │ ┌─UnitySDK.UnitySDKManager::HexStringToHex
0x00b06278 │ └─UnitySDK.UnitySDKManager::HexStringToHex
0x00b10fcc │ ┌─UnitySDK.UnitySDKManager::EncryptString
0x00b10fcc │ └─UnitySDK.UnitySDKManager::EncryptString
0x00b100a4 │ ┌─UnitySDK.UnitySDKManager::PostData
0x00a341fc │ │ ┌─UnitySDK.UnitySDKManager.d__25::.ctor
0x00a341fc │ │ └─UnitySDK.UnitySDKManager.d__25::.ctor
0x00b100a4 │ └─UnitySDK.UnitySDKManager::PostData
0x00a3422c │ ┌─UnitySDK.UnitySDKManager.d__25::MoveNext
0x00b12be4 │ │ ┌─UnitySDK.UnitySDKManager::GetUrl
0x00b12be4 │ │ └─UnitySDK.UnitySDKManager::GetUrl
0x0247db9c │ │ ┌─UnityEngine.Networking.UnityWebRequest::.ctor
0x0247db00 │ │ │ ┌─UnityEngine.Networking.UnityWebRequest::set_url
0x0247ef38 │ │ │ │ ┌─UnityEngine.Networking.UnityWebRequest::InternalSetUrl
0x0247ef38 │ │ │ │ └─UnityEngine.Networking.UnityWebRequest::InternalSetUrl
0x0247db00 │ │ │ └─UnityEngine.Networking.UnityWebRequest::set_url
0x0247dc1c │ │ │ ┌─UnityEngine.Networking.UnityWebRequest::set_method
0x0247e4e0 │ │ │ │ ┌─UnityEngine.Networking.UnityWebRequest::InternalSetMethod
0x0247e4e0 │ │ │ │ └─UnityEngine.Networking.UnityWebRequest::InternalSetMethod
0x0247dc1c │ │ │ └─UnityEngine.Networking.UnityWebRequest::set_method
0x0247db9c │ │ └─UnityEngine.Networking.UnityWebRequest::.ctor
0x0247e0f4 │ │ ┌─UnityEngine.Networking.UnityWebRequest::set_uploadHandler
0x0247e0f4 │ │ └─UnityEngine.Networking.UnityWebRequest::set_uploadHandler
0x0247dfdc │ │ ┌─UnityEngine.Networking.UnityWebRequest::set_downloadHandler
0x0247dfdc │ │ └─UnityEngine.Networking.UnityWebRequest::set_downloadHandler
0x0247f66c │ │ ┌─UnityEngine.Networking.UnityWebRequest::SetRequestHeader
0x0247f66c │ │ └─UnityEngine.Networking.UnityWebRequest::SetRequestHeader
0x0247e42c │ │ ┌─UnityEngine.Networking.UnityWebRequest::Send
0x0247e430 │ │ │ ┌─UnityEngine.Networking.UnityWebRequest::SendWebRequest
0x0247e42c │ │ │ └─UnityEngine.Networking.UnityWebRequest::Send
0x0247e430 │ │ └─UnityEngine.Networking.UnityWebRequest::SendWebRequest
0x00a3422c │ └─UnitySDK.UnitySDKManager.d__25::MoveNext
0x00a34cd4 │ ┌─UnitySDK.UnitySDKManager.d__25::System.Collections.IEnumerator.get_Current
0x00a34cd4 │ └─UnitySDK.UnitySDKManager.d__25::System.Collections.IEnumerator.get_Current
0x00b15b44 └─UnitySDK.UnitySDKManager::ServerVerifyApk 
```

通过调用栈可以初步对校验流程进行了解，主要通过 ServerVerifyApk 函数进行校验，经过一系列字符串加解密，最后进行网络请求

#### 3.3 加解密函数 hook 分析

对关键函数进行 hook 分析

```
0x00b10fcc │ UnitySDK.UnitySDKManager::EncryptString
0x00b0b13c │ UnitySDK.UnitySDKManager::DecryptString
il2cpp: EncryptString:"{"appid":"com.meta.peopleground.dream","appName":"com.meta.peopleground.dream","appVer":"1.0.1","sdkVer":"2","appFileList":"981lDpbOb+vjRn/3yI74fI+1S5cr08qKLGq0xg7TcfFOA7hRePAQjXPCgIzEC+p58RThRtl4PIsPEgiJZVYn1e2oZl06jeKGD6+C0Zy6lQaDiAVaOtO5szilZxfmN2j8XtlBmBaOzOVBBi6ctJamGsfM2XbkGqYtf/TlNyvOZ8p5YDdOgnhG+WonkjG1sGEbuqc4sm+hkCxOC7dX1rEM4/5S6wT1erPL5+iWJoRpiTuHok15zCQjdNFCXn97Vwg+/h8eXrtxrDvkkLqqCff2D0WrAISmIOi4fY97ulhpz5Pey2vJlkLK5MdSApENx4exv3t4Q1sVfoPojwDueXAW9T1RTOc1LeK0cYMrBjsE1W8="}"
il2cpp: EncryptString result:"xoJRLdGOAd21Q0JEIDiJqlciJFWy0jtiSLU7lNue9ZLAOxsuEkWvJwP8o3ub4CfcIjulssJC+ocK0ai3lkr1XfNXlLi2QJ0iTtYzbCDQqYmqXr7uTTdOrAK3Y6NKkd+mYgFF+J2w7tY/rl5NBPPlwRWWbvw45CRp3JX34BkKTA2KEXiWywj90T3Qw72AnpWgBOo1rD3USuAVQeN8EOhxVc1jTjsBn2qSfZX38tNC73wvceexa4w5wOea9rzaRshViPWjFkNko/BDuaAyWDJbB656FNatES6/WA3+f7qMlQmEU4BGrX72+StLzsNQYc+QZ4KDFnWwqfhuH1zCudxxDoNpi53fqv+DavLwmRRE19e1clgW5ANilhelSIGi9U9nq2BcNr3GMJ8eI2FAV76U1FHONbyMUggUYr6cDc9Vo9l5Wc4UVD4+tqDmulUW1L1s8NTkieyVDEJxb1o6166fVVWdUKNBlrIXDfbWjE6gLheTHfp1/ywUy4EcVSklERsEkdufZrdKM57XVkXcKFnA0hutpuKqCbbkthULifwkAtwyQ/qSdcZHIh6pNYGuLOFF5NiCsHhJ6D/O03/uUgsR/V5WwRGuOeZFsM2NK+NcyYjHDkoKh6SjoEnZZ/yfg1M4kOUJy5qG/V1jR8IqOmZ84w=="
il2cpp: DecryptString:"+CVWCEji2Lgf4nUTWb7J/Q=="
il2cpp: DecryptString result:"100"
```

发现之前抓包获得的请求体相对应，其中，appFileList 还是密文，继续分析

#### 3.4 追 EncryptString 函数

![](https://bbs.kanxue.com/upload/attach/202311/953698_MB4QVDF84E5GEZW.webp)

与 dump.cs 中的函数进行对照发现 v14 由字符串转 hex 转 base64 获得

```
System.Byte[] HexStringToHex(System.String inputHex); // 0x00b06278
```

hook HexStringToHex 函数获得参数

```
il2cpp: inputHex:"f7cd650e96ce6febe3467ff7c88ef87c8fb54b972bd3ca8a2c6ab4c60ed371f14e03b85178f0108d73c2808cc40bea79f114e146d9783c8b0f120889655627d5eda8665d3a8de2860faf82d19cba95068388055a3ad3b9b338a56717e63768fc5ed94198168ecce541062e9cb496a61ac7ccd976e41aa62d7ff4e5372bce67ca7960374e827846f96a279231b5b0611bbaa738b26fa1902c4e0bb757d6b10ce3fe52eb04f57ab3cbe7e896268469893b87a24d79cc242374d1425e7f7b57083efe1f1e5ebb71ac3be490baaa09f7f60f45ab0084a620e8b87d8f7bba5869cf93decb6bc99642cae4c75202910dc787b1bf7b78435b157e83e88f00ee797016f53d514ce7352de2b471832b063b04d56f"
```

IDA 继续查找调用发现其中字符串由 GetFileInfoList 函数获得

```
// static System.IntPtr GetFileInfoList(); // 0x00b04b3c
__int64 sub_B04B3C()
{
  __int64 (*v0)(void); // x8
  __int64 v2[5]; // [xsp+0h] [xbp-40h] BYREF
  int v3; // [xsp+28h] [xbp-18h]
  char v4; // [xsp+2Ch] [xbp-14h]
 
  v0 = (__int64 (*)(void))qword_30365A0;
  if ( !qword_30365A0 )
  {
    v3 = 0;
    v2[0] = (__int64)"UnitySDK";
    v2[1] = 8LL;
    v2[2] = (__int64)"GetFileInfoList";
    v2[3] = 15LL;
    v2[4] = 0x200000000LL;
    v4 = 0;
    v0 = (__int64 (*)(void))sub_812FB4(v2);
    qword_30365A0 = (__int64)v0;
  }
  return v0();
}
```

很明显，调用了 libUnitySDK.so 中的 GetFileInfoList 函数获得

#### 3.5 libUnitySDK.so 分析

```
__int64 GetFileInfoList()
{
  return FileInfoListStr;
}
```

FileInfoListStr 在 writeFileJson 函数中被赋值，而 writeFileJson 则是由 java 函数上文中的 IsHDR_DisplayBoot 调用（该函数首个参数为 base.apk 路径）

![](https://bbs.kanxue.com/upload/attach/202311/953698_XESC7BKW8G25F93.webp)

很明显，ll11l1l1ll 函数 对之前的明文字符串进行了加密，frida hook 打印参数

```
args: {
        "HashList" :
        {
                "AndroidManifest.xml" : " c72b1d2",
                "assets/bin/Data/Managed/Metadata/game.dat" : "a34a757d",
                "classes.dex" : "58b6cf21",
                "lib/arm64-v8a/libUnitySDK.so" : "b56d5af4",
                "lib/armeabi-v7a/libUnitySDK.so" : "3d548ba2"
        },
        "fileCount" : 1620
}
 
returnResult: f7cd650e96ce6febe3467ff7c88ef87c8fb54b972bd3ca8a2c6ab4c60ed371f14e03b85178f0108d73c2808cc40bea79f114e146d9783c8b0f120889655627d5eda8665d3a8de2860faf82d19cba95068388055a3ad3b9b338a56717e63768fc5ed94198168ecce541062e9cb496a61ac7ccd976e41aa62d7ff4e5372bce67ca7960374e827846f96a279231b5b0611bbaa738b26fa1902c4e0bb757d6b10ce3fe52eb04f57ab3cbe7e896268469893b87a24d79cc242374d1425e7f7b57083efe1f1e5ebb71ac3be490baaa09f7f60f45ab0084a620e8b87d8f7bba5869cf93decb6bc99642cae4c75202910dc787b1bf7b78435b157e83e88f00ee797016f53d514ce7352de2b471832b063b04d56f
```

其中 HashList 中文件对应的值为文件的 crc  
至此，除了具体的字符串加密算法，游戏的校验流程已经很清晰

4. 校验流程归纳
---------

*   签名校验
*   读取 base.apk 进行关键文件 crc 读取
*   进行服务器请求，对关键文件 crc 校验

5. 后记
-----

对该校验去除的思路：

*   对读取安装包的函数 zip_open 或 java 层的 IsHDR_DisplayBoot 函数进行 hook，进行参数替换，进行 io 重定向
*   基于 il2cpp 中的 ServerVerifyApk 函数进行更详细的分析，直接对其检测去除

ps. 仅限于学习交流，切勿用于商业用途，若造成侵权，请联系作者处理。  
样本链接：[https://pan.baidu.com/s/1awWYCF2nT50qPRj8_CRt-A?pwd=d8wn](https://pan.baidu.com/s/1awWYCF2nT50qPRj8_CRt-A?pwd=d8wn)

  

[[培训]《安卓高级研修班 (网课)》月薪三万计划](https://www.kanxue.com/book-section_list-84.htm)

最后于 2023-11-2 23:33 被 wx_嗨编辑 ，原因： 编辑标题 [#逆向分析](forum-161-1-118.htm)
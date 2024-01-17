> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7019114480877436965)

**小知识，大挑战！本文正在参与 “**  **[程序员必备小知识](https://juejin.cn/post/7008476801634680869 "https://juejin.cn/post/7008476801634680869")**  **” 创作活动**

**本文同时参与** **[「掘力星计划」](https://juejin.cn/post/7012210233804079141 "https://juejin.cn/post/7012210233804079141")**  **，赢取创作大礼包，挑战创作激励金**

替换 Boolean, Color, Integer, int[], String and String[] 类型的简单资源
--------------------------------------------------------------

### 1. 替换系统框架（Android Framwork）资源

替换系统框架资源（对所有 app 起作用）需要实现 `IXposedHookZygoteInit`接口的 `initZygote` 方法，并在该方法中调用`Resources.setSystemWideReplacement(...)` 方法替换资源

```
package de.robv.android.xposed.mods.tutorial;

import android.content.res.XResources;
import de.robv.android.xposed.IXposedHookZygoteInit;

public class Tutorial2  implements IXposedHookZygoteInit{

    @Override
    public void initZygote(StartupParam arg0) throws Throwable {
        XResources.setSystemWideReplacement("android", "bool", "config_unplugTurnsOnScreen", false);
    }

}
```

### 2. 替换 app 应用资源

替换 app 应用资源需要实现 `IXposedHookInitPackageResources` 类的 `andleInitPackageResources`方法，并在该方法中调用`res.setReplacement(...)`方法替换资源，注意在该方法中不要使用`XResources.setSystemWideReplacement` 方法

```
package de.robv.android.xposed.mods.tutorial;

import android.content.res.XResources;
import android.graphics.Color;
import android.graphics.drawable.ColorDrawable;
import android.graphics.drawable.Drawable;
import de.robv.android.xposed.IXposedHookInitPackageResources;
import de.robv.android.xposed.callbacks.XC_InitPackageResources.InitPackageResourcesParam;

public class Tutorial3 implements  IXposedHookInitPackageResources  {

    @Override
    public void handleInitPackageResources(InitPackageResourcesParam resparam) throws Throwable {
        //只替换systemui应用的资源
        if (!resparam.packageName.equals("com.android.systemui"))
            return;

        // 替换资源的不同方式
        resparam.res.setReplacement(0x7f080083, "YEAH!"); // WLAN toggle text. You should not do this because the id is not fixed. Only for framework resources, you could use android.R.string.something
        resparam.res.setReplacement("com.android.systemui:string/quickpanel_bluetooth_text", "WOO!");
        resparam.res.setReplacement("com.android.systemui", "string", "quickpanel_gps_text", "HOO!");
        resparam.res.setReplacement("com.android.systemui", "integer", "config_maxLevelOfSignalStrengthIndicator", 6);
        resparam.res.setReplacement("com.android.systemui", "drawable", "status_bar_background", new XResources.DrawableLoader() {
            @Override
            public Drawable newDrawable(XResources res, int id) throws Throwable {
                return new ColorDrawable(Color.WHITE);
            }
        });//你不能直接使用Drawble类进行替换，因为Drawble类可以影响其他引用Ddrawble类实例的ImageView ,最好使用一个包装器。
    }

}
```

替换复杂的资源
-------

对于复制的资源，如动画资源 ，我们也能够替换，下面我们来替换 battery icon

动画资源布局

```
<?xml version="1.0" encoding="utf-8"?>
<animation-list xmlns:android="http://schemas.android.com/apk/res/android"
     android:oneshot="true" >
    <item android:drawable="@drawable/icon1" android:duration="150"></item>
    <item android:drawable="@drawable/icon2" android:duration="150"></item>
    <item android:drawable="@drawable/icon3" android:duration="150"></item>
    <item android:drawable="@drawable/icon4" android:duration="150"></item>
    <item android:drawable="@drawable/icon5" android:duration="150"></item>
    <item android:drawable="@drawable/icon6" android:duration="150"></item>

</animation-list>
```

代码：

```
package de.robv.android.xposed.mods.tutorial;

import com.example.xposedmoduletest.R;

import android.content.res.XModuleResources;
import de.robv.android.xposed.IXposedHookInitPackageResources;
import de.robv.android.xposed.IXposedHookZygoteInit;
import de.robv.android.xposed.callbacks.XC_InitPackageResources.InitPackageResourcesParam;

public class Tutorial4 implements IXposedHookZygoteInit, IXposedHookInitPackageResources {

    private static String MODULE_PATH = null;

    @Override
    public void initZygote(StartupParam startupParam) throws Throwable {
        MODULE_PATH = startupParam.modulePath;

    }

    @Override
    public void handleInitPackageResources(InitPackageResourcesParam resparam) throws Throwable {
        if (!resparam.packageName.equals("com.android.systemui"))
            return;

        XModuleResources modRes = XModuleResources.createInstance(MODULE_PATH, resparam.res);
        resparam.res.setReplacement("com.android.systemui", "drawable", "stat_sys_battery",
                modRes.fwd(R.drawable.animation));
        resparam.res.setReplacement("com.android.systemui", "drawable", "stat_sys_battery_charge",
                modRes.fwd(R.drawable.animation));
    }
}
```

Xposed 框架会将模块请求资源的请求指向你模块中的资源

替换布局
----

你可以用替换资源的方法来替换布局文件，但这样你不得不将目标 apk 中的整个 layout 文件复制出来进行修改，这样会使模块的 Rom 兼容性降低。并且如果两个以上的模块修改布局后，最后修改布局的模块会起作用。更重要的是，布局中指向其它资源的 ID 很难确定下来。推荐使用下面的方法修改布局：

```
package de.robv.android.xposed.mods.tutorial;

import android.graphics.Color;
import android.widget.TextView;
import de.robv.android.xposed.IXposedHookInitPackageResources;
import de.robv.android.xposed.XposedBridge;
import de.robv.android.xposed.callbacks.XC_InitPackageResources.InitPackageResourcesParam;
import de.robv.android.xposed.callbacks.XC_LayoutInflated;
import de.robv.android.xposed.callbacks.XC_LayoutInflated.LayoutInflatedParam;

public class Tutorial5 implements   IXposedHookInitPackageResources{

    @Override
    public void handleInitPackageResources(InitPackageResourcesParam resparam) throws Throwable {
         if (!resparam.packageName.equals("com.android.systemui"))
                return;

            resparam.res.hookLayout("com.android.systemui", "layout", "status_bar", new XC_LayoutInflated() {
                @Override
                public void handleLayoutInflated(LayoutInflatedParam liparam) throws Throwable {
                    TextView clock = (TextView) liparam.view.findViewById(
                            liparam.res.getIdentifier("clock", "id", "com.android.systemui"));
                    clock.setTextColor(Color.RED);
                    XposedBridge.log("layout  resNames.fullname:"+liparam.resNames.fullName);
                    XposedBridge.log("layout  resNames.id:"+liparam.resNames.id);
                    XposedBridge.log("layout  resNames.name:"+liparam.resNames.name);
                    XposedBridge.log("layout  resNames.pkg:"+liparam.resNames.pkg);
                    XposedBridge.log("layout  resNames.type:"+liparam.resNames.type);

                    XposedBridge.log("layout  resNames.variant:"+liparam.variant);
                    XposedBridge.log("layout  resNames.view:"+liparam.view);




                }
            }); 
    }

}
```

回调方法`handleLayoutInflated` 会在 layout 文件被填充后回调，在方法的 LayoutInflatedParam 对象 参数中，你可以找到你想修改的 View 组件。你也可以通过调用`resNames`来确定加载的那一个布局文件。用 `variant`来确定加载的布局的目录’layout-land‘。`res` 同时也会帮你获取资源的 ID 和其它的资源。

五、用反射来 hook 方法
==============

每当应用加载的时候，IXposedHookLoadPackPage 接口的 handLoadPackage 方法就会被调用执行，为了让我们在正确的进程中执行，需要先判断被加载的包是不是正确的包

```
package de.robv.android.xposed.mods.tutorial;

import de.robv.android.xposed.IXposedHookLoadPackage;
import de.robv.android.xposed.callbacks.XC_LoadPackage.LoadPackageParam;

public class Tutorial6 implements IXposedHookLoadPackage {

    @Override
    public void handleLoadPackage(LoadPackageParam param) throws Throwable {
        if(!param.packageName.equals("com.android.systemui"))
            return;



        }


}
```

一旦我们进入到正确的进程进后，我们就能用 param 变中的 ClassLoad 来访问该进程中加载的类

```
package de.robv.android.xposed.mods.tutorial;

import android.webkit.WebView.FindListener;
import de.robv.android.xposed.IXposedHookLoadPackage;
import de.robv.android.xposed.XC_MethodHook;
import de.robv.android.xposed.XC_MethodHook.MethodHookParam;
import de.robv.android.xposed.callbacks.XC_LoadPackage.LoadPackageParam;

import static de.robv.android.xposed.XposedHelpers.findAndHookMethod;
public class Tutorial6 implements IXposedHookLoadPackage {

    @Override
    public void handleLoadPackage(LoadPackageParam param) throws Throwable {
        if(!param.packageName.equals("com.android.systemui"))
            return;
         findAndHookMethod("com.android.systemui.statusbar.policy.Clock",param.classLoader, "updateClock", new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                    // this will be called before the clock was updated by the original method
                }
                @Override
                protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                    // this will be called after the clock was updated by the original method
                }
        });


        }


}
```

[XposedHelpers](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Frovo89%2FXposedBridge%2Fwiki%2FHelpers "https://github.com/rovo89/XposedBridge/wiki/Helpers") 是一个重要的工具类，推荐用 Eclipse 的同学静态导入该类中的方法 `import static de.robv.android.xposed.XposedHelpers.findAndHookMethod;`。该类能够通过反射机制来访问方法、构造器、域。 `findAndHookMehthod(String packageName,Class clazz, String methodName, Object... args))`方法来对函数进行 Hook。如果在方法前和方法后 Hook, 该方法最后一个参数需要实现`XC_MethodHook`类的`beforeHookedMethod`和`afterHookedMethod`方法，如果想要替换整个方法，则需要实现`XC_MethodReplacement`类的`replaceHookedMethod`方法 XposedBridge 保存了每个 Hook 方法的回调方法。优先级高的回调方法被优先调用`A.before -> B.before -> original method -> B.after -> A.after`

```
package de.robv.android.xposed.mods.tutorial;

import android.graphics.Color;
import android.webkit.WebView.FindListener;
import android.widget.TextView;
import de.robv.android.xposed.IXposedHookLoadPackage;
import de.robv.android.xposed.XC_MethodHook;
import de.robv.android.xposed.XC_MethodHook.MethodHookParam;
import de.robv.android.xposed.callbacks.XC_LoadPackage.LoadPackageParam;

import static de.robv.android.xposed.XposedHelpers.findAndHookMethod;
public class Tutorial6 implements IXposedHookLoadPackage {

    @Override
    public void handleLoadPackage(LoadPackageParam param) throws Throwable {
        if(!param.packageName.equals("com.android.systemui"))
            return;
         findAndHookMethod("com.android.systemui.statusbar.policy.Clock",param.classLoader, "updateClock", new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                    // this will be called before the clock was updated by the original method
                }
                @Override
                protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                    TextView tv = (TextView) param.thisObject;//获取调用该方法类的对象
                    String text = tv.getText().toString();
                    tv.setText(text + " :)");
                    tv.setTextColor(Color.RED);
                }
        });


        }


}
```

「欢迎在评论区讨论，掘金官方将在[掘力星计划](https://juejin.cn/post/7012210233804079141 "https://juejin.cn/post/7012210233804079141")活动结束后，在评论区抽送 100 份掘金周边，抽奖详情见活动文章」
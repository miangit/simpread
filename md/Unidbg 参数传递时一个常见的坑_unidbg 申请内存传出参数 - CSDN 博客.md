> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/qq_38851536/article/details/118149948)

#### 一、问题描述

本篇描述一个常见且隐蔽的坑，新手很容易一头雾水。我写了个 demo，里面包含两个方法。快来看看吧！

JAVA 代码

```
package com.roysue.test623;

import androidx.appcompat.app.AppCompatActivity;

import android.os.Bundle;
import android.view.View;
import android.widget.Button;
import android.widget.TextView;

import java.util.TreeMap;

public class MainActivity extends AppCompatActivity {

    // Used to load the 'native-lib' library on application startup.
    static {
        System.loadLibrary("native-lib");
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Button btn1 = findViewById(R.id.btn1);
        TextView tv1 = findViewById(R.id.tv1);
        Button btn2 = findViewById(R.id.btn2);
        TextView tv2 = findViewById(R.id.tv2);
        btn1.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                tv1.setText(getFilePath());
            }
        });

        btn2.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                TreeMap<String, String> keymap = new TreeMap<String, String>();
                keymap.put("appkey", "123");
                if(getMapisEmpty(keymap)){
                    tv2.setText("传入map为空");
                }else {
                    tv2.setText("传入map不为空");
                };

            }
        });


    }


    public native String getFilePath();

    public native boolean getMapisEmpty(TreeMap map);
}
```

C 代码

```
#include <jni.h>
#include <string>

extern "C"
JNIEXPORT jstring JNICALL
Java_com_roysue_test623_MainActivity_getFilePath(JNIEnv *env, jobject thiz) {
    jclass ContextWrapper = env->FindClass("android/content/ContextWrapper");
    jmethodID getFileDirMethodId = env->GetMethodID(ContextWrapper, "getFilesDir", "()Ljava/io/File;");
    jclass fileClazz= env->FindClass("java/io/File");
    jmethodID fileGetAbsolutePathMethodId = env->GetMethodID(fileClazz, "getAbsolutePath", "()Ljava/lang/String;");

    jobject file = env->CallObjectMethod(thiz, getFileDirMethodId);
    jstring path = static_cast<jstring>(env->CallObjectMethod(file, fileGetAbsolutePathMethodId));
    return path;
}

extern "C"
JNIEXPORT jboolean JNICALL
Java_com_roysue_test623_MainActivity_getMapisEmpty(JNIEnv *env, jobject thiz, jobject map) {
    jclass c_map = env->FindClass("java/util/Map");
    jmethodID m_isempty = env->GetMethodID(c_map,"isEmpty","()Z");
    jboolean b= env->CallBooleanMethod(map,m_isempty);
    return b;
}
```

正常运行结果

![](https://img-blog.csdnimg.cn/20210623120717342.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

#### 二、Unidbg 模拟执行遇到的问题

先模拟执行 getFIleDir 这个函数

```
package com.error;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.android.dvm.array.ArrayObject;
import com.github.unidbg.memory.Memory;
import org.apache.log4j.Level;
import org.apache.log4j.Logger;

import java.io.File;
import java.util.ArrayList;
import java.util.List;

public class demo extends AbstractJni {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;
    public DvmClass cNative;

    demo(){
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.demo").build();
        final Memory memory = emulator.getMemory(); // 模拟器的内存操作接口
        memory.setLibraryResolver(new AndroidResolver(23)); // 设置系统类库解析

        vm = emulator.createDalvikVM(null); // 创建Android虚拟机
        vm.setVerbose(true); // 设置是否打印Jni调用细节
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\error\\libnative-lib.so"), true);

        module = dm.getModule();

        cNative = vm.resolveClass("com/roysue/test623/MainActivity");

        vm.setJni(this);
        dm.callJNI_OnLoad(emulator);

    }

    public static void main(String[] args) {
        demo test = new demo();
        test.getFilePath();
    }

    public void getFilePath(){
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // 第一个参数是env
        DvmObject<?> cnative = cNative.newObject(null);
        list.add(cnative.hashCode()); // 第二个参数，实例方法是jobject，静态方法是jclazz，直接填0这里是不行的，此样本参数2被使用了
        Number number = module.callFunction(emulator, 0x8F04 + 1, list.toArray())[0];
        StringObject result = (StringObject) ((DvmObject[])((ArrayObject)vm.getObject(number.intValue())).getValue())[0];
        System.out.println(result.getValue());
    };

}
```

需要注意的是，以往我一直直接把参数 2 填 0，这是偷懒但有风险的做法，还是建议老老实实初始化类或对象，传 hashCode 进去。

运行测试

![](https://img-blog.csdnimg.cn/20210623120707308.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

这是为什么呢？而且从报错中都看不出什么。

再试试判断 map 是否为空的函数

```
public void MapisEmpty(){
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // 第一个参数是env
        DvmObject<?> cnative = cNative.newObject(null);
        list.add(cnative.hashCode()); // 第二个参数，实例方法是jobject，静态方法是jclazz，直接填0这里是不行的，此样本参数2被使用了

        TreeMap<String, String> keymap = new TreeMap<String, String>();
        keymap.put("build", "6180500");

        DvmObject<?> input_map = vm.resolveClass("java/util/TreeMap").newObject(keymap);
        list.add(vm.addLocalObject(input_map));
        Number number = module.callFunction(emulator, 0x9044 + 1, list.toArray())[0];
    };
```

![](https://img-blog.csdnimg.cn/20210623120728865.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

为什么会这样？

#### 三、解决办法

到底是什么问题呢？

getMethodId 使用啥 class，对象要手动继承这个 class，从而打通调用链，如下的代码就可以不报错，继续运行往下了。

```
package com.error;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.android.dvm.array.ArrayObject;
import com.github.unidbg.memory.Memory;
import org.apache.log4j.Level;
import org.apache.log4j.Logger;

import java.io.File;
import java.util.ArrayList;
import java.util.List;
import java.util.TreeMap;

public class demo extends AbstractJni {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;
    public DvmClass cNative;

    demo(){
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.yrx").build();
        final Memory memory = emulator.getMemory(); // 模拟器的内存操作接口
        memory.setLibraryResolver(new AndroidResolver(23)); // 设置系统类库解析

        vm = emulator.createDalvikVM(null); // 创建Android虚拟机
        vm.setVerbose(true); // 设置是否打印Jni调用细节
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\error\\libnative-lib.so"), true);

        module = dm.getModule();

        DvmClass ContextWrapper = vm.resolveClass("android/content/ContextWrapper");
        cNative = vm.resolveClass("com/roysue/test623/MainActivity", ContextWrapper);

        vm.setJni(this);
        dm.callJNI_OnLoad(emulator);

    }

    public static void main(String[] args) {
        demo test = new demo();
        test.getFilePath();
        test.MapisEmpty();
    }

    public void getFilePath(){
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // 第一个参数是env
        DvmObject<?> cnative = cNative.newObject(null);
        list.add(cnative.hashCode()); // 第二个参数，实例方法是jobject，静态方法是jclazz，直接填0这里是不行的，此样本参数2被使用了
        Number number = module.callFunction(emulator, 0x8F04 + 1, list.toArray())[0];
        StringObject result = (StringObject) ((DvmObject[])((ArrayObject)vm.getObject(number.intValue())).getValue())[0];
        System.out.println(result.getValue());
    };

    public void MapisEmpty(){
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // 第一个参数是env
        DvmObject<?> cnative = cNative.newObject(null);
        list.add(cnative.hashCode()); // 第二个参数，实例方法是jobject，静态方法是jclazz，直接填0这里是不行的，此样本参数2被使用了

        TreeMap<String, String> keymap = new TreeMap<String, String>();
        keymap.put("build", "6180500");

        DvmClass Map = vm.resolveClass("java/util/Map");
        DvmObject<?> input_map = vm.resolveClass("java/util/TreeMap", Map).newObject(keymap);
        list.add(vm.addLocalObject(input_map));
        Number number = module.callFunction(emulator, 0x9044 + 1, list.toArray())[0];
    };
}
```
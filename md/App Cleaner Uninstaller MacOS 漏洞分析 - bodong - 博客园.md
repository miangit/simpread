> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/bodong/p/17312867.html)

> 这是一个 MacOS 下的软件卸载工具，这里是一些分析日记，若有侵权，请联系我，秒删。

     这是一个 MacOS 下的软件卸载工具，这里是一些分析日记，若有侵权，请联系我，秒删。　

     安装之后，直接使用 Hopper Disassembler 打开，接着搜索 "isunlock"，你可以找到这个函数：

```
[_TtC13App_Cleaner_822BaseFeaturesController isUnlocked]
```

     这个函数从 LicenseStateStroage 中获取注册状态，它是 LicenseManager 的一个字段，偏移是：

```
objc_ivar_offset__TtC13App_Cleaner_822BaseFeaturesController_licenseManager + 0x28
```

     在 isUnlocked 函数中，可以看到它调用了属性的 getter：

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

```
-[_TtC13App_Cleaner_822BaseFeaturesController isUnlocked]:        // -[App_Cleaner_8.BaseFeaturesController isUnlocked]
        000000010034150c         stp        x22, x21, [sp, #-0x30]!                     ; Objective C Implementation defined at 0x1005358cc (instance method)
        0000000100341510         stp        x20, x19, [sp, #0x10]
        0000000100341514         stp        fp, lr, [sp, #0x20]
        0000000100341518         add        fp, sp, #0x20
        000000010034151c         mov        x19, x0
        0000000100341520         adrp       x8, #0x10079b000                            ; 0x10079b148@PAGE
        0000000100341524         ldr        x8, [x8, #0x148]                            ; 0x10079b148@PAGEOFF, objc_ivar_offset__TtC13App_Cleaner_822BaseFeaturesController_licenseManager
        0000000100341528         ldr        x8, [x0, x8]
        000000010034152c         ldr        x20, [x8, #0x28]
        0000000100341530         ldr        x8, [x20]
        0000000100341534         ldr        x21, [x8, #0xc0]
        0000000100341538         mov        x0, x20                                     ; argument "ptr" for method imp___stubs__swift_retain
        000000010034153c         bl         imp___stubs__swift_retain                   ; swift_retain
        0000000100341540         mov        x0, x19                                     ; argument "instance" for method imp___stubs__objc_retain
        0000000100341544         bl         imp___stubs__objc_retain                    ; objc_retain
        0000000100341548         mov        x19, x0
        000000010034154c         blr x21                                         ; add breakpoint here.
        0000000100341550         mov        x21, x0
        0000000100341554         mov        x0, x19                                     ; argument "instance" for method imp___stubs__objc_release
        0000000100341558         bl         imp___stubs__objc_release                   ; objc_release
        000000010034155c         mov        x0, x20                                     ; argument "ptr" for method imp___stubs__swift_release
        0000000100341560         bl         imp___stubs__swift_release                  ; swift_release
        0000000100341564         sub        w8, w21, #0x1
        0000000100341568         and        w8, w8, #0xff
        000000010034156c         cmp        w8, #0x3
        0000000100341570         cset       w0, lo
        0000000100341574         ldp        fp, lr, [sp, #0x20]
        0000000100341578         ldp        x20, x19, [sp, #0x10]
        000000010034157c         ldp        x22, x21, [sp], #0x30
        0000000100341580         ret
```

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

      在 blr x21 处添加断点，开启调试运行。当断点触发时，使用 F7 进入目标函数。则可以进入到属性的 getter 函数了：

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

```
sub_1003dad4c:
        00000001003dad4c         sub        sp, sp, #0x30
        00000001003dad50         stp        fp, lr, [sp, #0x20]
        00000001003dad54         add        fp, sp, #0x20
        00000001003dad58         add        x0, x20, #0x28                              ; argument "pointer" for method imp___stubs__swift_beginAccess
        00000001003dad5c         add        x1, sp, #0x8                                ; argument "scratch" for method imp___stubs__swift_beginAccess
        00000001003dad60         mov        x2, #0x0                                    ; argument "flags" for method imp___stubs__swift_beginAccess
        00000001003dad64         mov        x3, #0x0                                    ; argument "arg3" for method imp___stubs__swift_beginAccess
        00000001003dad68         bl         imp___stubs__swift_beginAccess              ; swift_beginAccess
        00000001003dad6c ldrb w0, [x20, #0x28] 00000001003dad70         ldp        fp, lr, [sp, #0x20]
        00000001003dad74         add        sp, sp, #0x30
        00000001003dad78         ret
```

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

       你可以看到，注册状态就保存在 [x20, #0x28] 这个地址。因此，我们只需要修改这个函数，确保这个地址的数据总是 1，那么我们的目的就达到了，比如这样：

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

```
checkLisenseState:
        00000001003dad4c         sub        sp, sp, #0x30
        00000001003dad50         stp        fp, lr, [sp, #0x20]
        00000001003dad54         add        fp, sp, #0x20
        00000001003dad58         add        x0, x20, #0x28                              ; argument #1
        00000001003dad5c         add        x1, sp, #0x8                                ; argument #2
        00000001003dad60         mov x2, #0x1                                    ; argument #3
        00000001003dad64 strb w2, [x20, #0x28] 00000001003dad68         nop
        00000001003dad6c         ldrb       w0, [x20, #0x28]
        00000001003dad70         ldp        fp, lr, [sp, #0x20]
        00000001003dad74         add        sp, sp, #0x30
        00000001003dad78         ret
```

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

       这样再运行时，app 就变成已注册状态了。接下来我们需要跳过启动窗口，这个窗口提醒你 "buy a license"。窗口上有非常多的字符串，这些字符串都可以作为搜索目标进行搜索，然后查找引用它们的地址，再在这些地址上添加断点运行，即可找到接近窗口显示的代码。比如这些字符串：

```
yourFreeTrialHasEnded
                  Your free trial is ending less than 1 minute
                  Your free trial is ending <b>in %d hour(s)</b>!%@
                  ...
```

       当断点断下来后，你可以在调用堆栈中找到很多函数，比如这个：

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

```
int sub_1003eb5e4(int arg0, int arg1, int arg2, int arg3, int arg4) {
    var_50 = r28;
    stack[-88] = r27;
    var_40 = r26;
    stack[-72] = r25;
    var_30 = r24;
    stack[-56] = r23;
    var_20 = r22;
    stack[-40] = r21;
    r31 = r31 + 0xffffffffffffffa0 - 0x30;
    r19 = r20;
    r22 = arg4;
    r20 = arg3;
    r23 = arg2;
    r21 = arg1;
    r24 = arg0;
    r0 = objc_opt_self(@class(NSUserDefaults));
    r0 = [r0 standardUserDefaults];
    r29 = &saved_fp;
    r25 = [r0 retain];
    r26 = (extension in Foundation):Swift.String._bridgeToObjectiveC()();
    r27 = [r25 boolForKey:r26];
    [r25 release];
    r0 = [r26 release];
    if (r27 == 0x0) goto .l1;

loc_1003eb68c:
    swift_unknownObjectWeakAssign();
    *(int128_t *)(r19 + 0x98) = r24;
    *(int128_t *)(r19 + 0xa0) = r21;
    *(int128_t *)(r19 + 0xa8) = r23;
    *(int128_t *)(r19 + 0xb0) = r20;
    swift_bridgeObjectRetain(r21);
    swift_bridgeObjectRetain(r20);
    sub_1003eb960(r25, r26, r27, r28);
    r20 = *(r19 + 0x28) + 0x18;
    swift_beginAccess(r20, r29 - 0x68, 0x0, 0x0);
    r21 = *(r21 + 0x20);
    if (r21 == 0x0) goto loc_1003eb748;    // 这代码就是调用提醒窗口的位置。因此把jz或者cbz xxx改成nop就行了，它就不会再出现了

loc_1003eb6f0:
    swift_bridgeObjectRetain(r21);
    swift_retain(r19);
    *(int8_t *)(r31 + 0xfffffffffffffff0) = 0x1;
    sub_1003e6920(0x0, 0x0, 0x2, r20, r21, 0x0, 0x1003ecf60, r19, stack[-144]);
    swift_bridgeObjectRelease(r21);
    goto loc_1003eb7f4;

loc_1003eb7f4:
    r0 = swift_release(r19);
    return r0;

.l1:
    return r0;

loc_1003eb748:
    sub_1003ed2c8();
    if (*qword_10079f5e0 != -0x1) {
            swift_once(0x10079f5e0, 0x1003d26b0, 0x0);
    }
    r8 = *qword_1007c4238;
    *(int128_t *)(&stack[-144] + 0xfffffffffffffff0) = 0x1003ecfcc;
    *(int128_t *)(&stack[-144] + 0xfffffffffffffff8) = r8;
    sub_1003ebac4(0x0, 0x1007a1ad0, &@class(NSLock));
    r0 = sub_10037b240(0x1003ecfb4, &stack[-144] - 0x20);
    if ((var_70 & 0x1) != 0x0) goto .l1;

loc_1003eb7d0:
    swift_retain(r19);
    sub_1003ea8b0(r22, 0x0, 0x1003ecf54, r19);
    goto loc_1003eb7f4;
}
```

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

     通过调试尝试可以找到目标代码位置，把跳转改一下即可忽略启动时的那个提醒窗口。保存，完事。

原版安装包：

链接: [https://pan.baidu.com/s/1VAVCJ3IXwY7SO9Z-qUCSSQ?pwd=wjhc](https://pan.baidu.com/s/1VAVCJ3IXwY7SO9Z-qUCSSQ?pwd=wjhc) 提取码: wjhc
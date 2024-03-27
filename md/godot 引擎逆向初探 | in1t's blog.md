> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [in1t.top](https://in1t.top/2024/01/23/godot-%E5%BC%95%E6%93%8E%E9%80%86%E5%90%91%E5%88%9D%E6%8E%A2/)

> 以一款由 godot 制作的游戏 Buckshot Roulette 为例，讨论该类游戏如何解包和逆向

[](#缘起 "缘起")缘起
--------------

这两天刷 b 站直播，常看的几个恐游主播都在玩一款叫做 **Buckshot Roulette** 的游戏。

![](https://in1t.top/img/article/20240123/1.jpg)

该游戏说白了就是带点策略的俄罗斯轮盘赌，自己玩了几局确实上头，无尽模式打了 50 多万分还是顶不住道具刷的太烂，寄了。气愤之余，我就在想能不能在有需要的时候能透视枪管里的子弹顺序，这样就可以熬过一些倒霉的对局，于是就有了这篇文章。

游戏可以在 [itch](https://mikeklubnika.itch.io/buckshot-roulette) 上购买和下载，毕竟没有免费，我也不好直接提供下载链接，只需要 1 刀，咱还是支持一下吧。

[](#引擎识别 "引擎识别")引擎识别
--------------------

游戏下载下来是一个 300 多 MB 的 exe，没有其他任何资源文件：

![](https://in1t.top/img/article/20240123/2.png)

直接放 IDA 里查看字符串，可以找到一些有意思的路径：

![](https://in1t.top/img/article/20240123/3.png)

在 010 Editor 里也可以看到一个名为 **pck** 的很大的段：

![](https://in1t.top/img/article/20240123/4.png)

那基本就可以判断这东西是 godot 引擎打包出来的了。如果你有用 godot 开发过游戏，你会发现在导出成 exe 的时候它会把静态资源还有脚本啥的统统打包成一个后缀为 `.pck` 的文件，放在 exe 的同级目录。你还可以勾选**嵌入式打包**的选项，它会把 pck 文件放到 exe 里，就像我们讨论的这个游戏一样。

[](#解包 "解包")解包
--------------

翻了翻 github，找到一个好用的工具 [gdsdecomp](https://github.com/bruvzg/gdsdecomp)，搞笑的是这软件也是用 godot 做的（

在 Release 里把工具下下来，运行 gdre_tools.exe，选择 RE Tools -> Recover project：

![](https://in1t.top/img/article/20240123/5.png)

路径就选择 exe 所在的路径：

![](https://in1t.top/img/article/20240123/6.png)

点 Open 后，会弹一个 pck 文件的预览窗口，模式就选 `Full Recovery`，再选一个解压的目录解压即可：

![](https://in1t.top/img/article/20240123/7.png)

等待一段时间就得到 godot 的工程文件夹，一般导入 godot 编辑器里小修一下都可以直接用了（

![](https://in1t.top/img/article/20240123/8.png)

[](#脚本修改 "脚本修改")脚本修改
--------------------

脚本都在 **scripts** 目录下，后缀都是 `.gd`：

![](https://in1t.top/img/article/20240123/9.png)

用文本编辑器直接打开确实是可读的代码，风格比较像 Python：

![](https://in1t.top/img/article/20240123/10.png)

查了一下才知道这是 godot 自己实现的一种脚本语言，叫 [GDScript](https://docs.godotengine.org/zh-cn/4.x/tutorials/scripting/gdscript/gdscript_basics.html)，看下语法示例就能读懂这些代码了。经过一段时间的分析，在 `ShellSpawner.gd` 里找到了记录子弹顺序的数组 **sequenceArray**：

![](https://in1t.top/img/article/20240123/11.png)

这是一个字符串数组，实弹用 “live” 表示，空弹用 “blank” 表示，如果这个数组内容是 `["blank", "live", "blank"]`，那么接下来几枪就是 `空实空` 。

那么问题转变为如何将这个数组 dump 出来。第一步肯定是要改代码的，简单起见，我选择在拿起枪的时候将数组内容 dump 到本地文件。要实现这个效果，需要修改 `InteractionManager.gd` 的 _InteractWith_ 函数，当交互对象是枪时，调用新增的 _DumpBullets_ 函数：

![](https://in1t.top/img/article/20240123/12.png)

godot 的持久化存储需要用到 [FileAccess 类](https://docs.godotengine.org/zh-cn/4.x/classes/class_fileaccess.html)，并遵循 godot 的[文件系统约束](https://docs.godotengine.org/zh-cn/4.x/tutorials/io/data_paths.html#doc-data-paths)，这里的 `user://dump.txt` 实际对应到系统的 `%APPDATA%\Godot\app_userdata\Buckshot Roulette\dump.txt` 路径。

[](#应用修改 "应用修改")应用修改
--------------------

第二步就该使得修改后的脚本生效，有两种方案可以达到这个目的：

1.  重打包成 pck 文件，然后替换 exe 的 pck 段
2.  hook 引擎中加载脚本的函数，动态替换脚本内容

对于第一种方案，刚才用的 **gdsdecomp** 是支持将文件夹打包成 pck 并替换的：

![](https://in1t.top/img/article/20240123/13.png)

![](https://in1t.top/img/article/20240123/14.png)

这个太简单了，那我们肯定是要捣鼓第二种方案的，~不然文章就太水了~。

不得不说，开源的东西就是好，在代码仓库里分析后定位到 [gdscript.cpp](https://github.com/godotengine/godot/blob/master/modules/gdscript/gdscript.cpp)，找到 _GDScript::load_source_code_ 函数：

```
Error GDScript::load_source_code(const String &p_path) {
	if (p_path.is_empty() || p_path.begins_with("gdscript://") || ResourceLoader::get_resource_type(p_path.get_slice("::", 0)) == "PackedScene") {
		return OK;
	}

	Vector<uint8_t> sourcef;
	Error err;
	Ref<FileAccess> f = FileAccess::open(p_path, FileAccess::READ, &err);
	if (err) {
		const char *err_name;
		if (err < 0 || err >= ERR_MAX) {
			err_name = "(invalid error code)";
		} else {
			err_name = error_names[err];
		}
		ERR_FAIL_COND_V_MSG(err, err, "Attempt to open script '" + p_path + "' resulted in error '" + err_name + "'.");
	}

	uint64_t len = f->get_length();
	sourcef.resize(len + 1);
	uint8_t *w = sourcef.ptrw();
	uint64_t r = f->get_buffer(w, len);
	ERR_FAIL_COND_V(r != len, ERR_CANT_OPEN);
	w[len] = 0;

	String s;
	if (s.parse_utf8((const char *)w) != OK) {
		ERR_FAIL_V_MSG(ERR_INVALID_DATA, "Script '" + p_path + "' contains invalid unicode (UTF-8), so it was not loaded. Please ensure that scripts are saved in valid UTF-8 unicode.");
	}

	source = s;
	path = p_path;
	path_valid = true;
#ifdef TOOLS_ENABLED
	source_changed_cache = true;
	set_edited(false);
	set_last_modified_time(FileAccess::get_modified_time(path));
#endif 
	return OK;
}
```

先把参数 **p_path** 打印出来看看。为了方便，我直接用 frida 来 hook 了：

```
function readString(addr) {
    var result = '';
    var buf = addr.readPointer();
    var length = buf.sub(4).readU32() - 1;
    for (let idx = 0; idx < length; idx++) {
        var ascii = buf.add(idx * 4).readU8(ascii);
        result += String.fromCharCode(ascii);
    }
    return result;
}

function main() {
    var base = Module.findBaseAddress("buckshot roulette.exe");
    Interceptor.attach(base.add(0x184C90), {
        onEnter: function(args) {
            var scriptName = readString(this.context.rdx);
            console.log(scriptName);
        },
        onLeave: function(retval) {}
    });
}

setImmediate(main);
```

得到如下结果（部分已省略）：

```
res://scripts/MenuManager.gd
res://scripts/SaveFileManager.gd
res://scripts/CursorManager.gd
res://scripts/ButtonClass.gd
res://scripts/OptionsManager.gd
res://scripts/RoundManager.gd
res://scripts/PlayerData.gd
```

改动比较少的一种方式是直接把这个 **p_path** 改成自定义的路径，这样就不需要费劲去绕 _load_source_code_ 函数的第 19 到 22 行了。先把修改后的 `InteractionManager.gd` 放到 `%APPDATA%\Godot\app_userdata\Buckshot Roulette` 目录去，再将 **p_path** 指向 `user://InteractionManager.gd` 即可，上代码：

```
function readString(addr) {
    var result = '';
    var buf = addr.readPointer();
    var length = buf.sub(4).readU32() - 1;
    for (let idx = 0; idx < length; idx++) {
        var ascii = buf.add(idx * 4).readU8(ascii);
        result += String.fromCharCode(ascii);
    }
    return result;
}

function patchString(addr, str) {
    var buf = addr.readPointer();
    for (let idx = 0; idx < str.length; idx++) {
        var ascii = str.charCodeAt(idx);
        buf.add(idx * 4).writeU8(ascii);
    }
    buf.add(str.length * 4).writeU32(0);
    buf.sub(4).writeU32(str.length + 1);
}

function main() {
    var base = Module.findBaseAddress("buckshot roulette.exe");
    Interceptor.attach(base.add(0x184C90), {
        onEnter: function(args) {
            var scriptName = readString(this.context.rdx);
            if (scriptName == "res://scripts/InteractionManager.gd") {
                patchString(this.context.rdx, "user://InteractionManager.gd");
            }
        },
        onLeave: function(retval) {}
    });
}

setImmediate(main);
```

这个时候游戏可以正常启动，但是点 Start 开始游戏后报错了：

![](https://in1t.top/img/article/20240123/15.png)

读了下 [resource_format_binary.cpp](https://github.com/godotengine/godot/blob/master/core/io/resource_format_binary.cpp) 的相关代码后，发现大概意思是 `res://scripts/InteractionManager.gd` 这个资源是空的，再回头看下 _load_source_code_ 函数，发现第 32 行还用了 **p_path**，将它赋值给一个成员变量 **path**，那还是在 32 行执行前把 **p_path** 改回 `res://scripts/InteractionManager.gd` 好了：

```
function main() {
    var base = Module.findBaseAddress("buckshot roulette.exe");

    Interceptor.attach(base.add(0x184D79), {
        onEnter: function(args) {
            var scriptName = readString(this.context.r13);
            if (scriptName == "res://scripts/InteractionManager.gd") {
                patchString(this.context.r13, "user://InteractionManager.gd");
            }
        },
        onLeave: function(retval) {}
    });

    Interceptor.attach(base.add(0x184DFC), {
        onEnter: function(args) {
            var scriptName = readString(this.context.r13);
            if (scriptName == "user://InteractionManager.gd") {
                patchString(this.context.r13, "res://scripts/InteractionManager.gd");
            }
        },
        onLeave: function(retval) {}
    });
}
```

现在完全没问题了，在游戏里拿起枪 `%APPDATA%\Godot\app_userdata\Buckshot Roulette` 目录下也如愿地出现了 `dump.txt`：

![](https://in1t.top/img/article/20240123/16.png)

[](#关于加密 "关于加密")关于加密
--------------------

其实 godot 打包的时候其实还可以对脚本和一些元数据进行预编译和加密，[参考文档](https://docs.godotengine.org/zh-cn/4.x/contributing/development/compiling/compiling_with_script_encryption_key.html)：

![](https://in1t.top/img/article/20240123/17.png)

预编译后 `scripts` 目录下的文件都会以 `.gdc` 作为文件后缀，如果进一步加密，则以 `.gde` 结尾。只是预编译并且没有对 GDScript 的解释器实现进行修改，那么用 **gdsdecomp** 还是可以直接反编译的：

![](https://in1t.top/img/article/20240123/18.png)

`gde` 也可以反编译，但是需要先设置密钥：

![](https://in1t.top/img/article/20240123/19.png)

那么密钥怎么找呢，首先要明确一点，密钥肯定在导出的二进制文件里，因为程序要运行起来就必定要动态解密 pck 的内容。至于放在哪里了，还得读源码。[file_access_pack.cpp](https://github.com/godotengine/godot/blob/master/core/io/file_access_pack.cpp) 这个文件基本告诉了你引擎是怎么解析 pck 文件的。我们注意到，在 _FileAccessPack_ 类的构造函数中用到了一个全局变量 **script_encryption_key**，这个数组的内容是由 SCons 构建导出模板时根据环境变量 `SCRIPT_AES256_ENCRYPTION_KEY` 指定的，[代码参考](https://github.com/godotengine/godot/blob/74c32faa78b54863f8f25c538083907c2bf71791/core/SCsub#L39)。

所以要找到密钥就简单了，只需要根据字符串交叉引用先定位到 _FileAccessPack_ 的构造函数，再在其中找到一个长度为 32 的全局变量数组即可。
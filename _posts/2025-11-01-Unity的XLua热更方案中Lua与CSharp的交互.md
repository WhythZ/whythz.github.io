---
# author:
title: Unity的XLua热更方案中Lua与CSharp的交互
description: >-
  本文只包含XLua热更方案中客户端开发时用到的相关部分，不讨论服务端具体如何发布热更新、以及用户客户端具体如何载入热更包
date: 2025-11-01 00:00:00 +0800
categories: [游戏开发, 引擎相关]
tags: [Unity, XLua, Lua, CSharp]
# pin: true
# media_subpath: '/resources/'
# render_with_liquid: false
math: true
# mermaid: true
# image:
#   path: /resources/xxxxxx.png
#   lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
#   alt: Responsive rendering of Chirpy theme on multiple devices
---

## 一、关于Unity热更

### 1.1 CSharp热更方案
- CSharp其实可以实现热更新，需使用其反射机制
    - CSharp的反射指的是在运行时（通过程序集中的元数据）动态地获取和操作程序集、类型、成员等信息的机制（如实例化对象、调用方法、获取和设置属性等），而无需在编译时明确知道这些类型和成员的具体信息
    - 将需频繁更改的部分逻辑做成独立的动态链接库DLL（可包含类，函数等资源）以让其他程序动态加载使用，而主模块代码则不作修改
    - 通过反射机制加载这些DLL，用新DLL文件替换旧DLL，主模块加载DLL就实现了热更新
- **某些CSharp热更方案只适用Android热更新**，iOS由于其安全机制，不允许在新申请的内存（用于修改后的代码使用）上进行写操作，故目前CSharp的反射基本不被商业项目用作热更新

### 1.2 Lua热更方案
- 核心是CSharp与Lua代码间的相互调用
    - 需为Unity项目导入热更新库，如`XLua`或`toLua`等
    - 创建基于Lua解释器的管理器，用Lua解释器来解释执行Lua代码
    - XLua等库支持以热补丁形式将已用CSharp实现的逻辑替换为Lua实现
- 商业游戏大体有三种结合Lua热更新的成熟开发方式
    - 纯Lua开发：全部逻辑都用Lua实现，好处是灵活性强，坏处是性能较差（Lua毕竟是解释型语言，虽然当代的手机可能不差这点性能）
    - 半CSharp半Lua：核心逻辑CSharp业务逻辑Lua，好处是对比纯Lua性能会更好，坏处是灵活性较差（只动例如活动等业务逻辑时可热更，但若动核心逻辑就只能强更了，这一般会导致用户流失）
    - CSharp与热补丁：对于一开始就使用纯CSharp实现但中途决定要加Lua热更新的项目，那就只能通过热补丁实现了（这样会使得项目变得不优雅，故最好一开始就定好要不要Lua热更新）

### 1.3 两类方案对比
- CSharp是静态编译语言
    - 代码在编译时被CLR转换成机器码后才能够被计算机执行，故**运行时无法修改已编译的代码**，且CLR会在运行时对代码进行各种优化和安全检查，这使得热更新困难
- Lua是动态脚本语言
    - 代码在运行时被解释器逐行执行而无需静态编译，即运行时动态加载新脚本文件替换已加载代码模块，这使得Lua可在运行时修改已加载代码实现**热更新**，而无需重新启动整个程序

## 二、关于XLua方案
- [XLua](https://github.com/Tencent/XLua)基于Lua虚拟机实现逻辑热更（Lua脚本可作为资源参与热更，但核心代码如渲染管线仍以CSharp实现）
    - 通过自动生成CSharp-Lua桥接代码（Wrap文件）实现双向调用
    - 利用Unity的`LuaInterface`组件实现与引擎API的交互
- 应用场景例子如下
    - 频繁变动的配置表（例如以Lua脚本形式储存的配置表数据）
    - 大型项目多人协作（例如编写简单UI逻辑时无需修改CSharp核心代码，而是编写Lua脚本，提高效率且降低风险）
- 导入XLua只需将其Github仓库的`Assets`目录内的两个文件夹直接拖入你自己项目的对应目录下即可

## 三、Csharp调用Lua

### 3.1 管理器设计

#### 3.1.1 Lua解释器生命周期
- 想在Unity的CSharp环境中调用Lua就必须引入Lua解释器（本质上是运行了Lua虚拟机）

```cs
using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
//引用XLua相关命名空间
using XLua;

namespace WZ
{
    public class Test : MonoBehaviour
    {
        private LuaEnv luaEnv;

        void Start()
        {
            //初始化Lua解释器（启动Lua虚拟机）以使得能在Unity中运行Lua脚本
            luaEnv = new LuaEnv();

            //以字符串形式执行Lua脚本，还可往后多传几个信息参数以便Debug，详见API
            luaEnv.DoString("print('Test Call Lua')");
            //LUA: Test Call Lua
            //UnityEngine.Debug:Log(object)
            //XLua.StaticLuaCallbacks:Print(intptr)(at Assets / XLua / Src / StaticLuaCallbacks.cs:629)
            //XLua.LuaEnv:DoString(byte[], string, XLua.LuaTable)(at Assets / XLua / Src / LuaEnv.cs:268)
            //XLua.LuaEnv:DoString(string, string, XLua.LuaTable)(at Assets / XLua / Src / LuaEnv.cs:288)
            //WZ.Test:Start()(at Assets / Scripts / Test.cs:16)

            //调用名为TestLua的Lua脚本文件，默认读取的目录为Assets/Resources/下（后续可根据需求自定义）
            //但由于Lua脚本文件并非Unity原生资源，其.lua后缀无法被Load等方法识别，故需将.lua后缀改为.lua.txt或其他有效后缀
            //该Lua脚本内容为print("Run TestLua.lua.txt Successfully")
            luaEnv.DoString("require('TestLua')");
            //Unity打印内容为LUA: Run TestLua.lua.txt Successfully

            //垃圾回收，会清除Lua中未手动释放的对象，最好不要每帧执行，可定时调用或在跨场景时调用
            luaEnv.Tick();

            //销毁Lua解释器，若需保持解释器唯一性则不建议销毁
            luaEnv.Dispose();
        }
    }
}
```

#### 3.1.2 Lua加载路径重定向
- 前文通过CSharp语句`luaEnv.DoString("require('TestLua')");`调用Lua脚本时，默认从`Assets/Resources/`目录下读取`TestLua.lua.txt`文件，我们可以重定向该加载路径

```cs
using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
//引用文件操作相关命名空间（提供对如File.Exists等方法的支持）
using System.IO;
//引用XLua相关命名空间
using XLua;

namespace WZ
{
    public class Test : MonoBehaviour
    {
        private LuaEnv luaEnv;

        void Start()
        {
            luaEnv = new LuaEnv();

            //传入一个定义为public delegate byte[] CustomLoader(ref string filepath);的委托，用于重定向Lua文件的加载路径
            //只要被AddLoader自定义添加过的委托中有一个返回了非null的byte[]，那么这个byte[]就会被用作加载Lua文件，若全都返回无效byte[]，则会继续使用默认加载路径Assets/Resources/
            luaEnv.AddLoader(MyCustomLoader1);
            luaEnv.AddLoader(MyCustomLoader2);

            //该文件TestLua.lua.txt位于默认路径Assets/Resources/下（由输出可见，该脚本名被MyCustomLoader1和MyCustomLoader2接收过，但最终还是由默认加载器成功加载）
            luaEnv.DoString("require('TestLua')");
            //MyCustomLoader1接收到Lua脚本名TestLua
            //MyCustomLoader2重定向Lua文件加载路径失败D:/PROJECTS/FrameworkLuaUI/Assets/MyFolder/TestLua.lua
            //LUA: Run TestLua.lua.txt Successfully

            //该文件TestLuaRedirect.lua位于自定路径Assets/MyFolder/下（由输出可见，该脚本名被MyCustomLoader1和MyCustomLoader2接收过，最终由MyCustomLoader2成功加载）
            //该文件后缀仅为.lua而非.lua.txt是因为我们在MyCustomLoader2中拼接路径时对.lua后缀手动进行了处理，不会出现无法识别的情况
            luaEnv.DoString("require('TestLuaRedirect')");
            //MyCustomLoader1接收到Lua脚本名TestLuaRedirect
            //LUA: Run TestLuaRedirect.lua Successfully
        }

        //接收的_filepath参数是欲被执行的Lua脚本的文件名
        private byte[] MyCustomLoader1(ref string _filepath)
        {
            Debug.Log("MyCustomLoader1接收到Lua脚本名" + _filepath);
            return null;
        }
        private byte[] MyCustomLoader2(ref string _filepath)
        {
            //拼接路径，其中Application.dataPath是Assets文件夹的绝对路径，注意添加斜杠
            string _path = Application.dataPath + "/MyFolder/" + _filepath + ".lua";

            //根据路径查询脚本文件是否存在，存在则读取其字节流返回，不存在则打印错误日志并返回null
            if (File.Exists(_path))
                return File.ReadAllBytes(_path);
            else
            {
                Debug.LogError("MyCustomLoader2重定向Lua文件加载路径失败" + _path);
                return null;
            }
        }
    }
}
```

#### 3.1.3 Lua管理器核心功能
- 管理器封装如下

```cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using System.IO;
using XLua;

namespace WZ
{
    //该管理器用于提供唯一的Lua虚拟机（继承自纯CSharp单例类，其上无MonoBehaviour祖先也就无需挂载到GameObject上）
    public class LuaManager : SingletonPure<LuaManager>
    {
        private LuaEnv luaEnv;

        #region Interfaces
        //获取Lua环境的_G表
        public LuaTable Global
        {
            get
            {
                return luaEnv.Global;
            }
        }

        //初始化Lua虚拟机
        public void Init()
        {
            //确保唯一Lua虚拟机
            if (luaEnv == null)
                luaEnv = new LuaEnv();

            //重定向Lua文件加载
            luaEnv.AddLoader(MyCustomLoaderLua);
            //重定向AB包文件加载
            luaEnv.AddLoader(MyCustomLoaderAB);
        }

        //传入Lua脚本无后缀名以执行该脚本
        public void RunLuaScript(string _name)
        {
            luaEnv.DoString(string.Format("require('{0}')", _name));
        }

        //垃圾回收
        public void ReleaseGarbage()
        {
            luaEnv.Tick();
        }

        //销毁Lua虚拟机
        public void ReleaseLuaEnv()
        {
            luaEnv.Dispose();
            luaEnv = null;
        }
        #endregion

        #region CustomLoader
        //实际生产中可通过外部工具脚本进行后缀转换
        //该加载器加载AB包中的Lua脚本"_filepath.lua"，可在测试Lua脚本时使用（脚本可正常保持".lua"后缀以便调试代码）
        private byte[] MyCustomLoaderLua(ref string _filepath)
        {
            //拼接路径，其中Application.dataPath是Assets文件夹的绝对路径，注意添加斜杠
            string _path = Application.dataPath + "/ResourcesAB/LuaScripts/" + _filepath + ".lua";

            //根据路径查询脚本文件是否存在，存在则读取其字节流返回，不存在则打印错误日志并返回null
            if (File.Exists(_path))
                return File.ReadAllBytes(_path);
            else
                return null;

            //若在其它地方调用以下语句，则能成功执行对应Lua脚本
            //LuaManager.Instance.Init();
            //LuaManager.Instance.RunLuaScript("TestLua");
            //LUA: Run Test.lua.txt Successfully
        }

        //该加载器加载AB包中的Lua脚本"_filepath.lua.txt"，只在测试AB包时才使用（需将所有脚本改为".lua.txt"后缀以便被加载）
        private byte[] MyCustomLoaderAB(ref string _filepath)
        {
            //假设在Unity中已将所有Lua脚本打入"lua"标签的AB包，此处加载器目的是从该AB包中加载Lua脚本作为TextAsset类型对象（注意不要异步加载，因为加载结果需被立即返回）
            //此处加载"_filepath.lua.txt"文本文件，TextAsset无法直接识别".lua"后缀，故文件需添上".txt"后缀，而原本的".lua"也就被当作文件名的一部分，故此处传入名称时需添上".lua"
            TextAsset _luaFile = ABManager.Instance.LoadResSync<TextAsset>("lua", _filepath + ".lua");
            //返回该TextAsset的字节流
            if (_luaFile != null)
                return _luaFile.bytes;
            else
                return null;

            //若在其它地方调用以下语句，则能成功执行对应Lua脚本
            //LuaManager.Instance.Init();
            //LuaManager.Instance.RunLuaScript("TestLua");
            //ABManager: 加载AB包lua
            //LUA: Run Test.lua.txt Successfully
        }
        #endregion
    }
}
```

- 其所继承的单例类定义如下

```cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

namespace WZ
{
    //可继承的单例抽象基类，单例在被第一次访问Instance属性时创建
    //此处泛型约束要求继承该类的T需提供公共无参构造，包括未显式声明无参构造的类由编译器提供的默认公共无参构造，非公共的不符合
    public abstract class SingletonPure<T> where T : new()
    {
        private static T instance;
        public static T Instance
        {
            get
            {
                //若无new()泛型约束则此处创建实例会编译失败
                if (instance == null)
                    instance = new T();
                return instance;
            }
        }
    }
}
```

### 3.2 访问Lua普通变量
- `Main.lua`脚本内容如下

```lua
print("Main.lua")
-- 即便是在Lua脚本内执行其它Lua脚本，也会受到CSharp侧的CustomLoader重定向的路径影响，而不是靠相对路径查找文件
require("TestVariables")
```

- `TestVariables.lua`脚本内容如下

```lua
print("TestVariables.lua")
-- 用作CSharp调用Lua测试的全局变量，Lua中声明的所有全局变量都会被存到_G表内
testNumber = 666
testFloat = 3.14
testBool = true
testString = "Yui"
-- 测试本地局部变量
local testLocal = 555
```

- 在CSharp中可通过`_G`表访问Lua中的全局变量，无法直接获取Lua中的本地局部变量

```cs
using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

namespace WZ
{
    public class Test : MonoBehaviour
    {
        void Start()
        {
            LuaManager.Instance.Init();
            LuaManager.Instance.RunLuaScript("Main");

            //读取对应名称的变量（获取的是值而非引用）
            Debug.LogError("testNumber = " + LuaManager.Instance.Global.Get<int>("testNumber").ToString());
            Debug.LogError("testFloat = " + LuaManager.Instance.Global.Get<float>("testFloat").ToString());
            Debug.LogError("testBool = " + LuaManager.Instance.Global.Get<bool>("testBool").ToString());
            Debug.LogError("testString = " + LuaManager.Instance.Global.Get<string>("testString").ToString());
            //testNumber = 666
            //testFloat = 3.14
            //testBool = True
            //testString = Yui

            //改写对应名称的变量（可以改写为任意类型）
            LuaManager.Instance.Global.Set("testNumber", "newValue");
            Debug.LogError("testNumber = " + LuaManager.Instance.Global.Get<string>("testNumber").ToString());
            //testNumber = newValue
            
            //无法直接读取局部变量
            Debug.LogError("testLocal = " + LuaManager.Instance.Global.Get<int>("testLocal").ToString());
            //InvalidCastException: can not assign nil to int
            //XLua.LuaTable.Get[TKey,TValue] (TKey key, TValue& value) (at Assets/XLua/Src/LuaTable.cs:74)
            //XLua.LuaTable.Get[TValue] (System.String key) (at Assets/XLua/Src/LuaTable.cs:345)
            //WZ.Test.Start () (at Assets/Scripts/Test.cs:31)
        }
    }
}
```

### 3.3 访问Lua各种函数
- 更新`TestVariables.lua`脚本内容如下

```lua
print("TestVariables.lua")
-- 测试无参无返函数
func1 = function()
    print("func1")
end
-- 测试有参有返函数
func2 = function(parm)
    print("func2")
    return tostring(parm)
end
-- 测试多返回值函数
func3 = function(parm)
    print("func3")
    return "Yui", 520, true, parm
end
-- 测试变长参数函数
func4 = function(parm, ...)
    print("func4")
    arg = { ... }
    local result = tostring(parm) .. "/"
    for _, v in pairs(arg) do
        result = result .. tostring(v) .. "/"
    end
    return result
end
```

- 在CSharp中可通过多种方式接收到Lua侧的函数，官方建议不要使用`LuaFunction`，因为说是会产生一些垃圾

```cs
using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Events;
using XLua;

namespace WZ
{
    //自定义形式委托，需标识[CSharpCallLua]特性（在XLua命名空间内）并在Unity中重新生成代码（自动存放在Assets/XLua/Gen/目录下），否则会报错如下
    //InvalidCastException: This type must add to CSharpCallLua: WZ.DelegateForFunc2
    //XLua.LuaTable.Get[TKey, TValue] (TKey key, TValue& value) (at Assets/XLua/Src/LuaTable.cs:65)
    //XLua.LuaTable.Get[TValue] (System.String key) (at Assets/XLua/Src/LuaTable.cs:345)
    //WZ.Test.Start() (at Assets/Scripts/Test.cs:37)
    [CSharpCallLua]
    public delegate string DelegateForFunc2(int _parm);

    //由于CSharp没有多返回值委托，故除了第一个返回值（此处为string）外的其余返回值，都需通过传入对应类型的out参数来间接实现多返回值（应正常传入的参数则无需out修饰）
    [CSharpCallLua]
    public delegate string DelegateForFunc3(int _inParm, out int _returnVal2, out bool _returnVal3, out int _returnVal4);

    //该委托接收一个普通参数和一个params变长参数，后者类型取决于具体需要，若确定全为同类型T则直接传T[]数组即可（性能更好），否则只能用object[]数组
    [CSharpCallLua]
    public delegate string DelegateForFunc4(int _parm, params object[] _parms);

    public class Test : MonoBehaviour
    {
        void Start()
        {
            LuaManager.Instance.Init();
            LuaManager.Instance.RunLuaScript("Main");

            #region 无参无返函数
            //用Unity提供的UnityAction委托接收，定义为public delegate void UnityAction();（源自using UnityEngine.Events;），等价于用相同定义的自定义委托接收
            UnityAction _callFunc1Method1 = LuaManager.Instance.Global.Get<UnityAction>("func1");
            if (_callFunc1Method1 != null) { _callFunc1Method1(); }
            //LUA: func1

            //用CSharp提供的Action委托接收，定义为public delegate void Action();（源自using System;）
            Action _callFunc1Method2 = LuaManager.Instance.Global.Get<Action>("func1");
            if (_callFunc1Method2 != null) { _callFunc1Method2(); }
            //LUA: func1

            //用XLua提供的LuaFunction接收，定义为public class LuaFunction : LuaBase（源自using XLua;）
            LuaFunction _callFunc1Method3 = LuaManager.Instance.Global.Get<LuaFunction>("func1");
            if (_callFunc1Method3 != null) { _callFunc1Method3.Call(); } //Call()调用时参数和返回值都用object[]数组接收，可传入变长参数
            //LUA: func1
            #endregion

            #region 有参有返函数
            //使用自定义的委托接收
            DelegateForFunc2 _callFunc2Method1 = LuaManager.Instance.Global.Get<DelegateForFunc2>("func2");
            if (_callFunc2Method1 != null) { Debug.Log("CSharp _callFunc2Method1 return = " + _callFunc2Method1(999)); }
            //LUA: func2
            //CSharp _callFunc2Method1 return = 999

            //使用CSharp提供的Func委托接收，定义为public delegate TResult Func<in T, out TResult>(T arg);（源自using System;）
            Func<int, string> _callFunc2Method2 = LuaManager.Instance.Global.Get<Func<int, string>>("func2");
            if (_callFunc2Method2 != null) { Debug.Log("CSharp _callFunc2Method2 return = " + _callFunc2Method2(666)); }
            //LUA: func2
            //CSharp _callFunc2Method2 return = 666

            //用XLua提供的LuaFunction接收
            LuaFunction _callFunc2Method3 = LuaManager.Instance.Global.Get<LuaFunction>("func2");
            if (_callFunc2Method3 != null) { Debug.Log("CSharp _callFunc2Method3 return = " + _callFunc2Method3.Call(333)[0]); } //此处Call()的返回值只有一个，故取索引0即可
            //LUA: func2
            //CSharp _callFunc2Method3 return = 333
            #endregion

            #region 多返回值函数
            //使用自定义的委托接收，多返回值通过out参数间接体现
            DelegateForFunc3 _callFunc3Method1 = LuaManager.Instance.Global.Get<DelegateForFunc3>("func3");
            if (_callFunc3Method1 != null)
            {
                //其实用ref也可以（需同步修改委托定义也为ref），但out无需初始化作为返回值来说更贴切
                int _returnVal2; bool _returnVal3; int _returnVal4;
                string _returnVal1 = _callFunc3Method1(1314, out _returnVal2, out _returnVal3, out _returnVal4);
                Debug.Log("CSharp _callFunc3Method1 return = " + _returnVal1 + " " + _returnVal2 + " " + _returnVal3 + " " + _returnVal4);
                //LUA: func3
                //CSharp _callFunc3Method1 return = Yui 520 True 1314
            }

            //使用XLua提供的LuaFunction接收，多返回值通过Call()返回的的object[]数组体现
            LuaFunction _callFunc3Method2 = LuaManager.Instance.Global.Get<LuaFunction>("func3");
            if (_callFunc3Method2 != null)
            {
                object[] _returnVals = _callFunc3Method2.Call("Azusa");
                string _returnResult = "CSharp _callFunc3Method2 return = ";
                for (int _i = 0; _i < _returnVals.Length; _i++)
                    _returnResult += _returnVals[_i] + " ";
                Debug.Log(_returnResult);
                //LUA: func3
                //CSharp _callFunc3Method2 return = Yui 520 True Azusa 
            }
            #endregion

            #region 变长参数函数
            //使用自定义的委托接收
            DelegateForFunc4 _callFunc4Method1 = LuaManager.Instance.Global.Get<DelegateForFunc4>("func4");
            if (_callFunc4Method1 != null) { Debug.Log("CSharp _callFunc4Method1 return = " + _callFunc4Method1(520, "Mugi", true, 789.0f)); }
            //LUA: func4
            //CSharp _callFunc4Method1 return = 520/Mugi/true/789.0/
            if (_callFunc4Method1 != null) { Debug.Log("CSharp _callFunc4Method1 return = " + _callFunc4Method1(521, 789.0f, "Yui", true, false)); }
            //LUA: func4
            //CSharp _callFunc4Method1 return = 521/789.0/Yui/true/false/

            //使用XLua提供的LuaFunction接收
            LuaFunction _callFunc4Method2 = LuaManager.Instance.Global.Get<LuaFunction>("func4");
            if (_callFunc4Method2 != null) { Debug.Log("CSharp _callFunc4Method2 return = " + _callFunc4Method2.Call(520, "Mugi", true, 789.0f)[0]); }
            //LUA: func4
            //CSharp _callFunc4Method2 return = 520/Mugi/true/789.0/
            if (_callFunc4Method2 != null) { Debug.Log("CSharp _callFunc4Method2 return = " + _callFunc4Method2.Call(521, 789.0f, "Yui", true, false)[0]); }
            //LUA: func4
            //CSharp _callFunc4Method2 return = 521/789.0/Yui/true/false/
            #endregion
        }
    }
}
```

### 3.4 映射Lua表结构

>Lua的`table`表结构同样可通过`_G`表获取，根据`table`的结构不同可以选择使用不同的接收方式

#### 3.4.1 映射到列表或字典
- 更新`TestVariables.lua`脚本内容如下

```lua
print("TestVariables.lua")
-- 测试映射到CSharp的List
testList1 = { 111, 222, 333, 444, 555 }
testList2 = { true, 1.31, "Yui", 521, false, 520 }
-- 测试映射到CSharp的Dictionary
testDictionary1 = {
    ["one"] = 111,
    ["two"] = 222,
    ["three"] = 333,
    ["four"] = 444,
    ["five"] = 555
}
testDictionary2 = {
    [true] = "Hi",
    ["str"] = 100,
    [3] = false
}
```

- 此处使用CSharp的`List`和`Dictionary`分别接收两种对应类型的Lua的`table`

```cs
using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

namespace WZ
{
    public class Test : MonoBehaviour
    {
        void Start()
        {
            LuaManager.Instance.Init();
            LuaManager.Instance.RunLuaScript("Main");

            #region List
            //已知类型元素
            List<int> _list1 = LuaManager.Instance.Global.Get<List<int>>("testList1");
            string _strList1 = "testList1 = ";
            foreach (var item in _list1)
                _strList1 += item.ToString() + " ";
            Debug.Log(_strList1);
            //testList1 = 111 222 333 444 555 

            //未知类型元素
            List<object> _list2 = LuaManager.Instance.Global.Get<List<object>>("testList2");
            string _strList2 = "testList1 = ";
            foreach (var item in _list2)
                _strList2 += item.ToString() + " ";
            Debug.Log(_strList2);
            //testList1 = True 1.31 Yui 521 False 520 
            #endregion

            #region Dictionary
            //确定类型键值对
            Dictionary<string, int> _dic1 = LuaManager.Instance.Global.Get<Dictionary<string, int>>("testDictionary1");
            string _strDic1 = "testDictionary1 = ";
            foreach (var item in _dic1)
                _strDic1 += "(" + item.Key.ToString() + ":" + item.Value.ToString() + ") ";
            Debug.Log(_strDic1);
            //testDictionary1 = (four:444) (three:333) (two:222) (five:555) (one:111) 

            //未知类型键值对
            Dictionary<object, object> _dic2 = LuaManager.Instance.Global.Get<Dictionary<object, object>>("testDictionary2");
            string _strDic2 = "testDictionary2 = ";
            foreach (var item in _dic2)
                _strDic2 += "(" + item.Key.ToString() + ":" + item.Value.ToString() + ") ";
            Debug.Log(_strDic2);
            //testDictionary2 = (True:Hi) (3:False) (str:100) 
            #endregion
        }
    }
}
```

#### 3.4.2 映射到类或接口
- 更新`TestVariables.lua`脚本内容如下

```lua
print("TestVariables.lua")
-- 测试映射到CSharp的类（值拷贝）
testClass = {
    -- 测试普通成员
    memberInt = 520,
    memberFloat = 3.14,
    memberBool = true,
    memberString = "Yui",
    memberFunction = function()
        print("Run testClass.memberFunction")
    end,
    -- 测试嵌套类
    recursionClass = {
        memberInt = 999
    }
}
-- 测试映射到CSharp的接口（引用拷贝）
testInterface = {
    memberInt = 520,
    memberFloat = 3.14,
    memberBool = true,
    memberString = "Yui",
    memberFunction = function()
        print("Run testInterface.memberFunction")
    end,
}
```

- 在CSharp中的处理如下，注意此前的其它映射都是值拷贝，唯有映射到接口是引用拷贝

```cs
using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Events;
using XLua;

namespace WZ
{
    //测试映射到类
    public class MappingRecursionClass
    {
        public int memberInt;
    }
    public class MappingTestClass
    {
        //此处成员名称与类型必须与Lua脚本中对应的表中成员匹配，且需是public，否则无法被访问
        //允许缺少成员或存在多余成员（无非是相应的Lua变量无法被接收或类成员只能被初始化为默认值而已）
        public int memberInt;
        public float memberFloat;
        public bool memberBool;
        public string memberString;
        public UnityAction memberFunction;
        public MappingRecursionClass recursionClass;
    }

    //测试映射到接口，注意接口必须添加[CSharpCallLua]特性，并重新生成XLua代码
    [CSharpCallLua]
    public interface IMappingTestClass
    {
        //接口中不允许存在成员变量，但允许存在属性（本质是函数），故可用属性接收Lua表中的变量
        //同样需要是名称与类型匹配，且get和set都必须存在且为public（啥也不修饰默认就是public）
        int memberInt { get; set; }
        float memberFloat { get; set; }
        bool memberBool { get; set; }
        string memberString { get; set; }
        UnityAction memberFunction { get; set; }
    }

    public class Test : MonoBehaviour
    {
        void Start()
        {
            LuaManager.Instance.Init();
            LuaManager.Instance.RunLuaScript("Main");

            #region Class
            //将Lua的表映射到一个类对象上
            MappingTestClass _obj = LuaManager.Instance.Global.Get<MappingTestClass>("testClass");
            Debug.Log("testClass = " + _obj.memberInt.ToString() + " " + _obj.memberFloat.ToString() + " " + _obj.memberBool.ToString() + " " + _obj.memberString.ToString());
            //testClass = 520 3.14 True Yui
            _obj.memberFunction();
            //LUA: Run testClass.memberFunction
            Debug.Log("testClass.recursionClass = " + _obj.recursionClass.memberInt.ToString());
            //testClass.recursionClass = 999

            //修改类对象成员值不会反映到Lua表中，因其是值拷贝
            _obj.memberInt = 1314;
            Debug.Log("testClass.memberInt = " + LuaManager.Instance.Global.Get<MappingTestClass>("testClass").memberInt);
            //testClass.memberInt = 520
            #endregion

            #region Interface
            //将Lua的表映射到一个接口上，CSharp的接口本不能被实例化，但XLua会自动生成一个类来实现该接口，这也是为何需要使用[CSharpCallLua]修饰并生成代码
            IMappingTestClass _iObj = LuaManager.Instance.Global.Get<IMappingTestClass>("testInterface");
            Debug.Log("testInterface = " + _iObj.memberInt.ToString() + " " + _iObj.memberFloat.ToString() + " " + _iObj.memberBool.ToString() + " " + _iObj.memberString.ToString());
            //testInterface = 520 3.14 True Yui
            _iObj.memberFunction();
            //LUA: Run testInterface.memberFunction

            //修改接口成员值会直接反映到Lua表中，因其是引用拷贝
            _iObj.memberInt = 1314;
            Debug.Log("testInterface.memberInt = " + LuaManager.Instance.Global.Get<IMappingTestClass>("testInterface").memberInt);
            //testInterface.memberInt = 1314
            #endregion
        }
    }
}
```

#### 3.4.3 映射到`LuaTable`
- 更新`TestVariables.lua`脚本内容如下

```lua
print("TestVariables.lua")
-- 测试映射到LuaTable
testTable = {
    memberInt = 520,
    memberFloat = 3.14,
    memberBool = true,
    memberString = "Yui",
    memberFunction = function()
        print("Run testTable.memberFunction")
    end
}
```

- 在CSharp中的处理如下

```cs
using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Events;
using XLua;

namespace WZ
{
    public class Test : MonoBehaviour
    {
        void Start()
        {
            LuaManager.Instance.Init();
            LuaManager.Instance.RunLuaScript("Main");

            //LuaTable类依赖于XLua命名空间，官方不建议使用LuaTable和LuaFunction等类型，因其效率较低且易产生垃圾
            LuaTable _tab = LuaManager.Instance.Global.Get<LuaTable>("testTable");
            
            //读取LuaTable内元素的值
            Debug.Log("testTable.memberInt = " + _tab.Get<int>("memberInt").ToString());
            //testTable.memberInt = 520
            Debug.Log("testTable.memberFloat = " + _tab.Get<float>("memberFloat").ToString());
            //testTable.memberFloat = 3.14
            Debug.Log("testTable.memberBool = " + _tab.Get<bool>("memberBool").ToString());
            //testTable.memberBool = True
            Debug.Log("testTable.memberString = " + _tab.Get<string>("memberString").ToString());
            //testTable.memberString = Yui
            _tab.Get<UnityAction>("memberFunction")();
            //LUA: Run testTable.memberFunction

            //对LuaTable内元素的Set修改会影响Lua侧对应变量的值
            _tab.Set("memberInt", 20);
            LuaTable _tabRe = LuaManager.Instance.Global.Get<LuaTable>("testTable");
            Debug.Log("testTable.memberInt = " + _tabRe.Get<int>("memberInt").ToString());
            //testTable.memberInt = 20

            //用完就销毁，否则有垃圾
            _tab.Dispose();
            _tabRe.Dispose();
        }
    }
}
```

## 四、Lua调用Csharp

### 4.1 Lua初始化入口
- 游戏启动时只能先执行某些CSharp脚本入口，让其调用某些Lua脚本后，Lua才能够访问CSharp（不可能在最开始先调用Lua，其没法直接访问CSharp）

```cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

namespace WZ
{
    //Lua调用CSharp前需由此入口初始化Lua框架，否则无法调用
    public class LuaEntry : MonoBehaviour
    {
        void Awake()
        {
            //初始化Lua环境，往后无需再初始化
            LuaManager.Instance.Init();
            //执行Lua侧的初始化脚本
            LuaManager.Instance.RunLuaScript("Main");
        }
    }
}
```

### 4.2 调用CSharp的类与枚举
- 在Lua中调用已有类

```lua
-- 在Lua中调用CSharp的类直接通过以下方式即可，Unity的GameObject和Tranform等类在UnityEngine命名空间内
-- CS.命名空间.类名

-- 由于Lua中无法使用new，实例化一个对象直接类名括号即可
local obj1 = CS.UnityEngine.GameObject() -- 调用默认的无参构造
obj1.name = "YuiObject"
local obj2 = CS.UnityEngine.GameObject("MugiObject") -- 调用初始化GO名称的构造

-- 为了方便使用，且节约性能，可以使用全局变量存储CSharp中的特定类作为别名使用
GameObject = CS.UnityEngine.GameObject
local obj3 = GameObject("AzusaObject") -- 能够正常使用

-- 类中的静态成员方法可通过.调用，对象的成员也可通过.调用
local cam = GameObject.Find("MainCamera")
CS.UnityEngine.Debug.Log(tostring(cam.transform.position)) -- (0.00, 1.00, -10.00): 239599616

-- 类中的非静态成员方法需通过:调用
cam.transform:Translate(CS.UnityEngine.Vector3.right) -- 向右移动1个单位
CS.UnityEngine.Debug.Log(tostring(cam.transform.position)) -- (1.00, 1.00, -10.00): 835190784
```

- 在CSharp中声明自定义类

```cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class TestClassOutNamespace
{
    public void Print(string _str)
    {
        Debug.Log(_str + " TestClassOutNamespace");
    }
}

namespace WZ
{
    //有命名空间，用于测试Lua调用CSharp的类
    public class TestClassInNamespace
    {
        public void Print(string _str)
        {
            Debug.Log(_str + " TestClassInNamespace");
        }
    }

    public class TestClassMono : MonoBehaviour
    {
        void Start()
        {
            Debug.Log("TestClassMono Start");
        }
    }
}
```

- 在Lua中调用自定义类

```lua
GameObject = CS.UnityEngine.GameObject

-- 调用自定义类TestClassOutNamespace和WZ.TestClassInNamespace（非MonoBehaviour子类，可以直接创建新对象）
local ett1 = CS.TestClassOutNamespace()   -- 无命名空间
ett1:Print("ett1") -- ett1 TestClassOutNamespace
local ett2 = CS.WZ.TestClassInNamespace() -- 命名空间WZ
ett2:Print("ett2") -- ett2 TestClassInNamespace

-- 调用自定义类WZ.TestClassMono（MonoBehaviour子类，不能直接new新对象）
local obj = GameObject("NewGameObjectForTest")
-- 需将脚本AddComponent到新建出来的GameObject上，注意此处typeof是XLua提供的重要方法
obj:AddComponent(typeof(CS.WZ.TestClassMono)) -- TestClassMono Start
```

- 在CSharp中声明枚举

```cs
namespace WZ
{
    //用于测试Lua调用CSharp的枚举
    public enum TestEnumInNamespace
    {
        Type1,
        Type2,
        Type3
    }
}
```

- 在Lua中调用枚举

```lua
-- 调用Unity中的枚举，跟调用类的方法类似，此处调用PrimitiveType的Cube枚举项
local obj = CS.UnityEngine.GameObject.CreatePrimitive(CS.UnityEngine.PrimitiveType.Cube)

-- 注意命名空间
MyEnum = CS.WZ.TestEnumInNamespace
print(MyEnum.Type1) -- LUA: Type1: 0

-- 数值类型转换为枚举，使用固定方法__CastFrom(number)
print(MyEnum.__CastFrom(1)) -- LUA: Type2: 1

-- 字符串类型转换为枚举，使用固定方法__CastFrom(string)
print(MyEnum.__CastFrom("Type3")) -- LUA: Type3: 2
```

### 4.3 调用CSharp的数据集合
- 在CSharp中声明数组、列表、字典

```cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

namespace WZ
{
    //用于测试Lua调用CSharp的枚举
    public class TestContainer
    {
        //一维数组
        public int[] arr = new int[5] { 11, 22, 33, 44, 55 };
        //二维数组
        public int[,] map = new int[4,3] {
            { 11, 12, 13 },
            { 21, 22, 23 },
            { 31, 32, 33 },
            { 41, 42, 43 }
        };
        //列表
        public List<int> list = new List<int>() { 111, 222, 333, 444, 555 };
        //字典
        public Dictionary<string, int> dict = new Dictionary<string, int>()
        {
            { "one", 1111 },
            { "two", 2222 },
            { "three", 3333 },
            { "four", 4444 },
            { "five", 5555 },
        };
    }
}
```

- 在Lua中调用数组、列表、字典

```lua
local obj = CS.WZ.TestContainer()

---------------------------------------------------------------------------------------------------------------------------</一维数组
-- CSharp中怎么调用数组，此处就怎么调用，而不是像Lua的table那样调用
-- 不能通过#运算符获取长度
print("arr length = " .. tostring(obj.arr.Length)) -- LUA: arr length = 5
-- 索引从0开始而非从1开始
print("arr[0] = " .. tostring(obj.arr[0])) -- LUA: arr[0] = 11

-- 可通过对应类的静态创建方法绕过new关键字创建数据集合实例，注意命名空间是System
local arrNew = CS.System.Array.CreateInstance(typeof(CS.System.Int32), 3)
print(tostring(arrNew)) -- LUA: System.Int32[]: 1773503944
-- 可以直接像CSharp中一样对元素进行赋值
arrNew[0] = 100
arrNew[1] = 200
arrNew[2] = 300

-- 数组和列表都不能用pairs遍历，因XLua并未像对字典一样对pairs进行特殊处理
-- for k, v in pairs(arrNew) do
--     print("arrNew[" .. k .. "]" .. tostring(v))
-- end
for i = 0, arrNew.Length - 1 do
    print("arrNew[" .. i .. "] = " .. tostring(arrNew[i]))
end
-- LUA: arrNew[0] = 100
-- LUA: arrNew[1] = 200
-- LUA: arrNew[2] = 300
---------------------------------------------------------------------------------------------------------------------------/>

---------------------------------------------------------------------------------------------------------------------------</二维数组
-- 通过GetLength方法获取二维数组的长宽，0获取行数，1获取列数
print("map row = " .. tostring(obj.map:GetLength(0))) -- LUA: map row = 4
print("map col = " .. tostring(obj.map:GetLength(1))) -- LUA: map col = 3

-- Lua中不支持以[]获取按行列索引获取元素，需以成员方法获取
-- print("map[0,1] = " .. tostring(obj.map[0,1])) -- 这是CSharp中调用二维数组的方式，此处会语法报错
-- print("map[0,1] = " .. tostring(obj.map[0][1])) -- 报错LuaException: c# exception in ArrayIndexer:System.ArgumentException: Only single dimensional arrays are supported for the requested action
print("map[0,1] = " .. tostring(obj.map:GetValue(0, 1))) -- LUA: map[0,1] = 12
---------------------------------------------------------------------------------------------------------------------------/>

---------------------------------------------------------------------------------------------------------------------------</列表
-- 调用方法添加元素，注意冒号调用
obj.list:Add(777)
-- 对于List<T>同样需注意索引从0开始
for i = 0, obj.list.Count - 1 do
    print("list[" .. i .. "] = " .. tostring(obj.list[i]))
end
-- LUA: list[0] = 111
-- LUA: list[1] = 222
-- LUA: list[2] = 333
-- LUA: list[3] = 444
-- LUA: list[4] = 555
-- LUA: list[5] = 777

-- 在Lua中创建新的泛型List容器有新旧两种方法
-- 旧版本方法，较繁琐，注意此处System.String是CSharp的规则，故无需CS.前缀
local listOldMethod = CS.System.Collections.Generic["List`1[System.String]"]()
print(tostring(listOldMethod)) -- LUA: System.Collections.Generic.List`1[System.String]: 1418522420
-- 新版本方法，更简洁
local listNewMethod = CS.System.Collections.Generic.List(CS.System.String)()
print(tostring(listNewMethod)) -- LUA: System.Collections.Generic.List`1[System.String]: -922061158
---------------------------------------------------------------------------------------------------------------------------/>

---------------------------------------------------------------------------------------------------------------------------</字典
-- 新增键值对元素，无法直接通过obj.dict["newkey"] = newvalue的方式添加
obj.dict:Add("six", 6666)
-- 修改键值对元素，无法直接通过obj.dict["key"] = newvalue的方式修改
obj.dict:set_Item("one", nil) -- 在Lua中赋的nil会被当作CSharp中对应类型的默认值
-- 对于Dictionary<Key,Value>直接用pairs遍历更方便
for k, v in pairs(obj.dict) do
    print("dict[" .. k .. "] = " .. tostring(v))
end
-- LUA: dict[one] = 0
-- LUA: dict[two] = 2222
-- LUA: dict[three] = 3333
-- LUA: dict[four] = 4444
-- LUA: dict[five] = 5555
-- LUA: dict[six] = 6666

-- 在Lua中创建新的泛型Dictionary容器同样有新旧两种方法
-- 旧版本方法过于繁琐不写了，新版本方法如下
local dicNewMethod = CS.System.Collections.Generic.Dictionary(CS.System.String, CS.System.Int32)()
print(tostring(dicNewMethod)) -- LUA: System.Collections.Generic.Dictionary`2[System.String,System.Int32]: -395368172

-- 对于新创建的字典，无法通过["key"]方式访问，而需通过固定方法获取键对应的值
dicNewMethod:Add("key", 114514)
print("dicNewMethod[\"key\"] = " .. tostring(dicNewMethod["key"])) -- LUA: dicNewMethod["key"] = nil
print("dicNewMethod[\"key\"] = " .. tostring(dicNewMethod:get_Item("key"))) -- LUA: dicNewMethod["key"] = 114514
print("dicNewMethod[\"key\"] = " .. tostring(dicNewMethod:TryGetValue("key"))) -- LUA: dicNewMethod["key"] = true
-- CSharp中的声明为bool TryGetValue(TKey key, out TValue value)导致上下两打印结果不同，因为CSharp无多返回值，故用out传参模拟
print(dicNewMethod:TryGetValue("key")) -- LUA: true    114514
---------------------------------------------------------------------------------------------------------------------------/>
```

### 4.4 调用CSharp的拓展方法
- 在CSharp中声明

```cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using XLua;

namespace WZ
{
    //一个普通的类，拥有一些自己的方法
    public class TestOrigin
    {
        public void Speak(string _words)
        {
            Debug.LogError("TestOrigin speaks " + _words + " by TestOrigin");
        }
    }

    //扩展方法必须是在静态类中的静态方法，且若想在XLua中调用则必须以[LuaCallCSharp]修饰（添加修饰后需重新生成XLua）
    //语法上允许不同类的拓展方法写在同一静态类中，但实践中最好按照类型或功能进行分类
    [LuaCallCSharp]
    public static class TestExtend
    {
        //为已有的TestOrigin类添加新的拓展方法Eat，而无需修改原类的代码（避免重新编译）或创建其子类（避免冗余继承结构）
        //第一个参数前的this关键字表示这是一个扩展方法，此处表示扩展的目标类型是TestOrigin，该方法可直接通过目标类实例调用
        public static void Eat(this TestOrigin _obj)
        {
            Debug.LogError("TestOrigin eats " + _obj.ToString() + " by TestExtend");
        }
    }
}
```

- 在Lua中调用

```lua
local obj = CS.WZ.TestOrigin()

-- 测试TestOrigin类原有的方法
obj:Speak("Yui") -- TestOrigin speaks Yui by TestOrigin

-- 测试TestOrigin类在静态类TestExtend中的拓展方法，后者须以[LuaCallCSharp]修饰并重新生成XLua，否则报错LuaException: Main:4: attempt to call a nil value (method 'Eat')
obj:Eat() -- TestOrigin eats WZ.TestOrigin by TestExtend
```

### 4.5 调用CSharp的各类函数

#### 4.5.1 `ref/out`参数
- 在CSharp中声明

```cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

namespace WZ
{
    public class TestMethodsWithRefOrOut
    {
        public string FuncRef(int _a, ref int _b, int _c, ref int _d)
        {
            _b = _a;
            _d = _c;
            return "FuncRef Over";
        }

        public void FuncOut(int _a, out int _b, int _c, out int _d)
        {
            _b = _a * 10;
            _d = _c / 10;
            Debug.LogError("FuncOut Over");
        }

        public void FuncRefOut(int _a, ref int _b, int _c, out int _d)
        {
            _b = 999;
            _d = 111;
            Debug.LogError("FuncRefOut Over");
        }
    }
}
```

- 在Lua中调用

```lua
local obj = CS.WZ.TestMethodsWithRefOrOut()

-- 所有ref类型的参数都需传入一个占位的值，即不能空着
-- ref类型参数以多返回值形式给到Lua侧，若函数有返则体现到首个返回值上，往后依次是经调用后的ref传入参数的值
-- public string FuncRef(int _a, ref int _b, int _c, ref int _d)
local return1, b1, d1 = obj:FuncRef(10, 20, 30, 40)
print("return=" .. tostring(return1) .. " b=" .. tostring(b1) .. " d=" .. tostring(d1))
-- LUA: return=FuncRef Over b=10 d=30

-- 所有out类型的参数不需要传值，空着当他不存在就行
-- out类型参数同样以多返回值形式给到Lua侧，此处测试函数无返回值
-- public void FuncOut(int _a, out int _b, int _c, out int _d)
local b2, d2 = obj:FuncOut(50, 60) -- 此处相当于：50对应_a、60对应_c
print("b=" .. tostring(b2) .. " d=" .. tostring(d2))
-- FuncOut Over
-- LUA: b=500 d=6

-- 综合使用，注意ref不能空而out需空即可
-- public void FuncRefOut(int _a, ref int _b, int _c, out int _d)
local b3, d3 = obj:FuncRefOut(50, 0, 60) -- 此处相当于：50对应_a、0对应_b、60对应_c
print("b=" .. tostring(b3) .. " d=" .. tostring(d3))
-- FuncRefOut Over
-- LUA: b=999 d=111
```

#### 4.5.2 调用重载函数
- 在CSharp中声明

```cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

namespace WZ
{
    public class TestReloadedMethods
    {
        public int TestFunc1()
        {
            return 100;
        }

        public int TestFunc1(int _a, int _b)
        {
            return _a + _b;
        }

        public int TestFunc2(int _a)
        {
            return _a;
        }

        public float TestFunc2(float _a)
        {
            return _a;
        }
    }
}
```

- 在Lua中调用

```lua
local obj = CS.WZ.TestReloadedMethods()

-- 即便Lua本身语法不支持函数重载，但此处可调用CSharp中的重载
print(tostring(obj:TestFunc1())) -- LUA: 100
print(tostring(obj:TestFunc1(111, 222))) -- LUA: 333

-- 试图调用两个以int/float区分的重载函数，由于Lua只有number而不区分整型和浮点型，此处会不符预期
print(tostring(obj:TestFunc2(10))) -- LUA: 10
print(tostring(obj:TestFunc2(10.2))) -- LUA: 0
-- 所以最好需要尽量这种情况，比如统一全部使用float就得了，别搞这样的多精度重载

-- 但是你硬要写也有性能较低的办法，此处使用反射获取函数信息，传入函数名和参数类型列表（此处只有一个参数）
local infox = typeof(CS.WZ.TestReloadedMethods):GetMethod("TestFunc2", { typeof(CS.System.Int32) }) -- CSharp的int对应System.Int32
local infoy = typeof(CS.WZ.TestReloadedMethods):GetMethod("TestFunc2", { typeof(CS.System.Single) }) -- CSharp的float对应System.Single
-- 然后通过XLua提供的方法将上述信息转化为Lua侧的方法
local funcx = xlua.tofunction(infox)
local funcy = xlua.tofunction(infoy)
-- 注意成员方法的首个参数传对象，而静态方法则省略
print(tostring(funcx(obj, 10))) -- LUA: 10
print(tostring(funcy(obj, 10.2))) -- LUA: 10.199999809265
```

#### 4.5.3 调用泛型函数
- 在CSharp中声明

```cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

namespace WZ
{
    public class TestGenericMethods
    {
        public class TestFather { }
        public class TestChild : TestFather { }
        public interface ITestInterface { }
        public class TestInterfaceImpl : ITestInterface { }

        //泛型方法，有参数无约束
        public void TestFunc1<T>(T _a)
        {
            Debug.Log("测试有参数、无约束的泛型方法 " + _a.ToString());
        }

        //泛型方法，有参数有约束（T须为TestFather或其子类）
        public void TestFunc2<T>(T _a) where T : TestFather
        {
            Debug.Log("测试有参数、有子类约束的泛型方法 " + _a.ToString());
        }

        //泛型方法，无参数有约束（T须为TestFather或其子类）
        public void TestFunc3<T>() where T : TestFather
        {
            Debug.Log("测试无参数、有子类约束的泛型方法");
        }

        //泛型方法，有参数有约束（但约束为接口约束）
        public void TestFunc4<T>(T _a) where T : ITestInterface
        {
            Debug.Log("测试有参数、有接口约束的泛型方法 " + _a.ToString());
        }
    }
}
```

- 在Lua中一般认为只支持调用**有`class`泛型约束**且**接收泛型类型参数**的泛型方法，除非通过新版本XLua提供的`xlua.get_generic_method`方法，但不建议使用，因其对Unity打包时存在限制，即对于`Building Setting`->`Player Settings`->`Other Settings`->`Configuration`->`Scripting Backend`项的`Mono`和`IL2CPP`两个配置值
    - 若选`Mono`则`xlua.get_generic_method`支持对所有形式泛型方法的调用
    - 若选`IL2CPP`则`xlua.get_generic_method`要求泛型参数必须为引用类型，若为值类型则该值类型必须已经在CSharp侧被使用过，才能被支持在Lua侧作为泛型函数的泛型类型使用

```lua
local obj = CS.WZ.TestGenericMethods()
local objFather = CS.WZ.TestGenericMethods.TestFather()
local objChild = CS.WZ.TestGenericMethods.TestChild()
local objImpl = CS.WZ.TestGenericMethods.TestInterfaceImpl()

-- 不支持调用：无泛型约束的泛型方法（除非通过xlua.get_generic_method获取，并设置泛型类型）
-- obj:TestFunc1(999) -- LuaException: invalid arguments to TestFunc1
local testFunc1Generic = xlua.get_generic_method(CS.WZ.TestGenericMethods, "TestFunc1")
local testFunc1Integer = testFunc1Generic(CS.System.Int32) -- 例如指定泛型类型T为int
-- 通过xlua.get_generic_method中设置的该成员泛型函数所属的class的对象来调用（如果是静态方法则不用在第一个参数传类对象）
testFunc1Integer(obj, 999) -- 测试有参数、无约束的泛型方法 999

-- 支持调用：有子类泛型约束且有参数的泛型方法
obj:TestFunc2(objFather) -- 测试有参数、有子类约束的泛型方法 WZ.TestGenericMethods+TestFather
obj:TestFunc2(objChild) -- 测试有参数、有子类约束的泛型方法 WZ.TestGenericMethods+TestChild

-- 不支持调用：有泛型约束但无参数的泛型方法
-- obj:TestFunc3() -- LuaException: invalid arguments to TestFunc3

-- 不支持调用：非class泛型约束的泛型方法（此处以接口约束为例）
-- obj:TestFunc4(objImpl) -- LuaException: invalid arguments to TestFunc4
```

#### 4.5.4 调用协程函数
- 在CSharp中声明

```cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

namespace WZ
{
    //注意需是继承自MonoBehaviour的类，才能调用StartCoroutine以启动协程
    public class Test : MonoBehaviour
    {
    }
}
```

- 在Lua中调用

```lua
GameObject = CS.UnityEngine.GameObject
WaitForSeconds = CS.UnityEngine.WaitForSeconds -- 用于在协程内等待指定秒数

-- CSharp中需借助MonoBehaviour的类的StartCoroutine成员方法来启动协程，故需借助在GameObject上挂载的脚本组件来调用
local obj = GameObject("CoroutineHelper")
local scp = obj:AddComponent(typeof(CS.WZ.Test)) -- 挂载一个内有名为Test的MonoBehaviour子类的脚本

-- 定义协程函数
local coFunc = function()
    local count = 9
    while count > 0 do
        print("coFunc round " .. tostring(count))
        -- 此处无法使用CSharp的yield return xxx语法，但可以使用Lua的coroutine.yield(xxx)语法
        -- coroutine.yield(nil) -- 等待1帧
        coroutine.yield(WaitForSeconds(1)) -- 等待1秒
        count = count - 1
    end
end

-- 试图将coFunc作为协程函数直接启动，发现报错，因为不能将Lua侧的函数传给CSharp侧的StartCoroutine
-- scp:StartCoroutine(coFunc) -- LuaException: invalid arguments to UnityEngine.MonoBehaviour.StartCoroutine!

-- 引入XLua提供的工具模块，通过其cs_generator方法包裹Lua侧函数传给StartCoroutine
util = require("xlua.util")
scp:StartCoroutine(
    util.cs_generator(
        function()
            -- 想写啥写啥
            coFunc()
            print("coFunc finished")
        end
    )
)
-- 运行结果如下，每行均间隔1秒被打印出来
-- LUA: coFunc round 9
-- LUA: coFunc round 8
-- LUA: coFunc round 7
-- LUA: coFunc round 6
-- LUA: coFunc round 5
-- LUA: coFunc round 4
-- LUA: coFunc round 3
-- LUA: coFunc round 2
-- LUA: coFunc round 1
-- LUA: coFunc finished
```

### 4.6 调用CSharp的委托与事件
- 在CSharp中声明

```cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Events;

namespace WZ
{
    public class TestDelAndEvent
    {
        //接收一个string参数的委托/事件
        public UnityAction<string> testDel;
        public event UnityAction<string> testEvent;

        //由于事件只能在类的内部调用，故此处提供一个方法供Lua侧调用
        public void DoTestEvent(string _str)
        {
            if (testEvent != null)
                testEvent(_str);
        }

        //由于事件对外不提供清空方法，故此处提供一个
        public void ClearTestEvent()
        {
            testEvent = null;
        }
    }
}
```

- 在Lua中调用

```lua
local obj = CS.WZ.TestDelAndEvent()

-- 用于注册的函数
local testFunc = function(parm)
    print("testFunc = " .. tostring(parm))
end

-- 用CSharp中的委托来装Lua侧的函数，注意此处声明的函数必须与CSharp对应委托的签名保持一致
-- 初始化委托时由于是nil而无法使用加号注册，需先用等号赋值
obj.testDel = testFunc
-- 然后才可往该委托内多播更多的同签名函数（但最好不要用Lambda，因为不好从委托中取消注册，此处只是为了方便罢了）
obj.testDel = obj.testDel + function(parm)
    print("lambda = " .. tostring(parm))
end
-- 测试调用该委托，因为不是成员函数所以无需使用:调用
obj.testDel("call testDel")
-- LUA: testFunc = call testDel
-- LUA: lambda = call testDel
obj.testDel = obj.testDel - testFunc -- 取消注册
obj.testDel("call testDel")
-- LUA: lambda = call testDel
obj.testDel = nil -- 清空委托，此时调用会报错
-- obj.testDel("call testDel") -- LuaException: Main:23: attempt to call a nil value (field 'testDel')

-- 事件加减函数和委托不一样，需像成员函数一样通过以下方式加减
obj:testEvent("+", testFunc)
obj:testEvent("+", function(parm)
    print("lambda = " .. tostring(parm))
end)
obj:DoTestEvent("call testEvent")
-- LUA: testFunc = call testEvent
-- LUA: lambda = call testEvent
obj:testEvent("-", testFunc) -- 取消注册
obj:DoTestEvent("call testEvent")
-- LUA: lambda = call testEvent
obj:ClearTestEvent() -- 清空事件
obj:DoTestEvent("call testEvent") -- 调用后无输出
```

### 4.7 重要注意事项

#### 4.7.1 如何比较`null`与`nil`

```lua
-- 若一个物体身上没有某个组件则添加该组件，若有则无需添加
local go = CS.UnityEngine.GameObject("TestGO")

-- 正常思路就是先尝试获取该物体上的对应组件，若空则执行添加逻辑
local rb = go:GetComponent(typeof(CS.UnityEngine.Rigidbody))
print(tostring(rb)) -- LUA: null: 0
if rb == nil then
    print("Target component is empty") -- 这个根本没打印出来，说明执行时没进入此处if内
    rb = go:AddComponent(typeof(CS.UnityEngine.Rigidbody))
end
print(tostring(rb)) -- LUA: null: 0

-- 坑点在于上述rb是CSharp中的null类型，是不能与Lua侧的nil使用==判等的
-- 可对null使用Equals方法与nil进行比较，但若万一传入的就是nil则会导致报错，所以一般会写一个通用的全局判空方法如下
function IsNull(obj)
    if obj == nil or obj:Equals(nil) then
        return true
    else
        return false
    end
end
-- 然后再用上述方法来书写逻辑
if IsNull(rb) then
    print("Target component is empty")
    rb = go:AddComponent(typeof(CS.UnityEngine.Rigidbody))
end
-- LUA: Target component is empty
print(tostring(rb)) -- LUA: TestGO (UnityEngine.Rigidbody): -32252

-- 还有一种解决方法，是在CSharp侧为Object类拓展一个判空方法专供Lua侧使用
-- [LuaCallCSharp]
-- public static class ExtendLuaCheckNull
-- {
--     public static boll IsNull(this Object _obj)
--     {
--         return _obj == null;
--     }
-- }
-- 然后在Lua中如下使用即可
-- if rb:IsNull() then
-- end
```

#### 4.7.2 以XLua修饰系统类型
- 前文提到过Lua调用CSharp侧的拓展方法时需添加`[LuaCallCSharp]`修饰，虽然对于其它不承载拓展方法的类，即便不修饰`[LuaCallCSharp]`在Lua中被调用也不会报错，**但对于需要被Lua侧访问的类，建议都加上该修饰以提升性能**
    - 若无该修饰，Lua会默认在运行时**通过反射查找**被调用的CSharp中对应的类和方法等，效率较低（动态）
    - 若有该修饰，重新生成XLua后会为被标记的类生成某些对应的代码而避免反射，以提高性能（静态）
- 但是对于自定义类我们可以直接手动修饰，**但对于系统已有类或是第三方库的类，我们却不能直接去修改代码加上修饰**，对于`[CSharpCallLua]`同样存在该问题，如下所示

```lua
GameObject = CS.UnityEngine.GameObject
UI = CS.UnityEngine.UI

-- 假设当前场景上存在一个名为TestSlider的拥有Slider组件的游戏对象（UGUI）
-- 我们想在Lua侧找到该对象，然后为其Value变化的事件新增一个事件函数注册

-- 需先获取该对象身上的Slider组件脚本
local slider = GameObject.Find("Canvas/TestSlider"):GetComponent("Slider")
print(tostring(slider)) -- LUA: TestSlider (UnityEngine.UI.Slider): -32468

-- 然后通过Unity提供的方法AddListener将新委托注册到成员onValueChanged上，接收的参数即变化的值，此处采取默认设置即以0~1范围内的浮点数表示拖动进度
slider.onValueChanged:AddListener(function(f)
    print("slider.onValueChanged = " .. tostring(f))
end)
-- 此时运行会报错LuaException: c# exception:This type must add to CSharpCallLua: UnityEngine.Events.UnityAction<float>,stack:  at XLua.ObjectTranslator.CreateDelegateBridge
-- 上述报错提醒需为UnityEngine.Events.UnityAction<float>添加[CSharpCallLua]修饰，因为该类型就是前文onValueChanged委托的类型，但UnityAction是Unity系统提供的泛型委托，我们没办法直接修饰
```

- XLua提供了解决上述问题的办法，即如下所示在CSharp侧写一个静态类，然后声明一个静态列表将所需添加修饰的类型都写进去，然后重新生成XLua我们就会发现原来报错的功能恢复了，滑动`TestSlider`时能正常打印信息`LUA: slider.onValueChanged = 0.40952387452126`等

```cs
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Events;
using XLua;

namespace WZ
{
    public static class TestCall
    {
        //注意此处是固定写法，列表内存入想要添加[CSharpCallLua]修饰的系统类型，然后重新生成XLua
        [CSharpCallLua]
        public static List<System.Type> csharpCallLuaList = new List<System.Type>() {
            typeof(UnityAction<float>)
            //还可填入更多所需要的类型，此处只写一个来测试
        };

        //对于[LuaCallCSharp]同理
        [LuaCallCSharp]
        public static List<System.Type> luaCallCSharpList = new List<System.Type>() {
            typeof(GameObject),
            typeof(Rigidbody)
            //可填入更多其它需在Lua使用的系统类型
        };
    }
}
```
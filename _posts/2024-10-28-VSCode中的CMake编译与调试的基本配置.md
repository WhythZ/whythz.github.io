---
# author:
title: VSCode中的CMake编译与调试的基本配置
description: >-
  先介绍如何在VSCode中编写CMakeLists.txt以实现C++代码的多文件编译，然后介绍如何编写launch.json和tasks.json文件以实现代码的调试
date: 2024-10-28 19:33:00 +0800
categories: [编程相关, 高级语言]
tags: [C++, CMake, VSCode, 环境配置]
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

## 一、环境准备

### 1.1 MinGW安装
- 官方下载网址[在此](https://www.mingw-w64.org/downloads/)，记得配置环境变量

### 1.2 CMake安装
- 官方下载网址[在此](https://cmake.org/download/)，若下载免安装的`.zip`版本则可直接解压到某处，然后配置环境变量即可

![配置MinGW和CMake环境变量.png](/resources/2024-10-28-VSCode中的CMake编译与调试的基本配置/配置MinGW和CMake环境变量.png)

- 使用如下指令检查是否成功

```
cmake --version
```

### 1.4 插件安装
- C/C++：提供代码的提示、调试与浏览
- CMake：用于借助`CMakeLists.txt`文件生成`Makefile`，后者可以被Make工具用于编译
- CMake Tools：用于辅助提示编写`CMakeLists.txt`文件

### 1.5 注意事项

#### 1.5.1 确保项目路径用英文
- 我将Github上的一个C++项目下载到本地桌面（路径含有`桌面`这个中文字段）上运行的时候，发现居然编译失败，而我在另一个纯英文路径下的完全相同的项目却能够正常运行，我郁闷了半天，然后我将桌面的名称改为英文后，问题就解决了，如下图
![桌面改名.png](/resources/2024-10-28-VSCode中的CMake编译与调试的基本配置/桌面改名.png)
- 所以若在编译代码的过程中出现了编译失败的问题，并要求你添加`launch.json`文件时，不妨先检查一下你的代码文件夹所在的路径是否存在中文，问题很有可能就产生于此

#### 1.5.2 代码中的文件路径
- 写代码时我发现，对于同一个路径名，使用CMake编译和使用CompileRun插件编译的效果不同，参考如下项目结构

```
- Root/
    - Codes/
        - 01-XXX/
        - 02-XXX/
        - 03-XXX/
            - TestCase.csv
            - Loader.cpp
        - Main.cpp
    - Libs/...
    - Others/...
```

- 我在`Loader.cpp`内实现加载`Test.csv`内数据的函数如下，`Loader`类的单例对象会在`Main.cpp`内被创建，函数`LoadTestCase`也会在`Main.cpp`内被调用

```cpp
bool Loader::LoadTestCase(std::string _path)
{
    //加载路径下的文件
    std::ifstream _file(_path);
    //检查是否成功加载
    if (!_file.good())
        return false;

    //...

    //关掉文件
    _file.close();
    return true;
}
```

- 当我传入路径`../Codes/03-XXX/TestCase.csv`时，使用CMake编译和CompileRun都能正常加载目标文件，而当我使用路径`03-XXX/TestCase.csv`时，就只有CompileRun能正常加载
- 这可能是由于二者的`cwd`设置存在差异所导致？没搞懂

## 二、代码编译

### 2.1 单文件编译
- 写好一个简单的单文件程序后

```cpp
#include <iostream>
int main()
{
    int a = 11; int b = 22;
    std::cout << "a + b = " << a + b << "\n";
}
```

- 我们新建一个终端，然后我们就可以在终端内使用相关Linux命令来进行代码文件的编译了

![单文件编译之调出终端.png](/resources/2024-10-28-VSCode中的CMake编译与调试的基本配置/单文件编译之调出终端.png)

- 使用如下指令即可生成该文件的`.exe`可执行文件，注意C++和C的源文件对应的指令的区别

```
g++ file_name.cpp
gcc file_name.c
```

![单文件调试生成可执行文件.png](/resources/2024-10-28-VSCode中的CMake编译与调试的基本配置/单文件调试生成可执行文件.png)

- 我们也可以用`-o`来指定生成的可执行文件的文件名，若不指定则默认名称为`a.exe`

```
g++ file_name.cpp -o yourname
gcc file_name.c -o yourname
```

- 然后我们即可运行已有的可执行文件

![单文件调试运行可执行文件.png](/resources/2024-10-28-VSCode中的CMake编译与调试的基本配置/单文件调试运行可执行文件.png)

### 2.2 多文件编译

#### 2.2.1 基于命令
- 用于测试的多文件结构如下

```
- include/
    - Header.h
- src/
    - Source1.cpp
    - Source2.cpp
- Test.cpp
```

- 其内内容如下

```cpp
/* Header.h */
#ifndef _HEADER_H_
#define _HEADER_H_
void Function1();
void Function2();
#endif

/* Source1.cpp */
#include <iostream>
void Function1()
{
    std::cout << "Func1" << "\n";
}

/* Source2.cpp */
#include <iostream>
void Function2()
{
    std::cout << "Func2" << "\n";
}

/* Test.cpp */
#include "Header.h"
int main()
{
    Function1();
    Function2();
}
```

- 我们通过以下命令来链接上述四个文件并生成可执行文件`Main.exe`

```
g++ .\src\Source1.cpp .\src\Source2.cpp .\Test.cpp -o Main -I .\include\
```

- 其中`-o`用于指定可执行程序的文件名，`-I`则用于指定`.cpp`源文件中使用`#include`包含的头文件应当在何处（注意到我们包含头文件时并没有使用相对路径）进行检索，不填则默认在文件根目录`.\`进行检索

![多文件链接编译运行可执行文件结果展示.png](/resources/2024-10-28-VSCode中的CMake编译与调试的基本配置/多文件链接编译运行可执行文件结果展示.png)

#### 2.2.2 基于CMake
- 我们在文件根目录添加一个`CMakeLists.txt`文件，我们可在里面写一些函数实现同样的效果

```cmake
#必须放在CMakeLists.txt文件的开头，用于指定所需的CMake的最低版本，此命令只能执行一次
cmake_minimum_required(VERSION 3.10)
#该函数必须紧随cmake_minimum_required()函数之后，用于指定项目名称、版本、描述等信息
project(ProjectName)

#该函数第一个参数接收地址，第二个参数（可自定名称）用于搜索地址后记录该地址下的.cpp源文件
aux_source_directory(src SrcSub) #搜索根目录下的src文件夹内的源文件
aux_source_directory(. SrcCur)   #搜索根目录下的源文件

#该函数用于生成可执行文件，第一个参数指定文件名，往后的参数以空格间隔，传入先前导出的那几个变量
add_executable(Main ${SrcSub} ${SrcCur})

#该函数用于指定头文件的搜索路径，可以有多个路径，此处是指在根目录下的include文件夹内进行搜索
include_directories(include)
```

- 我们此时就可以在和`CMakeLists.txt`同目录下，新建一个`build`文件夹，然后在此处打开cmd命令行，输入以下指令来生成构建系统
    - 构建系统需要指定`CMakeLists.txt`所在路径，此时在`build`目录下，所以用`..`表示 其在上一级目录
    - Windows 下CMake 默认使用微软MSVC作为编译器，若想用MinGW编译器则可通过`-G`参数来进行指定，只有第一次构建项目时需要指定

```
cmake -G"MinGW Makefiles" ..
```

- 此时在`build`目录下会生成Makefile文件，然后我们需要使用如下命令来调用编译器实际编译和链接项目，运行成功后会在`build`目录下生成`.exe`可执行文件，在命令行中通过如`Main.exe`运行即可得到输出结果

```
cmake --build .
```

- 若是在VSCode中，按下`Ctrl + Shift + P`快捷键，然后如下图进行编译器选择（别选VS的）

![CMakeConfigure编译器选择.png](/resources/2024-10-28-VSCode中的CMake编译与调试的基本配置/CMakeConfigure编译器选择.png)

- 选择完成后就会在根目录自动生成（之后每修改该文件后进行保存时，都会重新生成）`build`文件夹，后续需要借助其中的`Makefile`文件内的规则进行代码的Linking以及可执行文件的生成

![Makefile的生成.png](/resources/2024-10-28-VSCode中的CMake编译与调试的基本配置/Makefile的生成.png)

- 打开终端，`cd`到`build`目录，然后使用指令`cmake ..`（若报错无法识别指令`cmake`，则卸载插件CMake和CMake Tools并重装后重启VSCode即可）

![cmake指令调用Makefile.png](/resources/2024-10-28-VSCode中的CMake编译与调试的基本配置/cmake指令调用Makefile.png)

- 上述步骤都执行成功后，我们即可在命令行中使用`mingw32-make.exe`指令（该工具可在MinGW的文件夹目录下的`bin`目录下找到，名字有可能有所不同），这就可以使得项目依据`Makefile`文件内的规则进行编译

![使用Make生成exe可执行文件.png](/resources/2024-10-28-VSCode中的CMake编译与调试的基本配置/使用Make生成exe可执行文件.png)

- 至此我们即可使用该`.exe`文件了，若我们需要对代码进行调试，则需要配置`tasks.json`和`launch.json`文件，若无需调试则可不管，每次直接`Ctrl + F5`运行可执行文件即可
- 若`Ctrl + F5`无法对进行了新的修改的项目重新生成，那么需要手动再次运行一下`mingw32-make.exe`指令，或者通过下图的按钮来重新生成最新版本可执行文件，然后再`Ctrl + F5`来运行该文件

![VSCodeCmake重新生成.png](/resources/2024-10-28-VSCode中的CMake编译与调试的基本配置/VSCodeCmake重新生成.png)

## 三、代码调试

>若只需要一个可执行文件，则这下面的内容于你无用；但项目的开发必然伴随着无数的调试，想要在VSCode中对C++代码进行调试，除非使用CompileRun插件直接一键搞定，否则都需要手动对下面这些文件进行配置

### 3.1 创建tasks.json与launch.json
- 打开你的`main()`函数所在的`.cpp`源文件，按`Ctrl + Shift + P`打开配置面板，进行如下操作便可生成`tasks.json`和`launch.json`两个文件

![CorC++AddDebugConfiguration.png](/resources/2024-10-28-VSCode中的CMake编译与调试的基本配置/CorC++AddDebugConfiguration.png)

- 此时默认生成的配置内容如下

```javascript
/* tasks.json */
{
    "tasks": [
        {
            "type": "cppbuild",
            "label": "C/C++: g++.exe 生成活动文件",
            "command": "D:\\VSCODE\\mingw64\\bin\\g++.exe",
            "args": [
                "-fdiagnostics-color=always",
                "-g",
                "${file}",
                "-o",
                "${fileDirname}\\${fileBasenameNoExtension}.exe"
            ],
            "options": {
                "cwd": "${fileDirname}"
            },
            "problemMatcher": [
                "$gcc"
            ],
            "group": "build",
            "detail": "调试器生成的任务。"
        }
    ],
    "version": "2.0.0"
}

/* launch.json */
{
    "configurations": [
        {
            "name": "C/C++: g++.exe 生成和调试活动文件",
            "type": "cppdbg",
            "request": "launch",
            "program": "${fileDirname}\\${fileBasenameNoExtension}.exe",
            "args": [],
            "stopAtEntry": false,
            "cwd": "${fileDirname}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "miDebuggerPath": "D:\\VSCODE\\mingw64\\bin\\gdb.exe",
            "setupCommands": [
                {
                    "description": "为 gdb 启用整齐打印",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                },
                {
                    "description": "将反汇编风格设置为 Intel",
                    "text": "-gdb-set disassembly-flavor intel",
                    "ignoreFailures": true
                }
            ],
            "preLaunchTask": "C/C++: g++.exe 生成活动文件"
        }
    ],
    "version": "2.0.0"
}
```

### 3.2 配置tasks.json
- 下面是一个参考模板，可见该文件需要依赖我们使用`CMakeLists.txt`生成的`build`内的内容来运作以链接各头文件与源文件，最终借助`launch.json`调用该文件内的一系列自动化命令生成项目的可执行文件

```javascript
{
    "version": "2.0.0",
    "options": {
        "cwd": "${fileDirname}/build/" //相当于一个cd命令，进入build文件的目录
    },
    "tasks": [
        {
            "label": "cmake", //标签设置为cmake
            "type": "shell", //在命令行中执行指令
            "command": "cmake", //（在build目录下的命令行）执行cmake指令
            "args": [ //上述命令的参数列表
                ".." //相当于使用"cmake .."命令
            ],
        },
        {
            "label": "make",
            "group":{
                "kind": "build",
                "isDefault": true
            },
            "command": "mingw32-make.exe", //执行该命令生成可执行文件
            "args":[ //无需参数
            ]
        },
        {
            "label": "Build The Project", //将此标签指定给launch.json中的"preLaunchTask"参数，使得自动化完成程序的链接与编译
            "dependsOn":[
                "cmake", //执行上述"label"为"cmake"的花括号内的"command"指令
                "make" //执行上述标签为"make"的"command"指令
            ]
        }
    ]
}
```

- 上面的这个配置设计了一个总任务`Build The Project`对两个步骤的子任务（`cmake`和`make`）进行管理，这个总任务会在`launch.json`文件中预先调用以生成可执行文件

### 3.3 配置launch.json
- 注意`"miDebuggerPath"`项参数要给到电脑本地的`gdb.exe`路径，不然无法调试

```javascript
{
    "version": "0.2.0",
    "preLaunchTask": "Build The Project", //对应tasks.json文件中标签为"Build The Project"的任务，即先完成tasks.json中的对应工作内容，再完成launch.json的工作内容
    "configurations": [
        {
            "name": "(gdb) 启动",
            "type": "cppdbg", //正在使用的调试器，使用Visual Studio Windows时必须为cppvsdbg；使用GDB或LLDB时必须为cppdbg
            "request": "launch", //表示此配置是用于启动程序还是附加到已运行的实例上
            // "program": "输入程序名称，例如 ${workspaceFolder}/a.exe", //要执行的可执行文件的完整路径
            "program": "${workspaceFolder}/build/Main.exe", //CMake系列方法生成的可执行文件位置，此处.exe文件名字要与CMakeLists.txt内的保持一致
            "args": [],
            "stopAtEntry": false,
            // "cwd": "${fileDirname}", //设置调试器启动的应用程序的工作目录
            "cwd": "${workspaceFolder}/build", //CMake生成的可执行文件目录
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "miDebuggerPath": "D:\\VSCODE\\mingw64\\bin\\gdb.exe", //指定gdb调试器位置
            "setupCommands": [
                {
                    "description": "为 gdb 启用整齐打印",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                },
                {
                    "description": "将反汇编风格设置为 Intel",
                    "text": "-gdb-set disassembly-flavor intel",
                    "ignoreFailures": true
                }
            ]
        }
    ]
}
```

- 此时我们就可以在VSCode的侧边栏的按钮进行调试

![CMake调试选项.png](/resources/2024-10-28-VSCode中的CMake编译与调试的基本配置/CMake调试选项.png)

- 此时不能直接对源文件使用右上角按钮进行运行（CompileRun也不行），大概率会出问题
    - 若`#include`未使用相对路径，就会导致报错，因为使用`CMakeLists.txt`时的头文件包含是可以不写路径而直接写头文件名字的
    - 此时的`tasks.json`内执行的是CMake相关指令，所以直接运行源文件的话可能导致无法链接所有源文件的情况（即只会编译被运行的源文件内的内容，而其它没被包含进来的代码都不会被Linking进来，这就会导致一些空实现的问题）

### 3.4 关于插件CompileRun的使用
- 使用插件`C/C++ Compile Run`提供的如下的第一个按钮即可一键运行单文件代码（多文件可能会产生问题），使用第二个按钮即可进行调试而无需配置`tasks.json`和`launch.json`文件
- 这种方式产生的运行时缓存文件会存放在被编译源代码目录下的`output`文件夹中，注意要添加到忽略列表中进行忽略

![CompileRun.png](/resources/2024-10-28-VSCode中的CMake编译与调试的基本配置/CompileRun.png)

- 但是该插件只会编译被运行的单个源文件的内容，而其它没被包含进来的源代码都不会被Link进来（也可能是我不会设置），这就可能会导致一些空实现的问题，所以要注意以下几点
    - `#include`的头文件应当同时包含声明与实现，若仅在`.h`内写了声明，在某个`.cpp`内写了实现，该插件在运行包含了该`.h`声明的源代码时就会导致无法链接到`.cpp`实现的错误
    - `#include`的头文件应当使用相对路径而不仅仅是头文件名，否则会报错
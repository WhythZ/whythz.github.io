---
# author:
title: 在Windows安装并测试Emscripten全流程
description: >-
  为了将SDL2编写的小游戏发布到网页上，需要使用到Emscripten，本文先介绍其基础知识，包括如何在Windows系统中安装emsdk、如何配置其环境变量、如何使用emcc/em++编译C/C++以及如何生成HTML
date: 2025-03-03 13:18:00 +0800
categories: [编程相关, 高级语言]
tags: [C++, Emscripten, 环境配置]
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

## 一、关于Emscripten
- Emscripten编译器是WebAssembly（一种可以使用非JavaScript语言编写代码并在浏览器上运行的技术）开发的重要工具，主要通过命令行工具emcc（Emscripten Compiler Frontend）工作
- 官方文档很详细地阐述了[安装的具体流程](https://emscripten.org/docs/getting_started/downloads.html)，以及[如何验证安装成果](https://emscripten.org/docs/getting_started/Tutorial.html)等信息

## 二、安装与配置流程

### 2.1 先决条件
- 此处我在Windows环境下进行安装，其余操作系统环境下需要准备的东西详见原文档
	- 按照官方文档，我至少需要准备好一定版本的Python环境
	- 至于Git则爱装不装（相信大部分人都有），不装你同样可以笨笨地从Github仓库中下载压缩包然后解压，但就无法使用Git进行新版本拉取了

### 2.2 获取emsdk并激活
- 在某目录下，用Git克隆emsdk（Emscripten SDK），或者从[原仓库](https://github.com/emscripten-core/emsdk)中下载压缩包解压（所得文件夹自带和版本分支同名的`-main`后缀，有洁癖的可以删除一下）

```
git clone https://github.com/emscripten-core/emsdk.git
```

![Emscripten克隆emsdk.png](/resources/2025-03-03-在Windows安装并测试Emscripten全流程/Emscripten克隆emsdk.png)

- 然后我们需要进入克隆下来的名为`emsdk`的目录中，在终端执行如下指令，非Windows系统则需将`emsdk.bat`替换为`./emsdk`，将``emsdk_env.bat``替换为`source ./emsdk_env.sh`

```
# 下载并安装最新的SDK工具
emsdk.bat install latest

# 为当前用户激活最新版本的SDK工具（改写.emscripten文件)
emsdk.bat activate latest
```

- 我使用Anaconda管理Python的虚拟环境，所以在Git Bash中的操作无法直接找到Python环境，故而我选择在Anaconda Prompt中进行后续操作

![Emscripten失败GitBash执行安装.png](/resources/2025-03-03-在Windows安装并测试Emscripten全流程/Emscripten失败GitBash执行安装.png)

![Emscripten安装SDK工具.png](/resources/2025-03-03-在Windows安装并测试Emscripten全流程/Emscripten安装SDK工具.png)

- 安装好后，我们就可以为当前终端激活最新版本

![Emscripten激活SDK.png](/resources/2025-03-03-在Windows安装并测试Emscripten全流程/Emscripten激活SDK.png)

- 可以通过如下指令测试当前终端是否激活SDK成功

```
# Windows系统中使用任意一种指令
emcc -v
em++ -v

# 非Windows系统中使用下述指令
./emcc -v
./em++ -v
```

![Emscripten测试安装成功.png](/resources/2025-03-03-在Windows安装并测试Emscripten全流程/Emscripten测试安装成功.png)

- 按照提示，我们关闭当前命令行后，下次重新打开时需要重新输入以下指令重新激活（仍然是临时激活），才能运行`emcc -v`得到正常输出

```
# 为当前终端控制台重新激活emsdk环境变量
emsdk_env.bat
```

### 2.3 配置永久环境变量
- 若想避免每次打开命令行都要重新激活临时环境变量，按照提示可使用如下指令永久激活

```
emsdk.bat activate latest --permanent
```

![Emscripten永久激活SDK.png](/resources/2025-03-03-在Windows安装并测试Emscripten全流程/Emscripten永久激活SDK.png)

- 但我们发现上图中无法读取其中一个环境变量，导致设置完后重新打开命令行输入`emcc -v`仍然报错，所以我们需要手动检查一下环境变量的配置出现了什么问题
- 如下图所示就是上图中自动配置的环境变量，但是不知为何其中`EMSDK`变量值的路径中出现了错误的符号，此处需要将其手动修正为`D:\EMSCRIPTEN\emsdk`（此处自动配置的是用户的环境变量而非系统环境变量，你可以按照自己的需求手动添加一下相同的系统环境变量）

![Emscripten配置环境变量Debug.png](/resources/2025-03-03-在Windows安装并测试Emscripten全流程/Emscripten配置环境变量Debug.png)

- 然后我们打开任意一个命令行测试一下，均成功

![Emscripten配置环境变量成功.png](/resources/2025-03-03-在Windows安装并测试Emscripten全流程/Emscripten配置环境变量成功.png)

## 三、代码编译生成

### 3.1 编译C/C++
- 我们找一块地方创建一个C语言的源文件用于测试，包含以下内容

```c
#include <stdio.h>

int main()
{
    printf("Kon Daisuki\n");
    return 0;
}
```

- 然后我们运行下面的指令来编译代码，若成功则会生成两个文件
	- `a.out.wasm`：一个包含编译后代码的WebAssembly文件
	- `a.out.js`：一个运行时支持以加载和执行`a.out.wasm`的JavaScript文件

```
# 编译C语言源文件
emcc Test.c
# 强制作为C++编译
em++ Test.c
```

- 然后可以使用[NodeJS](https://nodejs.org/en/)来运行上述JavaScript文件，在命令行中得到如下图的对应输出

![Emscripten编译C代码.png](/resources/2025-03-03-在Windows安装并测试Emscripten全流程/Emscripten编译C代码.png)

- 对于如下所示的C++源文件同理，使用`em++ Test.cpp`即可编译

```cpp
#include <iostream>

int main()
{
    std::cout << "Kon Daisuki\n";
}
```

![Emscripten编译C++代码.png](/resources/2025-03-03-在Windows安装并测试Emscripten全流程/Emscripten编译C++代码.png)

### 3.2 生成HTML
- 在C++源文件相同目录下创建一个空HTML文件，使用如下指令指定该文件即可将源文件输出到其上，并类似地新生成共两个`.js`和`.wasm`文件

```
em++ Test.cpp -o Test.html
```

![Emscripten生成HTML.png](/resources/2025-03-03-在Windows安装并测试Emscripten全流程/Emscripten生成HTML.png)

- 不能直接在浏览器中打开`Test.html`，会如下图报错

![Emscripten打开生成的HTML-P1.png](/resources/2025-03-03-在Windows安装并测试Emscripten全流程/Emscripten打开生成的HTML-P1.png)

- 我们先使用如下指令启动HTTP服务

```
emrun --no_browser --port 8080 Test.html
```

![Emscripten打开生成的HTML-P2.png](/resources/2025-03-03-在Windows安装并测试Emscripten全流程/Emscripten打开生成的HTML-P2.png)

- 然后再通过浏览器访问`http://localhost:8080/Test.html`即可在F12控制台中看到文本输出

![Emscripten打开生成的HTML-P3.png](/resources/2025-03-03-在Windows安装并测试Emscripten全流程/Emscripten打开生成的HTML-P3.png)
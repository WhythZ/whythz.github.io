---
# author:
title: 使用Emscripten+CMake构建代码的踩坑
description: >-
  大型项目不免要使用CMakeLists.txt来构建项目，本文以编译一个简单的代码库（无引入第三方库）为例，演示直到获取无漏洞构建结果的过程，并阐述了过程中遇到的问题（包括如何处理异常的抛出、如何将本地文件加载到虚拟文件系统等）以及是如何逐步解决的
date: 2025-03-04 03:01:00 +0800
categories: [编程相关, 高级语言]
tags: [C++, Emscripten, CMake, 环境配置]
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

## 一、基本编译流程
- 关于Emscripten的安装与环境配置，详见这篇：[在Windows安装并测试Emscripten全流程](https://whythz.github.io/posts/%E5%9C%A8Windows%E5%AE%89%E8%A3%85%E5%B9%B6%E6%B5%8B%E8%AF%95Emscripten%E5%85%A8%E6%B5%81%E7%A8%8B/)
- 此处以我的一个[数据结构代码库](https://github.com/WhythZ/DataStructure/tree/5b4587ca1bc337c6d85f3fab5b4bddbf450024c5)中编写完成的[CMakeLists.txt](https://github.com/WhythZ/DataStructure/tree/5b4587ca1bc337c6d85f3fab5b4bddbf450024c5/CMakeLists.txt)为例（两链接均索引到我写该博客时使用的相同提交版本，不受[最新提交版本](https://github.com/WhythZ/DataStructure)所影响，以便读者复刻），将该仓库克隆到本地

```
git clone https://github.com/WhythZ/DataStructure.git
```

- 然后在项目根目录打开cmd，新建一个`build`目录后进入

```
mkdir build
cd build
```

- 此时使用如下命令以使用Emscripten的CMake工具链文件来配置项目（`emcmake`是Emscripten提供的一个包装器，它会自动设置CMake使用Emscripten的工具链）

```
emcmake cmake ..
```

![emcmake构建项目.png](/resources/2025-03-04-使用Emscripten+CMake构建代码的踩坑/emcmake构建项目.png)

![emcmake构建项目生成的文件.png](/resources/2025-03-04-使用Emscripten+CMake构建代码的踩坑/emcmake构建项目生成的文件.png)

- 然后使用`emmake`来调用`make`进行编译（`emmake` 是另一个Emscripten提供的包装器，它会确保使用Emscripten的工具链进行编译）

```
emmake make
```

![emmake编译项目.png](/resources/2025-03-04-使用Emscripten+CMake构建代码的踩坑/emmake编译项目.png)

![emmake编译项目生成的文件.png](/resources/2025-03-04-使用Emscripten+CMake构建代码的踩坑/emmake编译项目生成的文件.png)

- 此时运行上图中的`Main.js`即可得到程序的输出结果，若修改了代码则需重新编译后运行

![emmake编译结果运行.png](/resources/2025-03-04-使用Emscripten+CMake构建代码的踩坑/emmake编译结果运行.png)

## 二、关于异常捕获
- 如果发现报错了如下内容，这是由于代码中抛出了异常，但Emscripten默认禁用了异常捕获功能，启用异常捕获会增加生成的WebAssembly文件的大小，并可能影响运行时性能

![emmake编译结果运行报错.png](/resources/2025-03-04-使用Emscripten+CMake构建代码的踩坑/emmake编译结果运行报错.png)

- 若无需异常处理则检查代码逻辑避免抛出异常，也可在`CMakeLists.txt`中的`add_executable`前添加如下内容强制启用异常捕获，然后重新执行正常的步骤编译代码，最终得到异常输出

```cmake
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -sNO_DISABLE_EXCEPTION_CATCHING")
```

![emmake编译结果运行文件加载报错.png](/resources/2025-03-04-使用Emscripten+CMake构建代码的踩坑/emmake编译结果运行文件加载报错.png)

## 三、文件加载方法

### 3.1 加载单个文件
- 如上图所示，报错源于如下代码，WebAssembly环境中的使用的是虚拟文件系统，若程序尝试打开本地文件则会报错

```cpp
SortingManager::SortingManager()
{
	//加载测试案例文件中的整数列表
	LoadTestCase("../Codes/03-Sorting/IntTestCase.csv");
}

void SortingManager::LoadTestCase(std::string _path)
{
	//加载路径下的文件
	std::ifstream _file(_path);
	//检查是否成功加载
	if (!_file.good())
		throw std::runtime_error("ERROR: File \"IntTestCase.csv\" Not Found!");

	//通过循环来遍历每行内容；getline函数第一参数位接收被读取的文件，第二参数位是存放读取的数据的对象，第三参数位以逗号为分隔符
	std::string _gridBuf;
	while (std::getline(_file, _gridBuf, ','))
	{
		srcList.emplace_back(stoi(_gridBuf));
	}

	//关掉文件
	_file.close();
}
```

- 其中一种解决方式是使用Emscripten的文件预加载功能将所需文件预先加载到虚拟文件系统，首先将被加载的文件放入`build`目录下
- 然后在编译时使用`--preload-file`选项将文件加载到虚拟文件系统中，在`CMakeLists.txt`中的`add_executable`前添加如下内容

```cmake
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -sFORCE_FILESYSTEM --preload-file IntTestCase.csv")
```

- 由于此时的被加载文件处于`build`目录， 但不应将加载路径改为`build/IntTestCase.csv`，这是相对于本地文件系统的路径而非Emscripten虚拟文件系统的路径，后者的根目录是`/`，所有文件路径都需要相对于这个根目录，所以应将代码中的路径修改为如下形式

```cpp
SortingManager::SortingManager()
{
	//加载测试案例文件中的整数列表
	LoadTestCase("/IntTestCase.csv");
}
```

- 然后重新执行`emcmake cmake ..`和`emmake make`后再运行`node Main.js`，程序即可成功加载目标文件并运行，得到如下结果，可以看到基于该`.csv`文件运行的代码成功运行了一部分，这表明该文件被成功读取了

![emmake编译结果读取文件成功但有新报错.png](/resources/2025-03-04-使用Emscripten+CMake构建代码的踩坑/emmake编译结果读取文件成功但有新报错.png)

- 但是仍有新的报错出现，经搜索得出这大概率是由于代码中出现了内存泄漏或是数组越界等问题，经过排查这是某处数组越界导致的（该漏洞于[此条提交](https://github.com/WhythZ/DataStructure/commit/74fc17e87867601da2392a4296386f0370d2887b)中被修复），我们修复该代码后重新编译，再`node Main.js`即可得到无报错的优美输出

![emmake完美编译成功.png](/resources/2025-03-04-使用Emscripten+CMake构建代码的踩坑/emmake完美编译成功.png)

- 可以发现成功构建所得的若干文件中，多了一个`Main.data`文件，其包含了预加载到虚拟文件系统中的数据，除了可以包含此处的`.csv`配置文件外，还可以是图片、音频等资源文件

![emmake虚拟文件系统数据.png](/resources/2025-03-04-使用Emscripten+CMake构建代码的踩坑/emmake虚拟文件系统数据.png)

### 3.2 文件放在子目录
- 当资源或配置文件多起来时，首先我不希望将它们零散地放在`build`目录下，那么可以在`build`内创建一个`Assets`目录，然后将被加载的文件放入，同时修改`CMakeLists.txt`内的加载路径

```cmake
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -sFORCE_FILESYSTEM --preload-file Assets/IntTestCase.csv")
```

- 最后修改代码中加载文件的路径，从`/`即虚拟文件系统根目录索引到`Assets`目录

```cpp
SortingManager::SortingManager()
{
	//加载测试案例文件中的整数列表
	LoadTestCase("/Assets/IntTestCase.csv");
}
```

- 然后重新执行必要的两条指令后即可同样构建运行成功

![emmake虚拟文件系统子目录.png](/resources/2025-03-04-使用Emscripten+CMake构建代码的踩坑/emmake虚拟文件系统子目录.png)

### 3.3 加载多个文件
- 如果有很多需要被加载的文件，我们最好不应该在`CMakeLists.txt`中为每个文件都进行如下的配置，这样十分繁琐且耦合

```cmake
`set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -sFORCE_FILESYSTEM --preload-file Assets/xxx.yyy")`
```

- 只需通过如下语句加载整个目录即可，无论`Assets/`中有多少文件CMake都会自动加载它们

```cmake
# 使用--preload-file加载整个Assets目录
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -sFORCE_FILESYSTEM --preload-file Assets")
```

## 四、生成HTML
- 完成上述流程并成功编译后，遂尝试将代码编译为HTML，需在`CMakeLists.txt`中添加如下指令

```cmake
set(CMAKE_EXECUTABLE_SUFFIX ".html")
```

- 然后同样在`build`中的终端执行完以下两句后，便可得到生成的额外一个`Main.html`文件

```
emcmake cmake ..
emmake make
```

![emmake编译生成HTML.png](/resources/2025-03-04-使用Emscripten+CMake构建代码的踩坑/emmake编译生成HTML.png)

- 然后同理先启用本地HTTP服务器，然后在浏览器中打开`http://localhost:8080/Main.html`网页（而不是直接打开文件）即可看到输出结果

```
emrun --no_browser --port 8080 Main.html
```

![emmake成功打开HTML文件.png](/resources/2025-03-04-使用Emscripten+CMake构建代码的踩坑/emmake成功打开HTML文件.png)
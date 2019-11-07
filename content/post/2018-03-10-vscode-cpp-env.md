---
layout:     post
title:      "在 Visual Studio Code 中构建一个C++开发环境"
subtitle:   "\"Write C++ Everywhere\""
date:       2018-03-10T11:00:00 
author:     "darjun"
image: "/img/post-bg-2015.jpg"
tags:
    - 环境搭建
URL: "/2018/03/10/vscode-cpp-env/"
categories: [ 
    "Tools"    
]
---

## 背景

有时候需要在Windows上编写`C++`代码，但是已经习惯了`linux`下`vim + gcc/clang`，并且不想安装体积庞大的`Visual Studio`。本文介绍如何一步步在Windows上使用`Visual Studio Code`(以下简称`VS Code`)搭建一个C++的开发调试环境。

## 安装 VS Code

> `VS Code`是`Microsoft`开发的免费、开源、跨平台的文本编辑器。它同时支持`Windows`、`Linux`和`MacOS`等操作系统。它支持调试、内置了Git版本控制功能，同时也具有代码补全、代码重构等功能。还支持扩展程序并在编辑器中内置了扩展程序管理的功能。

安装`VS Code`很简单，官网[下载][1]`Windows`版本，双击安装。

安装完成之后，通过扩展程序管理搜索安装`C/C++`扩展。如下：

![安装C/C++扩展](/img/in-post/vs-code-cpp-env/cpp-extension.png)

安装`VIM`扩展。如下：

![安装VIM扩展](/img/in-post/vs-code-cpp-env/vim-extension.png)

安装完成之后重新加载即可生效。

## 安装 msys2

利用`msys2`可以在`Windows`中使用`Linux/Unix`软件。`msys2`提供了一个包管理系统`Pacman`，可以很方便地安装各种软件。

### 1. 安装 msys2

去`msys2`官网[下载](https://www.msys2.org/)对应的安装程序。注意32位和64位系统的差别。

![下载msys2安装包](/img/in-post/vs-code-cpp-env/download-msys2.png)

双击安装，安装完成之后直接运行：

![运行msys2](/img/in-post/vs-code-cpp-env/finish-install-msys2.png)

或者从开始菜单运行:

![运行msys2](/img/in-post/vs-code-cpp-env/start_msys2.png)


### 2. 更新包数据库及核心系统包

在打开的Terminal窗口中，输入`pacman -Syu`:

![更新msys2](/img/in-post/vs-code-cpp-env/pacman-syu.png)

出现下面情况需要关闭Terminal，再次从开始菜单运行，然后输入`pacman -Su`更新剩余的部分:

![更新msys2中断](/img/in-post/vs-code-cpp-env/update-int.png)

![更新msys2剩余部分](/img/in-post/vs-code-cpp-env/update-left-msys2.png)

等待更新完成...

注意点：
* 可能出现获取错误是由于网络原因，会自动重新尝试，一般会成功。
* 有可能出现冲突，直接选`y`。
* 输出更新信息：下载大小、安装大小、净更新大小。输入`y`更新。
* 由于网络状况可能需要较长时间。

***下面`gcc`和`clang`按需安装！！！***

### 3. 安装 gcc

输入`pacman -Ss gcc`搜索`gcc`结果如下:

![搜索gcc](/img/in-post/vs-code-cpp-env/search-gcc.png)

选择安装`mingw-w64-x86_64-toolchain`(64位)，输入`pacman -S mingw-w64-x86_64-toolchain`:

![安装gcc](/img/in-post/vs-code-cpp-env/install-gcc.png)

### 4. 安装 clang
输入`pacman -Ss clang`搜索`clang`结果如下:

![搜索clang](/img/in-post/vs-code-cpp-env/search-clang.png)

选择安装`mingw-w64-x86_64-clang`(64位)，输入`pacman -S  mingw-w64-x86_64-clang`:

![安装clang](/img/in-post/vs-code-cpp-env/install-clang.png)

## 设置 Terminal
Windows上`VS Code`默认的Terminal为`PowerShell`。为了使用`msys2`需要改成`msys2`的`bash`。

选择`文件->首选项->设置`:

![设置](/img/in-post/vs-code-cpp-env/user-setting.png)

左侧是默认设置，我们需要修改右侧的用户设置来覆盖默认的设置。这里设置了以下值：
* `window.zoomLevel`: 窗口缩放，0为不缩放。
* `terminal.integrated.shell.windows`: 设置为`msys2`中`bash`的路径`C:\\msys64\\usr\\bin\\bash.exe`。
* `terminal.integrated.shellArgs.windows`: 启动`bash`的参数，设置为`["-i"]`表示启动`bash`后进入交互模式。
* `terminal.integrated.env.windows`: 启动`bash`的环境变量，设置为`{ "PATH": "/mingw64/bin:/usr/local/bin:/usr/bin:/bin:/c/Windows/System32:/c/Windows:/c/Windows/System32/Wbem:/c/Windows/System32/WindowsPowershell/v1.0/"}`

设置完成后，使用``Ctrl + ` ``打开的Terminal为`bash`。

## 编写程序
打开一个空目录，创建`main.cpp`文件，输入代码。然后`g++ -g main.cpp`编译，`./a.exe`运行:

![gcc编译](/img/in-post/vs-code-cpp-env/gcc-main.png)

也可以使用`clang++ -g main.cpp`编译，`./a.exe`运行：

![clang编译](/img/in-post/vs-code-cpp-env/clang-main.png)


## 调试程序

`VS Code`中输入`Ctrl + Shift + P`，然后选择`C/CPP: Edit Configurations`:

![cpp配置](/img/in-post/vs-code-cpp-env/cpp-edit-config.png)

在与`Win32`同层次上增加以下配置:
```
{
            "name": "MinGW",
            "intelliSenseMode": "clang-x64",
            "includePath": [
                "${workspaceRoot}",
                "C:/msys64/mingw64/include",
                "C:/msys64/mingw64/c++/7.3.0",
                "C:/msys64/mingw64/c++/7.3.0/tr1",
                "C:/msys64/mingw64/c++/7.3.0/x86_64-w64-mingw32",
                "C:/msys64/mingw64/x86_64-w64-mingw32/include"
            ],
            "defines": [
                "_DEBUG",
                "UNICODE",
                "__GNUC__=7",
                "__cdecl=__attribute__((__cdecl__))"
            ],
            "browse": {
                "path": [
                    "C:/msys64/mingw64/lib/*"
                ],
                "limitSymbolsToIncludedHeaders": true,
                "databaseFilename": ""
            }
        }
```

主要配置包含路径，宏定义等内容。如下：

![mingw配置](/img/in-post/vs-code-cpp-env/mingw-config.png)

然后选择`调试->添加配置`修改内容如下：

![调试配置](/img/in-post/vs-code-cpp-env/debug-config.png)

变量窗口，监视窗口，调用堆栈，一些控制按钮能完成基本的调试。

![调试](/img/in-post/vs-code-cpp-env/debug.png)

## 库安装

通过`msys2`的包管理器`pacman`可以很方便的安装一些库。一般先`pacman -Ss`查找，找到自己想要安装的指定版本的库，然后通过`pacman -S`安装。例如下面是如何安装`boost`库的：

输入`pacman -Ss boost`查找：

![查找boost库](/img/in-post/vs-code-cpp-env/search-boost.png)

选择安装`mingw-w64-x86_64-boost`，输入`pacman -S mingw-w64-x86_64-boost`安装：

![安装boost库](/img/in-post/vs-code-cpp-env/install-boost.png)

使用:

![使用boost库](/img/in-post/vs-code-cpp-env/boost-any.png)


## 参考资料
1. [GCC & clang on windows with Visual Studio Code + bash terminal + debugging][2]
2. [CppCon 2017: Rong Lu “C++ Development with Visual Studio Code”][3]
3. [MSYS2 homepage][4]

[1]: https://code.visualstudio.com/Download
[2]: https://www.youtube.com/watch?v=TLh--v8OxHE
[3]: https://www.youtube.com/watch?v=rFdJ68WbkdQ
[4]: https://www.msys2.org/
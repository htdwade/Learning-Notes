## 1. VSCode下载安装

[VSCode下载地址](https://code.visualstudio.com/download)

安装过程中已经将bin目录加入到环境变量Path路径.

## 2. 安装插件

* Chinese (Simplified) Language Pack for Visual Studio Code //简体中文包
* One Dark Pro //主题
* vscode-icons  //图标主题
* Material Icon Theme //图标主题
* Power Mode //打字特效
* Bracket Pair Colorizer 2 //彩虹花括号
* Code Runner //代码运行
* Color Highlight //颜色高亮
* TODO Highlight //todo高亮
* Markdown All in One
* Markdown Preview Enhanced
* Markdown PDF
* LeetCode

## 3. 配置开发环境

### C++

安装插件: C/C++

[VSCode配置C++运行环境](https://www.zhihu.com/question/30315894/answer/154979413)

launch.json:

```json
// https://github.com/Microsoft/vscode-cpptools/blob/master/launch.md
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "(gdb) Launch", // 配置名称，将会在启动配置的下拉菜单中显示
            "type": "cppdbg", // 配置类型，cppdbg对应cpptools提供的调试功能；可以认为此处只能是cppdbg
            "request": "launch", // 请求配置类型，可以为launch（启动）或attach（附加）
            "program": "${fileDirname}/${fileBasenameNoExtension}.exe", // 将要进行调试的程序的路径
            "args": [], // 程序调试时传递给程序的命令行参数，一般设为空即可
            "stopAtEntry": false, // 设为true时程序将暂停在程序入口处，相当于在main上打断点
            "cwd": "${fileDirname}", // 调试程序时的工作目录，一般为${workspaceFolder}；改成${fileDirname}可变为文件所在目录
            "environment": [], // 环境变量
            "externalConsole": false, // 为true时使用单独的cmd窗口，与其它IDE一致；18年10月后设为false可调用VSC内置终端
            "internalConsoleOptions": "neverOpen", // 如果不设为neverOpen，调试时会跳到“调试控制台”选项卡，你应该不需要对gdb手动输命令吧？
            "MIMode": "gdb", // 指定连接的调试器，可以为gdb或lldb。但我没试过lldb
            "miDebuggerPath": "gdb.exe", // 调试器路径，Windows下后缀不能省略，Linux下则不要
            "setupCommands": [
                { // 模板自带，好像可以更好地显示STL容器的内容，具体作用自行Google
                    "description": "Enable pretty-printing for gdb",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": false
                }
            ],
            "preLaunchTask": "Compile" // 调试会话开始前执行的任务，一般为编译程序。与tasks.json的label相对应
        }
    ]
}
```

task.json:

```json
// https://code.visualstudio.com/docs/editor/tasks
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Compile", // 任务名称，与launch.json的preLaunchTask相对应
            "command": "g++", // 要使用的编译器
            "args": [
                "${file}",
                "-o", // 指定输出文件名，不加该参数则默认输出a.exe，Linux下默认a.out
                "${fileDirname}/${fileBasenameNoExtension}.exe",
                "-g", // 生成和调试有关的信息
                "-Wall", // 开启额外警告
                "-static-libgcc", // 静态链接libgcc，一般都会加上
                "-std=c++17", // C++最新标准为c++17，或根据自己的需要进行修改
            ], // 编译命令参数
            "type": "process", // process是vsc把预定义变量和转义解析后直接全部传给command；shell相当于先打开shell再输入命令，所以args还会经过shell再解析一遍
            "group": {
                "kind": "build",
                "isDefault": true // 不为true时ctrl shift B就要手动选择了
            },
            "presentation": {
                "echo": true,
                "reveal": "always", // 执行任务时是否跳转到终端面板，可以为always，silent，never。具体参见VSC的文档
                "focus": false, // 设为true后可以使执行task时焦点聚集在终端，但对编译C/C++来说，设为true没有意义
                "panel": "shared" // 不同的文件的编译信息共享一个终端面板
            },
            "problemMatcher":"$gcc" // 此选项可以捕捉编译时终端里的报错信息
        }
    ]
}
```

c_cpp_properties.json:

```json
{
    "configurations": [
        {
            "name": "Win32",
            "includePath": [
                "${workspaceFolder}/**"
            ],
            "defines": [
                "_DEBUG",
                "UNICODE",
                "_UNICODE"
            ],
            "compilerPath": "E:\\mingw64\\bin\\gcc.exe",
            "cStandard": "c11",
            "cppStandard": "c++17",
            "intelliSenseMode": "gcc-x64"
        }
    ],
    "version": 4
}
```



另外需要在`E:\mingw64\lib\gcc\x86_64-w64-mingw32\8.1.0\include\c++\bits`文件夹中加入`stdc++.h`文件，才能使用万能头文件`<bits/stdc++.h>`。

```c++
// C++ includes used for precompiling -*- C++ -*-

// Copyright (C) 2003-2018 Free Software Foundation, Inc.
//
// This file is part of the GNU ISO C++ Library.  This library is free
// software; you can redistribute it and/or modify it under the
// terms of the GNU General Public License as published by the
// Free Software Foundation; either version 3, or (at your option)
// any later version.

// This library is distributed in the hope that it will be useful,
// but WITHOUT ANY WARRANTY; without even the implied warranty of
// MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
// GNU General Public License for more details.

// Under Section 7 of GPL version 3, you are granted additional
// permissions described in the GCC Runtime Library Exception, version
// 3.1, as published by the Free Software Foundation.

// You should have received a copy of the GNU General Public License and
// a copy of the GCC Runtime Library Exception along with this program;
// see the files COPYING3 and COPYING.RUNTIME respectively.  If not, see
// <http://www.gnu.org/licenses/>.

/** @file stdc++.h
 *  This is an implementation file for a precompiled header.
 */

// 17.4.1.2 Headers

// C
#ifndef _GLIBCXX_NO_ASSERT
#include <cassert>
#endif
#include <cctype>
#include <cerrno>
#include <cfloat>
#include <ciso646>
#include <climits>
#include <clocale>
#include <cmath>
#include <csetjmp>
#include <csignal>
#include <cstdarg>
#include <cstddef>
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <ctime>

#if __cplusplus >= 201103L
#include <ccomplex>
#include <cfenv>
#include <cinttypes>
#include <cstdalign>
#include <cstdbool>
#include <cstdint>
#include <ctgmath>
#include <cuchar>
#include <cwchar>
#include <cwctype>
#endif

// C++
#include <algorithm>
#include <bitset>
#include <complex>
#include <deque>
#include <exception>
#include <fstream>
#include <functional>
#include <iomanip>
#include <ios>
#include <iosfwd>
#include <iostream>
#include <istream>
#include <iterator>
#include <limits>
#include <list>
#include <locale>
#include <map>
#include <memory>
#include <new>
#include <numeric>
#include <ostream>
#include <queue>
#include <set>
#include <sstream>
#include <stack>
#include <stdexcept>
#include <streambuf>
#include <string>
#include <typeinfo>
#include <utility>
#include <valarray>
#include <vector>

#if __cplusplus >= 201103L
#include <array>
#include <atomic>
#include <chrono>
#include <codecvt>
#include <condition_variable>
#include <forward_list>
#include <future>
#include <initializer_list>
#include <mutex>
#include <random>
#include <ratio>
#include <regex>
#include <scoped_allocator>
#include <system_error>
#include <thread>
#include <tuple>
#include <typeindex>
#include <type_traits>
#include <unordered_map>
#include <unordered_set>
#endif
```

### Java

安装插件：Java Extension Pack(包括6个扩展)

[VSCode配置Java运行环境](https://code.visualstudio.com/docs/java/java-tutorial)

### Go

安装插件：Go

[VSCode配置Go运行环境](https://code.visualstudio.com/docs/languages/go)

### Python

安装插件：Python

[VSCode配置Python运行环境](https://code.visualstudio.com/docs/python/python-tutorial)

### Remote Development

安装插件：Remote Development(包括3个扩展)

[VSCode配置remote ssh](https://zhuanlan.zhihu.com/p/64849549)

配置`~.ssh/config`:

```bash
# Read more about SSH config files: https://linux.die.net/man/5/ssh_config
Host 腾讯云主机
    HostName 111.230.92.98
    User root
```

![](https://raw.githubusercontent.com/htdwade/PicBed/master/img/20191224215300.png)

将本机的公钥复制到远程机器的authorized_keys文件中，打开git bash，敲入以下命令：

```bash
ssh-copy-id root@111.230.92.98
```

现在无需密码就能连接远程主机

![](https://raw.githubusercontent.com/htdwade/PicBed/master/img/20191224220310.png)
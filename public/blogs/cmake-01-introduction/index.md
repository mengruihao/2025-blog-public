
> 本文系统讲解 CMake 在 C/C++ 项目中的作用，重点梳理 CMake、CMakeLists、Make、Makefile 之间的关系， 并以 Windows + MinGW 环境为例，完整展示从源码到可执行文件的构建流程。个人理解，如有错误欢迎指出!

**关键词**：CMake，CMakeLists.txt，Make，Makefile，构建系统，MinGW，gcc，g++

<br>

# 1.CMake、CMakeLists、Make、Makefile 的爱恨情仇

在正式介绍 CMake、CMakeLists、Make、Makefile 之前我们先规定一个说法：本文中所说的 “ 配置文件 ”，**特指 “ 构建配置文件 ”**，即用于描述编译与链接规则的文件。

## 1.1.Make 与 Makefile

> **Make** 是一个在软件开发中所使用的构建工具，用于自动化建构软件。

Make 通过名为 Makefile 的配置文件来描述源代码文件之间的依赖关系和构建规则。而 Make 会根据这些规则和依赖关系，判断哪些文件需要重新编译，并执行相应的编译命令，以确保最终生成可执行文件或其他目标文件（这些目标被称为“target”）。这一特点使得 Make 在较大型的项目中尤为方便。大多数情况下，Make 被用于将源代码（.c 或 .cpp）编译为目标文件（.o），再把目标文件链接起来生成可执行文件或者库文件。我们得出第一个关键理解：**计算机使用 make 按照 Makefile 中的内容去编译程序**。

值得注意的是，Make 这个工具的规范与概念来源于 Unix。如今我们在 Linux 或Windows 里使用的 Make 都是在此基础上重新实现的。比如 Linux 基本都默认装有 GNU make，而 Windows 其本身并没有提供 Make,因此我们需要下载 MinGW，使用里面的 mingw32-make 去实现 Make。此外 MinGW 还为我们提供了 gcc、g++ 等工具，用于构建完整的编译工具链（这里解释了在 Windows 下配置 C/C++ 环境需要安装MinGW 的根本原因）。简单来说，如今我们常说的 make 并不特指某一个具体的程序，而是一个 **泛指的概念**，代表 “ 由生成器所指定的构建工具 ”。在不同平台和生成器下，它可能对应 make、mingw32-make、nmake 等不同工具。

## 1.2.CMake（Cross platform Make）与 CMakeLists

> CMake 是是一个跨平台的 **构建系统生成器**（build-system generator）。

CMake 通过配置文件描述建构过程（build process）的方式和 Make 相似，只是 CMake 的配置文件为 CMakeLists。CMake 是通过配置文件 CMakeLists 来描述构建规则。例如 Unix 下 CMake 通过 CMakelists 生成 Makefile 描述构建规则 ，Windows Visual C++ 则生成工程文件（.vcxproj）和解决方案文件（.sln）描述构建规则。我们可以选择不同的生成器去生成不同的配置文件。也就是说 CMake 并不等同于 Makefile 生成器，而是可以通过选择对应的生成器生成多种不同构建系统所需的配置文件。不同构建工具在 CMake 体系中的角色如下表所示，这里我只举两个例子以便于理解：

|生成器|执行谁生成的规则|调用的编译器|常见平台|
| --- | --- | --- | --- |
|mingw32-make|CMake → MinGW Makefiles|gcc / g++|Windows + MinGW|
|nmake|CMake → NMake Makefiles|cl.exe|Windows + MSVC|

我们在工程中会让 CMake 读取 CMakeLists，并在 `build/` 目录中生成适用于所选构建系统（如 Makefile）的构建文件。但为了适用 mingw32-make 我们可以通过 `cmake -B build -G "MinGW Makefiles"` 中 `-G` 参数去选择生成器，这里我选择的就是适用于 mingw32-make 的生成器 MinGW Makefiles（在后面我们还会详细讲解）。这里我们只拿 C/C++ 举例，其构建系统流程如下：

进入 gcc/g++ 后，处理流程如下：

不难看出在 C/C++ 环境下，我们的建构过程是这样的：通过 CMake 去选择 MinGW Makefiles 生成器生成 Makefile 去描述源码。

由此我们得出第二个关键理解：**工程中 CMake 通过合适的生成器翻译 CMakeLists 生成对应的配置文件，如 Makefile**。（请思考这句话中 “ 对应的配置文件 ” 指的是什么）

现在不妨总结一下，我们将上文的两个理解结合起来可以得出 CMake 的工作流程：先由开发者写 CMakeLists ，然后通过 `cmake -B build -G "XXX"` （“ XXX ” 参数是生成器，后面会详细介绍）命令去选择生成器，并且让 CMake 将开发者写的 CMakeLists 翻译成对应的配置文件。最后我们调用 make 让计算机去按照配置文件编译程序，构建可执行文件。如果限定在 C/C++ 的环境下，CMake的工作流程为：**先由开发者写 CMakeLists ，然后通过 `cmake -B build -G "MinGW Makefiles"` 命令去选择生成器，并且让 CMake 将开发者写的 CMakeLists 翻译成 Makefile。最后我们通过`MinGW32-make` 命令调用 MinGW32-make 让计算机去按照 Makefile 编译程序，构建可执行文件。**（请按照此过程描述一下选择 nmake 生成器时 CMake 的工作流程）。这里还需要我们额外明确两个概念：

* CMake 执行 `cmake -B build -G "XXX"` 时我们称为 **Configure 阶段**（配置阶段） 。
* Make 执行 `MinGW32-make` 时我们称为 **build 阶段**。

**【本节总结】**

* 开发者写 CMakeLists
* CMake 根据生成器生成构建配置文件
* 构建工具（make / nmake）读取配置文件
* 编译器完成真正的编译与链接

现在我们已经了解 CMake 的工作流程了，下面将学习如何在 VS Code 中使用 CMake。

<br>

---

---
<br>

# 2.在 VScode 中使用 CMake

在此部分我将默认你已经下载安装 CMake、MinGW 和 VS Code 并完成添加环境变量，这里我仅提供验证你是否安装成功的方法。如果你还没有，请自行搜索教程并安装。

## 2.1.验证 CMake 与 MinGW

* `Win` + `R` 输入 `cmd` 打开终端，输入 `CMake --version`，输出类似如下则证明 CMake 安装成功。cmake version 后面的数字是你安装 cmake 的版本。
![CMake1|690x409](upload://uGCyJRK7qGBhItoqtglTtqTc8ee.png)

* 输入`gcc -v`，输出类似如下则证明 MinGW 安装成功。
![cmake2|690x458](upload://4zArIOXYZsQ24CWiQpZEhI8pjB9.png)

<br>

## 2.2.VS Code 中使用 CMake

* 打开 VS Code 点击左侧插件选项，在搜索栏里搜索如下插件并下载
![cmake9|690x411](upload://55hb74AAa1kBP3DTN966UavhGIV.png)


* 自行选择位置，新建名为 `HelloWorld` 的文件夹
![cmake3|690x386](upload://mxEZaWJsUmh0ccu1wB9TbiBaTxj.png)

* 打开 VS Code，点击 `打开文件夹` 选择之前新建的文件夹 `HelloWorld`，点击 `选择文件夹` 打开
![cmake4|690x411](upload://zGC2bGu8Czo9PFRH0ur6x8s1kDd.png)

* 点击新建文件，输入 `hello.c` 如果是 C++ 后缀改为 `.cpp`
![cmake5|690x411](upload://2qhye2H8YXIDkjV9CKnAzwtw21q.png)

* 复制下面代码，粘贴到 hello.c 文件里
```
#include<stdio.h>
int main(){
    printf("Hello World!");

    return 0;
}

```






* 点击右上角运行（这一步使用的是 Code Runner 插件，该方式仅用于快速验证环境是否可用，并未使用 CMake 构建系统）
![cmake6|690x411](upload://e4iaSaAOMlJpa7Vs8uL61j7cINX.png)

* 此时会跳出选择编译器，选择 gcc 编译器
![cmake7|690x411](upload://jXBEaVYNl3Ofet0EE3tRfD9zeE0.png)

* 点击终端，输出 Hello World 即为正常
![cmake8|690x411](upload://bdS29B1I7Il4Vwf16EL0giICrsQ.png)

* 在 HellowWorld 文件夹下新建文件，名为 CMakeLists.txt
![cmake10|690x411](upload://50ixKrfonaVBBFJqxbYnIUsZmZr.png)

* 此时你的 VS Code 大概率会提示你选择选择 **工具包（Kit）**，选择带有 mingw32 的编译器。如果没有先跳过，后面步骤出现时再选。（但一直没有出现或者不小心关闭请看 [2.3.2.主动选择生成器](###2.3.2.%E4%B8%BB%E5%8A%A8%E9%80%89%E6%8B%A9%E7%94%9F%E6%88%90%E5%99%A8)）![cmake15|690x431](upload://77HuUuP5O5bqhNcuSHL3ihjNS5y.png)

* 复制如下代码粘贴至 CMakeLists.txt 中，按下 `ctrl` + `s` 保存，VS Code 自动运行。HellowWorld 文件夹多出 build 文件夹，此时 CMake 已经构建出 Makefile。

```
cmake_minimum_required(VERSION 3.20) # 限定 CMake 的最低版本要求
​
project(hello) # 工程文件名称
​
add_executable(hello "hello.c") # hello 时生成可执行文件的名字，hello.c 是源码
```
![cmake11|690x411](upload://ek1JJhgh0sJvxR4yitLLE9GgyyM.png)

* 在终端输入 `cd .\build\` 按下回车，进入 `build` 文件。注意看箭头处两者的区别。
![cmake12|690x411](upload://fpxvntyw9kZyhGcNJWvvmC5NTpg.png)

* 在终端输入 `mingw32-make` 按下回车，出现类似输出证明编译完成。此时 make 按照 Makefile 的内容来编译文件，点开 build 文件可看见多出 hello.exe。
![cmake13|690x411](upload://gZngpudK1oQfq4ZgjhJvlTobHHo.png)

* 在终端输入 `.\hello.exe` 按下回车，终端输出 `Hello World！` 即成功
![cmake14|690x411](upload://pt9gsiN6ik9V6zzYU6v7bWURyXo.png)

<br>

## 2.3.CMake Tool

当按照 [2.2.VS Code 中使用 CMake 完成后](##2.2.VS%20Code%20%E4%B8%AD%E4%BD%BF%E7%94%A8%20CMake)，你也许会感到在 [1.CMake、CMakeLists、Make、Makefile 的爱恨情仇](#%201.CMake%E3%80%81CMakeLists%E3%80%81Make%E3%80%81Makefile%20%E7%9A%84%E7%88%B1%E6%81%A8%E6%83%85%E4%BB%87) 中所讲的 CMake 编译过程在 VS Code 中并没有一个直观的体现（虽然当你熟悉之后会感觉很明显），只有输入 `mingw32-make` 在前文中有直接点明。那我们自然会产生一个问题 —— `cmake -B build -G "MinGW Makefiles"` 命令去哪里了？还记得最开始我们在 VS Code 里下载的几个插件吗？其中有一个 CMake Tool 就是罪魁祸首。

### 2.3.1.CMake Tool 作用

我们在使用 VS Code 的 CMake Tools 插件时，当保存 `CMakeLists.txt` 后插件会自动触发一次 **CMake 配置（configure）过程**。该过程等价于执行 `cmake -B build -G "MinGW Makefiles"`。此外在使用中对 CMakeLists 编辑并保存时 CMake Tools 也会检测工程配置发生的变化，自动执行一次 CMake 配置。在 [2.2.VS Code 中使用 CMake](##2.2.VS%20Code%20%E4%B8%AD%E4%BD%BF%E7%94%A8%20CMake) 中 VS Code 提示过选择 **工具包（Kit）**，我们选择的是 mingw32 工具包。CMake Tools 会在后台执行 `cmake -B build` 命令，并根据选择的工具包自动添加 `-G` 参数。选择指定的工具包避免了手动输入 `cmake -B build -G "MinGW Makefiles"` 来选择生成器构建 Makfile，同时也不需要关心环境变量、路径等细节。此外 CMake Tool 还有图形化界面管理的作用，在 [2.3.2.主动选择生成器](###2.3.2.%E4%B8%BB%E5%8A%A8%E9%80%89%E6%8B%A9%E7%94%9F%E6%88%90%E5%99%A8) 中进行说明。

### 2.3.2.主动选择生成器

在使用中我们会遇到需要手动选择工具包的情况，请参考下面步骤：

* 使用快捷键 `Ctrl` + `shift` + `p` 在搜所栏里输入 `CMake:Select a Kit`![]
![cmake16|690x431](upload://bghCy0xSkSEEeqbTeKUw0wcFset.png)

* 选择你需要的工具包，这里我选用 mingw 工具包
![cmake17|690x431](upload://jkCSTgJMc0II8YVYUheyc4oJnFo.png)

在用快捷键 `Ctrl` + `shift` + `p` 搜索 `CMake:Select a Kit` 时会发现很多以 `CMake:` 为开头的项目，这些都是 CMake Tool 插件内置的图形化功能。其中 `CMake:Configure` 和 `CMake:build` 两个选项与 [1.CMake、CMakeLists、Make、Makefile 的爱恨情仇](#1.CMake%E3%80%81CMakeLists%E3%80%81Make%E3%80%81Makefile%20%E7%9A%84%E7%88%B1%E6%81%A8%E6%83%85%E4%BB%87) 末尾明确的 Configure 和 build 两个概念相对应。如果点击这两个选项则会执行对应阶段的操作，以免去手动输入命令。

![cmake18|690x431](upload://p5dknaXSsssDUvVWKBOH71UJwEq.png)


如果你想了解 CMakLists 的具体语法推荐前往：https://www.bilibili.com/video/BV1Tw411s7Pk/

<br><br><br>
**版权与许可说明**

© 2026 Meng Ruihao

© 2026 星期六不太累

本教程采用 知识共享署名-相同方式共享 4.0 国际许可协议（CC BY-SA 4.0）发布。

在遵守署名和相同方式共享的前提下，您可以自由复制、传播和修改本作品。

许可协议全文：https://creativecommons.org/licenses/by-sa/4.0/

<br>

### References

[1] CMake 官方文档（CMake-Generators）：https://cmake.org/cmake/help/latest/manual/cmake-generators.7.html

[2] VS Code CMake Tools 插件文档：https://github.com/microsoft/vscode-cmake-tools

[3] GNU Make 官方手册：https://www.gnu.org/software/make/manual/

[4] MinGW-w64 官方网站：https://www.mingw-w64.org/

[5] CMake 环境搭建与 VS Code 配置：https://www.bilibili.com/video/BV1FAkuBhE9W/

[6] VScode tasks.json 和 launch.json 的设置：https://zhuanlan.zhihu.com/p/92175757
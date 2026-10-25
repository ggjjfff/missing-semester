# 第二课学习笔记：CMake 与 catkin

> 课程：27 赛季控制视觉招新培训第二课
>
> 说明：这份笔记是我边跑命令边记的。命令都是我实际敲过的，输出也是从终端里直接拷的。
> `✍️` 标记的地方是我自己的理解和感受，还没写完的部分我会标 `TODO`。
>
> 环境：`uname -srm` → Linux x86_64，g++ 15.2.0、GNU Make 4.4.1、CMake 4.2.3。
> 命令的完整输出我自己留了一份在本机（没放进仓库）。
>
> 一个说明：**catkin 那一节我没有动手跑**，因为机器上没装 ROS。
> 那部分只写了读文档的理解，凡是没验证的地方我都标了「没验证」，不装懂。

## 0. 这节课要做什么

课堂主线是一个和机器人有关的小任务：**给底盘左右轮的角速度，算出车体的前进速度和转向角速度**。

按老师的顺序，同一个计算会被做四遍，一遍比一遍工程化：

1. 用一个 `.cpp` 文件直接 `g++` 编译
2. 拆成头文件 + 实现文件，手动编译和链接
3. 用 Makefile 保存构建规则
4. 用 CMake 描述库、应用和测试
5. 最后搬到 ROS 工作空间里，用 catkin 编译两个相互依赖的包

> ✍️ 我一开始的想法：直接 `g++ main.cpp` 不就行了吗，为什么要绕这么多层？
>
> 我现在的理解：TODO

## 1. 从源代码到可执行程序

计算机只认二进制。C++ 是给人看的，要经过编译器的四步翻译。

| 步骤 | 做什么 | 我跑的命令 | 产物 |
|---|---|---|---|
| 预处理 | 展开 `#include`、展开宏、去注释 | `g++ -E hello.cpp -o hello.i` | `hello.i` |
| 编译 | C++ → 汇编 | `g++ -S hello.i -o hello.s` | `hello.s` |
| 汇编 | 汇编 → 机器码 | `g++ -c hello.s -o hello.o` | `hello.o` |
| 链接 | 合并目标文件、接上库 | `g++ hello.o -o hello` | `hello` |

实际跑出来：

```
$ wc -l hello.cpp hello.i
     6 hello.cpp
 36938 hello.i
 36944 total
```

6 行的源文件，预处理之后变成 36938 行。这个数字让我第一次真正感觉到 `#include <iostream>` 到底做了什么——它把整个头文件的内容**原地抄进来**，一行不多一行不少。

体积对比：

| 文件 | 大小 | 说明 |
|---|---|---|
| `hello.cpp` | 95 B | 我写的 |
| `hello.i` | 903 K | 展开后，暴涨九千多倍 |
| `hello.s` | 960 B | 汇编，反而比 `.i` 小了将近一千倍 |
| `hello.o` | 1.9 K | 机器码，比汇编还小 |
| `hello` | 16 K | 链接后的可执行文件，比 `.o` 大了八倍 |

这五个数字连起来看很有意思：`.i` 最大，因为它还没被翻译，全是文本；`.s` 一下缩到 960 字节，因为编译器把没用到的东西都丢掉了；`.o` 更小；而最后链接出来的 `hello` 一下跳到 16K，多出来的是 C++ 标准库带进来的东西。

再往下拆一层，用 `readelf -S hello.o` 看这个目标文件里到底装了什么：

```
$ readelf -S hello.o
There are 14 section headers, starting at offset 0x3f0:

  [ 1] .text             PROGBITS   0000000000000000  00000040
       000000000000003a  0000000000000000  AX       0     0     1
  [ 2] .rela.text        RELA       0000000000000000  000002e8
       0000000000000078  0000000000000018   I      11     1     8
  [ 3] .data             PROGBITS   0000000000000000  0000007a
       0000000000000000  0000000000000000  WA       0     0     1
  [ 4] .bss              NOBITS     0000000000000000  0000007a
       0000000000000000  0000000000000000  WA       0     0     1
  [ 5] .rodata           PROGBITS   0000000000000000  0000007a
       000000000000000c  0000000000000000   A       0     0     1
```

我拿这几个数字对了一下，结果挺震：

- `.text` 的 Size 是 `0x3a`，也就是 58 字节——我那个 `main` 函数编出来只有 58 字节机器码。
- `.rodata` 是 `0x0c`，也就是 **12 字节**。我打印的字符串 `"hello cmake"` 是 11 个字符，加上结尾的 `\0` 正好 12 字节。**对得上**。
- `.data` 和 `.bss` 的 Size 都是 `0`。因为我的 `hello.cpp` 里**一个全局变量都没定义**，所以这两个段是空的。

这就把书上那句「`.data` 放已初始化的全局变量、`.bss` 放未初始化的」验证了——这两个段空着不是因为我理解错了，是因为我真的没写全局变量。

`.rela.text` 那个 0x78 字节是我第一次见到的：它记录「`.text` 里哪些位置还需要链接器来填」。因为 `cout` 和 `endl` 的地址此刻还不知道，只能先留个空，等链接时再补。

> ✍️ 我对「链接」这一步的理解（之前一直模糊）：TODO
>
> 提示自己：`hello.o` 里 `cout` 是怎么被找到的？为什么会报 `undefined reference`？
>
> 再提示：`.rela.text` 这个「重定位表」就是为了解决这件事——它就是「待填地址清单」。

## 2. Makefile

Makefile 是一份「自动编译说明书」，告诉 `make`：哪些文件要编译、顺序是什么、用什么命令。

我手写的第一个版本（故意写最傻的，先跑通再说）：

```makefile
hello: hello.o
	g++ hello.o -o hello

hello.o: hello.cpp
	g++ -c hello.cpp -o hello.o

clean:
	rm -f hello hello.o hello.i hello.s

.PHONY: clean
```

> ⚠️ 坑：缩进必须是 **Tab**，空格会报 `missing separator`。
> 怎么验证是 Tab：`cat -A Makefile`，行首应该看到一个 `^I`。

我特意先手写了一个用 4 个空格缩进的错误版本：

```
$ cat -A Makefile.bad
hello: hello.o$
    g++ hello.o -o hello$
```

两行都是 `$` 结尾，第二行开头是四个空格，**没有 `^I`**。然后：

```
$ make -f Makefile.bad
Makefile.bad:2: *** missing separator.  Stop.
```

改成 Tab 之后，`cat -A` 变成了这样：

```
$ cat -A Makefile.bad
hello: hello.o$
^Ig++ hello.o -o hello$
```

再跑就过了。区别只有第二行开头那个 `^I`。

我的理解：`make` 不是看「缩进」这个概念，它是靠**行首是不是 Tab** 来判断「这一行是上一行的命令」。不是 Tab 就当成新的规则，而新规则必须有冒号（`目标: 依赖`），字符串 `g++ hello.o -o hello` 里没有冒号，于是它就报「缺分隔符」。

跑通之后我换成了带变量的版本。因为头文件和源文件分开放了（`include/` 和 `src/`），编译时要加 `-Iinclude`：

```makefile
CXX      = g++
CXXFLAGS = -Wall -std=c++17 -Iinclude
TARGET   = diff_drive
OBJS     = src/main.o src/diff_drive.o

$(TARGET): $(OBJS)
	$(CXX) $(OBJS) -o $(TARGET)

src/%.o: src/%.cpp include/diff_drive.h
	$(CXX) $(CXXFLAGS) -c $< -o $@
```

`$<` 是「第一个依赖」，`$@` 是「目标名」，`src/%.o: src/%.cpp` 是模式规则——这三个符号我看了半天才记住。

这个正式版一共 15 行（`wc -l Makefile` 的结果），比第一个版本只多了几行。

关键体验：什么都不改直接 `make`，它什么都不做：

```
$ make
make: 'diff_drive' is up to date.
```

然后我 `touch src/main.cpp` 造了一次修改（只改时间戳，不动内容），再 `make`：

```
$ touch src/main.cpp && make
g++ -Wall -std=c++17 -Iinclude -c src/main.cpp -o src/main.o
g++ src/main.o src/diff_drive.o -o diff_drive
```

只重编了 `main.cpp`，`diff_drive.cpp` 那行没出现。

接着我又 `touch include/diff_drive.h`：

```
$ touch include/diff_drive.h && make
g++ -Wall -std=c++17 -Iinclude -c src/main.cpp -o src/main.o
g++ -Wall -std=c++17 -Iinclude -c src/diff_drive.cpp -o src/diff_drive.o
g++ src/main.o src/diff_drive.o -o diff_drive
```

这次两个 `.o` 都重编了。原因是我在规则里写了 `src/%.o: src/%.cpp include/diff_drive.h`，依赖里包含了头文件。如果漏写，改了头文件它就不会重编，会出现「明明改了却没生效」的鬼故事——这也是为什么 Makefile 里依赖关系写错比写慢更可怕。

三次 `make` 对比下来，我理解到 `make` 判断「要不要重编」的标准是：**目标文件和它依赖的文件，谁的修改时间更新**。`touch` 只改时间不改内容，但 `make` 看不出来，它照样重编。

> ✍️ 这句 `up to date` 是怎么判断出来的？我的猜测：TODO
>
> 提示自己：`make` 比的是文件修改时间（mtime），不是文件内容。所以 `touch` 一下就算「改过」了。
>
> 再想深一层：如果连时间戳都比不出来（比如从 git 刚 clone 下来，所有文件时间都一样），会发生什么？

PPT 里的原话是「Makefile 如同一份自动编译说明书」，我觉得这个比喻挺准的：它只说「谁依赖谁」，不说「怎么算」。

## 3. CMake

### 3.1 它到底解决什么问题

文件和依赖一多，Makefile 的规则行数会爆炸；更麻烦的是 Makefile **绑死了平台和编译器**（Linux 用 Makefile，Windows 用 VS 工程）。

CMake 的做法是：你只写一份 `CMakeLists.txt`，它去**生成**对应平台的构建文件。

```
CMakeLists.txt  ──cmake──>  Makefile  ──make──>  可执行文件
                            （Windows 上可能是 VS 工程）
```

所以 PPT 里那句「CMake 是对 make 工具的生成器」是准确的：**它不是构建工具，是「生成构建工具」的工具**。

### 3.2 最小可用的 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.10)
project(HelloDemo CXX)
add_executable(hello hello.cpp)
```

```bash
mkdir build && cd build
cmake ..
make
./hello
```

`cmake ..` 的输出我抄了一段下来，能看到它做了两件事：先检查编译器能不能用，再生成 `Makefile`：

```
$ cmake ..
-- The CXX compiler identification is GNU 15.2.0
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: /usr/bin/c++ - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Configuring done (0.5s)
-- Generating done (0.1s)
-- Build files have been written to: ~/lesson2_lab/build
```

我数了一下，`--` 开头的行有九行，全是「Detecting / Check / Configuring / Generating」这种状态提示。**没有一行在编译我的代码**——它到 `Generating done` 就停了。真正的编译是后面 `make` 干的事。

> ⚠️ 坑：`cmake ..` 里的两个点不能漏。我特意在 `build/` 里敲了一次 `cmake .`：

```
$ cmake .
CMake Warning:
  Ignoring extra path from command line:

   "."


CMake Error: The source directory "~/lesson2_lab/build" does not appear to contain CMakeLists.txt.
Specify --help for usage, or press the help button on the CMake GUI.
```

它去 `build/` 里面找 CMakeLists.txt，当然找不到。

有意思的是这个坑**只在空的 `build/` 目录里才复现**。如果 `build/` 里已经有上次的 `CMakeCache.txt`，`cmake .` 会直接从缓存里读出源码路径，一声不响就跑过去了。我第一次遇到这种情况还以为自己打对了，其实只是缓存救了它。

顺带记住：`CMakeCache.txt` 会记住上次的配置。所以改了 CMakeLists 里缓存变量的默认值却不生效时，删掉整个 `build/` 最省事。

这个「在外面建 build 目录」的做法叫**外部构建**，好处是生成的垃圾文件不会污染源码目录，删掉 `build/` 就等于彻底清理。

### 3.3 常用指令（对照菜鸟教程整理）

| 指令 | 作用 | 我跑过吗 |
|---|---|---|
| `cmake_minimum_required(VERSION x.y)` | 声明最低版本，必须放最前面 | ✅ |
| `project(名字 CXX)` | 项目名 + 语言 | ✅ |
| `add_executable(目标 源文件...)` | 生成可执行文件 | ✅ |
| `add_library(目标 STATIC/SHARED 源文件...)` | 生成库 | ✅（`STATIC` 试过，`SHARED` 没试） |
| `target_link_libraries(目标 库...)` | 把库链接到目标 | ✅ |
| `target_include_directories(目标 PUBLIC/PRIVATE 路径)` | 头文件搜索路径 | ✅（专门做了对照实验，见 3.4） |
| `set(变量 值)` | 定义变量，用 `${变量}` 取 | ✅ |
| `find_package(包名 REQUIRED)` | 查找外部依赖 | ❌ 还没试，因为手上没有现成的第三方库 |

### 3.4 PUBLIC / PRIVATE / INTERFACE

菜鸟教程里说这是「CMake 初学者最常见的坑」，我看了两遍才懂：

| 关键字 | 自己能用 | 依赖我的人能用 | 什么场景 |
|---|---|---|---|
| PRIVATE | ✅ | ❌ | 只有内部实现用到的头文件 |
| PUBLIC | ✅ | ✅ | 库的公开 API |
| INTERFACE | ❌ | ✅ | 纯头文件库（自己不用编译） |

光看表我还是糊的，所以我做了个对照实验。因为我把 `diff_drive.h` 放在了 `include/`，而不是和 `main.cpp` 同一个目录，这样差别就能暴露出来。

一开始写成 `PUBLIC`：

```cmake
add_library(diff_drive_lib STATIC src/diff_drive.cpp)
target_include_directories(diff_drive_lib PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)
add_executable(diff_drive_cmake src/main.cpp)
target_link_libraries(diff_drive_cmake PRIVATE diff_drive_lib)
```

编译通过。然后我只把 `PUBLIC` 改成 `PRIVATE`，其它一个字没动，重新 `cmake .. && make`：

```
$ cmake .. && make
[ 50%] Built target diff_drive_lib
[ 75%] Building CXX object CMakeFiles/diff_drive_cmake.dir/src/main.cpp.o
~/lesson2_lab/src/main.cpp:2:10: fatal error: diff_drive.h: No such file or directory
    2 | #include "diff_drive.h"
      |          ^~~~~~~~~~~~~~
compilation terminated.
make[2]: *** [CMakeFiles/diff_drive_cmake.dir/build.make:79: CMakeFiles/diff_drive_cmake.dir/src/main.cpp.o] Error 1
make[1]: *** [CMakeFiles/Makefile2:122: CMakeFiles/diff_drive_cmake.dir/all] Error 2
make: *** [Makefile:91: all] Error 2
```

注意第一行：`[ 50%] Built target diff_drive_lib` —— **库自己编译成功了**。它自己那条 `-I` 还在，`PRIVATE` 对「自己」当然生效。

挂掉的是 `[75%]` 那个可执行文件：`main.cpp` 找不到 `diff_drive.h`。

原因：`PRIVATE` 的意思是「这条 include 路径只给我自己用，谁链接我，我不告诉他」。而 `main.cpp` 需要 `#include "diff_drive.h"`，它是库的**使用者**，属于「下游」，所以拿不到这条路径。

改回 `PUBLIC` 之后立刻就好了，连报错都没有，直接 `[100%] Built target`。

> ✍️ 我的理解（打个比方）：TODO
>
> 我想到的类比是：PRIVATE 像自己抽屉里的说明书，PUBLIC 像贴在门上的公告……
>
> 我的补充：所以判断标准其实很简单——**问一句「别人 include 我的头文件时，需要这个路径吗」**。
> 需要就是 PUBLIC（或者 INTERFACE），不需要就是 PRIVATE。
> 而 `main.cpp` 之所以受影响，正因为它是「别人」。

### 3.5 我看 `build/Makefile` 的发现

我原以为 CMake 只是在 Makefile 外面套了个壳，结果打开一看：

```bash
cd build
wc -l Makefile
```

```
$ wc -l Makefile
222 Makefile
```

我手写的那个只有 15 行，CMake 生成了 **222 行**，差不多十五倍。

```bash
head -20 Makefile
```

```
$ head -20 Makefile
# CMAKE generated file: DO NOT EDIT!
# Generated by "Unix Makefiles" Generator, CMake Version 4.2

# Default target executed when no arguments are given to make.
default_target: all
.PHONY : default_target

# Allow only one "make -f Makefile2" at a time, but pass parallelism.
.NOTPARALLEL:

#=============================================================================
# Special targets provided by cmake.

# Disable implicit rules so canonical targets will work.
.SUFFIXES:

# Disable VCS-based implicit rules.
% : %,v
```

第一行就写着 `DO NOT EDIT!`。

> ✍️ 这里我本来以为 cmake 只是简单翻译一下，结果发现它生成了…… TODO
>
> 提示：看开头那些 `default_target`、`.NOTPARALLEL`、`.SUFFIXES`、`.PHONY`，
> 它顺手把「默认目标」「禁止并行」「关掉隐式规则」这些我根本没想到的活都干了。
> 我手写的时候只关心「怎么编」，它关心的是「怎么在各种环境下都编得对」。
> 所以它不是翻译，是把整套工程管理模板都铺了一遍。

## 4. catkin

> **先说清楚**：我这台机器上没装 ROS，`catkin_make` 是 `command not found`，所以这一节**全部是读文档和 PPT 的理解，没有动手跑过**。
> 凡是没跑过的地方我都写明「没验证」，不装懂。

### 4.1 catkin 是什么

ROS 用的是 CMake，但做了扩展，这套扩展 + 约定就是 catkin。

所以：**catkin 不是另一个构建系统，它是 CMake 的一层封装**，给 CMake 加了一堆 ROS 专用的宏（比如 `catkin_package()`、`generate_messages()`）。

这一点我是从「catkin 的包里面也有一个 `CMakeLists.txt`」推出来的——如果它是独立的构建系统，就不该复用 CMake 的配置文件格式。

### 4.2 建工作空间

按照官方 tutorial 和 PPT，流程是：

```bash
mkdir -p ~/catkin_ws/src
cd ~/catkin_ws/src
catkin_init_workspace
cd ~/catkin_ws
catkin_make
```

跑完之后应该有：

```bash
ls ~/catkin_ws
# build  devel  src
```

| 目录 | 放什么 | 我要不要管 |
|---|---|---|
| `src` | 源代码、功能包 | 天天在这 |
| `build` | 编译中间文件、二进制 | 基本不用看 |
| `devel` | 开发期的可执行文件、库、setup.sh | 要 source 它 |

> ⚠️ 易错点（查文档得到，不是我踩的）：PPT 第 6 页写的是 `catkin build`，但 ROS **自带的是 `catkin_make`**，`catkin build` 需要另外装 `catkin_tools` 才有。两者都能编译，区别是：
> - `catkin_make`：所有包共用一个 build 空间，一个包挂了全都编不动
> - `catkin build`：每个包独立的 build/devel 空间，互不影响
>
> 因为没装 ROS，这条我**没有实测**，只记下来提醒自己以后别照抄 PPT 里的命令。

### 4.3 别忘了 source

```bash
source ~/catkin_ws/devel/setup.bash
```

不 source 的话，`rosrun` 找不到你刚编译的包。想省事就写进 `~/.bashrc`。

我理解这一步的必要性来自上面对 `devel` 的定位：编译产物不在系统默认的搜索路径里，得靠这个脚本把路径塞进环境变量。

**这里有个我发现的问题**：我把 PPT 18 页的文字全过了一遍，`source`、`setup.bash`、`ROS_PACKAGE_PATH` 这几个词**一次都没出现**。PPT 只讲了 `devel` 是「放置开发过程中的可执行文件和库」，但没讲这些文件怎么才能被 `rosrun` 找到。

也就是说，照着 PPT 敲完 `catkin build` 就去 `rosrun`，是会找不到包的。这一步必须自己补。

按文档，`setup.bash` 做的事是往 `ROS_PACKAGE_PATH` 这类环境变量里追加 `devel` 相关路径。**因为没装 ROS，我没有实测过**，这一句只是我读文档的理解。

> ✍️ 这里我先不写「我忘了 source 然后报错」，因为真没跑过。等装了 ROS 再补。

### 4.4 建功能包

```bash
cd ~/catkin_ws/src
catkin_create_pkg test_pkg std_msgs rospy roscpp
```

按文档，生成的目录是：

```
test_pkg/
├── CMakeLists.txt
├── package.xml
├── include/test_pkg/
└── src/
```

两个核心文件：

| 文件 | 作用 | 类比 |
|---|---|---|
| `package.xml` | 包的身份 + 依赖清单 | 身份证 + 采购单 |
| `CMakeLists.txt` | 构建规则（调 catkin 的宏） | 车间装配图 |

> ✍️ 我打开这两个文件看到的内容：TODO（有 ROS 之后贴几行关键的，比如 package.xml 里的 `<depend>`，CMakeLists.txt 里 `find_package(catkin REQUIRED COMPONENTS ...)`）

### 4.5 验证依赖

```bash
rospack depends1 test_pkg
```

输出应该就是刚才 `catkin_create_pkg` 后面跟的那几个（`std_msgs`、`rospy`、`roscpp`）。

> ✍️ 这个命令有用在哪：TODO（我没 ROS 没跑过）
>
> 但我猜它和 CMake 的 `find_package` 是一对：`package.xml` 里声明依赖是为了让 ROS 知道「这个包要谁」，
> `rospack depends1` 就是去读那份声明把答案列出来——本质是在查清单，不是在编译。

## 5. 四篇补充资料

### 5.1 菜鸟教程：CMake 基础

读到的关键点：

- 变量分**普通变量**和**缓存变量**。缓存变量存在 `CMakeCache.txt` 里，会持久化；改了 CMakeLists 里 `set(... CACHE ...)` 的默认值**不会自动生效**，得删 cache 或重跑。
- `include_directories()` 是全局的，会污染其他目标；现代写法一律用 `target_include_directories()`。
- `find_package()` 找到包之后，推荐用**导入目标**（如 `Boost::Boost`）而不是手动拼 `${Boost_INCLUDE_DIRS}`。

命令行改缓存变量：

```bash
cmake .. -DENABLE_LOGGING=OFF
```

> ✍️ 我对「为什么 target_xxx 更好」的理解：TODO

### 5.2 知乎那篇：一条 gcc 背后到底发生了什么

链接：https://zhuanlan.zhihu.com/p/476697014

我原本以为这是一篇「CMake 入门」，读完发现讲的是**编译器是怎么工作的**——内容是《深入理解计算机系统》第七章（链接）的梳理。反而比 CMake 更底层，跟第 1 节跑的四条命令完全对得上。

几条我记住的：

1. **一条 `gcc -o hello hello.c` 其实是四个程序接力**：`cpp`（预处理）→ `ccl`（编译）→ `as`（汇编）→ `ld`（链接）。我在第 1 节分开敲的四条 `g++` 命令，就是把这四个程序单独拎出来跑，看得见每一步的产物。
2. **预处理阶段做的全是文本操作**：删注释、展开 `#define`、把 `#include` 的文件**直接插入**原位置、加行号（为了报错能定位）。所以 `hello.i` 从 6 行变 36938 行，它不是「编译」出来的，是「抄」出来的。
3. **编译阶段细分成六步**：词法分析 → 语法分析 → 语义分析 → 中间代码 → 目标代码生成 → 目标代码优化。文章提到前端/前后端的分界线就在「中间代码」这里——前端负责产生中间代码，后端负责变成目标机器的代码。这一段我原来完全不知道，一直以为「编译」就是一件事。
4. **链接可以发生在三个时机**：编译时、加载时、运行时。我们平时做的静态链接是编译时，`.so` 动态库是加载时。
5. **链接的真正价值是「分离编译」**：大程序拆成多个模块，改一个只重编一个、再链接一下就行。这正好解释了第 2 节 Makefile 为什么要关心「谁依赖谁」。

关于 ELF 格式，文章列了目标文件的各个节，我用 `readelf -S hello.o` 对了一遍（见第 1 节）：

| 节 | 放什么 | 我的 `hello.o` 里多大 |
|---|---|---|
| `.text` | 机器码 | 58 字节 |
| `.rodata` | 只读数据（格式串等） | 12 字节（正好是 `"hello cmake\0"`） |
| `.data` | 已初始化的全局/静态变量 | 0（我没写全局变量） |
| `.bss` | 未初始化或初始化为 0 的全局/静态变量 | 0 |
| `.symtab` | 符号表 | — |
| `.rela.text` | `.text` 里需要链接器改的位置 | 120 字节 |

文章特意解释了 `.data` 和 `.bss` 为什么要分开：**为了省磁盘空间**。`.bss` 在磁盘上不占空间（它只是个「运行时需要这么多字节、全填 0」的声明），只有加载到内存才真正分配。这也解释了为什么 `readelf` 里 `.bss` 的类型是 `NOBITS`——没有 bit，不占文件。

还有「强弱符号」的规则（Linux 链接器处理重名符号）：
- 不允许多个同名强符号
- 强符号和弱符号同名，选强符号
- 多个弱符号同名，随便挑一个

> ✍️ 这篇和我跑的命令对不对得上 / 有没有它讲了我没试过的：TODO
>
> 提示：对得上的部分是第 1 节四步 + `readelf`；
> 它讲了但我没试的是**动态链接**（`gcc -shared -fpic -o libvector.so ...`），以及 `.so` 在加载时怎么被重定位。

### 5.3 catkin 官方 Tutorials

链接：https://wiki.ros.org/catkin/Tutorials

官方教程的顺序大致是：

1. 装 ROS
2. 建工作空间（`Creating a workspace for catkin`）
3. 建功能包（`CreatingPackage`）
4. 写简单的 publisher / subscriber
5. 建 msg / srv
6. 写 launch 文件

一个 catkin 包必须满足三条：有 `package.xml`、有 `CMakeLists.txt`、**每个包一个独立文件夹**（不能嵌套）。

**和 PPT 讲的差别**（我把 PPT 18 页文字逐页过了一遍，下面这些是搜出来的结果）：

| 内容 | PPT | 官方教程 |
|---|---|---|
| 建工作空间 | ✅ P6 | ✅ |
| 建功能包 | ✅ P7 | ✅ |
| `source devel/setup.bash` | ❌ 完全没提 | ✅ 讲了 |
| `package.xml` 里写什么 | ❌ 没打开过 | ✅ 有专门章节 |
| `rospack` / `rosdep` 查依赖 | ❌ 没提 | ✅ |
| 写 publisher / subscriber | ❌ | ✅ |
| msg / srv 自定义消息 | ❌ | ✅ 占很大篇幅 |
| launch 文件 | ❌ | ✅ |
| 编译命令 | `catkin build`（P6） | 主要用 `catkin_make` |

搜 `msg`、`srv`、`launch`、`package.xml`、`rospack`、`source`、`setup.bash` 这几个词在 18 页 XML 里的出现次数，除了 `std_msgs` 里那个 `msg`，其余全是 **0**。

所以我的判断是：**PPT 只覆盖到「能建出一个空包」，官方教程才管「这个包能跑起来」。** 中间那段（写节点、定义消息、写 launch、让它被找到）全在官方教程里。

另外 PPT P6 那几行命令有个小笔误：

```bash
mkdir catkin_ws
cd /catkin_ws && mkdir src     # ← 这里多了个斜杠
```

前面 `mkdir catkin_ws` 是相对路径（建在当前目录），后面 `cd /catkin_ws` 却是绝对路径（去根目录）。中间只有一个 `/` 的差别，但后者会跑到文件系统的根上去，普通用户没权限在那里建 `src`。应该是 `cd catkin_ws` 或者 `cd ~/catkin_ws`。

> ✍️ 抄 PPT 命令的时候真的要看一眼，这种一个字符的错误抄下来就跑不通，还很难查。

### 5.4 The Emperor Has No Clothes

链接：https://www.mihaileric.com/The-Emperor-Has-No-Clothes/

这篇不是 CMake 教程，是用 200 行 Python 手搓一个 AI 编程助手，来说明「这些工具不是魔法」。

它的核心循环只有五步：

1. 你发消息
2. LLM 决定要用工具，回一个结构化的 tool call
3. **你的程序**在本地执行这个调用
4. 执行结果发回给 LLM
5. LLM 基于结果继续

最关键的作者原话：**The LLM never actually touches your filesystem. It just asks for things to happen, and your code makes them happen.**（LLM 从来没有真正碰过你的文件系统，它只是「请求」事情发生，是你的代码让它发生的。）

整个 agent 只需要三个工具：读文件、列目录、改文件。

三个实现细节我觉得最有意思：

- **工具的说明文档就是靠函数 docstring 生成的**。作者用 `inspect.signature()` 和 `tool.__doc__` 拼出「工具清单」塞进 system prompt。所以写 docstring 不是给人看的，是给模型看的。
- **工具调用格式是纯文本约定**：`tool: TOOL_NAME({JSON_ARGS})`，解析的时候就找以 `tool:` 开头的行，切出函数名和 JSON。没有什么神秘的协议格式。
- **`edit_file` 用一个反直觉的技巧**：`old_str` 传空字符串就表示「新建文件」，否则就是「找到 `old_str` 替换成 `new_str`」。整个编辑能力就靠这一个函数。

作者也说了生产级工具（Claude Code 之类）多出来的东西：更好的错误处理、流式输出、上下文管理、更多工具（`grep`、`bash`、`websearch`）、以及**危险操作的审批流程**。

最后一句挺打动我的：核心循环就是这些，剩下的都是工程包装。

> ✍️ 读完这段我改了哪些对 AI 的看法：TODO
>
> 提示：至少写清楚一点——原来「AI 直接改我代码」是错觉，实际是我装的 agent 软件替它执行了写文件的操作。所以权限在我这边，我让它读什么它才能读什么。

## 6. 用大模型帮助学习：我的实际用法和思考

### 6.1 我卡住过的三个问题

> ✍️ 下面三个问题是我真的卡住的地方。答案我核对过，但我为什么卡住、哪句话点醒我，得我自己写。
> 真实的对话记录（截图或原文）我另外贴在下面，不润色。

**Q1：`hello.o` 里没有 `cout`，为什么链接的时候能找到？**

查清楚的答案：`cout` 的定义在 `libstdc++` 里，`g++` 在链接阶段会**默认**带上这个库，所以不用手动写 `-lstdc++`。我自己写的 `diff_drive` 函数没这个待遇，所以漏掉 `diff_drive.o` 就报 `undefined reference`。

这也解释了我第 7 节踩坑记录第 3 条为什么会出现——我把「标准库自动链接」当成了「所有东西都自动链接」。

✍️ 我自己的补充理解：TODO

**Q2：`catkin build` 和 `catkin_make` 到底选哪个？**

事实部分：ROS 自带 `catkin_make`；`catkin build` 属于额外安装的 `catkin_tools`。
（因为我没装 ROS，这条我没有实测，只确认了文档是这么写的。）

✍️ 我最后的结论和理由：TODO

**Q3：为什么 `cmake ..` 要两个点？**

事实部分：`..` 是上一级目录。你在 `build/` 里执行 `cmake`，要告诉它「源码在上一级」，所以传 `..` 作为源码路径。

✍️ TODO（我一开始以为两个点是语法规定，后来才想明白它只是个路径）

### 6.2 我总结的提问方法

PPT 第 12 页讲了四个用法。下面这张表我自己填，没试过的就照实写没试：

| 方法 | 我试了吗 | 效果 / 我的评价 |
|---|---|---|
| 让它搜资料给来源 | 〔待填〕 | 〔待填〕 |
| 先告诉它我学过什么，让它设计课程 | 〔待填〕 | 〔待填〕 |
| 我自己讲一遍让它检查 | 〔待填〕 | 〔待填〕 |
| 让它出练习题 | 〔待填〕 | 〔待填〕 |

> ✍️ 哪一种对我最有效，为什么：TODO

### 6.3 用 AI 学习的边界

> ✍️ 这节是 TASK.md 明确要求写的思考，必须自己写。可以往这几个方向想：
>
> - 什么时候我发现自己「以为懂了其实没懂」？
> - AI 给的解释有没有让我更懒，还是更敢问？
> - 如果 AI 直接给我答案，我还会自己去跑那四条 g++ 命令吗？
>
> TODO

## 7. 踩坑记录

都是我自己遇到的，不是查来的。前三条是我故意先写错、再改对，故意踩出来的（不踩一遍根本记不住）。

| # | 现象 | 原因 | 怎么解决的 |
|---|---|---|---|
| 1 | `Makefile.bad:2: *** missing separator.  Stop.` | 缩进用了 4 个空格 | 改成 Tab，`cat -A` 里能看出 `^I` |
| 2 | `src/diff_drive.cpp:1:10: fatal error: diff_drive.h: No such file or directory` | 头文件在 `include/`，编译时没告诉编译器去哪找 | 加 `-Iinclude`（CMake 里对应 `target_include_directories`） |
| 3 | `undefined reference to 'diff_drive(double, double, double, double&, double&)'` | 只链了 `main.o`，漏了 `diff_drive.o` | 把两个 `.o` 都写进链接命令 |
| 4 | `CMake Error: The source directory "...build" does not appear to contain CMakeLists.txt` | 在 `build/` 里敲了 `cmake .` | 改成 `cmake ..` |
| 5 | 改成 `PRIVATE` 后 `src/main.cpp` 报找不到 `diff_drive.h` | `PRIVATE` 不把 include 路径传给下游，而 `main.cpp` 就是下游 | 改回 `PUBLIC`，见 3.4 的对照实验 |
| 6 | `cmake .` 在非空 `build/` 里居然不报错 | 上次的 `CMakeCache.txt` 记住了源码路径 | 想复现这个坑就用空的 `build/`；改配置不生效就删掉整个 `build/` |

> ✍️ 第 2 条和第 3 条的报错都出现了 `diff_drive` 和「找不到」这样的词，但原因完全不同：
> 第 2 条是**编译期**没给搜索路径（`-I`），
> 第 3 条是**链接期**没给目标文件（`.o`）。
> 以前我只会盯着报错里的文件名看，现在会先看它是在哪个阶段炸的：
> 出现 `fatal error: xxx.h: No such file` 就是编译，出现 `undefined reference` 就是链接。

## 8. 还没搞明白的

> ✍️ 这一节列你真没懂的问题，下一节课问老师。可以在笔记里留着，比装懂好。

- TODO
- TODO

## 参考

- 课程 PPT：27 赛季控制视觉招新培训第二课 cmake 与 catkin 基础
- [菜鸟教程 · CMake 基础](https://www.runoob.com/cmake/cmake-basic.html)
- [知乎 · CMake 相关文章](https://zhuanlan.zhihu.com/p/476697014)
- [catkin/Tutorials · ROS Wiki](https://wiki.ros.org/catkin/Tutorials)
- [The Emperor Has No Clothes · Mihail Eric](https://www.mihaileric.com/The-Emperor-Has-No-Clothes/)

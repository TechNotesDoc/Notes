# SCons构建系统

## SCons 内置函数

SConscript 文件可以控制源码文件的加入，并且可以指定文件的 Group（与 MDK/IAR 等 IDE 中的 Group 的概念类似）。

SCons 提供了很多内置函数可以帮助我们快速添加源码程序，利用这些函数，再配合一些简单的 Python 语句我们就能随心所欲向项目中添加或者删除源码。下面将简单介绍一些常用函数。

**GetCurrentDir()**

获取当前路径。

**Glob('*.c')**

获取当前目录下的所有 C 文件。修改参数的值为其他后缀就可以匹配当前目录下的所有某类型的文件。

**GetDepend(macro)**

该函数定义在 `tools/building.py` 下的脚本文件中，它会从 rtconfig.h 文件读取配置信息，其参数为 rtconfig.h 中的宏名。如果 rtconfig.h 打开了某个宏，则这个方法（函数）返回真，否则返回假。

**Split(str)**

将字符串 str 分割成一个列表 list。

**DefineGroup(name， src， depend，**parameters)**

这是 RT-Thread 基于 SCons 扩展的一个方法（函数）在`tools/building.py`。DefineGroup 用于定义一个组件。组件可以是一个目录（下的文件或子目录），也是后续一些 IDE 工程文件中的一个 Group 或文件夹。

`DefineGroup()` 函数的参数描述：

| **参数**   | **描述**                                                     |
| ---------- | ------------------------------------------------------------ |
| name       | Group 的名字                                                 |
| src        | Group 中包含的文件，一般指的是 C/C++ 源文件。方便起见，也能够通过 Glob 函数采用通配符的方式列出 SConscript 文件所在目录中匹配的文件 |
| depend     | Group 编译时所依赖的选项（例如 FinSH 组件依赖于 RT_USING_FINSH 宏定义）。编译选项一般指 rtconfig.h 中定义的 RT_USING_xxx 宏。当在 rtconfig.h 配置文件中定义了相应宏时，那么这个 Group 才会被加入到编译环境中进行编译。如果依赖的宏并没在 rtconfig.h 中被定义，那么这个 Group 将不会被加入编译。相类似的，在使用 scons 生成为 IDE 工程文件时，如果依赖的宏未被定义，相应的 Group 也不会在工程文件中出现 |
| parameters | 配置其他参数，可取值见下表，实际使用时不需要配置所有参数     |

parameters 可加入的参数：

| **参数**   | **描述**                                         |
| ---------- | ------------------------------------------------ |
| CCFLAGS    | C 源文件编译参数                                 |
| CPPPATH    | 头文件路径                                       |
| CPPDEFINES | 链接时参数                                       |
| LIBRARY    | 包含此参数，则会将组件生成的目标文件打包成库文件 |

**SConscript(dirs，variant_dir，duplicate)**

读取新的 SConscript 文件，SConscript() 函数的参数描述如下所示：

| **参数**    | **描述**                               |
| ----------- | -------------------------------------- |
| dirs        | SConscript 文件路径                    |
| variant_dir | 指定生成的目标文件的存放路径           |
| duiplicate  | 设定是否拷贝或链接源文件到 variant_dir |

## 安装scons

```bash
sudo apt-get install scons
```

## 测试

1. 新建helloscons.c 和 SConstruct 文件;

```
$ ls
helloscons.c  SConstruct
```



2. helloscons.c文件内容;

```
$ cat helloscons.c
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char* argv[])
{
     printf("Hello, SCons!\n");
     return 0;
}
```

3. SConstruct文件内容;

   ```
   $ cat SConstruct
   Program('helloscons.c')
   ```

4. 编译

   ```
   $ scons
   scons: Reading SConscript files ...
   scons: done reading SConscript files.
   scons: Building targets ...
   gcc -o helloscons.o -c helloscons.c
   gcc -o helloscons helloscons.o
   scons: done building targets.
   ```

5. 查看编译后生成的结果

   ```
   $ ls
   helloscons  helloscons.c  helloscons.o  SConstruct
   ```

6. 执行编译后的可执行程序 helloscons

   ```
   $ ./helloscons
   Hello, SCons!
   ```

   

7. 清除编译信息

   ```
   $ scons -c
   scons: Reading SConscript files ...
   scons: done reading SConscript files.
   scons: Cleaning targets ...
   Removed helloscons.o
   Removed helloscons
   scons: done cleaning targets.
   $ ls
   helloscons.c  SConstruct
   ```

   

   交叉编译嵌入式环境配置

   在coss文件夹下有src/main.c、comm/comm.c comm.h 和SConstruct文件，lib文件夹用来存放库文件，这里没有使用到，所以为空。SConstruct文件内容如下

   ```
   book@pc:~/Driver/coss$ vim SConstruct
   src = Glob('src/*.c')
   src = src + Glob('comm/*.c')
   include_path = [
       'src/',
       'comm/',
       'lib/'
   ]
   lib_path = 'lib/'
   #static_libs =[
   #     lib_path + 'libtestvector.a'
   #]
   
   execute = env.Program(
       target = 'test',
       #LIBPAHT = lib_path,
   #    LIBS = [File(lib) for lib in static_libs],
       source = src,
       CPPPATH = include_path,
   )
   
   env.Alias('all',[execute])
   Default('all')
   
   ```

   

## Scons命令

### scons文件编译

- scons -Q：进行代码文件编译，不显示Scons内部操作打印的信息，只显示编译信息

- scons -c：清除编译中间文件和可执行文件

###  Scons编译脚本

Scons对应的编译脚本名称为SConstruct，就如同make对应的编译脚本为makefile

#### 编译函数

- Program()：执行编译操作，生成可执行文件

- Library()：执行编译操作，生成静态库

- StaticLibrary()：执行编译操作，生成静态库

- SharedLibrary()：执行编译操作，生成动态库

- Environment()：编译环境

#### 编译参数

- target，生成的执行文件名字

- source，编译文件

- LIBS，依赖库

- LIBPATH，依赖库路径，有环境变量的可不添加，针对用户库或第三方库

- CPPPATH，头文件路径

- CCFLAGS，编译参数

#### 其他函数

- Split()：将字符串分隔为列表

- Glob('*.cpp')：加入所有文件
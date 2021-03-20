

# Makefile

## gcc常用选项

```
gcc常用选项：
  -v：查看gcc编译器的版本，显示gcc执行时的详细过程
  -o <file>               指定输出文件名为file，这个名称不能跟源文件名同名
  -E                      只预处理，不会编译、汇编、链接
  -S                      只编译，不会汇编、链接
  -c                      编译和汇编，不会链接    
```

最简单的编译命令 gcc -o hello hello.c  输出hello

实际分为以下几步 预处理 编译 汇编 链接 ，其中【预处理 编译 汇编】统称为编译

```
gcc -E -o hello.i hello.c  # 预处理    hello.c -》 hello.i
gcc -S -o hello.s hello.i  # 编译      hello.i -》 hello.s
gcc -c -o hello.o hello.s  # 编译和汇编 hello.s -》 hello.o
gcc -o hello hello.o       # 链接      hello.o -》 hello
```



## 规则

```
目标 : 依赖1 依赖2 ...
[TAB]命令
```

当"目标文件"不存在, 或某个依赖文件比目标文件"新",则: 执行"命令"

```bash
test:a.o b.o            # 目标test 依赖a.o和b.o  如果a.o或者b.o比test新，那么就执行下面的编译命令
	gcc -o test a.o b.o # 命令 ：链接a.o和b.o 生成 test的可执行文件
	
a.o : a.c               # a.o又依赖a.c ,同理当a.c比a.o新的时候，执行下面的命令
	gcc -c -o a.o a.c   # 命令：把a.c 编译为a.o

b.o : b.c               # b.o又依赖b.c ,同理当b.c比b.o新的时候，执行下面的命令
	gcc -c -o b.o b.c   # 命令：把b.c 编译为b.o
```

## 语法

### 通配符

```bash
%.o         匹配所有的.o文件
$@          表示目标
$<          表示第1个依赖文件
$^          表示所有依赖文件
例如
test：a.o b.o
	gcc -o test a.o b.o

当上面的test查找依赖a.o和b.o的时候会往下找这些.o的依赖规则，到里发现所有的.o文件依赖所有的.c文件
%.o:%.c                  
	gcc -c -o $@ $<     如果此时查找的是a.o 这里$@表示目标即a.o  $<表示第一个依赖即即a.c
```

**注：make 命令后面跟个目标名字，如果不跟名字，那么就找当前makefile的第一个目标**

### 假想目标: .PHONY

```
clean：
	rm *.o
```

如上代码的makefile，如果同级目录下有clean的文件，我们执行 `make clean`的时候会怎么处理。

答案是什么都没执行，为什么？

因为：我们编译规则是目标不存在或者依赖比目标新，才会执行命令，而这里的makefile里面的clean:并没有依赖，且同级目录下有clean文件即目标存在，不满足上面说的条件，所以不会执行该命令`rm *.o`，解决方式就是用假想目标。

makefile修改如下即可。假想有个这样的clean目标不存在。

```bash
clean：
	rm *.o

.PHONY：clean
```

### 即时变量、延时变量, export
####  变量定义方式

```bash
:=   # 即时变量
=    # 延时变量
?=   # 延时变量, 如果是第1次定义才起效, 如果在前面该变量已定义则忽略这句
+=   # 附加, 它是即时变量还是延时变量取决于前面的定义
```

##### 情况1

```bash
A :=$(C)  # 即使变量，由于此时C还没定义，所以理论上A应该等于空
B = $(C)  # 延时变量，由于B在用到的时候才定义，所以等B用到的时候会搜索整个makefile，找C的值，意味着C只要在此makefile定义，那么B就取C的值，无关C定义在这个makefile的哪个地方。
all:
	@echo A= $(C)
	@echo B= $(C)  # 此处用到了B，开始从makefile头开始搜索，找C的值。
C=123
执行make结果：
A=
B=123

```

##### 情况2

```bash
A :=$(C)  # 即使变量，
B = $(C)  # 延时变量，
C = abc
all:
	@echo A= $(C)
	@echo B= $(C)
C=123 #覆盖掉abc 

执行make结果：
A=
B=123

```

##### 情况3

```
A :=$(C)  # 即使变量，
B = $(C)  # 延时变量，
C = abc
all:
	@echo A= $(C)
	@echo B= $(C)
C+=123 #此时C=abc123 

执行make结果：
A=
B=abc123
```

##### 情况4

```bash
D = abc
D ?= CONGXIN #由于上面已经定义了D，所以这里不会起作用，那么D还是abc
all:
	@echo D= $(D)

执行make结果：
D=abc
```

##### 情况5

```bash
D ?= CONGXIN  
all:
	@echo D= $(D)

执行：make 后结果
D=CONGXIN
执行：make D=666 后的结果如下  #通过命令行传入参数 
D=666
```

### 函数

```
$(foreach var,list,text)       # 遍历list每个成员赋值给var，然后执行text 
$(filter pattern...,list)      # 在 list 中取出符合patten格式的值
$(filter-out pattern...,list)  # 在 list 中取出不符合patten格式的值
$(wildcard pattern)            # pattern定义了文件名的格式wildcard取出其中存在的文件
$(patsubst pattern,replacement,$(var))  # 遍历列表var，中取出每一个值如果符合pattern则用replacement 替换
```

#### 情况1

```bash
A = a b c
B = $(foreach f,$(A),$(f).o) #遍历A里面的每一个值赋值给f，然后执行 $(f).o ，
                             #例如如果当前匹配到a，那么f=a,然后执行$(f).o，实际就是 a.o

all:
	@echo B= $(B)
执行make结果
B=a.o b.o c.o
```

#### 情况2

```bash
C = a b c d/
D = $(filter %/,$(C))     #取出C中符合 %/  即是目录的项赋值给D
E = $(filter-out %/,$(C)) #取出C中不符合 %/  即不是目录的项赋值给D

all:
	@echo D= $(D)
	@echo E= $(E)

执行make结果
D=d/
E=a b c
```

#### 情况3

```bash
假如 目录下有 a.c b.c d.txt 等文件

SOURCES= $(wildcard *.c)  #查找当前目录下所有的.c文件，然后把名字赋值给SOURCES

all：
	@echo src = $(SOURCES) 

执行make结果如下：
src = a.c b.c
```

#### 情况4

```
files = a.c b.c 666 333
OBJS = $(patsubst %.c,%.o,$(files)) # 把files里面的所有匹配.c的文件替换为.o文件然后赋值给OBJS

all:
	@echo objs = $(OBJS)

执行make后结果
objs = a.o b.o

```

 ### 支持头文件依赖
http://blog.csdn.net/qq1452008/article/details/50855810

```bash
gcc -M c.c                     // 打印出依赖
gcc -M -MF c.d c.c             // 把依赖写入文件c.d
gcc -c -o c.o c.c -MD -MF c.d  // 编译c.o, 把依赖写入文件c.d

```



```bash
objs = a.o b.o c.o

dep_files := $(patsubst %,.%.d, $(objs)) # dep_files :=.a.o.d .b.o.d .c.o.d
dep_files := $(wildcard $(dep_files))    # dep_files :=上个dep_files里面正真存在的文件

CFLAGS = -Werror \        # 把警告信息当做错误输出，一般警告都隐藏着bug，所以建议加上
         -I./includes \   # 添加头文件的搜索路径

test: $(objs)            # 目标test依赖所   $(objs)
	gcc -o test $^       # $^ 所有依赖 即 $(objs)

ifneq ($(dep_files),)  # 因为第二个参数空就是NULL，所以 意思是如果dep_files!=NULL
include $(dep_files)
endif

%.o : %.c
	gcc $(CFLAGS) -c -o $@ $< -MD -MF .$@.d #编译目标$@依赖$< 并且把依赖写入.$@.d里面

# 举个例子假如是a.o,上面的代码就如同下面的
#a.o : a.c
#	gcc $(CFLAGS) -c -o a.o a.c -MD -MF .a.o.d
	
clean:
	rm *.o test

distclean:
	rm $(dep_files)   #删除所有依赖
	
.PHONY: clean	
```



### gcc选项

#### 编译选项CFLAGS

| 选项      | 说明                                                         |
| :-------- | :----------------------------------------------------------- |
| **-c**    | 用于把源码文件编译成 .o 对象文件,不进行链接过程              |
| **-o**    | 用于连接生成可执行文件，在其后可以指定输出文件的名称         |
| **-g**    | 用于在生成的目标可执行文件中，添加调试信息，可以使用GDB进行调试 |
| **-Idir** | 用于把新目录添加到include路径上，可以使用相对和绝对路径，“-I.”、“-I./include”、“-I/opt/include” |
| **-Wall** | 生成常见的所有告警信息，且停止编译，具体是哪些告警信息，请参见GCC手册，一般用这个足矣！ |
| **-w**    | 关闭所有告警信息                                             |
| **-O**    | 表示编译优化选项，其后可跟优化等级0\1\2\3，默认是0，不优化   |
| **-fPIC** | 用于生成位置无关的代码                                       |
| **-v**    | (在标准错误)显示执行编译阶段的命令，同时显示编译器驱动程序,预处理器,编译器的版本号 |

#### GCC链接选项LDFLAGS参数

| 选项           | 说明                                                         |
| :------------- | :----------------------------------------------------------- |
| **-llibrary**  | 链接时在标准搜索目录中寻找库文件，搜索名为`liblibrary.a` 或 `liblibrary.so` |
| **-Ldir**      | 用于把新目录添加到库搜索路径上，可以使用相对和绝对路径，“-L.”、“-L./include”、“-L/opt/include” |
| **-Wl,option** | 把选项 option 传递给连接器，如果 option 中含有逗号,就在逗号处分割成多个选项 |
| **-static**    | 使用静态库链接生成目标文件，避免使用共享库，生成目标文件会比使用动态链接库大 |

## 动态和静态库

**解决问题：**

1. 使用makefile生成静态库.a或者动态库.so，然后用户的app程序引用这些库文件；

2. 如果在生成库时候，又引用了其他库文件，makefile该怎么写。

**源码结构：**

```bash
Demo
├── App                 # 用户应用程序目录
|   ├── includes        	# 库中函数头文件目录，可以从LibProject/includes里面拷贝过来
|   ├── main.c              # 用户源代码
|   ├── Makefile            # 用户应用程序的makefile文件
|   └── thirdpart           # 库文件存放目录 可以从LibProject/thirdpart里面拷贝过来
|   
├── LibProject          # 库工程目录
    ├── build           	# 暂时没有用空的
    ├── common         		# 库工程源代码目录，里面存放着.c文件
    │   └── common.c
    ├── driver            	# 库工程源代码目录，同样里面存放着.c文件
    │   └── ledDrv.c
    ├── includes            # 库工程需要开放给用户的头文件，到时候可以和库文件一起拷贝给客户
    │   ├── common.h
    │   └── ledDrv.h
    ├── lib                 # 库工程 需要引用的外部库，就是问题2中描述的项
    │   └── libsqlite3.so
    ├── Makefile            # 库工程makefile，控制着该库的编译方式
    └── thirdpart           # 库工程最终生成的库文件存放路径
        ├── libtest.a
        └── libtest.so

```

### 应用程序相关

1. **Demo/App/main.c**

```c
#include <stdio.h>
#include <ledDrv.h>
#include <common.h>

int main(void)
{
     ledDrvInit();  // 调用libtest里面的ledDrvInit()函数
     commonInit();  // 调用libtest里面的commonInit()函数
	 return 0;
}
```

2. **Demo/App/Makefile**

```
# 设置交叉编译环境
CROSS_COMPILE_PATH=/usr/local/arm_linux_4.8/bin
CROSS_COMPILE=$(CROSS_COMPILE_PATH)/arm-linux-

CC=$(CROSS_COMPILE)gcc 

# 设置编译选项，并且制定头文件路径
CFLAGS = -g \
		-Wall \
 		-I$(CURDIR)/../LibProject/includes \
		-I$(CURDIR) \

# 源码目录 是当前路径下的所有.c文件，如果还有其它源码目录的请自行模仿添加
SRCS := $(wildcard ./*.c) \
# 去掉路径后的所有.c文件 
DIRS :=$(notdir $(SRCS))
# 把去掉路径后的所有.c文件转换为.o文件 也就是目标文件 
OBJS := $(patsubst %.c,%.o,$(DIRS))
# 引用库路径和库名字
LIBS := -L/$(CURDIR)/../LibProject/thirdpart \  # 就是自己编译的库的路径
       -lpthread\                               # 系统库，不需要指定路径
	   -ltest                                   # 自己编译的库名字

# 应用程序名字
APP_NAME = APP
# 编译生成自己的应用程序

all:
	$(CC) $(CFLAGS) -o $(APP_NAME) $(SRCS) $(LIBS)


clean:
	rm -rf *.o
	rm -rf $(OBJS) $(APP_NAME)


```

### 库工程相关

1. **Demo/LibProject/common.c**

   ```c
   #include <stdio.h>
   #include "common.h"
   void commonInit()
   {
       printf("I am common.c\r\n");  // 这里只打印一句话
   }
   ```

2. **Demo/LibProject/common/common.h**

   ```c
   #ifndef _COMMON_H
   #define _COMMON_H
   void commonInit();
   #endif
   ```

3. **Demo/LibProject/Drver/ledDrv.c**

   ```c
   #include <stdio.h>
   #include "ledDrv.h"
   void ledDrvInit()
   {
       printf("I am ledDrv.c\r\n"); // 这里只打印一句话
   }
   ```


   

4. **Demo/LibProject/Drver/ledDrv.h**

   ```c
   #ifndef _LED_DRV_H
   #define _LED_DRV_H
   void ledDrvInit();
   #endif 
   ```

   

5. **Demo/LibProject/Lib/libsqlite3.so**

6. **Demo/LibProject/Makefile**

   ```bash
   # 设置交叉编译环境
   CROSS_COMPILE_PATH=/usr/local/arm_linux_4.8/bin
   CROSS_COMPILE=$(CROSS_COMPILE_PATH)/arm-linux-
   CC=$(CROSS_COMPILE)gcc 
   AR=$(CROSS_COMPILE)ar
   
   # 设置需要生成库的的路径和名字
   LIB_NAME    ?= test
   LIB_PATH    ?= $(CURDIR)/thirdpart                                                                  
   STATIC_NAME ?= lib$(LIB_NAME).a
   SHARE_NAME  ?= lib$(LIB_NAME).so
   
   # 设置编译选项，这里指定了头文件的路径
   CFLAGS = -g \
   		-Wall \
    		-I$(CURDIR)/includes \
   		-I$(CURDIR) \
   
   # 源码目录 是当前路径下的所有.c文件，如果还有其它源码目录的请自行模仿添加
   SRCS := $(wildcard ./common/*.c) \
   	    $(wildcard ./driver/*.c)
   # 去掉路径后的所有.c文件 	
   DIRS :=$(notdir $(SRCS))
   # 把去掉路径后的所有.c文件转换为.o文件 也就是目标文件
   OBJS := $(patsubst %.c,%.o,$(DIRS))
   # 引用库路径和库名字
   LIBS := -L/$(CURDIR)/lib \ # 指定需要引用的库的路径就是libsqlite3.so路径
          -lpthread\          # 系统库，不需要指定路径
   	   -lsqlite3           # 该库工程又需要引用的库，即问题2中提到的
   
   
   # 执行 make all
   all: static_library shared_library
   
   # 编译所有的源码，生成目标.o，不链接
   $(OBJS):$(SRCS)
   	$(CC) $(CFLAGS) -c $(SRCS) $(LIBS)
   
   # 静态库的生成方式
   static_library:$(OBJS)
   	$(AR) -cr $(STATIC_NAME) $(OBJS)
    
   # 动态库的生成方式
   shared_library:$(OBJS)
   	$(CC) -shared -fpic -o $(SHARE_NAME) $(OBJS)
   
   # 执行make install 的时候可以把生成的库文件 移动到指定的目录，这里目录是$(LIB_PATH)
   install:
   	mv $(STATIC_NAME) $(LIB_PATH)
   	mv $(SHARE_NAME)  $(LIB_PATH)
   
   #执行make out的时候调试信息
   out:
   	@echo srcs  = $(SRCS)
   	@echo dir   = $(DIRS)
   	@echo libs  = $(LIBS)
   	@echo objs  = $(OBJS)
   
   clean:
   	rm -rf *.o
   	rm -rf $(STATIC_NAME) $(SHARE_NAME)
   
   
   ```

### 编译运行

```bash
# 1.生成库文件步骤：
# make clean
# make all
# make install
# 如下：
book@pc:~/Demo$ cd LibProject/
book@pc:~/Demo/LibProject$ make clean
rm -rf *.o
rm -rf libtest.a libtest.so
book@pc:~/Demo/LibProject$ make all
/usr/local/arm_linux_4.8/bin/arm-linux-gcc  -g -Wall -I/home/book/Demo/LibProject/includes -I/home/book/Demo/LibProject  -c ./common/common.c ./driver/ledDrv.c -L//home/book/Demo/LibProject/lib -lpthread -lsqlite3
/usr/local/arm_linux_4.8/bin/arm-linux-ar -cr libtest.a common.o ledDrv.o
/usr/local/arm_linux_4.8/bin/arm-linux-gcc  -shared -fpic -o libtest.so common.o ledDrv.o
book@pc:~/Demo/LibProject$ make install
mv libtest.a /home/book/Demo/LibProject/thirdpart                                                                  
mv libtest.so  /home/book/Demo/LibProject/thirdpart 

# 2.编译用户的应用程序步骤：
# make clean
# make all
# 如下：                       
book@pc:~/Demo/LibProject$ ls thirdpart/
libtest.a  libtest.so
book@pc:~/Demo/LibProject$ cd ../App/
book@pc:~/Demo/App$ ls
includes  main.c  Makefile  thirdpart
book@pc:~/Demo/App$ make clean
rm -rf *.o
rm -rf main.o APP
book@pc:~/Demo/App$ make
/usr/local/arm_linux_4.8/bin/arm-linux-gcc  -g -Wall -I/home/book/Demo/App/../LibProject/includes -I/home/book/Demo/App  -o APP ./main.c  -L//home/book/Demo/App/../LibProject/thirdpart -lpthread -ltest
book@pc:~/Demo/App$ ls
APP  includes  main.c  Makefile  thirdpart
book@pc:~/Demo/App$ 

```

最总把生成应用程序拷贝到开发板运行如下：

```bash
root@arm: sudo chmod +x ./APP
root@arm: ./APP
I am ledDrv.c
I am common.c

```

[该工程托管路径](https://github.com/RobotFly/MakefileLib.git)
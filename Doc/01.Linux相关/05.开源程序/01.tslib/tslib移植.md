# tslib移植

1.安装环境，并下载源码

```
book@100ask:~/AWTK/$ apt-get install automake libtool 
book@100ask:~/AWTK/$ apt-get install pkg-config
book@100ask:~/AWTK/$ git clone https://github.com/libts/tslib.git 
book@100ask:~/AWTK/tslib$ cd tslib/ 
```

2.编译

1.版本1：

```shell
book@100ask:~/AWTK/tslib$ ./autogen.sh 
book@100ask:~/AWTK/tslib$ mkdir build
注意：
arm-linux：是编译器，有些开发板是arm-fsl-linux  不一样的 
/home/book/AWTK/tslib/build ：编译的输出路径

book@100ask:~/AWTK/tslib$ ./configure --host=arm-linux --prefix=/home/book/AWTK/tslib/build
book@100ask:~/AWTK/tslib$ make
book@100ask:~/AWTK/tslib$ make install
```

2.版本2：

```shell
 
book@100ask:~/AWTK/tslib$ ./autogen.sh 
book@100ask:~/AWTK/tslib$ mkdir build
book@100ask:~/AWTK/tslib$ echo "ac_cv_func_malloc_0_nonnull=yes" >arm-linux.cache
book@100ask:~/AWTK/tslib$ ./configure --host=arm-linux --cache-file=arm-linux.cache --prefix=/home/book/AWTK/tslib/build/ 
book@100ask:~/AWTK/tslib$ make
book@100ask:~/AWTK/tslib$ make install
```

版本1和版本2不同之处：

echo "ac_cv_func_malloc_0_nonnull=yes" >arm-linux.cache 屏蔽编译过程报错；

make install后，会在/home/book/AWTK/tslib/build/目录生成以下子目录：

```
book@100ask:~/AWTK/tslib/build$ ls
bin  etc  include  lib  share

```

bin 目录：校准测试工具（如校准的 ts_calibrate ,测试用的 ts_tast）

lib 目录： 生成的库  该目录下还有一个子目录ts，它包含了许多校准用到的库(如input.so等) 

etc 目录： ts.conf为配置文件 

include 目录：头文件

拷贝bin etc include lib目录 到开发板的文件系统 /usr/local/tslib目录下 

然后修改开发板的文件系统 /usr/local/tslib/etc/ts.conf, 内容如下：

```
module_raw input
module pthres pmin=1
module variance delta=30
module dejitter delta=100
module linear
```



 module_raw有许多种，这里只使用input(即Linux的input子系统，设备文件名称为/dev/input/event0)，其它的删除掉。后面的几个module还没有深入了解，它们使用的库就在tslib/lib/ts中，最后三个模块的字面意思是“方差(滤波)”、“去抖动(去噪)”、“线性(坐标变换)”，对这些东西不了解 



然后修改开发板子的环境变量， 可以在开发板文件系统的/etc/profile文件里添加 ，也可以等板子启动后手动修改

如果手动修改 /etc/profile,要立即使这些变量生效，还需要修改完后输入命令source /etc/profile 

```shell

export TSLIB_ROOT=/usr/local/tslib
#触摸设备文件名
export TSLIB_TSDEVICE=/dev/input/event0
#配置文件名。
export TSLIB_CONFFILE=$TSLIB_ROOT/etc/ts.conf
#插件目录 
export TSLIB_PLUGINDIR=$TSLIB_ROOT/lib/ts
#校准数据文件，由ts_calibrate校准程序生成。
export TSLIB_CALIBFILE=/etc/pointercal
#控制台设备文件名
export TSLIB_CONSOLEDEVICE=none
#显示设备名
export TSLIB_FBDEVICE=/dev/fb0

export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$TSLIB_ROOT/lib
```

然后启动开发板子 运行校准程序 位于 /usr/local/tslib/bin目录下， 运行校准程序，触摸屏依次出现5个点，依次点击之： 

```shell
cd  /usr/local/tslib/bin
./ts_calibrate 
```

生成的校准文件名为pointercal，位于/etc目录下。

如果想运行ts的测试程序，在tslib/bin目录下输入

```shell
 ./ts_test 
```

注意

 清除编译命令： ./autogen-clean.sh 
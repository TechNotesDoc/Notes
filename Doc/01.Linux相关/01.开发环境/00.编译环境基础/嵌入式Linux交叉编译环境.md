# 嵌入式Linux交叉编译环境设置

- **平台**：Ubuntu虚拟机，下载的交叉编译工具路径为

  `/home/book/100ask_firefly-rk3288/ToolChain/gcc-linaro-6.2.1-2016.11-x86_64_arm-linux-gnueabihf`

##  临时生效

终端下直接执行下面的命令

```bash
export PATH=$PATH:/home/book/100ask_firefly-rk3288/ToolChain/gcc-linaro-6.2.1-2016.11-x86_64_arm-linux-gnueabihf/bin
export ARCH=arm
export CROSS_COMPILE=arm-linux-gnueabihf-
```

## 当前用户永久生效

终端下使用命令修改`~/.bashrc`或者 `~/.bash_profile`文件

```bash
vim ~/.bashrc
# 在行尾添加或修改：
export ARCH=arm
export CROSS_COMPILE=arm-linux-gnueabihf-
export PATH=$PATH:/home/book/100ask_firefly-rk3288/ToolChain/gcc-linaro-6.2.1-2016.11-x86_64_arm-linux-gnueabihf/bin
source ~/.bashrc # 使其生效
```

或者

```bash
vim ~/.bash_profile
# 在行尾添加或修改：
export ARCH=arm
export CROSS_COMPILE=arm-linux-gnueabihf-
export PATH=$PATH:/home/book/100ask_firefly-rk3288/ToolChain/gcc-linaro-6.2.1-2016.11-x86_64_arm-linux-gnueabihf/bin
source ~/.bash_profile # 使其生效
```

## 所有用户永久生效

修改`/etc/profile`文件

```bash
sudo vim /etc/profile
# 在行尾添加或修改：
export ARCH=arm
export CROSS_COMPILE=arm-linux-gnueabihf-
export PATH=$PATH:/home/book/100ask_firefly-rk3288/ToolChain/gcc-linaro-6.2.1-2016.11-x86_64_arm-linux-gnueabihf/bin
source /etc/profile
```

## 验证设置是否生效

```bash
arm-linux-gnueabihf-gcc -v
# 如果正常输出版本号，证明设置成功，如果没有，则失败
```


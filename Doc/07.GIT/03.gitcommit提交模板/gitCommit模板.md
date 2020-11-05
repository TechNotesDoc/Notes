# git commit 提交日志模块

## 编写模板

无论是linux还是win系统，使用下面的命令建立模板文件

```bash
 touch ~/.git-commit-template.txt
```

修改模板的内容,打开刚刚创建的【.git-commit-template.txt】文件，文件路径如下：win下在【C:\Users\Administrator】目录下，linux在【~/】目录下。内容如下：

```bash
#类型如下:
#   feat	: 新增feature
#   fix 	: 修复bug
#   docs	: 文档修改，比如readme等
#   style	: 代码格式修改，比如空格，缩进等
#   refactor: 代码重构，没有加新功能或者修复bug
#	perf	: 优化相关，比如提升性能
#   test	: 测试用，比如单元测试、集成测试
#   chore	: 其他修改, 比如构建流程, 依赖管理
#	revert	: 回退版本
#
#
#	修改内容:具体的修改，可以详细点，分条描述
#
#
#	备注: 可有可无
#
#
类型(影响范围):提交概述

修改内容:
    1:
    2:

备注:
    1:
    2:

```

![](media/328599-20190123104458539-2041029483.png)


## 设置git 默认的模块

```bash
#这个命令只能设置当前分支的提交模板
git config commit.template   ~/.git-commit-template.txt
#这个命令能设置全局的提交模板，注意global前面是两杠
git config --global commit.template   ~/.git-commit-template.txt
```

## 设置文本编辑器

```bash
git config --global core.editor  vim
```

## 提交代码

```bash
git commit   # 输入提交命令，进入vi模式。注意，#的内容会被忽略，不会显示在提交的日志里面
```






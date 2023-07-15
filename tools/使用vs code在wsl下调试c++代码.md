---
title: 使用vs code和wsl搭建C/C++开发环境
date: 2021-02-06 22:52:16
tags: 
	- vscode
	- wsl
categories: 其它
---

#### 安装WSL 2

按照[官方文档](https://docs.microsoft.com/zh-cn/windows/wsl/)安装子系统，建议使用WSL 2。WSL 2 使用最新、最强大的虚拟化技术在轻量级实用工具虚拟机 (VM) 中运行 Linux 内核。 但是，WSL 2 不是传统的 VM 体验。

<!--more-->
![image-20210206213930600](https://images.coxlong.cn/img/20210206213937.png)

除了跨操作系统文件系统的性能外，WSL 2 体系结构在多个方面都比 WSL 1 更具优势。

#### 安装VS Code及插件

在Windows上安装[VS Code](https://code.visualstudio.com/)并安装[远程开发扩展包](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack)。

VS Code有很多插件，建议安装语言包和Visual Studio IntelliCode。

![image-20210206215838016](https://images.coxlong.cn/img/20210206215838.png)

![image-20210206220013562](https://images.coxlong.cn/img/20210206220013.png)

![image-20210206220058492](https://images.coxlong.cn/img/20210206220058.png)

#### 安装编译器、调试器

在子系统里面安装编译器gcc和g++以及调试器gdb。

```bash
sudo apt -y install gcc g++ gdb
```

![image-20210206221905641](https://images.coxlong.cn/img/20210206221905.png)

#### 调试代码

在子系统里面输入以下命令打开VS Code。

```shell
code .
```

第一次从子系统打开VS Code会自动安装一些插件，等待安装完成即可。

![image-20210206222154233](https://images.coxlong.cn/img/20210206222154.png)

打开VS Code的插件安装界面，会提示在wsl里面安装相应插件，点击即可。

![image-20210206222430610](https://images.coxlong.cn/img/20210206222430.png)

然后打开资源管理器，新建一个cpp源文件并写个hello world。

![image-20210206222855891](https://images.coxlong.cn/img/20210206222856.png)

最后打开调试界面，点击运行和调试，选择GDB，然后选择g++。

![image-20210206223335217](https://images.coxlong.cn/img/20210206223335.png)

![image-20210206223858343](https://images.coxlong.cn/img/20210206223858.png)

此时，文件目录下会多出一个.vscode文件夹，包含两个配置文件。

![image-20210206224009387](https://images.coxlong.cn/img/20210206224009.png)

至此，配置大功告成，接下来就可以加断点调试代码啦！`F5`是开始运行，`F11`是单步调试，`F10`是单步跳过。
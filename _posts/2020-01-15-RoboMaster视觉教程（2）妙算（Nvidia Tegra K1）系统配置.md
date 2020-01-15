---
layout: post
title:  "RoboMaster视觉教程（2）妙算（Nvidia Tegra K1）系统配置"
date:   2020-01-15 20:33:56 +0800
categories: RoboMaster
---

# 概览

这里所说的妙算是大疆的第一代妙算，核心处理器为 Tegra K1 ，很轻颜值很高，现在已经停产了。

在实际应用中可以明显的感觉到妙算的性能不够。妙算主要是给 M100 设计的，对 M100 的 X3 摄像头做了优化，即可以通过 GPU 读取 X3 摄像头的数据。但对于比赛来说基本没用,而且我也从来没有成功用它读取过 X3 摄像头数据 (T_T) 。妙算的 GPU 只有在进行深度学习类的任务时才有用武之地，而在识别装甲板之类的任务时基本上是摆设，尽管有 zero copy 的特性，但是很不好用。

在这里吐槽了半天妙算，可能你会说：既然妙算这么垃圾，为啥还要用它？因为穷啊，当初买妙算的时候也是割了肉买的，还买了5个……

# 妙算资料链接汇总

+ <https://www.dji.com/cn/manifold/info#downloads> 妙算官方镜像以及说明书下载
+ <https://elinux.org/Jetson_TK1> 这里应该是 TK1 的百科全书，基本上碰到的问题都可以在这里找到答案，同时这个网站也有 TX1、TX2 的资料
+ <https://github.com/groundmelon/m100_x3> X3 摄像头相关资料
+ <https://developer.nvidia.com/embedded/downloads/archive> 这里可以下载 JetPack 也就是 Nvidia 官方的一些工具

# 妙算系统重置/克隆/恢复
妙算系统的安装在说明书中写的非常详细了，这里我说一下流程。

1. 首先要有一台装了 Ubuntu 的电脑，将需要的镜像和软件包都下载好。

2. 将镜像解压缩到某一个目录，我推荐在用户目录下解压。这里要强调一点**在解压缩的时候千万不要右键提取到此处，而一定要按照说明书中的命令来操作**即：

   ```shell
   sudo tar -xvpzf <your path>/manifold_image_v1.0.tar.gz
   ```

   如果不这样解压将会破坏各个文件之间的用户权限关系，在装好系统后会出现奇奇怪怪的bug。

3. 妙算进入恢复模式（就是刷机模式）

   有两种方法我推荐使用说明书中的**方法2**，方法2只要按一个按钮而方法1要同时按两个按钮。

4. 制作系统默认镜像，在说明书中**制作系统镜像**和**恢复系统镜像**这两个标题容易误导。我解释一下：

   如果你想恢复到**系统的默认镜像**只要运行

   ```shell
   sudo ./flash.shjetson-tk1 mmcblk0p1
   ```

   就可以了。

   而如果你想**将现有妙算的系统备份**则需要用到**制作系统镜像**中的内容。

   这里我推荐一种**批量刷机**的方法，在比赛中需要用到多个妙算，如果每个妙算都要从零开始配置工作量是巨大的。当我们在一台妙算上把所有的配置搞定之后就可以通过**制作系统镜像**功能来备份该妙算的系统，然后通过**恢复系统镜像**的功能来刷机。

   ```shell
   #系统镜像备份代码
   sudo ./nvflash --read APP system.img --bl ardbeg/fastboot.bin --go
   #将镜像中的系统恢复到妙算
   sudo ./flash.sh –r jetson-tk1 mmcblk0p1
   ```

   有关刷机的部分可以参考<https://elinux.org/Jetson/Cloning>

 # 妙算安装系统后要做的事

## 妙算通过网线直连电脑并共享电脑网络

在 Ubuntu 18.04 之前使用的网络管理器是 Network Manager，但是在 18.04 后换了，可能是为了符合 Gnome3 的审美吧，不过这样就缺失了一个超级好用的功能 **网络共享** 通过这个功能可以实现将电脑作为路由器将网络通过网线共享给妙算。

由于我没有怎么用过 Ubuntu 18.04 主要都在用 16.04 ，所以我不是很清楚怎样在 18.04 上安装 Network Manager 可以参考官方的这篇文章来安装 <https://help.ubuntu.com/community/NetworkManager>

下面进入正题：

1. 首先找到系统托盘中的 **网络图标** ，右键单击后再左键点击 **编辑连接**
![network](https://img-blog.csdnimg.cn/20190702185640814.png )

2. 在出现的网络连接界面中点击 **增加**
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019070218593027.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)

3. 选择 **以太网** 后点击 **新建**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190702190010874.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)
4. 点击 **ipv4 设置**，在 **方法** 上选择 「**与其他计算机共享**」，然后保存。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190702190056765.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)
5. 之后在终端输入`arp -a`就可以查看妙算对应的 ip 了。10.42.0.1 是电脑的 ip ，10.42.0.xx 是妙算的 ip 。

## 妙算远程调试

妙算可以直接接上显示器键盘鼠标当成正常电脑来用，但是如果手头没有多余的显示器键盘鼠标怎么办呢？可以通过 ssh 实现远程调试。

可能大家对 ssh 的印象只是停留在命令行中，其实 ssh 可以通过  X11 forwarding 来实现在本机中运行远程计算机中的图形界面程序，超级方便有没有，只要你输入 qtcreator 立刻一个编辑器的界面就弹出来了就像在运行你自己电脑中的程序一样。当然放视频会很卡。

```shell
ssh -XC ubuntu@10.42.0.178 #后面的是妙算的ip
```

与一般 ssh 命令不同的地方就是多了 -XC 选项，X 是启用 X11 forwarding ，C 是用来通过压缩传输所有信息。

## 妙算安装 FTP

如果需要经常在妙算与电脑之间传输数据，最方便的方式不是优盘而是通过 FTP ，鉴于网上安装 ftp 的资料非常多，这里就不给出了（允许我偷下懒 (\^-^*) ）。

## 妙算配置软件源

妙算用的 Ubuntu 与我们在桌面级电脑用的 Ubuntu 是不一样的，如果按照常规的桌面系统配置软件源的化一定无法使用。官方源的地址是 <http://ports.ubuntu.com/>  而由于国内网络劫持严重使用官方源有大概率会下载失败，所以推荐使用清华或者中科大的软件源，软件源的修改方式可以参看中科大的帮助链接 <https://mirrors.ustc.edu.cn/help/ubuntu-ports.html> 需要注意的是在更改软件源的时候一定要选择对应于自己系统版本的软件源，妙算系统是 14.04 的所以需要用 14.04 的源。

默认妙算是官方源，手动修改时只需要在 `/etc/apt/sources.list` 文件中，将软件源的地址改为 `http://mirrors.ustc.edu.cn/ubuntu-ports`即可。

## 妙算系统标题栏一跳一跳解决方案

官方的默认系统是存在 bug 的，就是系统标题栏会一跳一跳，具体原因不清楚，不过很好修复，只要将系统更新即可。

```shell
sudo apt update
sudo apt upgrade
```

## 妙算安装 OpenCV / CUDA
妙算最好安装官方的 **OpenCV4tegra** ，因为 **OpenCV4tegra** 是专门针对妙算优化过的。在我实际应用中跑装甲检测程序用官方的 OpenCV 可以跑到 6ms 一帧，而自己编译的最快跑到 8ms 。

CUDA 也直接按照说明书安装即可，不过 TK1 只能使用 CUDA6.5 再往上不支持了，所以很多新的深度学习框架就不能用了，我安装成功的只有 caffe1 的 rc5 版本再往上就有软件包依赖性问题无解，可以到 GitHub 上找到源码自行编译。

## 妙算安装 GCC5

由于我们用的视觉程序主框架是参考东南大学开源的程序，而他们使用了 C++14 中的新特性 unique_ptr 需要 GCC5 的支持，妙算由于使用的 Ubuntu 14.04 系统只提供 GCC 4.8 所以需要安装 GCC5 ，GCC5 的安装过程我忘了，当时可能参考了 <https://blkstone.github.io/2016/05/25/ubuntu-1404-gcc-5/> 。如果有误请留言或私信告诉我。

## 妙算安装配置Qt Creator

Qt Creator 应该是 linux 上最好用的 IDE 了。有人可能不服， Vim 也很牛逼啊加上插件吊打一切， Emacs 的快捷键用着多爽，但是 Qt Creator 是学习成本最少界面最友好的，而且 qmake 的编译管理做的非常棒，寥寥几句就可以描述一个完整的项目与 cmake 对比爽太多了。

妙算的 cpu 由于是 Arm 架构的而且系统为 Ubuntu 14.04 ，所以无法安装最新的 Qt Creator ，不过不要紧源中的 Qt Creator 已经足够用了。

```shell
sudo apt install qtcreator
```

安装完 Qt Creator 后编译器 Qt 环境什么的都是装好的，但是如果需要使用 GCC5 的话需要自行配置编译器，由于手头没有妙算，我就在自己电脑上截图凭印象说了，可能会与实际情况有出入。

首先点击：**工具-->选项**，之后弹出选项界面。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190702190205239.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)

在选项界面找到 **构建和运行中** 的 **编译器** 可以在这里更改编译器路径，将他们都改成 GCC5 对应的路径就可以啦。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190702190156316.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)
申请了一个自己的公众号**江达小记**，打算将自己的学习研究的经验总结下来帮助他人也方便自己。感兴趣的朋友可以关注一下。

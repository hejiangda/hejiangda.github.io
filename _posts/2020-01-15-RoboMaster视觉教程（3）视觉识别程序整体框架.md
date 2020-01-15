---
layout: post
title:  "RoboMaster视觉教程（3）视觉识别程序框架"
date:   2020-01-15 20:34:21 +0800
categories: RoboMaster
---

## 概览

RoboMaster 视觉识别是一个比较大的项目了，综合性太强。这里从程序框架的角度来粗略讲一下需要怎么做。比较好的框架有官方开源的视觉程序，东南大学开源的视觉程序，其中东南大学开源的程序可以认为是官方开源程序的加强版。他们的程序层次分明清晰易读非常具有扩展性，在其基础上可以很好地修改和扩展功能。

## 多线程

官方的开源程序和东南的开源程序最大的特点就是类封装和多线程。通过类封装让程序结构分明各部分功能清晰，而多线程通过并发执行增加 cpu 利用率提高算法速度。

一般我们在学习 opencv 的时候写程序通常的流程就是读入图片/视频/摄像头，对每张图片进行处理，处理后进行输出（显示图片/显示识别结果等等）。这种模式基本上是一条线走下来的，前一个步骤没有完成就无法进行下一步的操作。

而使用多线程后就可以把每个步骤拆分开，用单独的线程来完成对应的操作。有人会说不管怎么拆不还是需要先读图再处理再输出嘛。

但是由于使用了多线程就可以进行流水线作业，这样在图像处理线程处理本张图片的时候图像读取线程可以读入下一张图片。

一般而言图像读取速度会比图像处理慢得多，一个 120fps 的摄像头平均一帧所花费的时间是 8ms ，而图像处理所花的时间则在 1～3ms 左右（只识别装甲板灯条），这时候平均每帧的处理时间就是 8ms 。如果换 330fps 的摄像头则平均每帧的处理时间就是 3ms 。通过多线程可以极大地减少算法的用时，提高效率。

## 除了多线程，还可使用多进程

之前看到 <https://wzq.io/?p=345> 这篇文章，他通过两个进程来实现图像的获取和处理，我试过他的方法，这样做效率没有多线程高，获取图片时会增加 1ms 左右的延时，但是灵活性很强稳定性更高，可以让多个 client 进程通过 server 调取摄像头图片。

如果需要对同一张图片分别做各种处理，这种方式就很灵活简洁了，比如一个图像处理进程、一个图像传输进程就可以实现边处理图片边发送图片达到图传与处理同时进行的效果。

图像获取与处理通过进程来实现可以确保任意一方挂了对另一方没有影响，通过看门狗实现崩溃重启继续工作。由于时间关系在备赛中我没有采用这个方案，不过可以作为参考。

## 接下来以东南大学的开源程序为例讲一下他们的整体架构

东南大学2018年视觉开源程序 GitHub 地址：<https://github.com/SEU-SuperNova-CVRA/Robomaster2018-SEU-OpenSource>

他们的代码写得非常规范，而且也有很详细的注释和说明，现在把代码开源的队伍有很多，但像他们这样做得如此规范的队伍少有，很多队伍把代码开源后就不管了也没有注释什么的。

通过读他们的代码可以学到很多东西，首先他们的代码是通过 GitHub 来协作编写的，用 GitHub 的好处是多人编写代码时不会乱套。

我17年参加比赛时当时负责视觉和部分电控，当时代码协作就是靠优盘拷，经常会发生代码中的一些参数没改云台疯了之类的情况。

用 GitHub 的另一个好处是代码每个版本都有备份，可以随心所欲的写代码，不用担心之前的代码找不回来的窘境。 qtcreator 内置有一个git插件，只需要简单设置就可以图形化地使用 GIt 。

其次他们用面向对象的设计思路用若干个类来组织代码提高了程序的可读性和可维护性，他们将每个功能划分成类然后在调用的时候通过指针来实例化并调用相应的成员函数实现功能。

很多同学来问我说 SEU 的开源代码跑不起来，这其实是因为他们的代码默认的一些逻辑是需要与下位机配合的。

如果在用他们的代码中遇到了问题可以试试我改过的代码：[改过的代码](https://github.com/hejiangda/RM-Archive/raw/master/Robomaster2018-SEU.zip)

我仅仅注释了影响代码跑起来的部分没有做大的改动，这不是我们队实际用的代码。

我们队的开源可以看bbs上的这个帖子： [【哈尔滨工程大学】创梦之翼战队RM19全方位开源汇总贴](https://bbs.robomaster.com/thread-9233-1-1.html)  目前队里还没整理好。

#### 下面进入正题

GitHub 上下载源码解压后可以看到如下文件和文件夹，我在图上标注了用途，由于今年大符大改，以往的代码都作废所以装甲识别可以沿用并改进原有代码，大符的要自己写。在  README.md 中有项目说明和算法介绍，写得很好。
![seu2](https://img-blog.csdnimg.cn/20190710083629128.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70#pic_center)


#### 项目配置文件概览

在 qtcreator 中打开项目后可以看到项目全貌，双击 .pro 文件可以查看项目的配置情况，`CONFIG += c++ 14`是配置 qmake 支持 c++14  如果在妙算上用则由于版本问题该行失效，需要使用 `QMAKE_CXXFLAGS += -std=c++1y`  来支持 c++14 ，下面 CUDA 的部分可以删掉，因为今年视觉不需要用到显卡加速，无论是风车还是装甲识别算法都很简单不需要用到深度学习。 V4L2 是摄像头驱动，
![seu3](https://img-blog.csdnimg.cn/20190710083826818.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70#pic_center)

Darknet 是 yolo 作者写的一个开源深度学习框架，不需要使用可以删除。接下来是头文件和源文件，这些是添加相应文件后 qtcreator 自动生成的，当然也可以手动添加或者手动注释。如果不需要编译某个文件在前面加一个`#`就可以了，如果要跨行，需要在末尾添加一个`\`，否则会出错。
![seu4](https://img-blog.csdnimg.cn/20190710083908733.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)

#### ImgProdCons 类

该类可以看作是对整个系统的抽象，其成员函数包含了程序参数初始化，生产者消费者线程函数等，通过智能指针来调用其他类。

```c
/*
* @Brief:   This class aims at separating reading(producing) images and consuming(using)
*           images into different threads. New images read from the camera are stored
*           into a circular queue. New image will replace the oldest one.
*/
class ImgProdCons
{
public:
    ImgProdCons();
    ~ImgProdCons() {}

    /*
     * @Brief: Initialize all the modules
     */
	void init();

    /*
     * @Brief: Receive self state from the serail port, update task mode if commanded
     */
    void sense();

	/*
    * @Brief: keep reading image from the camera into the buffer
	*/
	void produce();

	/*
    * @Brief: run tasks
	*/
	void consume();

private:
    /*
    * To prevent camera from dying!
    */
    static bool _quit_flag;
    static void signal_handler(int);
    void init_signals(void);

    /* Camera */
    std::unique_ptr<RMVideoCapture> _videoCapturePtr;
    FrameBuffer _buffer;

    /* Serial */
    std::unique_ptr<Serial> _serialPtr;

    /* Angle solver */
    std::unique_ptr<AngleSolver> _solverPtr;

    /* Armor detector */
    std::unique_ptr<ArmorDetector> _armorDetectorPtr;

    /*Rune detector*/
    std::unique_ptr<RuneDetector> _runeDetectorPtr;

    /* @See: 'Serial::TaskMode' */
    volatile uint8_t _task;

	void updateFeelings();
};
```

如果自己写了算法，比如写一个打风车大符的算法也可以仿照类似的方式用智能指针来指向它，其实用普通指针也可以，不过用智能指针`unique_ptr`可以在管理资源同时保证安全性，创建对象的时候使用`make_unique`可以安全地创建对象，由于`make_unique`是 c++14 才支持的所以要用 gcc5 来编译。读东南大学的开源代码经常会看到一些最新的特性的使用和一些技巧，学到了很多，很佩服。

#### 主函数

主函数主要作为一个入口，创建 `ImgProdCons` 类，执行初始化成员函数，创建线程，之后主函数就完成了使命。这点和qt的编程很像，如果创建一个qt widget 应用，则会自动生成一个 `MainWindow` 类，自动生成的 `main` 中会帮写好初始化和显示的代码，只需在qt设计师中设计窗口添加槽函数补充 `MainWindow` 类就能很容易写出一个简单的qt程序。

```c
int main()
{
    rm::ImgProdCons imgProdCons;

    imgProdCons.init();

    std::thread produceThread(&rm::ImgProdCons::produce, &imgProdCons);
    std::thread consumeThread(&rm::ImgProdCons::consume, &imgProdCons);
    std::thread senseThread(&rm::ImgProdCons::sense, &imgProdCons);

    produceThread.join();
    consumeThread.join();
    senseThread.join();

    return 0;
}
```

produceThread 负责获取图像保存到缓存队列中，consumeThread 负责对图图片的处理和指令的发送，而 senseThread 用于接收数据。

其实除了这几个线程还可以添加一个保存视频线程和一个串口发送线程，经实测将串口发送也用线程来做可以省 1～2ms 左右的时间，保存视频的线程可以用来保存场上的比赛资料，现在的 minipc 的容量大的吓人随便来一个就有100多G的固态硬盘容量，就算是妙算也有10个G可以用，把视频保存下来后可以针对场上发生的各种情况来进行改进，这种第一视角的视频可以看到很多被忽略的细节。

#### 用类来包装算法

在写一个检测算法的时候我往往是怎么简单怎么来，很多要调的参数什么的就直接原样写在程序中了，整个算法也都堆在 main 中。

这么搞对于自己写着玩没问题，但是如果要把写好的算法拿来用那就要适当地包装一下自己的算法了。

通过合理地拆解自己的算法按照面向对象的方式组成类可以提高程序的可读性和健壮性。开源代码中的装甲检测、大符、位姿解算和串口通信都提供了很好的范例，照样画葫芦做就可以了。我本身也不太懂面向对象的东西，大学在课堂中只学过 c 语言，用到的一些 c++ 的东西都是现学现卖的。

现在以装甲检测举例，分析一下他们是如何用类来包装的。

在 ArmorDetector.h 中，有一个 ArmorParam 结构体和三个类 LightDescriptor 、 ArmorDescriptor 、 ArmorDetector 。

其中 ArmorParam 结构体用来存放一些算法用到的常量并用构造函数初始化， LightDescriptor 类用来描述检测到的灯条， ArmorDescriptor 类用来描述装甲板， ArmorDetector 是最终用来检测装甲的类。

可以依识别步骤先将灯条抽象出来建一个类，用自定义的类来描述灯条，每个灯条都会有宽高面积角度等属性。

再将装甲板抽象出来，装甲板也有宽高面积倾斜角度以及装甲的种类等属性。

而整个识别过程也通过类来描述，识别的整个流程包括初始化参数、定义敌方颜色、加载图像、检测装甲、返回装甲信息等，将这些步骤每一步都用成员函数来封装，可以让代码更易读易改。

申请了一个自己的公众号 **江达小记** ，打算将自己的学习研究的经验总结下来帮助他人也方便自己。感兴趣的朋友可以关注一下。

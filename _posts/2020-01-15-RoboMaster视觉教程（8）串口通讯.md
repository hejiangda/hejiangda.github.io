---
layout: post
title:  "RoboMaster视觉教程（8）串口通讯"
date:   2020-01-15 20:38:01 +0800
categories: RoboMaster
---

## 概览

这几天一直在做一个小车打算做好了再往下写的，但是由于我两年没写stm32的程序了，写好程序还是很吃力的。再加上这几天要准备考科目三（考驾照好辛苦 `T_T` ）准备开学考试（没错！开学就要考试，还要考三门我没学过的课 `T_T` ）事比较多，就停更了两个星期，下一篇我也不清楚什么时候发不过有时间会一点一点地写。

在视觉识别中一般是用妙算或者其他迷你电脑作上位机完成复杂的识别功能，在识别到目标后通过串口向下位机传送命令指挥小车云台运动。

妙算或者其他使用linux系统的机器直接用DJI或者东南大学开源代码中的串口部分就可以了，使用windows系统的机器可以参考东北林大的开源代码中的串口部分。

## DJI开源代码串口部分

大疆的开源代码中串口部分写的比较简洁，主要就四个函数。openPort、configurePort、sendXYZ和praseDatafromCar。

串口的操作其实和文件读写类似，或者说IO相关的操作其实都差不多，都是先获取文件描述符fd再使用read和write函数进行读写操作。

```cpp
#include <stdio.h>      // standard input / output functions
#include <string.h>     // string function definitions
#include <unistd.h>     // UNIX standard function definitions
#include <fcntl.h>      // File control definitions
#include <errno.h>      // Error number definitions
#include <termios.h>    // POSIX terminal control definitionss

int openPort(const char * dev_name){
    int fd; // file description for the serial port
    fd = open(dev_name, O_RDWR | O_NOCTTY | O_NDELAY);
    if(fd == -1){ // if open is unsucessful
        printf("open_port: Unable to open /dev/ttyS0. \n");
    }
    else  {
        fcntl(fd, F_SETFL, 0);
        printf("port is open.\n");
    }
    return(fd);
}

int configurePort(int fd){                      // configure the port
    struct termios port_settings;               // structure to store the port settings
    cfsetispeed(&port_settings, B115200);       // set baud rates
    cfsetospeed(&port_settings, B115200);

    port_settings.c_cflag &= ~PARENB;           // set no parity, stop bits, data bits
    port_settings.c_cflag &= ~CSTOPB;
    port_settings.c_cflag &= ~CSIZE;
    port_settings.c_cflag |= CS8;

    tcsetattr(fd, TCSANOW, &port_settings);     // apply the settings to the port
    return(fd);
}

bool sendXYZ(int fd, double * xyz){
    unsigned char send_bytes[] = { 0xFF,0x00,0x00,0x00,0x00,0x00,0x00,0xFE};
    if(NULL == xyz){
        if (8 == write(fd, send_bytes, 8))  //Send data
            return true;
        return false;
    }
    short * data_ptr = (short *)(send_bytes + 1);
    data_ptr[0] = (short)xyz[0];
    data_ptr[1] = (short)xyz[1];
    data_ptr[2] = (short)xyz[2];
    if (8 == write(fd, send_bytes, 8))      //Send data
        return true;
    return false;
}
//这个函数是在RemoteController.cpp中的
void RemoteController::praseDatafromCar(){
    char buf[255]={0};
    size_t bytes = 0;
    ioctl(fd_car, FIONREAD, &bytes);
    if(bytes > 0 && bytes < 255)
        bytes = read(fd_car, buf, bytes);
    else if(bytes >= 255)
        bytes = read(fd_car, buf, 255);
    else
        return;

    praseData(buf, bytes);
}
```

在打开串口时需要提供串口设备的文件地址类似于`/dev/ttyUSB0`。

如果使用妙算的话可以使用妙算自带的GPIO上的几个串口。

如果使用USB串口，在插拔的过程中有可能会出现串口号变化的情况，比如上次是ttyUSB0然后程序挂了或串口出错了，插拔usb转串口之后串口号可能变成ttyUSB1。对于这种情况可以先将当前系统中有效的串口找出来然后再打开串口，可以参考stakoverflow中

https://stackoverflow.com/questions/2530096/how-to-find-all-serial-devices-ttys-ttyusb-on-linux-without-opening-them

串口通信为保证上下位机数据准确需要制定一个通信协议，我最开始做比赛的时候是用字符串来对发送数据进行描述的，类似于“Y010P020"来代表yaw轴转10度pitch轴转20度，在下位机stm32上用最原始的加法和乘法把字符组合成数据。

这种方式不仅不好看效率也很低也很容易出错，合理的方法是定义发送帧和接收帧，用帧头和帧尾来校验帧是否正确，帧头和帧尾中间放数据。

这是DJI开源代码中定义的上位机向下位机传输的帧的格式，可以看到中间的数据部分是用两个字节组合成一个16位的整数。

![frame1](https://img-blog.csdnimg.cn/20190821203848962.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)

串口发送时一般是发送一段unsigned char数组，其中的每一个字节都可以通过强制类型转换来表示其他的类型。这有点类似于c语言中的union用一段内存来表示不同的数据类型。

以上述帧为例第一个字节为帧头0xFF，中间6个字节组成三个16位整数，就可以像下面这样写。

```cpp
unsigned char data[] = { 0xFF,0x00,0x00,0x00,0x00,0x00,0x00,0xFE};
*(short*)(data+1)=(short)value1;
*(short*)(data+3)=(short)value2;
*(short*)(data+5)=(short)value3;
```

下位机在接收的时候首先找0xFF找到后再接收7个字节，对比最后一个字节是否是0xFE若不是则丢弃，若是则本次数据有效，之后同样可以采用强制类型转换的方式来提取数据。

如果想发送其他类型的数据的话同样可以用这种方式，但要根据类型的大小分配好需要的字符数组大小以防越界。

大疆开源代码中下位机对上位机的帧的格式如下：

![frame2.png)](https://img-blog.csdnimg.cn/20190821203946118.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)

对应的串口接收和命令解析由`void RemoteController::praseData(const char * data, int size)`函数完成。

这个函数中比较有意思的一段是

```cpp
case 2:{                        // pitch angle
    int a = 0;
    a |= (0x7f & cmd2);
    a = (0x80 & (cmd2)) == 0 ? a : a | 0xffffff80;
    other_param->angle_pitch = (int)a / 4.0;
    //std::cout << "angle_pitch:" << other_param->angle_pitch << '\n';
    break;
}
```

 `a |= (0x7f & cmd2);`这行干啥的呢？就是将cmd2这个字节的后7位赋值给a。而下一句`a = (0x80 & (cmd2)) == 0 ? a : a | 0xffffff80;`则是判断cmd2是否是负数如果是就把a的剩余位数都赋1变成负数。

我觉得这段代码写得不好，明明有更好看易懂的写法，直接`int a=*(char*)&cmd2`不就好啦 `^_^`

另外串口波特率需要设置合理，如果波特率太高则误码率会增加，波特率太低则发送速度太慢。

以波特率115200为例，它表示每秒发送115200位，换算成字节每秒是11520（不加校验位）也就是除以十，按上例每次发送的数据为8字节，则除8得到每秒最大可发送指令1440次，这样对于100多帧的摄像头来说是够用的（我觉得串口的发送速度至少要比摄像头的帧率大10倍以上）。

如果想每次多传些数据，那就需要提高波特率了，在东南大学的开源代码中他们把波特率设置为460800也就是115200的4倍，他们定义的帧每次发送需要传输16字节接收需要20字节在这个波特率下可以满足性能需要。

## 东南大学开源代码串口部分

东南大学的串口部分的开源代码兼顾了调试需要，对一些异常情况也考虑的比较周到。最值得称赞的地方就是他们定义的帧格式考虑很周全，这样在设计自己的通信协议时极大地减少了工作量。

```cpp
/*
 * @Brief: 控制战车帧结构体
 */
struct ControlFrame
{
    uint8_t  SOF;
    uint8_t  frame_seq;
    uint16_t shoot_mode;
    float    pitch_dev;
    float    yaw_dev;
    int16_t  rail_speed;
    uint8_t  gimbal_mode;
    uint8_t  EOF;
}_controlFrame;

/*
 * @Brief: 战车回传数据帧结构体
 */
struct FeedBackFrame
{
    uint8_t  SOF;
    uint8_t  frame_seq;
    uint8_t  task_mode;
    uint8_t  bullet_speed;
    uint8_t  rail_pos;
    uint8_t  shot_armor;
    uint16_t remain_HP;
    uint8_t  reserved[11];
    uint8_t  EOF;
}_feedBackFrame;

/*
 * @Brief: 比赛红蓝方
 */
enum TeamName
{
    BLUE_TEAM       =   (uint16_t)0xDDDD,
    RED_TEAM        =   (uint16_t)0xEEEE
};

/*
 * @Brief: control frame mode
 */
enum ControlMode
{
    SET_UP          =   (uint16_t)0xCCCC,
    RECORD_ANGLE    =   (uint16_t)0xFFFF,
    REQUEST_TRANS   =   (uint16_t)0xBBBB
};

/*
 * @Brief: 发射方式
 */
enum ShootMode
{
    NO_FIRE         =   (uint16_t)(0x00<<8),//不发射
    SINGLE_FIRE     =   (uint16_t)(0x01<<8),//点射
    BURST_FIRE      =   (uint16_t)(0x02<<8) //连发
};

/*
 * @Brief: 发射速度
 */
enum BulletSpeed
{
    HIGH_SPEED      =   (uint16_t)(0x01),   //高速
    LOW_SPEED       =   (uint16_t)(0x02)    //低速
};

/*
 * @Breif:所需控制模式
 */
enum TaskMode
{
    NO_TASK         =   (uint8_t)(0x00),    //手动控制
    SMALL_BUFF      =   (uint8_t)(0x01),    //小符模式
    BIG_BUFF        =   (uint8_t)(0x02),    //大符模式
    AUTO_SHOOT      =   (uint8_t)(0x03)     //自动射击
};

/*
 * @Brief: 哨兵云台工作模式
 */
enum GimbalMode
{
    PATROL_AROUND   =   (uint8_t)(0x01),    //旋转巡逻
    PATROL_ARMOR_0  =   (uint8_t)(0x02),    //巡逻装甲板0
    PATROL_ARMOR_1  =   (uint8_t)(0x03),    //巡逻装甲板1
    SERVO_MODE      =   (uint8_t)(0x04)     //伺服打击
};

/* @Brief:
     *      SYSTEM_ERROR:   System error catched. May be caused by wrong port number,
     *                      fragile connection between Jetson and STM, STM shutting
     *                      down during communicating or the sockets being suddenly
     *                      plugged out.
     *      OJBK:         Everything all right
     *      PORT_OCCUPIED:  Fail to close the serial port
     *      READ_WRITE_ERROR: Fail to write to or read from the port
     *      CORRUPTED_FRAME: Wrong frame format
     *      TIME_OUT:       Receiving time out
     */
enum ErrorCode
{
    SYSTEM_ERROR    = 1,
    OJBK            = 0,
    PORT_OCCUPIED   = -1,
    READ_WRITE_ERROR= -2,
    CORRUPTED_FRAME = -3,
    TIME_OUT        = -4
};
```

## Qt编写串口助手

Qt是非常好用的跨平台开源的GUI程序开发库，我非常喜欢Qt。Qt有详细的文档和大量的示例程序，很多示例程序只需要稍微改一改就可以写出我们想要的功能，对比之下用GTK开发就困难多了。

这里我们就来用Qt自带的串口终端的例子来实现一个串口助手，Qt编写的代码是跨平台的也就是三大主流系统Windows、Linux、macOS都可以用一套代码实现相同的功能，这个例子我在Windows和Linux下测试都是好用的。

打开Qt Designer 在Welcome界面中点击Example 在搜索框中搜索 terminal 就能找到串口终端的例子

![/8/qt1.png)](https://img-blog.csdnimg.cn/20190821204033712.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)![/8/qt2.png)](https://img-blog.csdnimg.cn/20190821204059405.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)双击后选择**复制项目并打开**会进入配置界面，按默认配置即可

![/8/qt3.png)](https://img-blog.csdnimg.cn/20190821204131950.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)
![/qt4.png)\]](https://img-blog.csdnimg.cn/20190821204151891.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)
在弹出配置界面的同时也会弹出该例子的说明帮助，Qt的帮助文档都写得非常规范，读了会很有收获
![qt5.png)\]](https://img-blog.csdnimg.cn/20190821204211465.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)
这个例子直接编译运行就能得到一个简易的串口助手。
![/8/qt6.png)\]](https://img-blog.csdnimg.cn/20190821204228633.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)
点击齿轮按钮可以打开串口配置窗口，左边列出了目前系统中存在的串口，右边是波特率校验位等的设置。配置好后点击连接按钮就可以打开串口了。
![/8/qt8.png)\]](https://img-blog.csdnimg.cn/20190821204252305.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)
这个示例程序实现了数据发送和接收功能，在黑框中可以直接输入要发送的数据，同时接收数据也会传到黑框中。

数据的接收和发送就是调用read和write函数

```cpp
void MainWindow::writeData(const QByteArray &data)
{
    m_serial->write(data);
}
void MainWindow::readData()
{
    const QByteArray data = m_serial->readAll();
    m_console->putData(data);
}
```

如果希望一打开软件就能自动连接串口可以在`MainWindow`的构造函数中加入`openSerialPort();`因为Qt的这个例程默认会将搜到的第一个有效串口设置为要打开的串口，所以加上这句后每次只要提前把串口插入再打开软件就会以默认参数打开这个串口（这非常的方便，不用选串口不用设置波特率什么的，插上就能用）。

网上有很多串口软件很傻屌，有的把所有com号都列出来让用户自己找有效的，有的只给四个com号让用户选，如果com号刚好分配到这四个号以外还要到设备管理器里改com号。

而通过调用`QSerialPortInfo::availablePorts()`就可以把这个问题完美解决了。

```cpp
void SettingsDialog::fillPortsInfo()
{
    ui->serialPortInfoListBox->clear();
    QString description;
    QString manufacturer;
    QString serialNumber;
    //查找有效的串口
    const auto infos = QSerialPortInfo::availablePorts();
    //遍历填充窗口信息
    for (const QSerialPortInfo &info : infos) {
        QStringList list;
        description = info.description();
        manufacturer = info.manufacturer();
        serialNumber = info.serialNumber();
        list << info.portName()
             << (!description.isEmpty() ? description : blankString)
             << (!manufacturer.isEmpty() ? manufacturer : blankString)
             << (!serialNumber.isEmpty() ? serialNumber : blankString)
             << info.systemLocation()
             << (info.vendorIdentifier() ? QString::number(info.vendorIdentifier(), 16) : blankString)
             << (info.productIdentifier() ? QString::number(info.productIdentifier(), 16) : blankString);

        ui->serialPortInfoListBox->addItem(list.first(), list);
    }

    ui->serialPortInfoListBox->addItem(tr("Custom"));
}
```

通过简单的修改，我们就可以实现一个交互式的控制软件以方便调试，也可以结合Qt中的其他模块实现数据可视化、数据分析等功能。

例如我给我的小车方便调试做的一个简单的控制车轮速度的软件如下：
![/8/qt7.png)\]](https://img-blog.csdnimg.cn/20190821204320495.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)
给闲鱼上淘来的写字机器人写的控制软件（就是那个小朋友买来抄作业的）：
![/8/qt9.png)\]](https://img-blog.csdnimg.cn/20190821204335948.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)



申请了一个自己的公众号**江达小记**，打算将自己的学习研究的经验总结下来帮助他人也方便自己。感兴趣的朋友可以关注一下。

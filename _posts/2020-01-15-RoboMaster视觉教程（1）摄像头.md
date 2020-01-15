---
layout: post
title:  "RoboMaster视觉教程（1）摄像头"
date:   2020-01-15 20:32:35 +0800
categories: RoboMaster
---

# 摄像头参数
摄像头应该是机器视觉中最重要的部分了，选择一款成像质量好稳定可靠的摄像头可以极大地减少识别算法设计的难度。主流的摄像头分为CMOS摄像头和CCD摄像头。一般而言CCD摄像头体积大造价高精度高，而CMOS摄像头由于集成度高造价远低于CCD摄像头，同时CMOS摄像头的体积功耗等参数也相应地优于CCD摄像头。本文主要总结CMOS摄像头的选型（因为没玩过CCD摄像头）。
## 卷帘曝光与全局曝光
通常我们在网上买到的摄像头都是卷帘曝光的摄像头，在日常使用时很难看出这两种摄像头的区别，但是在对速度要求高的领域这两种曝光方式的优劣就很明显了，尤其是对于廉价的摄像头卷帘曝光的果冻效应更加明显。
有关卷帘曝光和全局曝光的区别可以参看：[聊聊快门：卷帘/全局快门与果冻效应](https://zhuanlan.zhihu.com/p/33920660)

除了果冻效应外，全局曝光的摄像头在低曝光时间的情况下的颜色饱和度更高，在实际测试比较中发现同样将曝光值调到最小，在拍摄裁判系统红色灯条的时候，KS2A17（200万像素卷帘曝光摄像头）与KS1A552（100万像素全局曝光摄像头）对比明显，KS2A17所拍摄的灯条偏暗偏灰，并且会闪烁，而KS1A552色彩鲜艳。如图1图2所示：（由于现在已经不做比赛了，就以之前拍摄的视频截图举例）



|![在这里插入图片描述](https://img-blog.csdnimg.cn/20190608151148680.png =200x )  | ![KS1A552拍摄能量机关](https://img-blog.csdnimg.cn/2019060814423486.png =200x) |
|--|--|
| 图 1  KS2A17低曝光拍摄装甲板 |图2 KS1A552低曝光拍摄能量机关 |

通过以上两图对比，在颜色方面全局曝光摄像头完爆卷帘曝光摄像头。
## 曝光
在装甲识别中，曝光度是决定能否成功识别到装甲板的关键因素。我大二做比赛时，由于是从零开始并不知道摄像头有那么多的参数可以调节，当时以为分辨率高颜色逼真的摄像头就是好摄像头，于是买了个微软的LifeCam Studio，这个摄像头应该是我买过的最失败的摄像头，又贵又难用。第一次知道调节曝光的威力还是在看了官方开源的代码后，当时的感觉就是那种突然开窍的感觉。在不知道曝光的时候用的方法是YOLO，这算法很牛逼，但很慢，连电脑上跑起来都卡就别说妙算上了。当时跑YOLO的帧率为电脑上28fps，妙算上8fps。下面直接上图对比一下摄像头正常曝光和低曝光的巨大差别。
|![在这里插入图片描述](https://img-blog.csdnimg.cn/20190608151020469.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70 )  | ![在这里插入图片描述](https://img-blog.csdnimg.cn/2019060815112570.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70#pic_center) |
|--|--|
|图3 正常曝光下拍摄的图片  | 图4 低曝光下拍摄的图片 |

所以当我们拍摄能自主发光的物体时，降低曝光时间可以减少环境光的影响。这时通过阈值处理得到二值图就可以进一步处理。

## Gamma矫正
在降低装甲板的误识别率的过程中，我们可以通过各种约束来过滤误匹配的情况，而更聪明的方法其实是识别装甲板中间的数字。但是当我们把摄像头的曝光调到很低的情况下数字就看不见了（如图5），这时候怎么办呢？一种不太容易想到的方法是提高摄像头的Gamma值。当Gamma提高时图像中亮度较低的区域的亮度会被提高。如图5～图7所示：

| ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190608185243704.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70) |![在这里插入图片描述](https://img-blog.csdnimg.cn/20190608153830936.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)  |![在这里插入图片描述](https://img-blog.csdnimg.cn/20190608153838573.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)|
|--|--|--|
|图5 没有提高Gamma |图6 提高Gamma |图7 进一步提高Gamma|
图5中几乎看不出中间的数字，而在提高Gamma后可以看清中间的数字，当进一步提高Gamma后数字可以明显被认出。有人可能会有疑问，既然暗的区域的亮度提高了，那不就和降低曝光前是一样的了吗。并不一样，提高Gamma后图像中被提升亮度的部分为灰色而红色部分的灯条没有明显变化。如图8，图中被提高亮度的部分基本上都为灰色，由于灰色的rgb值相同，所以只要将图片的两个通道相减就能去除背景的干扰。如图9，要识别红色只需要红色通道减去蓝色通道，留下来的便是需要的。在找到灯条后框出数字区域给识别函数识别即可。
| ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190608185931821.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)|![在这里插入图片描述](https://img-blog.csdnimg.cn/20190608190122461.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)|
|--|--|
|图8 Gamma矫正结果|图9 红蓝通道相减|
那摄像头的Gamma矫正到底做了些什么呢？其实做的事情很简单，就是将图像的每一个像素点通过一个幂函数进行了转换，而幂函数的指数的倒数就是通常所说的gamma值，该函数的图像如图10所示：
|![在这里插入图片描述](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9wLWJsb2cuY3Nkbi5uZXQvaW1hZ2VzL3BfYmxvZ19jc2RuX25ldC9jb2xvcmFudC9hMDYyNjNjMTIzNjk0NmVkYjkyZmI3NWU5YWMzNjcxNi5wbmc)  |  
|--|
| 图10 Gamma矫正函数曲线 |
从曲线中可以看到输入数值较小时输出对数入的比值较大，由此可以提高图像暗处的亮度。
有关曝光和Gamma可以参看[这篇博文](https://blog.csdn.net/colorant/article/details/1913271)
## 帧率与摄像头选型
对于RoboMaster比赛而言，识别的速度当然是越快越好，所以摄像头帧率也是越高越好。由于带宽的限制帧率和分辨率通常互相制约，想要高帧率分辨率就会低。下表是我使用过的摄像头的分辨率与帧率表：
|型号|分辨率  |帧率|
|--|--|--|
| KS2A17 彩色卷帘| 640*480 |120 |
||1280*720|60|
||1920*1080|30|
| KS1A293 黑白全局 | 640*480 | 240|
||1280*800|120
| KS1A552 彩色全局| 640*480 |60 |
||1280*800|60|
| WX605 彩色卷帘| 640*320 |330 |
||1280*720|120|
||1920*1080|60
可以看到分辨率取最低的时候通常可以达到最大帧率，事实上对于妙算（TK1）而言也仅能处理分辨率为640*480左右的图片，分辨率再高就跑不动了，选用WX605时可以达到最快单幅图片6ms左右的处理速度也就是说妙算最快可以跑到150帧，故综合考虑在实际使用中结合滤光片选择了KS1A293兼顾了图像质量与速度。

**更新：复活赛期间我微信问队里同学视觉做的怎么样了，他们告诉我不用黑白摄像头了，黑白的无法过滤白光的干扰，最后还是采用WX605。**
# 镜头
镜头我觉得是最容易被忽视的一点了，因为通常买来摄像头也不会去想到换镜头。不同焦距的镜头所呈现的视角是不一样的，焦距越大视角越窄。

在硬件和金钱条件限制下，可以通过选择适合的镜头来提高图像的成像质量，在实测中战车枪管上使用6mm或8mm左右的镜头比较合适，在这个焦距下3m左右的装甲板拍的很清楚。使用1280*720分辨率8m的能量机关也可以拍得较清楚。

镜头与拍摄视角的关系大致如下图，由于该图针对的是数码相机故数据不适合我们的这个场景，但焦距与视角间的关系还是可以看一看对比一下的。
| ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190608194439771.jpeg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70) |
|--|
|图11 镜头焦距与视角  |

# 滤光片
由于装甲板只有红色和蓝色，所以在识别的时候我们只需要把这两种光过滤出来就可以了，其实不用滤光片也是可以的，通过将图片由BGR转换到HSV识别颜色或者红蓝通道互减都可以达到过滤颜色的目的。

滤光片主要是用来配合和黑白摄像头使用的。之前分析各种摄像头时说道最终选型为KS1A293全局曝光黑白摄像头，使用滤光片就可以很好地结合其240fps的帧率和全局曝光的优势

# Linux摄像头驱动
Linux下的摄像头驱动是V4L2，这没什么好说的。这里我主要要讲一下OpenCV调用摄像头的方法，以及一些摄像头相关的工具。
## 摄像头调试工具
最好用的工具当属qv4l2了，通过这个工具可以很方便地调节摄像头的参数。如果你想在程序运行时调节摄像头曝光，只要打开这个工具，就可以直接修改摄像头的曝光、Gamma等参数，非常好用。
| ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190608200737751.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70) |  
|--|
|图12 qv4l2软件截图 |
可以看到该软件几乎列出了摄像头的所有参数，调节参数非常方便。
```shell
安装方法：sudo apt install qv4l2
```
还有一套命令行工具
```shell
sudo apt install v4l-utils
```
安装后可以使用`v4l2-ctl`来控制摄像头。
## 使用RMVideoCapture类调用摄像头
DJI在2017年开源了一套视觉代码，其中包含了一个很好用的摄像头驱动RMVideoCapture。通过RMVideoCapture，可以绕过OpenCV自带的VideoCapture来调用摄像头，进而实现对摄像头的各种设置包括曝光、分辨率、帧率、图像格式等，具体的调用方法可以参看2018年东南大学开源的代码。
## 使用OpenCV的VideoCapture来调用摄像头
上面说道DJI搞了一个RMVideoCapture，那么是不是说VideoCapture就真的不好用呢？不是的，有大概率是参数没设置对或者编译OpenCV的时候没有选中编译V4L2。

下面来介绍以下怎样正确使用VideoCapture。
首先必须在编译OpenCV的时候选中V4L2（我一般是用CMake GUI来编译OpenCV的）
| ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190608204014365.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70) |
|--|
| 图13 CMake添加V4L2 |
编译过后就可以正常地使用VideoCapture来更改摄像头参数了。
之前OpenCV是不支持更改BufferSize的，但是在2018年3月9号以后添加了对更改BUFFERSIZE的支持
查看OpenCV[相应的代码](https://github.com/opencv/opencv/blob/master/modules/videoio/src/cap_v4l.cpp)可以看到
```c
12th patch: March 9, 2018, Taylor Lanclos <tlanclos@live.com>
 added support for CV_CAP_PROP_BUFFERSIZE

    case cv::CAP_PROP_BUFFERSIZE:
        if (bufferSize == value)
            return true;

        if (value > MAX_V4L_BUFFERS || value < 1) {
            fprintf(stderr, "V4L: Bad buffer size %d, buffer size must be from 1 to %d\n", value, MAX_V4L_BUFFERS);
            return false;
        }
        bufferSize = value;
        return v4l2_reset();
```
也就是说只要用最新的库之前RMVideoCapture所解决的问题都可以用VideoCapture来解决。
为了方便我以python说明下如何正确使用。

```python
#显示OpenCV的编译信息，可以查看有没有包含V4L2支持，如果没有需要重新编译
print(cv2.getBuildInformation())
#打开摄像头
cap = cv2.VideoCapture(1)
#显示缓存数，默认为4最大为10
print(cap.get(cv2.CAP_PROP_BUFFERSIZE))
#设置缓存帧为2
cap.set(cv2.CAP_PROP_BUFFERSIZE,2)
print(cap.get(cv2.CAP_PROP_BUFFERSIZE))
#调节摄像头分辨率
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 800)
cap.set(cv2.CAP_PROP_FRAME_WIDTH, 1280)
#设置图像为MJPG格式，大多数情况下只有这个格式能达到最大帧率
print('setform', cap.set(cv2.CAP_PROP_FOURCC, cv2.VideoWriter.fourcc('M', 'J', 'P', 'G')))
#设置FPS
print('setfps', cap.set(cv2.CAP_PROP_FPS, 60))
#以上两句打印出来的是设置有没有成功，成功的话为True否则是False
gamm = 100
expo = 1
#设置Gamma
cap.set(cv2.CAP_PROP_GAMMA, gamm)
#设置成手动曝光
cap.set(cv2.CAP_PROP_AUTO_EXPOSURE, 1)
#设置曝光
cap.set(cv2.CAP_PROP_EXPOSURE, expo)
#设置成自动曝光
 cap.set(cv2.CAP_PROP_AUTO_EXPOSURE,2.6)
```
所需要设置的参数大概就是上面的这些。这里我要强调以下CAP_PROP_AUTO_EXPOSURE的设置，这个参数的范围在OpenCV源码中写的时0到4，也就是说设置是否自动曝光的参数并不是0或者1。通过实验发现在使用KS1A552摄像头时[0,2.6)范围为关闭自动曝光，[2.6,4]为开启自动曝光。

```c
OpenCV源代码中对参数范围的说明
    if (normalizePropRange) {
        switch(property_id)
        {
        case CAP_PROP_WB_TEMPERATURE:
        case CAP_PROP_AUTO_WB:
        case CAP_PROP_AUTOFOCUS:
            range = Range(0, 1); // do not convert
            break;
        case CAP_PROP_AUTO_EXPOSURE:
            range = Range(0, 4);
        default:
            break;
        }
    }
```

最近申请了一个微信公众号，名字叫**江达小记**。打算将自己的学习研究的经验总结下来帮助他人也方便自己。感兴趣的朋友可以关注一下。

 版权声明：本文为博主原创文章，未经博主允许不得转载。 https://blog.csdn.net/u010750137/article/details/90698203

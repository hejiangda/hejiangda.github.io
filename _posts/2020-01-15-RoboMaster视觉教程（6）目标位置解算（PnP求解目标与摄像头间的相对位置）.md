---
layout: post
title:  "RoboMaster视觉教程（6）目标位置解算（PnP求解目标与摄像头间的相对位置）"
date:   2020-01-15 20:35:43 +0800
categories: RoboMaster
---

## 概览

上篇文章讲到了可以用小孔成像原理得到图像中某点相对于摄像头的转角，可以用这个来对所需要转角的测量。

但是这个方法有很大的局限性，它只能得到**相对于摄像头中心的转角**。而在实际应用中摄像头肯定不会在云台的轴上，每次以相对摄像头中心的转角来指挥云台运动就会有误差。

如果摄像头是放在枪管上的，那其实这点误差没有大碍。因为一般会用闭环控制算法来控制云台跟踪目标，这样如果目标静止不动则稳态时摄像头中心点正对目标，枪管则稍偏下，可以根据实际情况调整摄像头所瞄准的中心位置使枪管对准目标。

如果摄像头是放在车上的，那这种测转角的方法就不能用了。上篇文章的方法最大的缺陷在于它**缺失了深度信息**，从而无法变换坐标系。这里讲另一种求解目标位置的算法，通过求解 **Perspective N-Point (PNP) Problem** 来得到两者间的坐标。

与上文的方法对比这个方法可以得到**三维坐标**，缺点在于需要4个点才能计算而上文算法只要1个点即可，所以目标**距离太远**适合用**单点解算角度**，目标**距离合适**的情况下使用**PnP解算角度**。

## 算法原理

有关算法原理方面的讲解和一些使用案例可以参看以下几篇文章：

「Head Pose Estimation using OpenCV and Dlib」https://www.learnopencv.com/head-pose-estimation-using-opencv-and-dlib/

「Real Time pose estimation of a textured object」 https://docs.opencv.org/3.4.6/dc/d2c/tutorial_real_time_pose.html

「《视觉SLAM十四讲》学习笔记-3D->2D: PnP问题的由来」 https://blog.csdn.net/luohuiwu/article/details/80722542

## solvePnP的使用流程

要用这个算法首先要标定摄像头，得到相机的内参矩阵和畸变参数。之后需要**测量物体的尺寸**，得到物体在世界坐标系中的坐标。

这个坐标是**自定义的**，由于装甲板可以看成是一个平板，其z轴坐标可以设为0，x轴和y轴的坐标分别可以设置为正负二分之一的长和宽，这样装甲板的中心就为原点了。OpenCV中的坐标系可参看下图：
![pinhole_camera_model.png](https://img-blog.csdnimg.cn/2019080412314136.png)
（图片来源：「OpenCV文档」https://docs.opencv.org/3.4.6/d9/d0c/group__calib3d.html）

在检测程序检测到装甲板后就可以得到装甲板在图像中的坐标，之后通过solvePnP函数即可得到平移向量和旋转向量。

旋转向量可以不用管，因为我们不需要得到姿态信息而只需要得到位置信息。求得的平移向量就是以当前摄像头中心为原点时物体原点所在的位置。

因为我们的目的是让枪管瞄准装甲板的中央，而得到的坐标是以摄像头中心为原点的，要得到正确的转角需要对这个坐标进行**平移**，将原点平移到**云台的轴**上。

一般而言x坐标可以不用动，因为摄像头一般放在枪管的上方这样x方向就没有偏差。y方向和z方向需要修正。修正后即可通过反三角函数得到角度值。

## 实验：测量二维码相对于摄像头的位置

为啥用二维码不用装甲板呢，因为不做比赛了没有装甲板可以用。用二维码可以直接使用现成的库完成识别，而不需要自己写识别函数了。

二维码识别我是参看「Barcode and QR code Scanner using ZBar and OpenCV」https://www.learnopencv.com/barcode-and-qr-code-scanner-using-zbar-and-opencv/ 使用ZBar库来做的，如果用较新版的OpenCV也可以使用OpenCV自带的二维码识别函数。

参考的代码：https://github.com/spmallick/learnopencv/tree/master/barcode-QRcodeScanner

我只是在其基础上增加了读取摄像头内参和用solvePnP得到二维码的位置的代码。

其中解算位置的部分只需要以下几行即可：

```c
#define HALF_LENGTH 29.5
//自定义的物体世界坐标，单位为mm
vector<Point3f> obj=vector<Point3f>{
    cv::Point3f(-HALF_LENGTH, -HALF_LENGTH, 0),	//tl
    cv::Point3f(HALF_LENGTH, -HALF_LENGTH, 0),	//tr
    cv::Point3f(HALF_LENGTH, HALF_LENGTH, 0),	//br
    cv::Point3f(-HALF_LENGTH, HALF_LENGTH, 0)	//bl
};
cv::Mat rVec = cv::Mat::zeros(3, 1, CV_64FC1);//init rvec
cv::Mat tVec = cv::Mat::zeros(3, 1, CV_64FC1);//init tvec
//进行位置解算
solvePnP(obj,pnts,cam,dis,rVec,tVec,false,SOLVEPNP_ITERATIVE);
//输出平移向量
cout <<"tvec: "<<tVec<<endl;
```

`HALF_LENGTH` 为我手机上显示的二维码的一半的长度，点坐标的定义是按照从左上角开始顺时针依次定义的，在解算完PnP后得到的平移向量用`cout`输出。

这是我的实验环境，摄像头与手机平面平行（当然肯定还是有点歪的），手机用一本书夹住并且可以在夹槽中移动改变位置。
![img01](https://img-blog.csdnimg.cn/20190804123243802.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)
将手机放在右侧，如下图：
![img03](https://img-blog.csdnimg.cn/20190804123303237.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)
此时在摄像头拍摄的图片中，可以得到二维码中心的坐标为（53.04,43.74,266.77）
![img08](https://img-blog.csdnimg.cn/20190804123326683.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)
将手机放在左侧，如下图：
![img04](https://img-blog.csdnimg.cn/20190804123343222.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)
此时在摄像头拍摄的图片中，可以得到二维码中心的坐标为（-113.49,39.58,266.07）
![img07](https://img-blog.csdnimg.cn/20190804123401905.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)
可以看到二维码的z坐标基本上没有变，而x坐标变化非常大，y坐标按照理想情况不应该变，但是由于摄像头还是有点歪就有了些许变化。

接下来用尺子测量摄像头和二维码间的距离，看看我们测得的摄像头与二维码平面的距离是否正确：
![img05](https://img-blog.csdnimg.cn/20190804123419575.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)
离近一点看
![img06](https://img-blog.csdnimg.cn/20190804123436797.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)
可以看到在26.6cm左右的地方恰好接近摄像头镜头的中心处，说明测得的结果是准确的。

测试代码我放在GitHub上了：https://github.com/hejiangda/pnpTest

## RoboMaster视觉程序中的位置解算

在东南大学的开源代码中，位置解算部分位于Pose文件夹。

在`AngleSolver.cpp`文件中，定义了大小装甲板的世界坐标，这里需要注意的是，通过识别函数得到的装甲板四个点的坐标是**装甲板中间图案**的坐标，也就是装甲板两个光条靠近中间部分的坐标，所以在定义装甲板的世界坐标时应以中间的长宽为准，否则算法不能正常运行，测得的数据都是错的。

因为在摄像头以及预处理参数不同的情况下识别到的装甲板的位置都不尽相同，所以需要根据实际情况来确定装甲板的世界坐标，我的做法是每隔一块地砖（一块地砖长宽为1.15m）查看程序在该距离下测得的装甲板距离，如果在不同距离下测得的装甲板距离都误差不大，则采用该组世界坐标，否则进行调整。

另外`void AngleSolver::setResolution(const cv::Size2i& image_resolution)`这个函数我觉得有问题，看样子是根据图像的分辨率来修改摄像头内参的，它改变了内参中的中心点位置和焦距。按照它那样改肯定有问题，其中心点不是按比例来改的，也就是除非用1920×1080的分辨率否则中心点会偏很多，在实际使用中我把它更改内参的几句话屏蔽了。不同分辨率就再标定一次不就好了，而且反正妙算只跑得动640×480分辨率的图像，那就只标定这个分辨率下的内参就行了啊。

摄像头内参和畸变系数是放在`angle_solver_params.xml`这个文件中的，可以照样子把自己测得的参数加进去。

位置偏移以及重力补偿可以参考`void AngleSolver::compensateOffset()`以及`void AngleSolver::compensateGravity()`

另外在RoboMaster官方开源的RoboRTS的代码中也有重力补偿的部分可以参考

https://github.com/RoboMaster/RoboRTS/blob/ros/roborts_detection/armor_detection/gimbal_control.cpp

## 扩展

单目测距除了用pnp外还可以用相似三角形原理，可以参看「Find distance from camera to object/marker using Python and OpenCV」https://www.pyimagesearch.com/2015/01/19/find-distance-camera-objectmarker-using-python-opencv/ 不过相似三角形只适合被测物体与摄像头平行的情况，局限性比较大。

双目测距的准确性会更好，因为自己水平以及精力有限没有进行研究，有兴趣的可以看大连交通大学的开源代码「**RM2018-DJTU-VisionOpenSource**」https://github.com/Ponkux/RM2018-DJTU-VisionOpenSource

申请了一个自己的公众号**江达小记**，打算将自己的学习研究的经验总结下来帮助他人也方便自己。感兴趣的朋友可以关注一下。

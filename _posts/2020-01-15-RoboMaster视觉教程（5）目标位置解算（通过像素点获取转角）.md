---
layout: post
title:  "RoboMaster视觉教程（5）目标位置解算（通过像素点获取转角）"
date:   2020-01-15 20:35:43 +0800
categories: RoboMaster
---

## 概览

在识别到目标后，有一个很重要的问题：我们的最终目的是瞄准、跟踪、打击，怎样利用识别到目标后得到的目标在图像中的像素坐标来确定在真实世界中目标的位置呢？更清楚点说就是我识别得到的是图像中点的坐标，而我要输出告诉下位机的是它应该旋转或者移动到的目的地。

## 直接使用像素坐标的缺陷

在RoboMaster视觉辅助瞄准中我们需要让枪管一直瞄准装甲板，由于摄像头是装在枪管上的，一种显而易见的方法就是直接利用目标所在像素的坐标与图像中心坐标对比，然后利用一套控制算法移动摄像头使目标在摄像头的中心区域（东北林业大学ARES战队的开源代码是这种方式 GitHub：https://github.com/moxiaochong/NEFU-ARES-Vison-Robomaster2018）。

直接这样做的话PID调节会很麻烦，可以看到东林的代码中PID部分根据像素差的不同使用了不同的PID值。

像素差如果直接喂给PID，那PID只能知道我有偏差了需要照着这个方向转减小偏差，如果偏差大了就给大点偏差小了就少给点。

而发给战车云台的数据是旋转的角度，所以像素差与角度之间的关系需要不断调节PID来控制。并且因为像素差与角度并不是线性关系，导致在图像的不同地方需要不同的PID值。如下图：
![像素点与角度关系](https://img-blog.csdnimg.cn/20190729154810557.gif)


可以看到如果角度在30度以下的时候角度与像素差的关系还可以近似线性，当大于30度后就无法近似了。

那怎样将像素差转化成角度呢？根据上面的图可以看出如果能够得到OF的距离就可以通过反正切函数来得到角度值了，而OF其实就是焦距，当然上图是理论情况，在实际应用中我们需要通过相机标定来得到需要的相机内参。

## 摄像头标定

因为我们需要从图像中得到关于物理世界的信息，所以需要标定摄像头来得到摄像头内参。

摄像头标定的工具有很多，OpenCV自带例子中就有标定工具，也可以用MATLAB来进行标定，标定方法可以参考：https://blog.csdn.net/Loser__Wang/article/details/51811347

## 根据小孔成像原理得到需要的转角

在标定完摄像头后我们会得到摄像头的内参矩阵和畸变参数：
$$
cameraMatrix=\left[
\begin{matrix}
f_x & 0 & c_x\\
0 & f_y & c_y\\
0 & 0 &1
\end{matrix}
\right]
distortionCoefficients=(k_1 \ k_2 \ p_1 \ p_2 \ k_3)
$$
首先根据摄像头内参矩阵和畸变参数矫正x和y的像素值，之后通过计算可以得到转角，原理在《Learning OpenCV》的第11章「摄像头模型与标定」中有。

像素点与物理世界坐标的关系：其中X、Y、Z为物理世界中点的坐标
$$
x_{screen}=f_x(\frac{X}{Z})+c_x\\ \\
y_{screen}=f_y(\frac{Y}{Z})+c_y
$$

我们要想得到转角只要得到X/Z和Y/Z，然后通过反三角函数就可以得到需要的角度了。
$$
\tan\theta_x=\frac{X}{Z}=\frac{x_{screen}-c_x}{f_x}\\
\tan\theta_y=\frac{Y}{Z}=\frac{y_{screen}-c_y}{f_y}\\
 \theta_x=\arctan(\tan\theta_x)\\
 \theta_y=\arctan(\tan\theta_y)
$$
在东南大学的开源代码中的位姿解算部分中的单点解算角度部分就是利用这个原理。

```c
if(angle_solver_algorithm == 0)//One Point
{
    double x1, x2, y1, y2, r2, k1, k2, p1, p2, y_ture;
    x1 = (centerPoint.x - _cam_instant_matrix.at<double>(0, 2)) / _cam_instant_matrix.at<double>(0, 0);
    y1 = (centerPoint.y - _cam_instant_matrix.at<double>(1, 2)) / _cam_instant_matrix.at<double>(1, 1);
    r2 = x1 * x1 + y1 * y1;
    k1 = _params.DISTORTION_COEFF.at<double>(0, 0);
    k2 = _params.DISTORTION_COEFF.at<double>(1, 0);
    p1 = _params.DISTORTION_COEFF.at<double>(2, 0);
    p2 = _params.DISTORTION_COEFF.at<double>(3, 0);
    x2 = x1 * (1 + k1 * r2 + k2 * r2*r2) + 2 * p1*x1*y1 + p2 * (r2 + 2 * x1*x1);
    y2 = y1 * (1 + k1 * r2 + k2 * r2*r2) + 2 * p2*x1*y1 + p1 * (r2 + 2 * y1*y1);
    y_ture = y2 - _params.Y_DISTANCE_BETWEEN_GUN_AND_CAM / 1000;
    _xErr = atan(x2) / 2 / CV_PI * 360;
    _yErr = atan(y_ture) / 2 / CV_PI * 360;
    if(is_shooting_rune) _yErr -= _rune_compensated_angle;

    return ONLY_ANGLES;
}
```

不过这段代码在我测试的时候感觉有问题，它矫正畸变的过程可能有误。

我在实现的时候直接使用OpenCV的矫正函数对点进行矫正，得到的结果是正确的

```c
void calAngle(Mat cam,Mat dis,int x,int y)
{
    double fx=cam.at<double>(0,0);
    double fy=cam.at<double>(1,1);
    double cx=cam.at<double>(0,2);
    double cy=cam.at<double>(1,2);
    Point2f pnt;
    vector<cv::Point2f> in;
    vector<cv::Point2f> out;
    in.push_back(Point2f(x,y));
    //对像素点去畸变
    undistortPoints(in,out,cam,dis,noArray(),cam);
    pnt=out.front();
    //没有去畸变时的比值
    double rx=(x-cx)/fx;
    double ry=(y-cy)/fy;
    //去畸变后的比值
    double rxNew=(pnt.x-cx)/fx;
    double ryNew=(pnt.y-cy)/fy;
    //输出原来点的坐标和去畸变后点的坐标
    cout<< "x: "<<x<<" xNew:"<<pnt.x<<endl;
    cout<< "y: "<<y<<" yNew:"<<pnt.y<<endl;
    //输出未去畸变时测得的角度和去畸变后测得的角度
    cout<< "angx: "<<atan(rx)/CV_PI*180<<" angleNew:"<<atan(rxNew)/CV_PI*180<<endl;
    cout<< "angy: "<<atan(ry)/CV_PI*180<<" angleNew:"<<atan(ryNew)/CV_PI*180<<endl;
}
```

## 角度测量验证

如何知道我们测出的角度是正确的呢？可以通过一把三角尺来进行测量。

将三角尺的30度尖对准摄像头，不断调整位置，使尖对准cx的像素值位置，之后不断调整摄像头，使三角尺的两条边在图像中平行于y轴，此时就可以选取三角尺边上的像素点来测量角度了，如下图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190729154920907.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)
拍摄出的图像如图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190729154952702.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA3NTAxMzc=,size_16,color_FFFFFF,t_70)
这个时候选择左边那条边测得的角度就是30度，选择右边的那条边测得的角度就是0度。

角度测量的代码我放在GitHub上了 https://github.com/hejiangda/angleMeasure

申请了一个自己的公众号**江达小记**，打算将自己的学习研究的经验总结下来帮助他人也方便自己。感兴趣的朋友可以关注一下。

---
layout: post
title:  "RoboMaster视觉教程（7）风车能量机关识别"
date:   2020-01-15 20:37:01 +0800
categories: RoboMaster
---

今年的能量机关在识别的难度上降低了，难在怎么打中。能量机关我只写了识别部分，因为没有道具可以做测试，焊灯条的同学焊的千辛万苦也没焊出可以用的灯条。当然纯手工焊这么大面积的灯条非常困难，前几天打算做一个迷你版的风车能量机关，只焊了一个扇叶就放弃了，太耗时间了。

因为关于风车能量机关的教程已经有很多了，而且我自己写的程序没有经过实战不好说好不好使，这篇就划划水给几个链接吧
**------2019/9/14更新--------**
之前说能量机关的教程有很多了打算不写了，但是总有同学来问，想了想还是写一下吧。
[**「RoboMaster视觉教程（9）风车能量机关识别2」** ](https://blog.csdn.net/u010750137/article/details/100825793)
**---------------------------------**
今年我写的能量机关识别主要是参考**西交RoboMaster机器人队**公众号上的教程：

**「内附代码｜今年的大风车能量机关识别就是这么地so easy！」** https://mp.weixin.qq.com/s/3B-iR32GX7jfVyxvNQVRXw

官方圆桌

**「RM圆桌008 | 如何击打大风车」** https://www.robomaster.com/zh-CN/resource/pages/1015?type=newsSub

**「RM圆桌005 | 抢人头要靠自瞄」** https://www.robomaster.com/zh-CN/resource/pages/1009?type=newsSub

**「RoboRTS/roborts_detection/armor_detection/constraint_set」** https://github.com/RoboMaster/RoboRTS/blob/ros/roborts_detection/armor_detection/constraint_set/constraint_set.cpp

接下来几篇都是**RoboMaster论坛**里的。我在写代码前有个习惯，先到网上搜一搜别人有没有做过，有没有可以借鉴的代码，如果有那就拿来用，不足的地方再自己改。

**RoboMaster论坛** http://bbs.robomaster.com 是一个非常好的资源宝库，没事搜一搜总会有惊喜。

**「110行代码大风车识别卡尔曼预测算法自己写的附效果图」** https://bbs.robomaster.com/thread-9085-1-1.html

**「 RM2019大风车识别算法」** https://bbs.robomaster.com/thread-9092-1-1.html

**「全角度高速稳定装甲板识别卡尔曼预测算法自己写的附图」** https://bbs.robomaster.com/thread-8623-1-1.html

申请了一个自己的公众号**江达小记**，打算将自己的学习研究的经验总结下来帮助他人也方便自己。感兴趣的朋友可以关注一下。

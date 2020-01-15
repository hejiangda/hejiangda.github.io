---
layout: post
title:  "RoboMaster视觉教程（4）装甲板识别算法"
date:   2020-01-15 20:35:08 +0800
categories: RoboMaster
---

## 概览

装甲板识别是RoboMaster视觉识别中比较成熟的了，到现在有很多战队开源了他们的算法。

基本上的思路都是一样的：利用装甲板灯条发光的特性将摄像头曝光值调低屏蔽环境光干扰，二值化处理图像得到只含灯条的二值图，根据装甲板灯条的几何特征来设置约束筛选灯条，灯条匹配筛选装甲板。

**每年都有很多战队将他们的算法开源，善于利用他人的成果可以极大地减少自己工作量。**

我17年参加比赛的时候基本上是闭门造车，一开始不知道官方已经在16年开源了一套非常好的代码。

最开始是按照车牌识别的套路来做的，自己写的代码基本上无法正确地识别装甲板。

后来折腾YOLO，终于在标记了六千多张图片训练了两天后可以识别装甲板了。识别归识别速度太慢了，那时候心情非常失落，又加上带队老师总是催进度，就弃坑学习去了。

等到学期快结束了，发现了大疆的开源代码，下下来读了读，这个真的靠谱，各个方面都想到了，甚至还有通过妙算GPIO控制LED来指示当前程序状态的代码，觉得可以做，就在考完模电后重回实验室折腾了。

换句话说那年我之前做的努力全都作废，识别装甲板、大符、小符的程序是在一个月内参考官方的（官方装甲板识别当装甲板上贴有数字就失效了）赶出来的……

### 下面是一些资料链接，篇篇经典！

RoboMaster论坛中总结的历届开源资料：<https://bbs.robomaster.com/forum.php?mod=viewthread&tid=6979&fromuid=14>

RM圆桌是这届RoboMaster推出的技术分享活动，**全是干货**。<https://www.robomaster.com/zh-CN/resource/news>

RM圆桌005 抢人头要靠自瞄 <https://www.robomaster.com/zh-CN/resource/pages/1009?type=newsSub>

RM圆桌008 如何打击大风车 <https://www.robomaster.com/zh-CN/resource/pages/1015?type=newsSub>

另外一些队伍的官方公众号也会发布一些教程，例如公众号「西交RoboMaster机器人队」里有很多干货，今年大符的识别算法我就是按照他们的教程一步步做的。

「内附代码｜今年的大风车能量机关识别就是这么地so easy！」<https://mp.weixin.qq.com/s/3B-iR32GX7jfVyxvNQVRXw>

昨天看到一句话觉得很好：**一个复杂的系统并不是全部需要从0到1，把优势的资源整合在一起才能发挥最大作用**，用别人的代码或思路并不是可耻的事情（要遵守对方许可证协议），站在巨人的肩膀上才可能走的更远。

今年帮这届做视觉时看到学弟在重新造轮子从零开始写装甲识别，并且也了解到去年写装甲识别的研究生也是从零开始用zed+tx2做的装甲识别最终在赛场上也没发挥作用。

我当时听了很震惊，明明我17年都解决了啊，虽然当时没有做预测没有写好基地电控导致基地自瞄很慢、没有考虑到一些意外情况在场上摄像头歪了导致打大符全打偏了，但是视觉识别的代码为什么要从头开始呢，我还花了一个星期把用到的东西算法整理了一个pdf文档，难道大家都不care前人的经验吗？

## 装甲板识别

由于主要参考东南大学的开源代码，他们的算法思路在readme里写的比较详细<https://github.com/SEU-SuperNova-CVRA/Robomaster2018-SEU-OpenSource/tree/master/Armor>。

我这里讲一下我对算法做的一些改进并将整个流程过一遍。

## test_sentry.cpp

在代码文件夹中的Main中有test_infantry.cpp和test_sentry.cpp，前者是步兵的程序模板，后者是哨兵的程序模板。

默认在项目文件中没有添加test_sentry.cpp，可以右键添加现存文件或者手动输入进项目文件中。这两个文件只能同时只有一个有效（即一个需要在项目文件中注释掉）。

我主要讲test_sentry.cpp，包括之后的教程都是主要用这个文件。因为test_infantry.cpp比较复杂而且其中的大符识别不需要，改起来比较麻烦，而test_sentry.cpp相对简洁，增改代码也比较方便。

在main函数里出现的三个线程中除了produce放在ImgProdCons.cpp中其他的两个都在本文件中。

```cpp
void ImgProdCons::init()//各种参数的初始化
void ImgProdCons::consume()//视觉识别主程序
```

在识别主程序中首先定义要用到的一些变量，定义时间变量t1记录当前时间用于之后的算法耗费时间的度量，之后获取待识别图片，图片可以通过produce线程获得也可以通过视频获得，之后将图片载入到armorDetector中进行识别，识别到后的返回值在ArmorDetector.h中有定义：

```cpp
enum ArmorFlag
{
	ARMOR_NO = 0,		// not found
	ARMOR_LOST = 1,		// lose tracking
	ARMOR_GLOBAL = 2,	// armor found globally
	ARMOR_LOCAL = 3		// armor found locally(in tracking mode)
};
```

若识别到则获取关于装甲类型、装甲板四点的图像坐标来解算装甲相对摄像头的空间坐标，之后将数据发给stm32就可以啦（发送给stm32的是拍下这张图片时以摄像头或敌方装甲为原点的相对坐标值转化成的角度值，依靠这个可以随动跟踪，但永远跟不上（─.─||）。预测比较难做我做的也不好，关于预测以后再说）。

这里可以看出来最核心的识别算法放在了ArmorDetector类中，接下来将详细讲解这部分。

## 分析一下装甲板

在分析代码前先来分析一下装甲板，下面这幅图是装上装甲板后的步兵轴测图（图片来自于官方RM2019裁判系统规范手册）。

![carWithArmor](https://img-blog.csdnimg.cn/20190718115344163.jpg)

在图中可以看到：

1. 装甲板竖直固定在小车的四周
2. 同一装甲板两灯条平行、灯条长宽确定、两灯条间的间距确定
3. 在小车转过45度后会有两个装甲板出现在摄像头画面中
4. 两个装甲板的倾斜角度相差不大，容易将中间两个灯条误识别成大装甲板

据此可以初步构思出识别思路。

1. 找出图片中所有的灯条
2. 根据长宽比、面积大小和凸度来筛选灯条
3. 对找出的灯条进行匹配找到合适的配对
4. 配对灯条作为候选装甲板，提取其中间的图案判断是否是数字进行筛选

以上几步就是识别装甲板的步骤。

## 识别函数 int ArmorDetector::detect()

这个函数是算法的核心，类中的其他的各种数据结构各种成员函数都是为它服务的。

算法第一步是存储灯条对应上面分析的第一步和第二步。

我对原来的代码做了些修改，去掉了颜色识别的部分，用红蓝通道相减得到的差作为识别用的灰度图，这个方法在第一篇教程「摄像头」中提过一下。

对于图像中红色的物体来说，其rgb分量中r的值最大，g和b在理想情况下应该是0，同理蓝色物体的b分量应该最大。

如果识别红色物体可以直接用r通道-b通道。由于在低曝光下只有灯条是有颜色的，两通道相减后，其他区域的部分会因为r和b的值差不多而被减去，而蓝色灯条部分由于r通道比b通道的值小，相减后就归0了，也就是剩下的灰度图只留下了红色灯条。

```cpp
// 把一个3通道图像转换成3个单通道图像
split(_roiImg,channels);//分离色彩通道
//预处理删除己方装甲板颜色
if(_enemy_color==RED)
    _grayImg=channels.at(2)-channels.at(0);//Get red-blue image;
 else _grayImg=channels.at(0)-channels.at(2);//Get blue-red image;
```

得到灰度图后需要阈值化处理得到二值图，之后可以进行膨胀处理让图像中的轮廓更明显。

```cpp
Mat binBrightImg;
//阈值化
threshold(_grayImg, binBrightImg,_param.brightness_threshold
              , 255, cv::THRESH_BINARY);
Mat element = cv::getStructuringElement(cv::MORPH_ELLIPSE, cv::Size(3, 3));
//膨胀
dilate(binBrightImg, binBrightImg, element);
```

找轮廓，这步是整个算法中最耗时的部分，如果预处理做的好，可以极大地减少找轮廓中花费的时间。

```cpp
vector<vector<Point>> lightContours;
//找轮廓
findContours(binBrightImg.clone(), lightContours, CV_RETR_EXTERNAL, CV_CHAIN_APPROX_SIMPLE);
```

找到轮廓后开始遍历轮廓提取灯条
```cpp
for(const auto& contour : lightContours)
{
          //得到面积
	float lightContourArea = contourArea(contour);
	//面积太小的不要
          if(contour.size() <= 5 ||
	   lightContourArea < _param.light_min_area) continue;
	//椭圆拟合区域得到外接矩形
          RotatedRect lightRec = fitEllipse(contour);
          //矫正灯条
          adjustRec(lightRec, ANGLE_TO_UP);
          //宽高比、凸度筛选灯条
          if(lightRec.size.width / lightRec.size.height >
             _param.light_max_ratio ||
	   lightContourArea / lightRec.size.area() <
             _param.light_contour_min_solidity
            )continue;
	//对灯条范围适当扩大
	lightRec.size.width *= _param.light_color_detect_extend_ratio;
	lightRec.size.height *= _param.light_color_detect_extend_ratio;
          Rect lightRect = lightRec.boundingRect();
          const Rect srcBound(Point(0, 0), _roiImg.size());
          lightRect &= srcBound;
          //因为颜色通道相减后己方灯条直接过滤，不需要判断颜色了,可以直接将灯条保存
          lightInfos.push_back(LightDescriptor(lightRec));
      }
//没找到灯条就返回没找到
if(lightInfos.empty())
{
	return _flag = ARMOR_NO;
}
```

对灯条进行匹配筛选

```cpp
      //按灯条中心x从小到大排序
sort(lightInfos.begin(), lightInfos.end(), [](const LightDescriptor& ld1, const LightDescriptor& ld2)
      {//Lambda函数,作为sort的cmp函数
	return ld1.center.x < ld2.center.x;
});
for(size_t i = 0; i < lightInfos.size(); i++)
{//遍历所有灯条进行匹配
	for(size_t j = i + 1; (j < lightInfos.size()); j++)
          {
		const LightDescriptor& leftLight  = lightInfos[i];
		const LightDescriptor& rightLight = lightInfos[j];
		/*
		*	Works for 2-3 meters situation
		*	morphologically similar: // parallel
						 // similar height
		*/
              //角差
              float angleDiff_ = abs(leftLight.angle - rightLight.angle);
              //长度差比率
              float LenDiff_ratio = abs(leftLight.length - rightLight.length) / max(leftLight.length, rightLight.length);
              //筛选
              if(angleDiff_ > _param.light_max_angle_diff_ ||
		   LenDiff_ratio > _param.light_max_height_diff_ratio_)
		{
			continue;
		}

		/*
		*	proper location:  y value of light bar close enough
		*			  ratio of length and width is proper
		*/
              //左右灯条相距距离
		float dis = cvex::distance(leftLight.center, rightLight.center);
              //左右灯条长度的平均值
              float meanLen = (leftLight.length + rightLight.length) / 2;
              //左右灯条中心点y的差值
              float yDiff = abs(leftLight.center.y - rightLight.center.y);
              //y差比率
              float yDiff_ratio = yDiff / meanLen;
              //左右灯条中心点x的差值
              float xDiff = abs(leftLight.center.x - rightLight.center.x);
              //x差比率
              float xDiff_ratio = xDiff / meanLen;
              //相距距离与灯条长度比值
              float ratio = dis / meanLen;
              //筛选
              if(yDiff_ratio > _param.light_max_y_diff_ratio_ ||
		   xDiff_ratio < _param.light_min_x_diff_ratio_ ||
		   ratio > _param.armor_max_aspect_ratio_ ||
		   ratio < _param.armor_min_aspect_ratio_)
		{
			continue;
		}

		// calculate pairs' info
              //按比值来确定大小装甲
		int armorType = ratio > _param.armor_big_armor_ratio ? BIG_ARMOR : SMALL_ARMOR;
		// calculate the rotation score
		float ratiOff = (armorType == BIG_ARMOR) ? max(_param.armor_big_armor_ratio - ratio, float(0)) : max(_param.armor_small_armor_ratio - ratio, float(0));
		float yOff = yDiff / meanLen;
		float rotationScore = -(ratiOff * ratiOff + yOff * yOff);
              //得到匹配的装甲板
              ArmorDescriptor armor(leftLight, rightLight, armorType, channels.at(1), rotationScore, _param);

		_armors.emplace_back(armor);
		break;
	}
}
//没匹配到装甲板则返回没找到
if(_armors.empty())
{
	return _flag = ARMOR_NO;
}
```

对找到的装甲板进行筛选

```cpp
//delete the fake armors
   _armors.erase(remove_if(_armors.begin(), _armors.end(), [this](ArmorDescriptor& i)
   {//lamdba函数判断是不是装甲板，将装甲板中心的图片提取后让识别函数去识别，识别可以用svm或者模板匹配等
       return 0==(i.isArmorPattern(_small_Armor_template,_big_Armor_template,lastEnemy));
}), _armors.end());
//全都判断不是装甲板
if(_armors.empty())
{
	_targetArmor.clear();

	if(_flag == ARMOR_LOCAL)
	{
		//cout << "Tracking lost" << endl;
		return _flag = ARMOR_LOST;
	}
	else
	{
		//cout << "No armor pattern detected." << endl;
		return _flag = ARMOR_NO;
	}
}
```

判断是不是装甲板我用的是模板匹配的方法，模板匹配特别适合待识别图片不会变化的场景，选择合适的模板可以得到很高的准确率并且花费时间远小于svm等机器学习方法。

模板匹配用到的模板下载地址：[数字模板](https://github.com/hejiangda/RM-Archive/raw/master/Template.zip)

```cpp
//模板匹配 根据装甲板中心的图案判断是不是装甲板
bool ArmorDescriptor::isArmorPattern(std::vector<cv::Mat> &small,
                                     std::vector<cv::Mat> &big ,
                                     LastenemyType &lastEnemy)
{
    //若需要判断装甲中间数字
#ifdef IS_ARMOR
    vector<pair<int,double>> score;
    map<int,double> mp;
    Mat regulatedImg=frontImg;

    for(int i=0;i<8;i++){
        //载入模板，模板是在初始化的时候载入ArmorDetector类，
        //因为ArmorDescriptor与其非同类需要间接导入
        Mat tepl=small[i];
        Mat tepl1=big[i];
        //模板匹配得到位置，这里没用
        cv::Point matchLoc;
        //模板匹配得分
        double value;
		//匹配小装甲
        value = TemplateMatch(regulatedImg, tepl, matchLoc, CV_TM_CCOEFF_NORMED);
        mp[i+1]=value;
        score.push_back(make_pair(i+1,value));
		//匹配大装甲
        value = TemplateMatch(regulatedImg, tepl1, matchLoc, CV_TM_CCOEFF_NORMED);
        mp[i+11]=value;
        score.push_back(make_pair(i+11,value));
    }
    //对该装甲与所有模板匹配后的得分进行排序
    sort(score.begin(),score.end(), [](const pair<int,double> &a, const pair<int,double> &b)
    {
        return a.second > b.second;
    });
    //装甲中心位置
    cv::Point2f c=(vertex[0]+vertex[1]+vertex[2]+vertex[3])/4;
	//装甲数字即为得分最高的那个
    int resultNum=score[0].first;
    //得分太低认为没识别到数字
    if(score[0].second<0.6)
    {
        if(//与上次识别到的装甲板位置差不多，且丢失次数不超过一定值
                std::abs(std::abs(lastEnemy.center.x)-std::abs(c.x))<10&&
                std::abs(std::abs(lastEnemy.center.y)-std::abs(c.y))<10&&
                lastEnemy.lostTimes<100
                )
        {//认为该装甲的数字与上次相同
            lastEnemy.lostTimes++;
            lastEnemy.center=c;
            enemy_num=lastEnemy.num;
            return true;
        }
        else
        {//认为不是装甲
            return false;
        }
    }
    //当装甲板识别为小装甲，而得到的号码为11、22……时，说明把大装甲识别为了小装甲。
    if(type==SMALL_ARMOR )
    {
        if(resultNum>10)
        {
            type=BIG_ARMOR;
        }
    }
    enemy_num=resultNum%10;
    lastEnemy.num=enemy_num;
    lastEnemy.center=c;
    lastEnemy.lostTimes=0;
    return true;
#endif
    //若不需要判断匹配的装甲板中的数字则整个函数直接返回true
#ifndef IS_ARMOR
    return true;
#endif
}
```

对装甲板筛选后可能会有多个装甲板，这时候需要选定一个进行跟踪和打击

```cpp
   //找出历史装甲中次数最多装甲号
   int targetNum=0;
   if(!enemy_nums.empty())
   {
       //一共8个装甲号
       int count[9]={0};
       for(auto num:enemy_nums)
          count[num]++;
       // 找出最多出现的次数
       int maxCount=0;
       for(int i = 1; i < 9; i++)  
       {
           if(count[i] > maxCount)
               maxCount = count[i];
       }
       // 找出出现最多次的那个数字
       for(int i = 1; i < 9; i++)            
       {
           if(count[i] == maxCount)
               targetNum = i;
       }
       //保留最近的10个装甲号
       if(enemy_nums.size()>10)
       {
           enemy_nums.erase(enemy_nums.begin());
       }
   }
   //选择装甲板
   bool findFlag=false;
   if(targetNum!=0)
   {
       for(auto & armor : _armors)
       {
           //跟踪之前出现的那辆车的装甲板
           if(armor.enemy_num==targetNum)
           {
               findFlag=true;
               _targetArmor=armor;
               break;
           }
       }
   }
//之前没有跟踪或者之前的车不见了
   if(findFlag==false)
   {
       //calculate the final score
       for(auto & armor : _armors)
       {
           armor.finalScore = armor.sizeScore + armor.distScore + armor.rotationScore;
       }

       //choose the one with highest score, store it on _targetArmor
       std::sort(_armors.begin(), _armors.end(), [](const ArmorDescriptor & a, const ArmorDescriptor & b)
       {
           return a.finalScore > b.finalScore;
       });
       //选择得分最高的目标装甲板
       _targetArmor = _armors[0];
   }
//update the flag status
_trackCnt++;

   enemy_nums.push_back(_targetArmor.enemy_num);
return _flag = ARMOR_LOCAL;
```

装甲识别算法讲完了！觉得不错点个赞呗(\^_^)

申请了一个自己的公众号 **江达小记** ，打算将自己的学习研究的经验总结下来帮助他人也方便自己。感兴趣的朋友可以关注一下。

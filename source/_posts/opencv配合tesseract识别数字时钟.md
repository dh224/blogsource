---
title: opencv配合tesseract识别数字时钟
date: 2021-08-19 12:10:56
tags: [cv,python,opencv]
thumbnail: http://satt.oss-cn-hangzhou.aliyuncs.com/img/ss.png
---



前两天突然被安排了一个任务。不过与其说是任务，倒不如说是测试。大约是因为导师看我在家里太过悠闲，一直在打游戏，于是给了我点事情做做。

不过这个任务实际上很简单。大致目的就是，识别摄像头里的数字时钟。我大约花了一个晚上的时间看了看opencv的基本用法，然后就有大致的思路了。

一开始，我是想做的高大上一点的，也就是通过DL来识别摄像头拍摄到的数字区域，然后再通过opencv对区域内的数字进行处理，最后再用基于DL的算法来识别数字。

但是后来发现，对于导师的要求。我直接用opencv处理了一下图像，然后交给tesseract库来识别（虽然这个库也是用的DL算法来识别）。直接实现了功能，然后录了屏给导师，他居然说不错，自然，我也就懒得继续下去了。封面和下图即为实现的效果。

![ss](http://satt.oss-cn-hangzhou.aliyuncs.com/img/ss.png)

---
title: dcm文件的排序问题
date: 2021-09-24 21:25:59
tags: 
- AR
- DICOM
thumbnail: http://satt.oss-cn-hangzhou.aliyuncs.com/img/aa.png

---

开学后一直很忙。今天无心学习，打了一天的游戏，想着是时候更新一下博客了。

前段时间（中秋），我一直在看[MRTK](https://docs.microsoft.com/zh-cn/windows/mixed-reality/mrtk-unity/?view=mrtkunity-2021-05)的文档，想要用它来实现论文里提到的一些功能。

![Exploring and Slicing Volumetric Medical Data in Augmented Reality
Using a Spatially-Aware Mobile Device](http://satt.oss-cn-hangzhou.aliyuncs.com/img/image-20210924213054565.png)

起初，一些都很顺利。unity的基本操作难不倒我，MRTK的奇怪组件，在不细究的情况下也简单易用。在运行了官方文档里提供的demo后，我信心大增，觉得是时候实干了。

论文的思路其实相当简单，那就是传统的ct的slice是2D的，但是AR模型是3D的，想要将其对应起来实际上是不符合直觉的。但是，对于普通人来说，3D模型或许更加容易理解，但对于经验丰富的医生来说，他们曾经受过的训练已经让他们习惯看这种2D的图形了。因此， 鉴于实用性考虑，论文想到了一个方法，也就是把现实中的ipad当成切板，去切虚幻的AR模型，切面(ipad屏幕)显示的就是对应位置的2D slice。想法很美好。实现起来却很困难。因为论文中实际上并没有提到他是怎么实现的，只是提到了他所用到的一些工具包和困难。

>It consists of an AR HMD (Microsoft HoloLens 2) that is coupled
>with a tablet (Microsoft Surface Pro 6). To track the tablet, we em-
>ployed an OptiTrack motion capturing system and attached infrared
>reflecting markers to both devices. Based on this technical foun-
>dation we implemented our core concepts in the Unity 3D engine,
>utilizing the Mixed Reality Toolkit (MRTK). The tablet shows a
>scan of a liver model, generated from a 3D texture containing the
>volumetric data. The data we used in our prototype was acquired
>from the open-source prototype IMHOTEP1by Michael Pfeiffer.

不过这给了我信息，我先去下载了它提到的[3D数据](http://imhotep-medical.org/downloads#Data)——一个肝脏模型，附带上了一系列dcm文件（也许是ct吧）。将其导入到unity中。看着133张ct图，我不禁陷入沉思。

一段时间的思考后，我决意先实现最简单的功能，即从上到下切模型。毕竟，我看那几个dcm文件就是自上而下的横切面。

实现相对简单——我先去一个名叫作[DICOMGO](https://www.dicomgo.com/#/studylist)的网站打开了那133张dcm文件，并且按照网站上的提供的顺序，选取了包含肝脏的30多张图。记下序号。

![image-20210924214712665](http://satt.oss-cn-hangzhou.aliyuncs.com/img/image-20210924214712665.png)

随后，先后获取肝脏模型在Y轴上的position和切面（由于我没有平板，只能模拟代替了）的position。计算差值，根据当前模型的的高度，计算比例，再用这个比例去提取那133张dcm文件中对应的图。逻辑很简单，但实现却非常奇怪。因为我发现，实现后的ct图并不是按高度播放的。

又过了一小段时间的思考，我认识到，是之前的那个网站（DICOMGO)默默地替我将那些dcm文件按顺序排好了。随后我就去查看fo-dicom(unity上可用的一个插件，用来加载dcm文件)的文档，去寻找对应的tag。显然我高估了我的查文档能力，查了快一个小时，仍然写不出有用的代码。最终还是由于运气好，看到了别人的提问中附带的代码，里面恰好有提取tag内容的代码，因此复制、粘贴，获得了dcm文件的sliceLocation的信息，接下去就很简单了，将我要的文件改个名字，其他的全删掉，其余逻辑和之前一样，并最终实现了功能——仅仅是最基础的功能。

总之，总算是跑起来了。接下去我也懒得做了。看了看傅老师的unity视频，做做游戏不是比做ar好玩多了？

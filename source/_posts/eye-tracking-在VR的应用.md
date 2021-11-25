---
title: eye-tracking 在VR的应用
date: 2021-11-24 20:08:07
tags: [VR,eye-tracking,paper]
thumbnail: https://satt.oss-cn-hangzhou.aliyuncs.com/img/image-20211124211241275.png
---



前段时间被抓去做了一段时间的地图可视化，一番似懂非懂的拼凑，至昨天终于做出了点初步成果，可以把那个项目暂时放一下。

得了空，这次就来记录一下今天刚看的有关Eye-tracking在VR领域的一篇综述。这篇综述的题目如下：

> Prospective on Eye-Tracking-based Studies in Immersive Virtual Reality
>
> Fan Li et al.

该文回顾了2000~2019的大部分沉浸式VR领域中有关Eye-tracking的文章，将它们的主题归纳为两大类：Eye-tracking的方法和Eye-tracking的应用。其中，前者的“科研含量”似乎要高一点，但以我短短两年的研究生生涯而论，似乎没时间做出成绩。因此，我关注的重心是后者，即Eye-tracking的应用部分。

### 论文的选择

该文用以筛查的关键词为：（eye || eye movement || eye-tracking || eye tracking） && （immersive || virtual reality)。这个筛选方法似乎比较好，值得记录。

### 虚拟现实的定义和相关思考

Steuer认为，定义何为虚拟现实的关键是定义好"presence"即存在的概念。presence即存在、在场。这也不奇怪大多数人谈论起虚拟现实的优势，就会说是“沉浸式”。要利用好沉浸感，就要让使用者沉浸其中，即对虚拟环境的沉浸——人不可能同时沉浸于两种环境，自然——用户将缺乏对于显示环境的感知。缺乏的这部分或许可以利用起来。目前，在虚拟现实中，主要提供的是视觉和听觉，触觉则需要昂贵的设备来提供。而视觉和Eye-tracking的关联比较大。

### Eye-tracking提供双向的信息

用户通过视觉对VR提供的虚拟环境进行感知，但同时，用户也向VR环境传输了他的眼动信息。如何收集和利用好这些数据？似乎需要一些思考。

### Eye-tracking需要记录哪些信息？

首先是双眼的注视点，这几乎是必须的。其次就是眼睛的动作。比如：注视、扫视、眨眼、闭眼等信息，都可以用来帮助我们进行分析。

### 基于Eye-tracking的控制

再一次，Eye-tracking提供双向信息。相比于用手柄进行交互，基于Eye-tracking的交互方式应该会更类似手势，操作直观且无心智负担。当然，基于Eye-tracking的交互方式也有很多，该文介绍了几种新的基于眼睛跟踪的交互技术，如双十字线、径向追踪、点头和滚动[32]。论文待看。这种方式还能够提高上下文的阅读水平[12]。论文待看。还可以用来生成路径[32]。论文待看。还可以用以用户移动。Xu等人[36]提出将显著性图和历史扫描路径输入卷积神经网络和长-短期记忆，以预测用户将要看的地方。论文待看。

一个值得注意的小点，如果VR环境阻止眼睛扫视，就能够促进使用者进行无意识的转头操作。

### 基于Eye-tracking的信息收集和评估

又又强调：Eye-tracking提供双向信息。我们要利用好用户的输入。一个好的想法是：对用户进行评估。如[41]借此了解用户的认知状态。然而，眼睛跟踪在沉浸式环境中测量满意度、挫折感和厌倦感的应用很少。或许可以利用上。同时，用来评估的指标也是没有一个广泛使用的标准，往往自行其是。这方面也需要思考。

当然，对用户进行评估，一个比较大的限制就是VR收集到的用户参数是有限的，很多数据实际上并不能获得。因此还是要等待技术的升级。

我认为该文对我最大的帮助就是整理了这张表格。能让我一一对照查看。不过这些都需要一些时间。

![image-20211124211241275](https://satt.oss-cn-hangzhou.aliyuncs.com/img/image-20211124211241275.png)![image-20211124211316566](https://satt.oss-cn-hangzhou.aliyuncs.com/img/image-20211124211316566.png)

基于这些评估参数，或许可以建立一个利用上这些特征的深度学习模型，不过这方面我还是个门外汉，需要时间去学习。



## 总结

总之，目前还需要多看论文才有更好的认识和方向。似乎可行的研究方向是：1.想办法在现有设备下，收集和利用好更多的眼动数据（注视熵、眼跳峰值速度和眼跳持续时间）等。2.提取合适的信息当作特征，用以训练模型。





### 待看paper list

**A. McNamara, K. Boyd, D. Oh, R. Sharpe, and A. Suther,**
**"Using Eye Tracking to Improve Information Retrieval in**
**Virtual Reality," in 2018 IEEE International Symposium on**
**Mixed and Augmented Reality Adjunct (ISMAR-Adjunct),**
**2018, pp. 242-243: IEEE.**

 

 

**A. Haffegee, V. Alexandrov, and R. Barrow, "Eye tracking
and gaze vector calculation within immersive virtual
environments," in Proceedings of the 2007 ACM symposium
on Virtual reality software and technology, 2007, pp. 225-226:
ACM.**

 

 

**S. Stellmach, L. Nacke, and R. Dachselt, "Advanced gaze
visualizations for three-dimensional virtual environments," in
Proceedings of the 2010 symposium on eye-tracking research
& Applications, 2010, pp. 109-112: ACM.**

------

> **T. Piumsomboon, G. Lee, R. W. Lindeman, and M.
> Billinghurst, "Exploring natural eye-gaze-based interaction
> for immersive virtual reality," in 2017 IEEE Symposium on 3D
> User Interfaces (3DUI), 2017, pp. 36-39: IEEE.**

该文介绍了三种基于眼动的交互方式。分别为：1.双十字线DR（Duo-Reticles）2.半径追踪RP（Radial Pursuit ）3.点头与旋转NR（Nod and Roll）。

DR：即有两个十字线，一个是实时同步用户的注视点，另一个则根据下图的公式，进行惯性移动，具有一定的延迟。当双十字线重合时，认为是选中目标。![image-20211125105415489](https://satt.oss-cn-hangzhou.aliyuncs.com/img/image-20211125105415489.png)

RP：该方法的逻辑是：用户注视某个预设区域时，会产生产生一个逐渐扩张的圆形（即半径追踪的Radial），伴随着圆的扩张，半径内的物体会进行小范围的移动。此时，双眼的注视点应跟随要选中的物体。当到达某个预设时间后，根据下图的公式判定用户选择了哪个物体。其中p为物体位置，p'为注视点位置，基本就是选择注视点最近的物体。

![image-20211125105515185](https://satt.oss-cn-hangzhou.aliyuncs.com/img/image-20211125105515185.png)

RP用在很多其他的场合，但在VR中并无应用。RP特别适合用于选择固定场景内的杂乱小物体。

NR：当我们注视某物体的同时转头，眼球会不自觉地朝转头方向相反的方向运动。根据这点，我们可以通过进行交互。如盯着某物体进行转头操作，就可以旋转该物体等。

研究者们设计了几组实验用以对比。其中GD1是凝视选择，即注视一段时间后即为选中。GD2是扩张完成后再选择，无追踪。

其中，双十字线的速度和精度和GD差不多，但是由于没有时间限制，会给使用者较少的心理压力。但同时，要留有惯性十字线的辨识度问题，如果它太过显眼，或许会造成干扰。

对于RP。认为它更加无感（与完全拓展后进行选择相比）。但这种方法需要你提前了解你要选择的物体才可以，不然边动边选择，可能时间不过多。

对于点头和摇头。使用者认为他们很有趣，在一些场合比手势更加直观。但是由于HMD的重量，导致并不是很舒适。并且，在做一些迅猛的运动的时候，有可能导致HMD松动，造成错误。

![image-20211125105757669](https://satt.oss-cn-hangzhou.aliyuncs.com/img/image-20211125105757669.png)

------

**34** 

**S. Vickers, H. Istance, and A. Hyrskykari, "Performing
locomotion tasks in immersive computer games with an
adapted eye-tracking interface," ACM Transactions on
Accessible Computing (TACCESS), vol. 5, no. 1, p. 2, 2013.**

 

**35**

**B. Bolte and M. Lappe, "Subliminal reorientation and
repositioning in immersive virtual environments using
saccadic suppression," IEEE transactions on visualization and
computer graphics, vol. 21, no. 4, pp. 545-552, 2015.**

 

 **36**

**Y. Xuet al., "Gaze prediction in dynamic 360 immersive
videos," in proceedings of the IEEE Conference on Computer
Vision and Pattern Recognition, 2018, pp. 5333-5342**

 **39**

**P. Menezes, J. Francisco, and B. Patrão, "The Importance of
Eye-Tracking Analysis in Immersive Learning-A Low Cost
Solution," in Online Engineering & Internet of Things:
Springer, 2018, pp. 689-697.**

 **40** 

**W. Steptoeet al., "Eye tracking for avatar eye gaze control
during object-focused multiparty interaction in immersive
collaborative virtual environments," in 2009 IEEE Virtual
Reality Conference, 2009, pp. 83-90: IEEE.**

**41**

**J. Jordan and M. Slater, "An analysis of eye scanpath entropy
in a progressively forming virtual environment," Presence:
Teleoperators and Virtual Environments, vol. 18, no. 3, pp.
185-199, 2009.**


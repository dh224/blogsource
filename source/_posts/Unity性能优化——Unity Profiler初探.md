---
title: Unity性能优化——Unity Profiler初探
date: 2023-03-04 21:56:01
tags: [Unity, IK, OMC, 性能优化]
thumbnail: https://ob-1253229442.cos.ap-shanghai.myqcloud.com/ob/202303042214931.png
---
## 起因
这两天OMC大概搭建了个框架，发现在运行时帧数不高，基本卡着90fps运行，常常出现卡顿。为了搞明白这一点，我在哔哩哔哩找了个使用入门Profiler的[教程](https://www.bilibili.com/video/BV1xz4y167rx/?vd_source=068c2a124b2fb99b31ca74c34e2ad41e)。感觉学到了不少以前从未想过的问题，现将学习经过记录下来，以供复习。
> 在找教程的时候发现了一个新工具——内存分析器（Memory Profiler）[内存分析器的介绍，看上去很酷的样子](https://blog.unity.com/cn/engine-platform/everything-you-need-to-know-about-memory-profiler)，不过这个工具在Unity2022及以后的版本中才进入验证(Verified)阶段，对于我这个尚未了解Profiler的人来说过于先进，还是在这里留个记录，后面有时间再学习研究吧。
--------------------
## Profiler介绍
在Unity中使用的Profiler是一种 Instrumentation-based profiler（IBP），与Sample-based profiler（SBP）不同的是，*IBP*在每个函数调用的开始和结束时都会自动插入marker，而不是*SBP*中使用snapshots（快照）技术。这两种方法的区别在于*IBP*能够捕捉所有时刻的变化，但是会引入额外开销。而*SBP* 则可能会错过某些信息.
>不过我不清楚什么是快照技术,似乎之前提到的内存分析器就是使用到了快照.
## 准备工作
总之,让我们使用Profiler对项目进行分析. 果然,我的项目中常常出现奇怪的卡顿,但是发现占用主要时间的居然是editor,这或许是因为我是在editor中的playmode中进行调试的. 我们可以通过构建develop build的方式解决这个问题.
![image.png](https://ob-1253229442.cos.ap-shanghai.myqcloud.com/ob/202303042214931.png)

具体来说,我们需要在Build Setting中勾选 Development Build 以及autoconnect profiler选项,这样我们就可以在外部运行的游戏的同时对其使用profiler进行性能分析了.在打开游戏后,profiler将会自动开始捕获游戏.此时我们可以看到Editor Loop不再出现了.
![image.png](https://ob-1253229442.cos.ap-shanghai.myqcloud.com/ob/202303042218669.png)
![image.png](https://ob-1253229442.cos.ap-shanghai.myqcloud.com/ob/202303042220801.png)
### 开始分析
尽管让游戏运行在我们的目标帧数(例如90fps)是我们性能调优的最终目的,但是对于性能调优本身来说,我们更需要关注的是单帧的预算时间.我们的目的是合理配置,使得每一帧的开销都小于等于预算时间.这种单帧预算时间给了我们一个好的衡量标准.
![image.png](https://ob-1253229442.cos.ap-shanghai.myqcloud.com/ob/202303042223589.png)
作者介绍了什么是Rendering API calls,总之就是我们游戏的运行需要CPU和GPU协同计算,当CPU性能不足时,就会出现等待问题，浪费性能。下两图是两种不同的等待情况：1. GPU等待CPU 2. CPU等待GPU。此外，我们还可以使用`Application.targetFrameRate`对预期帧速率进行设置。
![image.png](https://ob-1253229442.cos.ap-shanghai.myqcloud.com/ob/202303051011652.png)
![image.png](https://ob-1253229442.cos.ap-shanghai.myqcloud.com/ob/202303051011841.png)
这张图显示了VSyunc的工作方式，在极端情况下可能导致帧率减半。
![image.png](https://ob-1253229442.cos.ap-shanghai.myqcloud.com/ob/202303051044797.png)

### 垃圾收集 GC
垃圾回收是导致游戏卡顿的重要原因，然而，在C#中我们没有什么能力对其进行控制。在Profiler中，我们关注分配和回收两个过程，分别为`GC.Alloc`和`GC.Collect`.我们可以使用搜索功能查看其过程，以了解回收机制的运行，尽管我们难以对其进行控制。但是我们可以开启incremental GC（增量垃圾回收）来平滑波峰，减少卡顿。
![image.png](https://ob-1253229442.cos.ap-shanghai.myqcloud.com/ob/202303052214911.png)

### 进行多帧分析
尽管了解某一帧的开销十分重要，但是更重要的是了解一段时间内的情况。因此我们可以使用Profile Analyzer对多个帧进行分析。我们可以在package manager中找到它。在其中，我们选中多个帧进行分析，甚至对两个帧序列进行比较分析。


### Freame Debugger
帧调试器允许我们对渲染过程进行有限度的分析。
…… 待续
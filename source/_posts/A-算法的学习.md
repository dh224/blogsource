---
title: A-star算法的学习
date: 2021-10-07 14:14:30
tags: [算法,Uinty]
thumbnail: http://satt.oss-cn-hangzhou.aliyuncs.com/img/image-20211007141621215.png

---

A\*算法的名字我在其他各种地方听过不少次了，这次国庆总算有功夫稍微学习一下。我看的[教学视频](https://www.youtube.com/watch?v=-L-WgKMFuhE&list=PLFt_AvWsXl0cq5Umv3pMC9SPnKjfp9eGW&index=1)大致讲述了在Unity引擎下实现的A\*算法，同时还有一些优化。但是这个系列视频部分episode没有字幕，看的颇慢。不过也有很多收获，记录一下。更好的是，这系列的教程都有 [源码](https://github.com/SebLague/Pathfinding)可以参考。

A*算是静态路网中求解最短路径最有效的**直接**搜索方法，也是解决许多搜索问题的有效算法。A\*算法是一种启发式探索的算法。什么是启发式探索呢？

> 启发式探索是利用问题拥有的启发信息来引导搜索，达到减少探索范围，降低问题复杂度的目的。

## A*法的逻辑概述

算法是点对点之间的寻路算法，因此，将路网划归成网格是一个很容易理解的手法。

在算法中，每个网格记录三个值。分别是Gcost、Hcost、Fcost。如图所示。Gcost是当前网格与起始网格的距离。Hcost是当前网格到终点网格的距离。Fcost则是Gcost和Hcost的和。

距离可以有欧几里得表示法或者是曼哈顿表示法，这里用符合直觉的欧几里得表示法。即D = sqrt(x1-x2)^2 + (y1-y2)^2)。

![image-20211007143015088](http://satt.oss-cn-hangzhou.aliyuncs.com/img/image-20211007143015088.png)



有了这三个值，我们就可以进行A\*算法的逻辑部分了。A\*算法的伪代码如下图。其实看图就很容易理解了，但是还是用文字来描述一下。

![伪代码](http://satt.oss-cn-hangzhou.aliyuncs.com/img/image-20211007142813462.png)



首先，我们需要准备OPEN和CLOSDE两个集合，用来存储网格。随后，将起始网格存入OPEN里。随后，找到OPEN中拥有最小Fcost的网格，设定为current，随后将current网格移出OPEN集合并添加到CLOSED集合中。如果此时的current就是目标网格，那么算法就此结束。如果不是，则要依次判断current所有的相邻网格。判断他们在经过current后，是否有更小的Fcost，若有，则更新数据，并将其parent网格设置为current。当然，还要讲邻居网格添加到OPEN集合中。

经过以上的逻辑，A\*算法就能够实现功能了，看上去并不困难，接下去就是实现。

## A*算法在Untiy中的实现和表达

### 建立Node类和grid类

首先，我们需要创建Node类和grid类。用来显示网格和存放节点数据。

出于某种我不了解的原因，视频作者使用gizmos的方式来显示网格，因此，那些网格只有在scene场景下才能看到。

总之大概的逻辑就是通过3d的物体来创建网格。如下图所示。红色区域为不可通行区域。搭好了框架，接下去就是实现算法的逻辑了。

![image-20211007155839402](http://satt.oss-cn-hangzhou.aliyuncs.com/img/image-20211007155839402.png)

### 实现算法

为了实现算法。我们创建了FindPath类。随后是如下代码：

```c#
	void FindPath(Vector3 startPos, Vector3 targetPos) {
		Node startNode = grid.NodeFromWorldPoint(startPos);
		Node targetNode = grid.NodeFromWorldPoint(targetPos);

		List<Node> openSet = new List<Node>();
		HashSet<Node> closedSet = new HashSet<Node>();
		openSet.Add(startNode);

		while (openSet.Count > 0) {
			Node node = openSet[0];
			for (int i = 1; i < openSet.Count; i ++) {
				if (openSet[i].fCost < node.fCost || openSet[i].fCost == node.fCost) {
					if (openSet[i].hCost < node.hCost)
						node = openSet[i];
				}
			}

			openSet.Remove(node);
			closedSet.Add(node);

			if (node == targetNode) {
				RetracePath(startNode,targetNode);
				return;
			}

			foreach (Node neighbour in grid.GetNeighbours(node)) {
				if (!neighbour.walkable || closedSet.Contains(neighbour)) {
					continue;
				}

				int newCostToNeighbour = node.gCost + GetDistance(node, neighbour);
				if (newCostToNeighbour < neighbour.gCost || !openSet.Contains(neighbour)) {
					neighbour.gCost = newCostToNeighbour;
					neighbour.hCost = GetDistance(neighbour, targetNode);
					neighbour.parent = node;

					if (!openSet.Contains(neighbour))
						openSet.Add(neighbour);
				}
			}
		}
	}

	void RetracePath(Node startNode, Node endNode) {
		List<Node> path = new List<Node>();
		Node currentNode = endNode;

		while (currentNode != startNode) {
			path.Add(currentNode);
			currentNode = currentNode.parent;
		}
		path.Reverse();

		grid.path = path;

	}

	int GetDistance(Node nodeA, Node nodeB) {
		int dstX = Mathf.Abs(nodeA.gridX - nodeB.gridX);
		int dstY = Mathf.Abs(nodeA.gridY - nodeB.gridY);

		if (dstX > dstY)
			return 14*dstY + 10* (dstX-dstY);
		return 14*dstX + 10 * (dstY-dstX);
	}
```

基本上就是用C#代码实现了之前提到的伪代码的逻辑。经过这样的操作，我们的寻路就完成了。接下去就是优化性能和功能。



待续…………

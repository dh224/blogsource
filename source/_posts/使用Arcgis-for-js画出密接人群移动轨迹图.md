---

title: 使用Arcgis for js画出密接人群移动轨迹图
date: 2021-12-06 21:29:32
tags: [arcgis,vue,web]
thumbnail: https://satt.oss-cn-hangzhou.aliyuncs.com/img/image-20211206213201817.png
---

前段时间，做了个小项目。其中一个主要的需求是要根据真实经纬度的路径来可视化行程，目的是画出密接人群。写完后记录一下（虽然已经过去大半个月了）。

## 构思

考虑到实用性，需要用到真实的地图来显示行程。一开始我是想到使用Arcgis绘制地图。不过做了一部分后，发现用高德地图、百度地图等的API也可以做，对于这种体量的项目来说并不需要用到ArcgisForJS。不过……写都写了。只能硬着头皮写下去。

有了地图，我们可以将需求分为三个小点，分别是：*产生路径*、*画出动画*、*判断密接*。

### 产生路径

首先是产生路径。由于理论上该项目是要展示真实的疫情，但目前没有数据。因此我需要先自行创建一系列*随机*的路径。但又不能过于*随机*，出现小点不沿着路行走的情况。经过一番查找，我发现可以地图服务商（高德地图、百度地图等）提供的路径规划功能。通过WebAPI，发送终点和起点，就会返回对应路径的经纬度信息。不过，对于起点和终点，似乎不好确定。一个简单的想法是手动在地图上打点，记录下不同位置的经纬度，到时候随机选取两个分别作为起点和终点即可。用来模拟应该足够了。

### 画动画

在解决路径后，接下去就是要根据路径画出小点来。通过查找Arcgis for js的文档，发现它确实有提供根据经纬度画点的API，不错！至于动画，似乎可以通过JS的SetInterval，即设置一个定时器来画出动画。在每一帧都更新点的位置即可。不过似乎要考虑*性能*问题。

### 密接判定

最后是密接人群的判定。由于不需要考虑真实科学的传染方式，那么一个简单的思路就是根据距离判断，在定时器上添加一个距离判定，如果在某个半径内，就判定为密接，同时改变小点的颜色以示区分。

思路有了，接下去就是实现了。

## 实现

地图的绘制说来简单，但实际上会有一些问题。而且问题并不是在一开始就出现，这就导致了代码管理的问题。经过这次教训，我觉得git真是个非常好用的工具，哪怕不进行多人协作。

### 地图的绘制

在一开始，我使用Arcgis自带的底图和导航服务，很快就实现了产生路径。然而却发现产生的路径会有很大的偏移，如下图所示。后来发现是中国特色的[火星坐标系](https://zhuanlan.zhihu.com/p/62243160)导致的。实际上我们在页面点击所获取到的经纬度并不是正确的经纬度，而是会有随机的一些偏差。这导致产生的路径也会有所偏移。然而arcgis显然没有为这一套东西做修正，这就导致出现路径偏移的情况出现。

![image-20211210203936757](https://satt.oss-cn-hangzhou.aliyuncs.com/img/image-20211210203936757.png)

然而自己去修正这个偏移几乎是不可能实现的，尝试了几个小时后，还是决定换底图来源。最后经过一番查找，找到了一个使用arcgis调用高德地图、百度地图等作为底图的demo。大约是通过分片请求然后合并起来的，虽然不懂原理，但是能用就行。不错，在上面改改就成了！底图能够显示出来了。接下去就是获取路径了。

### 产生路径的实现

最终选用高德地图作为地图，那么自然，需要用高德地图所提供的WEBAPI来获取路径。不过值得一提的是，在使用高德地图前，我还尝试使用了百度地图进行测试，发现百度地图获取到的路径也有偏移。后来发现，原来百度地图也给经纬度加了一层自己的偏移！太麻烦了，直接选择使用高德地图。而且高德地图的文档也比百度地图的那种几乎等于没写的文档好上不少。就这点小事浪费了我两天。

找到[高德地图WEBAPI](https://lbs.amap.com/api/webservice/guide/api/newroute)的网页，查看如何请求参数。随后在代码里手动拼接请求。

![image-20211210204525131](https://satt.oss-cn-hangzhou.aliyuncs.com/img/image-20211210204525131.png)

在这里使用了Vue所推荐的axios发送异步请求。

在获得返回值后对获得的数据进行处理。

```javascript
        .then((res) => {
          // res.data.result.routes[0].steps.forEach(element => {
          //     paths.push(element.)
          // })
          // console.log(res)
          res.data.route.paths[0].steps.forEach((element) => {
            temppath = element.polyline.split(";");
            var myarr = new Array(); //先声明一维
            for (var i = 0; i < temppath.length; i++) {
              myarr[i] = new Array(); //再声明二维
              for (var j = 0; j < 2; j++) {
                //二维长度为3
                myarr[i] = temppath[i].split(",");
              }
            }
            temppaths.push(myarr);
          });
          console.log(temppaths);

          var templongpath = new Array();
          for (let j = 0; j < temppaths.length; j++) {
            for (let k = 0; k < temppaths[j].length; k++) {
              var tempa = new Array();
              tempa.push(temppaths[j][k][0]);
              tempa.push(temppaths[j][k][1]);
              templongpath.push(tempa);
            }
          }
          this.longPath.push(templongpath);
        })
```

这里将返回值搞成了一个二维数组的形式。每个请求都获取一整条路径，然而这条路径会分为很多个长短不一的小段，一开始，我将他们弄成了一个三维数组的形式。将小段分开存储。但是做到后面动画的时候，发现这样很不好写，需要将路径整合成一条。因此就有了后半段代码，将所有小段整合在一起。不过这也是经过修改后的版本。

在最初开发的时候，我是先初始化，然后再进行的修改操作的。但是这样会有异步请求的问题，即出现数据为空报错。异步请求也卡了我好几天的时间。完全没想到即便在VUE不同的生命周期里执行操作，异步请求仍然不会执行，而是在所有同步操作执行完后才执行异步请求。这就导致出现初始化时，异步的数据还没有获取到的情况，导致了错误。然而那个时候一些代码已经写好了，懒得再改，简单的操作就是设立一个新的数据，这就是longPath这种这么奇怪的命名的由来。原本的变量名称是Path(也不对，应该是复数形式)。不过这样改了很久仍然不成功，最后还是狠下心来将代码改成了现在的样子，这才解决了这个问题。

可以得出两个教训：1.代码在设计之初就要考虑到多种情况，不能选择最简单、最容易实现的。2.如果发现完善的解决方案比较麻烦，而简单的凑合的方法很容易实现，千万不要选择后者，因为到后来就会积重难返，再起不能。应该在问题发现的时候就解决掉他。

总之，我实现了获取路径。接下去就是做出动画效果了。

### 动画的实现

依旧是老样子，找了不少源码，还真找到了一个根据路径显示移动的动画的。虽然那仅仅是显示一个点的动画，不过改变起来似乎不难（实际上还是花了一个下午的时间去做的……）。

就像之前说的，一开始获取到的一条路径是分段存储的，然而动画的时候处理分段很麻烦，最好需要将所有小段的路径整合到一长条路径上去。然后花了几天解决问题，成了。很简单地画出了一个小点地移动轨迹动画。并且可以设置速度和播放速率。 

接下去就是画出多个点了。我对原来的动画代码进行了修改。联想到Unity等游戏引擎的做法。将整个动画设置了一个定时器。并且在定时器里依次执行逻辑。伪代码如下:

```
update(){
	updatepoints()
	updategraphic()
	updateSomethings()
	…………
}

```

写了一个下午就搞定了，而且很让人吃惊，居然没什么Bug，一次就运行成功。

```javascript
   updateAllPoint() {
      for (let i = 0; i < this.longPath.length; i++) {
        if (this.peopleMove[i].end >= this.longPath[i].length) {
          this.peopleGra[i] = new Graphic({
            geometry: {
              type: "point",
              longitude: this.peopleGra[i].geometry.longitude,
              latitude: this.peopleGra[i].geometry.latitude,
            },
            symbol: this.simpleMarkerSymbol[this.peopleMove[i].stat],
          });
          continue;
        }
        var startX = this.longPath[i][this.peopleMove[i].start][0];
        var startY = this.longPath[i][this.peopleMove[i].start][1];
        var endX = this.longPath[i][this.peopleMove[i].end][0];
        var endY = this.longPath[i][this.peopleMove[i].end][1];
        var p = (endX - startX) / (endY - startY);
        var v = 0.0005;
        if (startX == endX && startY == endY) {
          // this.peopleGra[i].geometry.longitude = endX;
          // this.peopleGra[i].geometry.latitude = endY;
          this.peopleGra[i] = new Graphic({
            geometry: {
              type: "point",
              longitude: this.peopleGra[i].geometry.longitude,
              latitude: this.peopleGra[i].geometry.latitude,
            },
            symbol: this.simpleMarkerSymbol[this.peopleMove[i].stat],
          });
          this.peopleMove[i].end++;
          this.peopleMove[i].start++;
          // console.log(
          //   "有两个相同的点，更新位置" +
          //     this.peopleGra[i].geometry.longitude +
          //     "  " +
          //     this.peopleGra[i].geometry.latitude
          // );
        } else {
          var newX, newY;
          if (Math.abs(p) == Number.POSITIVE_INFINITY) {
            endY > startY
              ? (newY = this.peopleGra[i].geometry.latitude + v)
              : (newY = this.peopleGra[i].geometry.latitude - v);
            newX = this.peopleGra[i].geometry.longitude;
          } else {
            if (endX < startX) {
              newX =
                this.peopleGra[i].geometry.longitude - (1 / Math.sqrt(1 + p * p)) * v;
              newY = this.peopleGra[i].geometry.latitude - (p / Math.sqrt(1 + p * p)) * v;
            } else {
              newX =
                this.peopleGra[i].geometry.longitude + (1 / Math.sqrt(1 + p * p)) * v;
              newY = this.peopleGra[i].geometry.latitude + (p / Math.sqrt(1 + p * p)) * v;
            }
          }
          if (
            (this.peopleGra[i].geometry.longitude - endX) * (newX - endX) <= 0 ||
            (this.peopleGra[i].geometry.latitude - endY) * (newY - endY) <= 0
          ) {
            // this.peopleGra[i].geometry.longitude = endX;
            // this.peopleGra[i].geometry.latitude = endY;
            //上面的两行不知为何不起作用，一定要新建graphic才可以
            this.peopleGra[i] = new Graphic({
              geometry: {
                type: "point",
                longitude: endX,
                latitude: endY,
              },
              symbol: this.simpleMarkerSymbol[this.peopleMove[i].stat],
            });
            this.peopleMove[i].end++;
            this.peopleMove[i].start++;
            // console.log(
            //   "到达第二段位置，更新位置" +
            //     this.peopleGra[i].geometry.longitude +
            //     "  " +
            //     this.peopleGra[i].geometry.latitude
            // );
          } else {
            // this.peopleGra[i].geometry.longitude = newX;
            // this.peopleGra[i].geometry.latitude = newY;
            this.peopleGra[i] = new Graphic({
              geometry: {
                type: "point",
                longitude: newX,
                latitude: newY,
              },
              symbol: this.simpleMarkerSymbol[this.peopleMove[i].stat],
            });
            // console.log(
            //   "尚未到达第二段位置，更新位置" +
            //     this.peopleGra[i].geometry.longitude +
            //     "  " +
            //     this.peopleGra[i].geometry.latitude
            // );
          }
        }
      }
    },
```

```javascript
    updateGraphic() {
      this.moving = setInterval(() => {
        this.moveLayer.removeAll();
        this.updateAllPoint();
        this.updateStatus();
        for (let i = 0; i < this.longPath.length; i++) {
          this.moveLayer.add(this.peopleGra[i]);
        }
      }, 100);
    },
```

以上是主要代码。这里还有一个问题，即一开始，我选择使用修改Graphic的经纬度信息来更新点，发现并没有效果，点仍然停留在原地。而如果在每次更新点信息的时候，都重新创建一个点的Graphic。那么就可以正确更新动画位置。这或许是因为Graphic声明的作用域的问题。这样反复创建新变量，一定造成了大量的性能损失，在最后的画面中可以看到明显的卡顿。一个好的想法是使用对象池来管理这些点。只需要在一开始根据申请的路径数量，创建点Graphic，随后只是读取和更新即可，无需再次声明。这是一个可改进的点。

### 密接人群判定的实现

由于不需要考虑科学性，一个最简单的方法就是考虑距离。如果一个传染者在范围内，那么就会变成密接人群。同时，如果是高风险的密接人群，那么他也会具有传染性。我们可以设置代码，将不同等级的密接人群按不同的颜色进行表示，这样就好了！代码实现起来很容易，但是有很多值得修改的地方。

```javascript
    updateStatus() {
      for (let i = 0; i < this.longPath.length; i++) {
        for (let j = 1; j < this.longPath.length; j++) {
          if (this.peopleMove[i].stat >= 3 && this.peopleMove[j].stat < 3) {
            if (i != j) {
              if (
                0.006 >=
                  Math.sqrt(
                    (this.peopleGra[i].geometry.longitude -
                      this.peopleGra[j].geometry.longitude) *
                      (this.peopleGra[i].geometry.longitude -
                        this.peopleGra[j].geometry.longitude) +
                      (this.peopleGra[i].geometry.latitude -
                        this.peopleGra[j].geometry.latitude) *
                        (this.peopleGra[i].geometry.latitude -
                          this.peopleGra[j].geometry.latitude)
                  ) &&
                Math.floor(Math.random() * 30) >= 22 // 用以减慢传染速度
              ) {
                this.peopleMove[j].stat++;
                console.log("增加了传染者" + this.peopleMove[j].stat);
              }
            }
          }
        }
      }
    },
```

```javascript
    initAllPoints() {
      for (let i = 0; i < this.longPath.length; i++) {
        if (i == 0) {
          this.peopleMove.push({
            start: 0,
            end: 1,
            PathLen: this.longPath[i].length,
            stat: 4,
          });
        } else {
          this.peopleMove.push({
            start: 0,
            end: 1,
            PathLen: this.longPath[i].length,
            stat: 0,
          });
        }
      }
      for (let i = 0; i < this.longPath.length; i++) {
        if (i == 0) {
          this.peopleGra.push(
            new Graphic({
              geometry: {
                type: "point",
                longitude: this.longPath[0][0][0],
                latitude: this.longPath[0][1][1],
              },
              symbol: this.simpleMarkerSymbol[4],
            })
          );
        } else {
          this.peopleGra.push(
            new Graphic({
              geometry: {
                type: "point",
                longitude: this.longPath[i][0][0],
                latitude: this.longPath[i][1][1],
              },
              symbol: this.simpleMarkerSymbol[0],
            })
          );
        }
      }
      console.log(this.peopleMove);
    },
```

首先，这里一个问题是，在一次遍历中，会出现一个点被多次累加感染风险的情况。这就导致几乎在一瞬间就变成了最高等级的密接人群。这虽然有一定现实性，但是却失去了可视化的优势。因此在这里我使用随机数，将密接比例设置为26.67%左右。也就是说，一个点即便在感染者的范围内。每一帧也只有26.67%的概率提升密接等级。这个数字还值得进一步调整，但我懒得改了。

其次，这里的运行效率颇为低下。看最后的动图，可以看到卡顿的现象。除了之前提到的对象池，另一部分原因可能是这段逻辑写的不够高效。

在最后，我还花了一点时间做了几个交互用的按钮。

下图就是目前的成果，虽然已经半个月没动了。但是还有不少可以改进的地方。一个是点Graphic的对象池，这是一个非常明确的点，应该放在主要更新方向上。

接下去是有关图表、数据的显示问题。光有动画可不够，还需要显示一些统计数据。不过这个有几个难点，一是怎么做出丰富的数据来，因为这完全是模拟出来的。二是如何设计好的交互方式，让这些数据更好的显示。三是能不能创建更大尺度的，比如跨市跨省之间的密接人群流动。毕竟现在是2021年而不是1202年，密接人群并不是只会用走路的方式进行移动。这些是我暂时想到的未来实现的目标和方向。图表有关的可以用Echarts来做，大范围移动，我也在Echarts上看到了实例。

![传染啊啊](https://satt.oss-cn-hangzhou.aliyuncs.com/img/传染啊啊.gif)





## 后记

这个项目在第二次会议后，方向就发生了大偏转。原因主要是合作的一方进度太慢，这让原本是给他们打下手的我们感到颇为无聊。为了打发时间（交差），我们只能另辟蹊径。

原本这个项目的主要内容应该是根据已有的行程轨迹显示密接人群。只不过由于合作方迟迟没有动作，导致我们没有轨迹。因此，在一开始我就做了一点轨迹的生成，主要是用高德地图来做的。当然考虑到项目的重点是行程的显示，所以生成方面我并没有下什么功夫，只是简单的随机选择初始点和重点，然后发送请求获得数据而已。但是这显然是不合理的，有谁的形成轨迹只会是一条线的？（其实真有。总之，在显示方面，由于没有可靠的成功，导致进度无法推进下去。因此我们转变了项目的方向，将重点放在了模拟轨迹的生成上了。

具体如何做呢？一个好的想法是使用行为模式的概念。

行为模式具体来说，就是将某类人的行为抽象出来，并将其赋予我们随机生成的路径。这样就能够产生一种符合常见行为模式的路径了。

举个例子来说，比如说退休的老年人。他们可能很早就会离开家，去公园或者是什么地方，使用的交通工具是公共交通工具或者是步行。相比于年轻人来说，他们会更加倾向于前往公园等老年人常去的地方，而不是像KTV这类年轻人常去的地方。

这就又引出了第二个问题，那就是路径的时间问题。此前我们只是将已有的数据进行显示，因此，我们并不需要考虑时间。然而既然我们需要自己生成可靠的数据，那么对于轨迹上的每一点上的时间戳就是我们必须要考虑的了。

综合以上的想法，我写了一个详细方案，附在下面。这就是我的实现思路。我前两天也一直在写（1.7~1.11)，总算是基本按照方案实现了。



详细方案

**意义及定义：**

随着全球疫情的反复，越来越多研究者的研究方向转向了疫情防控。然而，由于隐私等问题，相关方向的研究者难以获得真实的行程信息。这使得相关的研究难以正常开展。因此，如果我们能够生成一些拟真的行程数据代替真实的行程数据。将其用以科研，或许可以更好的提高疫情防控的效果和效率。

为此，我们提出了一种基于真实路网的随机人群轨迹生成方法，它能依据不同的行为模式，随机生成带有时间信息的拟真人群行程轨迹序列。同时，我们还提出了一种密接人群传播的可视化模拟方法，能够将生成的移动轨迹或是真实行程信息以动画的方式呈现，并模拟密接人群的传播和扩散。

**方法概述：**

## 带行为模式的轨迹生成方法：

为了模拟人群的轨迹，我们需要以个人的行程为单位，生成轨迹数据，这些轨迹数据需要按照真实的路网进行移动。这些轨迹数据组成了人群轨迹数据信息。高德地图等地图服务商，可以为我们提供基于真实路网的轨迹导航服务。出于对数据真实性的要求，我们还使用行为模式这一概念来让我们的生成更加可信。此外，我们还对轨迹中的每一坐标点都生成了可靠的时间序列信息。在最后，为了让数据更加丰富可信，我们模拟产生了每一条轨迹所代表的个人的基本属性。

## 轨迹数据的可视化方法：

   为了改进密接人群筛查的方法，提高密接人群研判的效率。我们提出了一种可视化方法，它能够将轨迹序列以动画的形式呈现。为了提高动画的流畅度，我们使用了插值的手段加密轨迹路径。由于我们使用的是真实的经纬度信息，因此，如果我们拥有真正的行程数据，那么也能对其进行动画呈现。

## 密接人群模拟方法：

在可视化方法中，我们还需要在动画过程中对密接人群的扩张进行模拟。利用生成方法所产生各类数据，结合距离模拟出可靠的密接人群传播效果。

**算法详述：**

## 带行为模式的轨迹生成方法：

在本方法中，我们主要使用高德地图提供的路径规划服务来获得轨迹点。我们需要给出起点和目的地等信息，就能够获得多条符合真实路网的移动轨迹数据。只要我们依次生成各段轨迹路径的起点和终点，依次发送请求，并将返回的数据拼接在一起，就能够获得最终的轨迹数据。因此生成方法的重点在于如何合理地产生起点和终点，我们为此引入了行为模式的概念。

行为模式可以影响轨迹所代表的个人的偏好。具体来说，行为模式能够影响个人对于目的地类型的偏好。比如，对于“工人”这一行为模式，他们在早上会倾向于前往工作地点进行工作，他们会一直停留在工作场所直至工作结束，下班后则倾向于前往娱乐场所进行活动。行为模式不仅仅会影响个人对于目的地的选择，它还会影响个人对于交通工具的选择。比如说，成年人在远距离移动的情况下会倾向于自驾进行移动。同样的，行为模式还会影响用户进行移动的动力，即他们是更偏爱呆在室内，还是闲逛。

为了生成更加可靠的时间数据，在我们的方法中，我们根据轨迹使用的交通方式（亦由行为模式决定），设置不同的平均移动速度。比如，步行的速度为每秒1.25米。这样，我们就可以先根据轨迹中相邻两点的经纬度计算出距离，随后依据速度值依次计算出各点的时间戳。值得一提的是，不同纬度的经纬度值代表的现实距离并不相同，需要额外的计算计算才能获得相对准确的距离。

   为了让数据更加丰富和可信，我们还生成了每一条轨迹信息所代表的个人的基本属性，如年龄、性别、是否接种疫苗、是否戴口罩等。这些属性在后面的疫情模拟下会有更多作用。

![路径生成](https://satt.oss-cn-hangzhou.aliyuncs.com/img/路径生成.png)                               

图 1 路径生成算法

## 轨迹数据的可视化方法：

   对于可视化方法，该方法使用ArcGis For JS来绘制地图和代表轨迹的图形。得益于该库提供的方法，我们可以在动画执行的同时对地图进行操作。我们使用此前介绍的生成方法所产生的行程轨迹数据，快速地依次刷新轨迹中各点的位置信息，从而实现动画的效果。由于此方法使用的是真实的经纬度信息，因此，如果我们拥有真实的行程数据，那么也能够对其进行动画呈现。该方法大致可以分为两个部分：位置的更新及动画的呈现。

  首先是位置的更新。由于行程轨迹序列的间隔不定，为了避免动画出现跳跃、闪烁等情况，我们采用了插值的方法细化轨迹点。具体的，我们在更新数据时需要计算两点间距，即根据当前位置和轨迹中下一点位置的距离。随后判断是否需要在两点间插入一点。通过这种插值方法，我们能够让位置坐标进行小步移动式的更新，提高了画面的流畅性。

 ![image-20220115225016960](https://satt.oss-cn-hangzhou.aliyuncs.com/img/image-20220115225016960.png)

图 2 路径插值算法

   然后是动画的显示。为了让动画更加流畅，我们的地图动画以每秒25帧的速度更新。对于轨迹所代表的个人的行为，我们使用不同的外框颜色表示。比如，角色状态是工作中，那么使用蓝色外框表示；状态是移动中，则使用白色外框表示。为了直观显示出不同的感染指数，我们也使用不同颜色的小点表示。比如，正常者用灰色小点表示，感染者使用红色小点表示。对于不同的交通工具，我们使用一个保存停留时间的数组来进行计算。即当某点的停留时间不为零时，该点不进行移动。如此就能实现点的变速移动。为了能够显示点的实时状态，我们还使用了一个数组来存储其在某点的状态信息。

## 密接人群模拟方法：

为了让可视化方法更加真实，我们设计了一种基于距离的密接人群模拟方法。我们使用距离来作为是否为密接人群的判断依据。即当两点的距离小于设定的阈值的时候，我们就会认为该点具有成为密接人员的可能性。为了让模拟的数据更加可信，我们还用到了各类数据，根据之前提到的生成方法，我们可以获得代表该轨迹的个人的属性信息。如是否戴口罩、是否接种疫苗、当前身体状况等。这些信息都将会影响密接人群的扩散和模拟。我们根据各类数据对于疫情传播的影响程度，设置了不同的权值。比如，接种了疫苗的模拟人会有更小的概率成为感染者。交通工具也是判断的依据之一，因为即便感染者是在自己驾驶的车辆内靠近他人，那么也很难会影响到车外步行的人。同时，场所的属性也会影响密接人群的传播。比如，在饭馆遭遇感染者被判定密接的概率会大于在公园等开放性场所。总之，我们使用到了各类数据，尽可能地模拟出可信的疫情传播。

 ![生成路径](https://satt.oss-cn-hangzhou.aliyuncs.com/img/生成路径.png)

图 3 密接人群模拟算法



以上就是详细方案，具体的实现大部分已经完成。接下去可以考虑的问题有以下三点

1.考虑修改行为模式，使得其随机性更强。

2.考虑自动获取带有不同tag的场所坐标，而非手点，这样能够增加实用性。

3.修改生成时间，使其更加符合现实。

以上三点是我目前想到的可以改进的地方，我会在未来几天内着手改进。

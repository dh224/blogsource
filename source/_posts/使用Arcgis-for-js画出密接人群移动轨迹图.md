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

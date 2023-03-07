---
title: Lerp的误解和正确使用
date: 2023-03-03 12:43:46
tags: [Unity, IK, OMC]
thumbnail: https://ob-1253229442.cos.ap-shanghai.myqcloud.com/ob/202303071048386.png
---

### Lerp的误解和正确使用
Unity中由于`Lerp`方法用于在两个值之间进行线性插值，但是常见的用法往往是错误的。比如该例[使用Lerp旋转的用法](https://gwb.tencent.com/community/detail/127454)：
``` C#
using UnityEngine;
using System.Collections;
public class ExampleClass : MonoBehaviour ｛
    public Transform from;
    public Transform to;
    public float speed = 0.1F;
    void Update() ｛
       transform.rotation = Quaternion.Lerp(from.rotation, to.rotation, Time.time * speed);
    ｝
｝
```
实际上，作者也指出该方法存在以下三个问题，
1.  这样的旋转不是匀速的旋转，这种是逐渐减速的旋转。
2.  永远无法旋转到目标角度，可以无限的接近。
3.  旋转速度与帧数相关，请注意from.rotation在变化，举例来说单位时间内移动2次2%和移动1次4%并不相同。

想要正确使用Lerp(线性插值），我们需要预先保存起始阶段和目标阶段的角度、位置等，取决于我们如何使用Lerp。
``` C#
private Vector3 endPosition = new Vector3(1f,1f,1f);
private Vector3 startPosition;
private float desireDuration = 3f;
private float elapsedTime;
void Start(){
	startPosition = transform.position; // 需要预先存储起始位置
}
void Update(){
	elapsedTime += Time.deltaTime;
	float percentageComplete = elapsedTime / desireDuration;
	transfrom.position = Vector3.Lerp(startPosition, endPosition, percentageComplete);
}
```
使用以上代码就能够正确使用Lerp进行线性插值了。可见其要点在于不要在插值过程中输入改变后的起始位置。许多人使用实时更新的起始位置，目的是为了实现逐渐减速的移动过程，尽管肉眼上是可行的，但会面临前例出现的**三个问题**，即不是匀速转动——这是可以我们的目的，永远无法到达目标位置/角度——这是一个隐藏的问题，往往需要进行额外的判定，比如，当位置与目标位置的差值小于某阈值后停止插值等，实际上让代码变得更加复杂了。旋转速度与帧数相关——同样是个问题，特别是面临高性能机器的时候。
实际上，我们可以用以下的方法实现逐渐减速的移动、旋转效果:
``` C#
transfrom.position = Vector3.Lerp(startPosition, endPosition, Mathf.SmoothStep(0, 1, percentageComplete));
```
在这里，我们使用了Mathf.SmoothStep()方法来让平滑我们的percentageComplete变量，我们可以选用任何我们想要的方法，比如`Math.pow(percentage, 0.5)`。通过这种方法实现的旋转和平移不受帧率的影响，并且更加可靠。

#### 利用动画曲线实现
为了实现更加可控的曲线，我们可以使用动画曲线。
``` C#
[SerializeField] private AnimationCurve curve; // 我们只需要在编辑器中设定该值即可。
```
同时将Lerp的输入修改为：
``` C#
transfrom.position = Vector3.Lerp(startPosition, endPosition, curve.Evaluate(percentageComplete));
```
即可实现功能。
#Unit #Lerp 
### 参考
https://www.youtube.com/watch?v=MyVY-y_jK1I
[blueraja.com/blog/404/how-to-use-unity-3ds-linear-interpolation-vector3-lerp-correctly](https://www.blueraja.com/blog/404/how-to-use-unity-3ds-linear-interpolation-vector3-lerp-correctly)
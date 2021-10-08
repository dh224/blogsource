---
title: 看Object Management有感
date: 2021-10-08 13:57:14
tags: [Unity]
thumbnail: http://satt.oss-cn-hangzhou.aliyuncs.com/img/image-20211008182646614.png
---



看了[Catlike coding的教程](https://catlikecoding.com/unity/tutorials/object-management/persisting-objects/)，感觉有所收获，在这里记录一下.

## 对象的持久化



对于一个预制件，我们可以使用Instantiate()来实例化。如

```C#
	void Update () {
		if (Input.GetKeyDown(createKey)) {
			Instantiate(prefab);
		}
	}
```

即当按下createKey所记录的键后，创建一个prefab。如果再用随机数修改这些prefab的localposition，那么我们就可以创建一系列随机的Cube。那么如何保存这些随机生成的？这就引出了本篇教程的主题，即对象的持久化。

### 文件的路径、打开、写入和读取

要想要保存这些游戏物体，最符合直觉的方式是将其数据保存在一个文件中，当我们需要加载这些预先保存的物体的时候，我们就读取这个文件，获得数据，随后生成物体。

所以，第一步就是要设定保存数据的文件。

```c#
using System.Collections.Generic;
using System.IO;
using UnityEngine;

public class Game : MonoBehaviour {

	…

	void Awake () {
		objects = new List<Transform>();
		savePath = Path.Combine(Application.persistentDataPath, "saveFile");
	}

	…
}
```

我们可以用这种方式来设置用来保存文件的路径。由于C#的原因，我们在写入数据之前还需要对文件进行“OPEN”操作。我们可以用二进制来写数据。因此如下代码可用：

```c#
	void Save () {
		BinaryWriter writer =
			new BinaryWriter(File.Open(savePath, FileMode.Create));
	}
```

同样的，由于c#的原因，我们还需要在文件打开和关闭之间进行异常判断，这在java里非常复杂，而C#为我们提供了好用的语法糖。即

```c#
	void Save () {
		using (
			var writer = new BinaryWriter(File.Open(savePath, FileMode.Create))
		) 
        {
           //todo
            writer.Write(objects.Count);
        }
	}
```

我们可以在大括号中对writer进行操作。不用再写麻烦的try……catch。

……终于，在打开文件后，我们*终于*可以写数据了！接下去的代码平平无奇。

```c#
			writer.Write(objects.Count);
			for (int i = 0; i < objects.Count; i++) {
				Transform t = objects[i];
				writer.Write(t.localPosition.x);
				writer.Write(t.localPosition.y);
				writer.Write(t.localPosition.z);
			}
```

通过以上的代码，我们写入了Cube的位置，但是没有记录他们的缩放和旋转，因此，读取后他们的角度是固定的。我们会在之后进行修改。

![image-20211008155824435](http://satt.oss-cn-hangzhou.aliyuncs.com/img/image-20211008155824435.png)

接下去就是读取数据了。不过在记录读取数据之前，我们要明白，我们为啥不用Unity提供的BinaryFormatter来存储文件。教程的作者给出的答案是：我们这样做更加灵活和可理解。

读取数据的代码和存数据的代码类似。不赘。

总之，根据教程，我们实现了以下功能：

按C创建一个随机位置的方块，按N消除所有方块。按S将所有方块的位置信息保存在一个二进制文件中。按L读取文件，并重新创建方块。

接下去就是对于代码逻辑的优化。

### 代码的抽象

尽管我们实现了上述的功能，但是却不符合软件工程的高内聚低耦合的原则。我们不想在读取和加载的时候，都要多次调用write和read方法，我们也不想固定用二进制存储我们的数据。因此，我们需要抽象出Writer和Reader类，来帮助我们实现分离。

```c#

public class GameDataWriter  
{
    BinaryWriter writer;
    public GameDataWriter(BinaryWriter writer)
    {
        this.writer = writer;
    }
    public void Write(float value)
    {
        writer.Write(value);
    }
    public void Write(Quaternion value)
    {
        writer.Write(value.x);
        writer.Write(value.y);
        writer.Write(value.z);
        writer.Write(value.w);
    }
    public void Write(Vector3 value)
    {
        writer.Write(value.x);
        writer.Write(value.y);
        writer.Write(value.z);
    }
}

```

由于writer和reader的代码过于雷同，这里只贴writer的部分。总之，我们将一些低级操作封装了起来。

接下去，我们要让我们的调用者来保存数据，而不是引入writer来做这件事。因此，我们还需要创建PersistableObjet类。

```c#
using UnityEngine;

public class PersistableObject : MonoBehaviour {

	public void Save (GameDataWriter writer) {
		writer.Write(transform.localPosition);
		writer.Write(transform.localRotation);
		writer.Write(transform.localScale);
	}

	public void Load (GameDataReader reader) {
		transform.localPosition = reader.ReadVector3();
		transform.localRotation = reader.ReadQuaternion();
		transform.localScale = reader.ReadVector3();
	}
}
```

创建这个类的目的是:对于需要持久化的物体，我们让这个物体继承PersistableObject类，就将这个（类）物体设置成持久化对象类型。它（们）获得了一些方便地接口。这样就可以方便（具体）地设置这类持久化物体需要保存的参数了。

有了PersistableObject类还不够，我们还需要创建PersistentStorage类来保存这类对象。理论上，对于不同类型的PersisitableObject类（如有必要），我们都需创建对应的PersistentStorage来操作这类对象。

```
using System.IO;
using UnityEngine;

public class PersistentStorage : MonoBehaviour {

	string savePath;

	void Awake () {
		savePath = Path.Combine(Application.persistentDataPath, "saveFile");
	}

	public void Save (PersistableObject o) {
		using (
			var writer = new BinaryWriter(File.Open(savePath, FileMode.Create))
		) {
			o.Save(new GameDataWriter(writer));
		}
	}

	public void Load (PersistableObject o) {
		using (
			var reader = new BinaryReader(File.Open(savePath, FileMode.Open))
		) {
			o.Load(new GameDataReader(reader));
		}
	}
}
```

再之后，我们重写一部分我们写好的代码。

![image-20211008161858119](http://satt.oss-cn-hangzhou.aliyuncs.com/img/image-20211008161858119.png)

为了多次保存和读取，我们还需要让我们的脚本自身继承persistableObject类。为了方便操作，我们引入了stroage。

```c#
public class Game : PersistableObject {

	…

	public PersistentStorage storage;

	…

	void Update () {
		if (Input.GetKeyDown(createKey)) {
			CreateObject();
		}
		else if (Input.GetKeyDown(saveKey)) {
			storage.Save(this);
		}
		else if (Input.GetKeyDown(loadKey)) {
			BeginNewGame();
			storage.Load(this);
		}
	}

	…
}
```

目前，如果我们按S保存，我们保存下来的是我们的Game脚本，这是没用的，我们还需要在Game脚本里重写load和save方法，即保存每一个创建出来的方块。

```c#
	public void Save (GameDataWriter writer) {
		writer.Write(objects.Count);
		for (int i = 0; i < objects.Count; i++) {
			objects[i].Save(writer);
		}
	}
	public override void Load (GameDataReader reader) {
		int count = reader.ReadInt();
		for (int i = 0; i < count; i++) {
			PersistableObject o = Instantiate(prefab);
			o.Load(reader);
			objects.Add(o);
		}
	}
```

如此，我们完成了对于之前功能的抽象。

## 物体的种类

### 形状工厂

为了能让我们生成不同形状的物体，我们需要创建一个继承了PersistableObject类的Shape类。值得注意的是，由于我们之前的设置，persistableObject类和Shape类不能共存。

接下去，我们继续创建ShapeFactiory类。对于这个工厂，他的作用是交付形状实例。我们不需要让他设置位置、旋转和缩放。也不需要改变状态。因此它可以作为一个asset存在。实现的方法是继承ScriptableObject类.

```
[CreateAssetMenu]
public class ShapeFactory : ScriptableObject {}
```

使用[CreateAssetMenu]标签可以让它出现在create里。

![image-20211008204206185](http://satt.oss-cn-hangzhou.aliyuncs.com/img/image-20211008204206185.png)

在ShapeFactory类里，我们创建预制件。并且实现其交付功能

```c#
	public Shape Get (int shapeId) {
		return Instantiate(prefabs[shapeId]);
	}
		public Shape GetRandom () {
		return Get(Random.Range(0, prefabs.Length));
	}
```

此时有个值得注意的点。产生随机形状的时候，代码中居然写的是0~prefabs.Length。我们都知道，实际上应该是0~prefabs.Length-1才对。这只是因为unity为他特意做了设置而已。为了能让ShapeFactory实现功能，我们还需要修改原来的一些代码，不赘。

### 保存形状和材质

之前我们已经能够做到保存生成的cube。既然已经实现了生成不同形状的物体，那么势必要修改一下我们的存取了。

显然，为了知道当前物体的形状，我们必须要在Shape类里添加一个shapeId属性。理论上，这个属性应该是readonly的。但是，考虑到我们实现了形状工厂以及其他的抽象，我们必然要在其他地方设置物体的形状，因此我们需要添加get和set方法。并且，可以添加一条简单的if语句来判断形状是否正确分配了。

```c#
	public int ShapeId {
		get {
			return shapeId;
		}
		set {
			if (shapeId == 0) {
				shapeId = value;
			}
			else {
				Debug.LogError("Not allowed to change shapeId.");
			}
		}
	}
```

此处又是一个很细的点。由于默认值一开始设置的是0，而0有可能代表false。为了避免这种情况，我们将默认值设置为int的最小值。

![image-20211008205613493](http://satt.oss-cn-hangzhou.aliyuncs.com/img/image-20211008205613493.png)

由于添加了形状属性。所以，我们保存的文件也需要增加这一部分。这与之前设置的存档起了冲突。

一个简单的想法是：添加version属性，用来判断不同版本的存档，便于我们读取。即，新版本支持读取旧版本的存档。

```c#
	public override void Save (GameDataWriter writer) {
		writer.Write(saveVersion);
		writer.Write(shapes.Count);
		…
	}
	
	public override void Load (GameDataReader reader) {
		int version = reader.ReadInt();
		int count = reader.ReadInt();
		…
	}
```

此处，教程用了一种相当巧妙的方法。由于原先存档的第一个int是物体的数量，那么必然>=0。因此，我们在保存version的时候，可以将其反转正负号。当我们读取第一个数字后，若是正数，那么我们就能立刻判断出这是个老版本的存档。此处还添加了一个版本判断报错。

```c#
int version = reader.Version;
if (version > saveVersion)
{
	Debug.LogError("Unsupported future save version " + version);
     return;
}
int count = version <= 0 ? -version : reader.ReadInt();
```

当然，接下去还有一系列逻辑代码的修改，在教程里写的很清楚。

除了可以改变创建物体的形状，还可以修改创建物体的材质。不过方法与之类似。不赘述。

### 随机颜色

为了创建随机颜色的物体。我们需要在Shape里创建新的字段，并增加set方法。

```
	Color color;

	public void SetColor (Color color) {
		this.color = color;
		GetComponent<MeshRenderer>().material.color = color;
	}
```

并且在之前的抽象类,GameDataWriter\reader里增加一个方法：

```
public void Write (Color value) {
	writer.Write(value.r);
	writer.Write(value.g);
	writer.Write(value.b);
	writer.Write(value.a);
}
	public Color ReadColor () {
		Color value;
		value.r = reader.ReadSingle();
		value.g = reader.ReadSingle();
		value.b = reader.ReadSingle();
		value.a = reader.ReadSingle();
		return value;
	}
```

为了增加向后兼容的能力，即若打开旧版本存档，我们就要跳过存储颜色的部分。这部分教程里写的很清楚。


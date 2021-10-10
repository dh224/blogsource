---
title: 看Object Management有感
date: 2021-10-08 13:57:14
tags: Uinty
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

## 重用对象——对象池

### 摧毁物体及优化

有了创造物体的能力后，我们就得要有摧毁物体的能力，不然就有点误区了。要实现摧毁物体。我们首先需要写一个销毁物体的方法。逻辑很简单：

```c#
	void DestroyShape () {
		if (shapes.Count > 0) {
			int index = Random.Range(0, shapes.Count);
			Destroy(shapes[index].gameObject);
		}
	}
```

此处值得注意的是，shapes存储的是Shape组件，并不是一个游戏物体，因此要在后面加上Destroy(shapes[index].*gameObject*)。然后，如同之前创建物体那样，给摧毁物体也绑定一个按键，这样就实现了摧毁随即物体的功能。但是，如果我们多次按下摧毁按键，有时候反而会报错。仔细看代码就会发现：

我们删除的凭据是shapes中的随机物体。但是，在我们Destroy那个物体后，并没有将其移出shapes的List。这样就有可能在某次删除时，选择了某个早已被删除的物体，这样就会报错。因此，我们还需要在删除物体之后，将其编号移出shapes。

```c#
	void DestroyShape () {
		if (shapes.Count > 0) {
			int index = Random.Range(0, shapes.Count);
			Destroy(shapes[index].gameObject);
			shapes.RemoveAt(index);
		}
	}
```

如此，我们实现了删除物体的功能。但是，这样还不够，我们需要对其进行优化。有什么地方可以被优化呢？还是从shapes入手。

我们注意到，shapes是一个List。而List是由数组实现的，因此不能实现链表的操作。即：

![image-20211009125451883](http://satt.oss-cn-hangzhou.aliyuncs.com/img/image-20211009125451883.png)

与图中的高效方法相比，我们的list反而会用逐次移动的方式来处理中间项的删除。即：

![image-20211009125547679](http://satt.oss-cn-hangzhou.aliyuncs.com/img/image-20211009125547679.png)

这是非常低效的。因为我们目前实际上并不关心数组中物体的序号，维护这个顺序毫无意义。因此，为了加快处理速度，教程中给出了如下方法：

![image-20211009125659733](http://satt.oss-cn-hangzhou.aliyuncs.com/img/image-20211009125659733.png)

即确定是中间项要被删除后，就把最后一个和被删除项交换。这样避免了逐次移动的操作。

```c#
	void DestroyShape () {
		if (shapes.Count > 0) {
			int index = Random.Range(0, shapes.Count);
			Destroy(shapes[index].gameObject);
			int lastIndex = shapes.Count - 1;
			shapes[index] = shapes[lastIndex];
			shapes.RemoveAt(lastIndex);
		}
	}
```

这种做法，如果是我一定想不到，还需要多多思考才能发现。

### 自动化生成和摧毁

我们可以通过添加滑块的方式来删除和产生物体。这就是自动化！这给我们后续进行优化提供了方便。

![image-20211009130025373](http://satt.oss-cn-hangzhou.aliyuncs.com/img/image-20211009130025373.png)

这一段代码相对简单，没啥可说的。唯一要注意的是

```c#
// false
creationProgress += Time.deltaTime * CreationSpeed;
		if (creationProgress >= 1f) {
			creationProgress -= 1f;
			CreateShape();
		}	
//true
creationProgress += Time.deltaTime * CreationSpeed;
		while (creationProgress >= 1f) {
			creationProgress -= 1f;
			CreateShape();
		}
```

我们需要用while来代替if。因为有时候变化过于巨大，以至于-1后仍然大于1.

### 内存池

在应用内存池后，可以让内存的调用不再那么频繁。因为内存池减少了那一瞬间的大量的垃圾回收机制的运行，因为物体实际上并没有被删除，而是被隐藏了起来，不会触发垃圾回收。如下图的对比。

![image-20211009210447274](http://satt.oss-cn-hangzhou.aliyuncs.com/img/image-20211009210447274.png)

![image-20211009210520689](http://satt.oss-cn-hangzhou.aliyuncs.com/img/image-20211009210520689.png)

为了实现内存池，我们要在shapeFactory里创建一个list，并写上创建方法。

```
List<Shape>[] pools;

	void CreatePools () {
		pools = new List<Shape>[prefabs.Length];
		for (int i = 0; i < pools.Length; i++) {
			pools[i] = new List<Shape>();
		}
	}
```

内存池的具体思路就是，当我们创建一个物体时，首先判断内存池中还有没有对应形状的物体，如果有，就从其中激活一个物体。如果没有，那么就只能创建一个新物体。摧毁物体的时候也是同理，我们不对物体进行销毁，而是将其隐藏。具体的代码在这里省略了。

## 多场景

我们想要让创建的物体生成在一个专门的场景中。既然物体是在运行时创建的，那么容纳它们的场景自然也是如此。也就是说，我们不能在编辑器里新建场景，而是应该用代码来创建它。

要实现它，我们先创建一个池场景来容纳所有可以被回收的物体实例。所有创建的物体将进入这里，并且再也不会被移出去。我们可以通过如下代码来创建一个场景。

```C
Scene poolScene; //声明的池场景

	void CreatePools () {
		…
		poolScene = 	SceneManager.CreateScene(name);
	}
```

这里需要注意的是，我们创建场景用的名字是当前shapefactory的名字。因此，当我们拥有多个工厂的时候，就需要注意，要给这些工厂以不同的命名。

有了新创建的场景，那么就需要修改代码，来让新创建的物体放在新场景下。

```c#
	public Shape Get (int shapeId = 0, int materialId = 0) {
		Shape instance;
		if (recycle) {
			…
			if (lastIndex >= 0) {
				…
			}
			else {
				instance = Instantiate(prefabs[shapeId]);
				instance.ShapeId = shapeId;
				SceneManager.MoveGameObjectToScene(
					instance.gameObject, poolScene
				);
			}
		}
		…
	}
```

目前，我们实现了效果。

但是仍然有一些问题。比如，在播放模式下，如果我们修改了C#代码，那么在再编译后，我们的池场景就会被弄乱。这里的原因是：

unity不会再次编译ScriptableObject类。也就是我们的工厂的类型。也就是说，当再编译后，我们的池场景连同内存池之类的都销毁了。那么自然，CreatePools()方法将会再次被调用，产生错误。

因此，我们似乎可以简单地在CreatePools方法中加上一句判断来避免这个问题。

```c
		if (poolScene.isLoaded) {
			return;
		}
```

但是这不起作用，那是因为场景是一种结构体，而非引用。并且由于之前提到的不可序列化性质，所以在再编译后，我们已经丢失了poolScene的引用了。因此，我们需要再次获得它。

同时，我们需要注意到，我们只会在编辑器里才会出现再编译的情况，没必要在构建后的版本中如此做，因此，再加上一个当前是否在编辑器模式的判断是更好的选择。

此外，还有一个更隐蔽的问题，那就是，既然我们已经丢失了之前的引用，那么那些被设置为非活动状态的物体就再不能被激活了。因此，我们还需要对代码做一些修改。

```c
		if (Application.isEditor) {
			poolScene = SceneManager.GetSceneByName(name);
			if (poolScene.isLoaded) {
				GameObject[] rootObjects = poolScene.GetRootGameObjects();
                				for (int i = 0; i < rootObjects.Length; i++) {
					Shape pooledShape = rootObjects[i].GetComponent<Shape>();
if (!pooledShape.gameObject.activeSelf) {
						pools[pooledShape.ShapeId].Add(pooledShape);
					}
				}
				return; 
			}
		}

		poolScene = SceneManager.CreateScene(name);
```

至此，我们终于实现了一开始的目标：在新场景中创建物体。

### 多场景编辑

有了一个场景，自然而然地就需要考虑多场景的问题了，这也是本章的重点。

在创建一个新场景后，我们将其移动到hierarchy中。值得注意的是，每个场景都有一个摄像机和光照系统，我们保留主场景的摄像头，但是光照系统则由其他的各个场景负责。

此处要注意，为了在构建后的版本中也能调用多个场景，我们需要在*File / Build Settings*里添加索引。

![image-20211010200710281](http://satt.oss-cn-hangzhou.aliyuncs.com/img/image-20211010200710281.png)

在添加了索引后，我们在hierarchy中删除level 1场景也没关系了（因为我们在代码中会创建它）。

为了让光照系统工作正常，我们需要将LoadSceneMode设置为Additive。

```
	void LoadLevel () {
		SceneManager.LoadScene("Level 1", LoadSceneMode.Additive);
		SceneManager.SetActiveScene(SceneManager.GetSceneByName("Level 1"));
	}
```



但是只是这样还不够，因为加载场景需要一定的时间。后面的SetActiveScene方法在运行的时候，往往场景还没有加载好。因此，这就需要我们用到协程。



接下去还有一些细节的问题，比如判断是否加载之类的。不赘述。

### 多场景切换

有了level 1，我们还需要有level 2。

仿照之前的做法，创建场景，在build里构建索引。

同时，给loadLevel方法增加参数，让他支持加载不同level。

```c++
	IEnumerator LoadLevel (int levelBuildIndex) {
		enabled = false;
		yield return SceneManager.LoadSceneAsync(
			levelBuildIndex, LoadSceneMode.Additive
		);
		SceneManager.SetActiveScene(
			SceneManager.GetSceneByBuildIndex(levelBuildIndex)
		);
		enabled = true;
	}
```

为了方便我们操作，我们还需要实现按下键盘上的1、2、3、4……来实现场景的切换。这些功能的实现都和往常一样。

```c#
		else {
			for (int i = 1; i <= levelCount; i++) {
				if (Input.GetKeyDown(KeyCode.Alpha0 + i)) {
					StartCoroutine(LoadLevel(i));
					return;
				}
			}
		}
```

支持加载不同的场景，但是我们发现，切换后的场景并不会被删除。因此还要加上如下代码

```c#
	int loadedLevelBuildIndex; // 记录当前的场景
				if (loadedScene.name.Contains("Level ")) {
					SceneManager.SetActiveScene(loadedScene);
					loadedLevelBuildIndex = loadedScene.buildIndex;
					return;
				}
				
					IEnumerator LoadLevel (int levelBuildIndex) {
		…
		loadedLevelBuildIndex = levelBuildIndex;
		enabled = true;
	}
	
		IEnumerator LoadLevel (int levelBuildIndex) {
		enabled = false;
		if (loadedLevelBuildIndex > 0) {
			yield return SceneManager.UnloadSceneAsync(loadedLevelBuildIndex);
		}
		yield return SceneManager.LoadSceneAsync(
			levelBuildIndex, LoadSceneMode.Additive
		);
		SceneManager.SetActiveScene(
			SceneManager.GetSceneByBuildIndex(levelBuildIndex)
		);
		loadedLevelBuildIndex = levelBuildIndex;
		enabled = true;
	}
	
					if (Input.GetKeyDown(KeyCode.Alpha0 + i)) {
					BeginNewGame();
					StartCoroutine(LoadLevel(i));
					return;
				}

```

最后，实现了多个场景的切换，我们还需要让我们的存档文件也能记录我们当前打开的是哪个场景。因此，不得不的，我们的saveVersion++了。同时，在save和load方法中添加一个新的值，用以记录场景编号。

## 生成区

在有了多场景切换后，我们下一个需要实现或者说改进的功能就是生成物体的区域了。还记得在一开始，我们将物体随机生成的区域设定为

```
t.localPosition = Random.insideUnitSphere * 5f;
```

也就是(0,0,0)半径为5f的球体里。这是写死的数据。因此我们需要改进他。即使用生成区的概念。生成区，顾名思义就是生成物体的区域。

我们先创建一个新的脚本，命名为SpawnZone。并在hierarchy中新建一个物体并绑定上脚本。它将用来代表生成区。

在SpawnZone类中，我们返回一个点。这就是生成物体的点。它可以由一个变量控制是否生成在球体表面。

```
    public Vector3 SpawnPoint
    {
        get
        {
            return transform.TransformPoint(surfaceOnly?Random.onUnitSphere: Random.insideUnitSphere);
        }
    }
```

这样我们就可以通过改变spawnZone的形状来控制生成区的大小了。

接下去，我们在game类中创建一个对spawnZone的引用。然后修改CreateShape()方法。

```c#
	void CreateShape () {
		Shape instance = shapeFactory.GetRandom();
		Transform t = instance.transform;
		//t.localPosition = Random.insideUnitSphere * 5f;
		t.localPosition = spawnZone.SpawnPoint;
		…
	}
```

但是，生成区毕竟是隐形的（出于各种理由，我们都不应该让玩家看到他），为了方便我们调试。我们可以使用如下代码来让其可见。即使用Gizmos.

```c
	void OnDrawGizmos () {
		Gizmos.color = Color.cyan;
		Gizmos.matrix = transform.localToWorldMatrix;
		Gizmos.DrawWireSphere(Vector3.zero, 1f);
	}
```

![image-20211010212723074](http://satt.oss-cn-hangzhou.aliyuncs.com/img/image-20211010212723074.png)


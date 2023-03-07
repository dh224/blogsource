---
title: 基于PlayerPrefs的存档机制
date: 2023-02-28 11:43:46
tags: [Unity, IK, OMC]
thumbnail: 
---

[PlayerPrefs](https://docs.unity3d.com/ScriptReference/PlayerPrefs.html)用于存储玩家偏好设置，支持字符串、int以及float三种类型数据的存储，数据存储在注册表内，不同系统的存储路径不同。尽管本意是用于存储偏好设置，PlayerPrefs也可以用于保存简单的存档。
主要API主要是SetXXX()以及GetXXX()系列，比较简单。如：
``` c#
	PlayerPrefs.SetString(key, value);
```
这种方式只适合存储基本的数据类型，如果需要存储稍复杂一点的数据，我们可以定义一个类，如：PlayerData，随后使用Unity内建的JsonUtility将PlayerData类转换为JSON格式，这样就可以将多个数据封装成为单个数据类型并以字符串的形式存储了，读取也是类似，以字符串的类型读取，再使用JsonUtility转回PlayerData类即可，代码示例如下：
``` C#
// 设定的Data类，用于封装多种类型的数据
[System.Serializable]  
class SaveData  
{  
    public string playerName;  
    public int playerLevel;  
    public int playerCoin;  
    public Vector3 playerPosition;  
}

public static void SaveByPlayerPrefs(string key, object data)  // 存储数据，先转成JSON格式再以String格式存储。
{  
    var json = JsonUtility.ToJson(data);  
    PlayerPrefs.SetString(key, json);  
    PlayerPrefs.Save();  
#if UNITY_EDITOR  
    Debug.Log("Successfully saved data to PlayerPrefs.");  
#endif  
}
public static string LoadByPlayerPrefs(string key)  // 从注册表中读取数据
{  
    return PlayerPrefs.GetString(key, null);  
}

```
## 基于JSON的存档机制
在实现PlayerPrefs的存档之后，可以很容易地想到：我们可以直接将JSON文件存储下来。这样做的好处是可以方便地建立起云存档服务。与之前的代码类似，只是多了文件操作的部分.如下：
``` C#
public static void SaveByJSON(string saveFileName, object dataObject)  
{  
    var json = JsonUtility.ToJson(dataObject);  
    var path = Path.Combine(Application.persistentDataPath, saveFileName);  
    try  
    {  
        File.WriteAllText(path,json);  
    #if UNITY_EDITOR  
        Debug.Log($"Successfully saved data to {path}");  
    #endif  
    }  
    catch (System.Exception e)  
    {    #if UNITY_EDITOR  
        Debug.LogError($"Failed to save data to {path}. \n because of Exception : {e}");  
    #endif  
    }  
}  
  
public static T LoadFromJSON<T>(string saveFileName)  
{  
    var path = Path.Combine(Application.persistentDataPath, saveFileName);  
    try  
    {  
        var json = File.ReadAllText(path);  
        var data = JsonUtility.FromJson<T>(json);  
        return data;  
    }    catch (System.Exception e)  
    {    #if UNITY_EDITOR  
        Debug.LogError($"Failed to load data from {path}. \n because of Exception : {e}");  
    #endif  
        return default;   
    }  
}  
  
#region Deleting  
  
public static void DeleteSaveFile(string saveFileName)  
{  
    var path = Path.Combine(Application.persistentDataPath, saveFileName);  
    try  
    {  
        File.Delete(path);  
    }    catch (System.Exception e)  
    {    #if UNITY_EDITOR  
        Debug.LogError($"Failed to delete data from {path}. \n because of Exception : {e}");  
    #endif  
    }  
}
```
代码都很容易懂，就不需要注释了。值得一提的是我们可以使用Path.persistenDataPath获得存储路径，这个路径会在项目Build时自动适配不同的操作系统，我们不再需要手动设置了。

>整篇文章的内容主要照抄自阿严的[*存档系统*](https://www.bilibili.com/video/BV1Cb4y1b71G).
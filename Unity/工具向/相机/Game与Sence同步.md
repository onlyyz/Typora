<font color=#4db8ff>Link：</font>：https://blog.csdn.net/xinjay1992/article/details/108065220

1.通过<font color=#4db8ff>SceneView.lastActiveSceneView.camera</font>获取当前Scene视图中的<font color="red">观察相机</font>；

2.每帧调整Game视图相机的<font color=#66ff66>position和rotation</font>；

3.为了保证脚本在编辑器模式下也能达到我们想要的效果，因此需要给脚本添加<font color=#66ff66>ExecuteInEditMode</font>特性或者<font color=#66ff66>ExecuteAlways</font>特性；

4.为了确保每帧都能正常调用相机调整的代码，因此需要将调整的代码放到<font color="red">OnRenderObject</font>系统回调函数中

（至于为什么不放到Update系统回调函数中，是因为Update函数在编辑器模式下即使添加了<font color=#bc8df9>ExecuteInEditMode/ExecuteAlways</font>特性，也只有当场景有变化时才调用（比如某个对象位置变动），而不是每帧都调用，即在编辑器模式下不能确保脚本的<font color=#66ff66>Update</font> 函数每帧都能被调用）。

```C#
using UnityEngine;
using UnityEditor;
[ExecuteInEditMode]
public class CameraFollow : MonoBehaviour
{
    private void OnRenderObject()
    {
        transform.position = SceneView.lastActiveSceneView.camera.transform.position;
        transform.rotation = SceneView.lastActiveSceneView.camera.transform.rotation;
    }
```

![img](https://img-blog.csdnimg.cn/20200817220817648.gif)
#### 一、组件

##### 1.1 m_CameraDistance

```c#
public class MoveScript: MonoBehaviour
{
    private GameObject camObj;
    void Start()
    {
        camObj = GameObject.Find("Vertical Follow Camera");
        camObj.GetComponent<CinemachineVirtualCamera>(); // <- but I get error saying, The type or namespace name 'CinemachineVirtualCamera' could not be found 
    }
}
```

##### 1.2 Body

```c#
CinemachineVirtualCamera cmvc = gameobject.AddComponent<CinemachineVirtualCamera>();
CinemachineTrackedDolly body = cmvc.AddCinemachineComponent<CinemachineTrackedDolly>();
CinemachinePOV aim = cmvc.AddCinemachineComponent<CinemachinePOV>();
```

#### 二、功能

##### 2.1 Blend Camera

<font color=#4db8ff>Question：</font>https://forum.unity.com/threads/how-to-change-blend-speed-between-two-cinemachinevirtualcameras-through-code.1464638/

<font color=#4db8ff>documentation ：</font>https://docs.unity3d.com/Packages/com.unity.cinemachine@2.3/api/Cinemachine.CinemachineCore.html#Cinemachine_CinemachineCore_GetBlendOverride

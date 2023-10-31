#### 1.1 m_CameraDistance

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

#### 1.2 Body

```c#
CinemachineVirtualCamera cmvc = gameobject.AddComponent<CinemachineVirtualCamera>();
CinemachineTrackedDolly body = cmvc.AddCinemachineComponent<CinemachineTrackedDolly>();
CinemachinePOV aim = cmvc.AddCinemachineComponent<CinemachinePOV>();
```


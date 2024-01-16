<font color=#4db8ff>Link：</font>https://zhuanlan.zhihu.com/p/413733828

<font color=#4db8ff>Link： </font>https://www.youtube.com/watch?v=jQgwEsJISy0&t=19s

<font color=#4db8ff>Link： </font>https://www.bilibili.com/video/BV1Pr4y1f7rv/?spm_id_from=333.337.search-card.all.click&vd_source=fc3995fc21714a345f57a951f8232fb7

### 一、 委托和事件

事件可以充当通信机制

##### 1.1 Title

```c#
namespace EventandDelegates
{
    public class Video 
    {
        public string Title { get; set; }
    }
}
```

##### 1.2 EventandDelegates

```c#
using System;
using System.Threading;

namespace EventandDelegates
{
    
    public class VideoEventArgs: EventArgs
    {
            public Video Video { get; set; }  
    }
    
    public class VideoEncoder
    {
        //1 - define delegate
        //2 - define an event based on that delegate
        //3 - Raise the event

        
        //1 - 决定了订阅的形式
        public delegate void VideoEncodeEventHandler(object source, VideoEventArgs args);
        //2 - 创建事件
        public event VideoEncodeEventHandler VideoEncoded;

        public void Encode(Video video)
        {
            Console.WriteLine("Encoding Video...");
            Thread.Sleep(2000);

            //3 - 发起事件 - 通知所有订阅者
            OnVideoEncode(video);
        }

        protected virtual void OnVideoEncode(Video video)
        {
            if (VideoEncoded != null)
                VideoEncoded(this, new VideoEventArgs(){Video = video});
        }
    }
}
```

##### 1.3 Program

```c#
using System;

namespace EventandDelegates
{
    public class Program 
    {
        static void Main(string[] args)
        {
            var video = new Video() { Title = "Video 1" };
            var videoEncoder = new VideoEncoder();       //发布
            var MailService = new MailService();        //订阅
            var MassageService = new MassageService();        //订阅

            //像方法的指针列表
            videoEncoder.VideoEncoded += MailService.OnVideoEncoded;
            videoEncoder.VideoEncoded += MassageService.OnVideoEncoded;
            //调用前订阅
            VideoEncoder.Encode(video);
        }
    }

    //可以位于其他文件夹
    public class MailService
    {
        public void OnVideoEncoded(object source, VideoEventArgs args)
        {
            Console.WriteLine("MailService ");
            
        }
    }
    public class MassageService
    {
        public void OnVideoEncoded(object source, VideoEventArgs args)
        {
            Console.WriteLine("MassageService ");
            
        }
    }
}
```

1.4 委托形式

```c#
//1 - 决定了订阅的形式
public delegate void VideoEncodeEventHandler(object source, VideoEventArgs args);
//EventHandler
//EventHandler<TEventArgs>
//2 - 创建事件
public event VideoEncodeEventHandler VideoEncoded;
```



```c#
public event EventHandler<VideoEventArgs> VideoEncoded;

//object source, EventArgs args
public event EventHandler VideoEncoded;
```





##### for example

```C#
using UnityEngine;
using UnityEngine.Events;

public class GameObjectFilter : MonoBehaviour
{
    [System.Serializable]
    public class ScriptFilterEvent : UnityEvent<GameObject>
    {
    }

    [SerializeField]
    private ScriptFilterEvent onFilterEvent;

    void Start()
    {
        // 在这里执行过滤操作，你可以在Start方法中调用，或者在编辑器中手动触发
        FilterGameObjects();
    }

    void FilterGameObjects()
    {
        foreach (Transform child in transform)
        {
            // 触发事件，将子物体传递给订阅者
            onFilterEvent.Invoke(child.gameObject);
        }
    }
}
```

##### 1.4 Delegate

<font color=#4db8ff>Link：</font>https://www.youtube.com/watch?v=J01z1F-du-E

委托可以从类外部设置，调用委托

需要声明委托类型，随后创建委托实例

```C#
//1 - 决定了订阅的形式
public delegate void TriggerEncodeEventHandler(object source, TriggerEventArgs args);
//2 - 创建事件
public TriggerEncodeEventHandler TriggerEncoded;
```

##### 1.5 Event

事件只可以从类内部设置，调用委托

```C#
//1 - 决定了订阅的形式
public delegate void TriggerEncodeEventHandler(object source, TriggerEventArgs args);
//2 - 创建事件
public event TriggerEncodeEventHandler TriggerEncoded;
```

### 二、Action、UnityEvent

<font color=#4db8ff>Link：</font>https://www.youtube.com/watch?v=8fcI8W9NBEo

Action是现成的无参委托

#### 2.1 Action

```c#
public static event Action OnUnVariable;
public static event Action<float> OnVariable;

OnVariable?.Invoke(float);
```



#### 2.2 UnityEvent

UnityEvent 并不需要判断是否为空

```c#
public UnityEvent OnUnityEvent;
	OnUnityEvent.Invoke();
```





 UnityEvent才是我们游戏中更加常用的事件类型，从功能上来说，UnityEvent和C#的事件（Event）有一些相似之处，首先它也允许你定义自己的事件，并将其绑定到特定的方法或函数，就像使用Event那样去使用。

```c#
public class Test : MonoBehaviour
{
    // 定义一个 Unity Event
    public UnityEvent onClickEvent;

    private void Start()
    {
        // 获取按钮组件
        Button button = GetComponent<Button>();

        // 绑定点击事件
        button.onClick.AddListener(OnButtonClick);
    }

    private void OnButtonClick()
    {
        // 触发 Unity Event，执行绑定的方法
        onClickEvent.Invoke();
    }
}
```


<font color=#4db8ff>Link：</font>https://zhuanlan.zhihu.com/p/413733828

<font color=#4db8ff>Link： </font>https://www.youtube.com/watch?v=jQgwEsJISy0&t=19s

### 一、 委托和事件

事件可以充当通信机制

#### 1.1 Title

```c#
namespace EventandDelegates
{
    public class Video 
    {
        public string Title { get; set; }
    }
}
```

#### 1.2 EventandDelegates

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

#### 1.3 Program

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

            //只想方法的指针列表
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





### for example

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
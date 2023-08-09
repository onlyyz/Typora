

### 

1、不透明纹理

#### 1.1 <font color="red">ConfigureInput </font>

使用 ScriptableRenderPassInput.Color 参数调用 ConfigureInput 时
确保渲染传递可以使用<font color=#4db8ff>不透明纹理</font>.

#### 1.2 <font color=#FFCE70>ConfigureTarget</font>

配置本次渲染传递的渲染目标。调用此方法可替代 CommandBuffer.SetRenderTarget 方法。
此方法应在 Configure.SetRenderTarget 中调用。





1、Volume 添加组件

2、getcomponent 获得组件参数

3、RenderFeature

作用是可以让我们扩展渲染的pass，官方的插件完全解耦，可以灵活的在渲染的各个阶段插入commandbuffer，这个插入点由RenderPassEvent决定。

每个RenderFeature必须要继承ScriptableRendererFeature抽象类，并且实现AddRenderPasses跟Create函数。在ScriptableRendererFeature对象被初始化的时候，首先会调用Create方法，我们可以在这构建一个ScriptableRenderPass实例，然后在AddRenderPasses方法中对其进行初始化并把它添加到渲染队列中，这里大家或许已经明白，ScriptableRendererFeature对象跟ScriptableRenderPass对象需要搭配使用。创建好AdditionPostProcessRendererFeature，我们就可以在ForwardRenderer中添加新定义的RenderFeature了。

#### 1、VolumeComponent

```C#
//add to the Volume componentMenu 
[Serializable, VolumeComponentMenu("URP14/Test")]
public class ClassName : VolumeComponent,IPostProcessComponent,IDispose
{
    	//set the parameters UI ：form VolumeParameter
        public FloatParameter threshold = new ClampedFloatParameter(0.5f, 0.0f, 1.0f);
        public FloatParameter softThreshold = new ClampedFloatParameter(0.5f, 0.0f, 1.0f);
        public FloatParameter intensity = new ClampedFloatParameter(1.0f, 1.0f, 10.0f);
        public ColorParameter colorTint = new ColorParameter(Color.white);
        public FloatParameter blurRange = new ClampedFloatParameter(0f, 0f, 15f);
        public IntParameter iteration = new ClampedIntParameter(4, 1, 8);
        public FloatParameter downSample = new ClampedFloatParameter(1f, 1f, 10f);
}
```

1.1、IPostProcessComponent

IsActive是为了告知是否需要渲染后期处理

IsTileCompatible告知后期处理是否可以在平铺时运行特效，或者是否需要一次完全通过

```c#
 /// Tells if the post process needs to be rendered or not.
public bool IsActive()
{
    return active;
}
//Tells if the post process can run the effect on-tile or if it needs a full pass
public bool IsTileCompatible()=> false;
```

![image-20230809174142591](E:\Typora-Note\assets\image-20230809174142591.png)


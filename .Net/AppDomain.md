```c++
http://www.cnblogs.com/kongyiyun/archive/2011/08/01/2123459.html
```

#### 注：

<font color=#4db8ff>CLR</font>：common language runtime（公共语言运行库）

### 一、Domain（域）

<font color="red">CLR</font> （<font color=#FFCE70>公共语言运行库</font>）在启动的时候会创建<font color=#4db8ff>系统域(System Domain)、共享域(Shared Domain)和默认域(Default Domain)</font>、<font color=#FFCE70>系统域与共享域</font>对于用户是不可见的

<font color=#66ff66>默认域</font>也可以说是当前域，它承载了当前应用程序的各类信息(堆栈)，所以，我们的一切操作都是在这个默认域上进行."插件式"开发很大程度上就是依靠AppDomain来进行.

当加载了一个<font color=#4db8ff>程序集（Assemblies）</font>之后，该程序集就会被加入到指定<font color="red">AppDomain</font>中，按照原来的想法，要实现"<font color=#4db8ff>热插拔</font>"，只要在需要使用该"插件"的时候，加载该"插件"的程序集(dll)，使用结束后，卸载掉该程序集便可达到我们预期的效果。

加载程序集很简单，C#提供一个<font color="red">Assembly</font>类，方便又快捷.

<font color=#4db8ff>Assembly link ：</font> https://docs.unity3d.com/ScriptReference/Compilation.Assembly.html

然后，<font color=#FD00FF>C#却没有提供卸载程序集的方法</font>，唯一能卸载程序集的方法只有卸载该程序集所在的<font color="red">AppDomain</font>，这样，该AppDomain下的程序集都会被释放。知道这一点，我们便可以利用AppDomain来达到我们预期的效果。

### 二、程序集锁定

```c#
setup.ShadowCopyFiles = "true";
```

这句很重要，其作用就是启用影像复制程序集，什么是[影像复制程序集](http://msdn.microsoft.com/library/ZH-CN/DE8B8759-FCA7-4260-896B-5A4973157672(VS.100))，复制程序集是保证"热插拔"

实现的主要工作.AppDomain加载程序集的时候，如果没有ShadowCopyFiles，那就直接加载程序集，结果就是程序集被锁定，相反，如果启用了ShadowCopyFiles，则CLR会将准备加载的程序集拷贝一份至CachePath，再加载CachePath的这一份程序集，这样原程序集也就不会被锁定了. AppDomain.CurrentDomain.SetShadowCopyFiles();的作用就是当前AppDomain也启用ShadowCopyFiles，在此，当前AppDomain也就是前面我们说过的那个默认域(Default Domain)，为什么当前域也要启用ShadowCopyFiles呢?

>   主AppDomian在调用子AppDomain提供过来的类型，方法，属性的时候，也会将该程序集添加到自身程序集引用当中去，所以，"插件"程序集就被主AppDomain锁定，这也是为什么创建了单独的AppDomain程序集也不能删除，替换(释放)的根本原因
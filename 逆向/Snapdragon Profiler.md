## snapdragon profiler

### 1、配置环境

 **download**:https://developer.qualcomm.com/software/snapdragon-profiler/tools

环境配置：环境需要的是安卓环境，查看是否激活该环境 利用window + R 或者搜索栏命令提示符。

输入adb devices。进行调试

![image-20230414160300952](C:\Users\tangjianpeng\AppData\Roaming\Typora\typora-user-images\image-20230414160300952.png)

Android Debug Bridge，即Android 调试桥

adb命令失败则说明，应该是你没有android sdk环境，需要安装一下 android stuido 

**Android  SDK DownLoad**：https://www.androiddevtools.cn/

![image-20230414160438256](E:\Typora-Note\逆向\assets\image-20230414160438256.png)

有.zip以及.exe，选择zip，解压即可食用。

打开解压包里面的 SDK Manager.exe。安装相应包体。

![image-20230414160619188](E:\Typora-Note\逆向\assets\image-20230414160619188.png)

一般选择最新版下载即可

![image-20230414160645975](E:\Typora-Note\逆向\assets\image-20230414160645975.png)

选择完成之后点击右下角的instal packages。网络环境需要科学上网

![image-20230414160719280](E:\Typora-Note\逆向\assets\image-20230414160719280.png)

当下载完成之后，还需要其他的操作，才可以配置完成。

<img src="E:\Typora-Note\逆向\assets\image-20230414160818038.png" alt="image-20230414160818038" style="zoom:50%;" />

搜索栏，搜索高级系统设置/高级/环境变量。在用户变量处新建

<img src="E:\Typora-Note\逆向\assets\image-20230414161000300.png" alt="image-20230414161000300" style="zoom:50%;" />

输入路径为adb.exe的文件路径

![image-20230414161103944](E:\Typora-Note\逆向\assets\image-20230414161103944.png)

即：D:\Android-sdk-windows\platform-tools

点击确定

![image-20230414161131878](E:\Typora-Note\逆向\assets\image-20230414161131878.png)

系统变量，新建变量名

name：ANDROID_HOME

变量值：为adb.exe Path

即：D:\Android-sdk-windows\platform-tools

![image-20230414161224294](E:\Typora-Note\逆向\assets\image-20230414161224294.png)

点击Path

![image-20230414161319626](E:\Typora-Note\逆向\assets\image-20230414161319626.png)

新建三个变量：

%ANDROID_SDK_HOME%\platform-tools

%ANDROID_SDK_HOME%\tools

%ANDROID_SDK_HOME%\build-tools\28.0.0

![image-20230414161353604](E:\Typora-Note\逆向\assets\image-20230414161353604.png)

环境配置完成，进入控制台，输入adb devices 回车调试。显示以下内容即为环境配置完成

![image-20230414161510951](E:\Typora-Note\逆向\assets\image-20230414161510951.png)

snapdragon profiler若无法识别到手机，则需要重启电脑

### 2、基础操作
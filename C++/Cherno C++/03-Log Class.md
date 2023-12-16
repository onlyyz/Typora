<font color=#4db8ff>Link：</font>https://www.youtube.com/watch?v=3dHBFBw13E0&list=PLlrATfBNZ98dudnM48yfGUldqGD0S4FFb&index=24

#### 一、Static 

##### 1.1 含义

Staic 有两种含义，内容取决于具体的<font color=#4db8ff> Context </font>

##### 1.2 类外静态

Static 的意思是<font color=#4db8ff> internal</font>内联含义，仅在内部使用

##### 1.3 类内静态

与类的所有实例共享内存，即拥有唯一一个<font color=#4db8ff>Static instance</font>，即单例模式或全局变量

##### 1.4 for example

Static.cpp 

仅存在于编译OBJ文件中，即程序集差异

```c++
static int s_Variable = 5;
```

此时Debug 结果为10 ，但是当我们取消 **Static** 中的 <font color=#4db8ff>static</font> 关键字后，编译会出现**link**错误，因为变量已经在不同的翻译单元中定义，不能有两个同名的全局变量

```c++
int s_Variable = 10;
int main()
{
	std::cout << s_Variable << std::endl;
	std::cin.get();
}
```

##### 1.5 extern internal

可以将<font color=#4db8ff> extern </font>变量将会查找外部数据，这被称为 <font color=#4db8ff>extern internal</font> 外部链接，

```c++
//
int s_Variable = 5;
//
extern int s_Variable = 10;
int main（）{}
```

此时如果将 **Static** 中的 <font color=#4db8ff>static</font> 关键字还原，则会出现查找错误

```c++
static int s_Variable = 5;
```

这很像在类中定义 **私有变量** 没有其他翻译单元会得到这个私有变量，<font color=#4db8ff> Linker </font>也不会在全局范围中找到它，因为已经有效地将变量设置为私有

##### 1.6 Staic Function

如果在两个Cpp之间设置一个为<font color=#4db8ff> Staic </font> ，一个不设置，当<font color=#4db8ff> Linker </font>填充内容的时候，将不会得到静态函数，因此不会得到编译错误

如果将静态变量头文件包含在其他文件中，则对文件可见

#### 二、Log Level

log level分为三层，<font color=#4db8ff> Warning、Error、Trace(Message) </font>



```c++
#include <iostream>
#define Log(x) std::cout<<x<<std::endl;

class Log {
public:
	const int LogLevelError = 0;
	const int LogLevelWarning = 1;
	const int LogLevelInfo = 2;
private:
	int m_LogLevel = LogLevelInfo;

public:
	void SetLevel(int Level)
	{
		m_LogLevel = Level;
	}
	void Error(const char* message)
	{
		if (m_LogLevel >= LogLevelError)
			std::cout << "[Error]: " << message << std::endl;
	}
	void Warn(const char* message)
	{
		if (m_LogLevel >= LogLevelWarning)
			std::cout << "[WARNING]: " << message << std::endl;
	}
	void Info(const char* message)
	{
		if (m_LogLevel >= LogLevelInfo)
			std::cout << "[INFO]: " << message << std::endl;
	}
};

int main()
{
	Log log;
	log.SetLevel(log.LogLevelWarning);
	log.Warn("Hello");

	log.SetLevel(log.LogLevelInfo);
	log.Info("Hello");
	std::cin.get();
}

```

#### 三、Static

<font color=#4db8ff>Link：</font>https://www.youtube.com/watch?v=V-BFlMrBtqQ&list=PLlrATfBNZ98dudnM48yfGUldqGD0S4FFb&index=22

##### 3.1 Class Struct

Class内静态意味着，Class所有实例中存在唯一实例

```c++
struct Entity
{
	static int x, y;
    void Print()
	{
		std::cout << x << ", " << y << std::endl;
	}
};

int main()
{
	Entity e;
	e.x = 2;
	e.y = 3;
	Entity e1;
    e1.x = 5;
	e1.y = 8;
}
```

![image-20231214235747110](./assets/image-20231214235747110.png)

此时会出现错误，因为我们必须在一些位置上定义这些静态变量

```c++
int Entity::x;
int Entity::y;
int main()
{}
```

他们被定义为链接，因为将X和Y设置为静态变量以后，所有的实体类的所有实例中X和Y只存在一个实例。意味着当我修改其中一个实例的时候，所有实例都会被修改，因为他们本质是相同的

<font color=#4db8ff> 指向相同内存的两个不同的实例，但是x y 是否事项共享，指向相同的x y </font>

因此，下列代码并无意义

```c++
e1.x = 5;
e1.y = 8;
```

可以引用他们，就像创建了两个变量，基本只存在于命名空间

```c++
Entity::x = 5;
Entity::y = 8;
```

跨类拥有变量时，很有用，如果将方法改为静态也可以调用

```c++
static  void Print()
{
    std::cout << x << ", " << y << std::endl;
}
...

Entity::Print();
Entity::Print();
```

如果x y 不是静态的，那么函数将会在这里中断，因为静态函数不能访问非静态变量

原因是：<font color=#4db8ff>静态方法没有类实例，因为在类中每个非静态函数，都会获取一个实例，当前的类作为函数，这也是类的幕后实际工作方式</font>

不存在类之类的东西，他们是带有隐藏参数的函数，但是静态方法无法获得隐藏参数

![image-20231215004339172](./assets/image-20231215004339172.png)

```c++
struct Entity
{
	int x, y;
		
	static  void Print()
	{
		std::cout << x << ", " << y << std::endl;
	}
};

static void Print(Entity e)
{
	std::cout << e.x << ", " << e.y << std::endl;
}


int main()
{
	Entity e;
	e.x = 2;
	e.y = 3;
	Entity e1;
	Entity::x = 5;
	Entity::y = 8;

	Entity::Print();
	Entity::Print();
}
```

但是这样修改就可以继续工作，本质是一个非类方法

如果去掉 “ Entity ” 则会出现错误，因为没有给予一个实体的引用，程序不知道，要访问实例的 x y 是那个

##### 3.2 Enums

<font color=#4db8ff>Link：</font>https://www.youtube.com/watch?v=x55jfOd5PEE&list=PLlrATfBNZ98dudnM48yfGUldqGD0S4FFb&index=23

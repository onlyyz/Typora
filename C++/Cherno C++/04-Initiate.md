<font color=#4db8ff> Link：</font>https://www.youtube.com/watch?v=1nfuYMXjZsA&list=PLlrATfBNZ98dudnM48yfGUldqGD0S4FFb&index=35

#### 一、Initiate

##### 1.1 构造函数

构造赋值，因为会按照定义的顺序进行赋值，因此需要按<font color=#4db8ff>顺序初始化</font>

打破顺序会有依赖性问题出现

```c++
class Entity
{
private:
std::string m_Name;
int m_Socre;
public:
Entity(const std::string& name): m_Name(name)
{
m_Name = name;
}
};
```

构造函数二次创建

```c++
class Example
{
	public:
	Example(int x)
	{}
};
class Entity
{
private:
	std::string m_Name;
	//这里创建一个
	Example m_Example;
public:
	Entity()
	{
		//这里创建一个 此处会扔掉旧的实例
		m_Example = Example(1);
	}
};
```

优化利用赋值

```c++
Entity():m_Example(Example(1))
{
	//这里只会赋值
}

Entity():m_Example(1)
{
	//这里只会赋值
}
```

##### 1.2 Ternary Operators

<font color=#4db8ff>Link：</font>https://www.youtube.com/watch?v=ezqsL-st8qg&list=PLlrATfBNZ98dudnM48yfGUldqGD0S4FFb&index=36

##### 1.3 New

<font color=#4db8ff>Link：</font> https://www.youtube.com/watch?v=NUZdUSqsCs4&list=PLlrATfBNZ98dudnM48yfGUldqGD0S4FFb&index=39

<font color=#4db8ff>New</font> 本质是在堆上分配内存，并且在申请内存的同时调用构造函数

```c++
Entity* e = new Entity();
Entity* e = (Entity*)malloc(sizeof(Entity));

delete e;
```

两个代码本质区别在于，<font color=#4db8ff>New </font> 会调用类的构造函数，而<font color=#4db8ff>Malloc</font> 只是单纯的分配内存，随后返回指向该内存的指针

New 关键字内存不会被释放，不会被标记为空闲，也不会被放回到空闲列表。

堆分配的内存需要使用<font color=#4db8ff>Delete</font> 去释放，实际上它调用了C++ 的 <font color=#4db8ff>free()</font> 函数，释放了内存块

指定内存

```c++
int* b = new int[50];
Entity* e = new(b) Entity();
```

##### 1.4 Implicit and Explicit 

<font color=#4db8ff>Link：</font> https://www.youtube.com/watch?v=Rr1NX1lH3oE&list=PLlrATfBNZ98dudnM48yfGUldqGD0S4FFb&index=39

显示声明<font color=#4db8ff>Explicit </font> 禁止隐式转换即构造函数前添加显示关键字

##### 1.5 this

<font color=#4db8ff>this </font> 是一个指针

#### 二、Object

##### 2.1 Lifetime 

堆栈分配离开作用域内存就被清除，但是如果是通过堆分配，则一直存在，除非应用程序结束

```c++
var stock = New var();		//堆分配
```

可以通过智能指针，在析构函数中释放堆分配

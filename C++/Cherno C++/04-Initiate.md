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
# 第十五章 友元、异常和其他



本章内容包括：

- 友元类；
- 友元类方法；
- 嵌套类；
- 引发异常、`try`块和`catch`块；
- 异常类；
- 运行阶段类型识别（RTTI）；
- `dynamic_cast` 和 `typeid`；
- `static_cast`、`const_cast` 和 `reiterpret_cast`。

## 1、15.1  友元

让<font color="RoyalBlue">Remote::set_chan( )</font>成为Tv类的友元的方法是，在Tv类声明中将其声明为友元：

```C++
Class TV
{
	friend void Remote::set_chan( TV & t, int);
}
```

### 15.1.2 前向声明(forward declaration)

```C++
class TV;
class Remote{...}
class TV;
```

![image-20210815204948714](https://static.fungenomics.com/images/2021/08/image-20210815204948714.png)



## 15.2、嵌套类

## 15.3、 异常

#### 15.3.3、异常机制

>   引发异常
>
>   使用处理程序捕获异常
>
>   使用try块

程序使用异常处理程序（exception handler）

<font color="red">catch</font>关键字表示<font color="RoyalBlue">捕获异常</font>。<font color="RoyalBlue">处理程序</font>以关键字<font color="RoyalBlue">catch</font>开头，随后是位于括号中的类型声明，它指出了异常处理程序要响应的异常类型；

然后是一个用花括号括起的代码块，指出要采取的措施。<font color="green">catch关键字和异常类型</font>用作标签，指出当异常被引发时，程序应跳到这个位置执行。

<font color="RoyalBlue">异常处理程序也被称为catch块</font>。

<font color="red">try块标识</font>其中特定的异常可能被激活的代码块，它后面跟一个或多个catch块。<font color="green">try块是由关键字try指示的</font>，关键字try的后面是一个由花括号括起的代码块，表明需要注意这些代码引发的异常。

要了解这3个元素是如何协同工作的，最简单的方法是看一个简短的例子，如程序清单15.9所示。

```C++
// error3.cpp -- using an exception
#include <iostream>
double hmean(double a, double b);

int main()
{
    double x, y, z;
    std::cout << "Enter two numbers: ";
    while (std::cin >> x >> y)
    {
        try {                   // start of try block
            z = hmean(x,y);
        }                       // end of try block
        catch (const char * s)  // start of exception handler
        {
            std::cout << s << std::endl;
            std::cout << "Enter a new pair of numbers: ";
            continue;
        }                       // end of handler
        std::cout << "Harmonic mean of " << x << " and " << y
            << " is " << z << std::endl;
        std::cout << "Enter next set of numbers <q to quit>: ";
    }
    std::cout << "Bye!\n";
    return 0;
}

double hmean(double a, double b)
{
    if (a == -b)
        throw "bad hmean() arguments: a = -b not allowed";
    return 2.0 * a * b / (a + b); 
}
```

<font color="red">try</font>	特定异常激活代码块 <font color="green">-></font><font color="red">    throw</font>	引发异常 <font color="green">-></font> <font color="red">catch</font>	捕获异常 <font color="green">-></font>	<font color="RoyalBlue">执行catch 块</font> 处理异常

<font color="red">char*s</font>则表明该处理程序与字符串异常匹配。

s与函数参数定义极其类似，因为匹配的引发将被赋给s。另外，当异常与该处理程序匹配时，程序将执行括号中的代码。

<img src="E:/dev/Typora-Note/C++/CPP%20Primer%20Plus%20-%206/chapter15.assets/image-20230729003328388.png" alt="image-20230729003328388" style="zoom:50%;" />

2．bad_alloc异常和new

对于使用new导致的内存分配问题，C++的**最新处理方式是让new引发bad_alloc异常**。头文件new包含bad_alloc类的声明，它是从exception 类公有派生而来的。但**在以前，当无法分配请求的内存量时，new返回 一个空指针**，然后我们可以 `exit(EXIT_FAILURE)`。


## 15.6 总结

友元使得能够为类开发更灵活的接口。类可以将其他函数、其他类 和其他类的成员函数作为友元。在某些情况下，可能需要使用前向声 明，需要特别注意类和方法声明的顺序，以正确地组合友元。

嵌套类是在其他类中声明的类，它有助于设计这样的助手类，即实 现其他类，但不必是公有接口的组成部分。

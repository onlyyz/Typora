# 第十四章 C++中的代码重用

本章内容包括：

- has-a 关系；
- 包含对象成员的类；
- 模板类valarray
- 私有和保护继承；
- 多重继承；
- 虚基类；
- 创建类模板；
- 使用类模板；
- 模板的具体化。

## 14.1 包含对象成员的类
### 14.1.1 valarray类简介

valarray类是由头文件 `valarray` 支持的。**这个类用于处理数值，它支持诸如将数组中所有元素的值相加以及在数组中找出最大和最小的值等操作**。valarray被定义为一个模板类，以便能够处理不同的数据类型。例子：

```C++
double gpa[5] = {3.1,3.5,3.8,2.9,3.3};

valarrat<double> v1;		//an array of double size 0
valarrat<int> v2(8);		//of int size 8
valarrat<int> v3(10,8);		//of int size 8 ,each set to 10
valarrat<double> v4(gpa,4);	//of double size 8 ,each set to the gpa
```

从中可知，可以创建长度为零的空数组、指定长度的空数组、所有元素度被初始化为指定值的数组、用常规数组中的值进行初始化的数组。

这个类有一些内置方法：

- operator：访问各个元素；
- `size()`: 返回包含的元素数；
- `sum()`: 返回所有元素的总和；
- `max()`: 返回最大的元素；
- `min()`: 返回最小的元素。

等等，https://cplusplus.com/reference/valarray/valarray/ 。


### 14.1.2 Student类的设计

![image-20210815170944251](https://static.fungenomics.com/images/2021/08/image-20210815170944251.png)

![image-20210815172143416](https://static.fungenomics.com/images/2021/08/image-20210815172143416.png)

## 14.2 私有继承

C++还有另一种实现has-a关系的途径——私有继承。使用私有继 承，基类的公有成员和保护成员都将成为派生类的私有成员。这意味着 基类方法将不会成为派生对象公有接口的一部分，但可以在派生类的成 员函数中使用它们。

要进行私有继承，请使用关键字 `private` 而不是 `public` 来定义类（实 际上，`private`是默认值，因此省略访问限定符也将导致私有继承）。

使用私有继承时，只能在派生类的方法中使用基类的方法。

用类名显式地限定函数名不适合于友元函数，这是因为友元不属于 类。然而，可以通过显式地转换为基类来调用正确的函数。

如果两个基类都提供了函数operator<<( )，由于这个类使用的是多重继承，编译器将无法确定 应转换成哪个基类。

<font color="red">包含时使用对象名来调用函数，使用私有继承时使用类名和作用域解析符来调用方法</font>

### 14.2.2 使用包含还是私有继承

大多数C++程序员倾向于使用包含。首先，它易于理 解。类声明中包含表示被包含类的显式命名对象，代码可以通过名称引用这些对象，而使用继承将使关系更抽象。其次，继承会引起很多问题，尤其从多个基类继承时，可能必须处理很多问题，如包含同名方法 的独立的基类或共享祖先的独立基类。总之，使用包含不太可能遇到这 样的麻烦。另外，包含能够包括多个同类的子对象。如果某个类需要3 个string对象，可以使用包含声明3个独立的string成员。而继承则只能使 用一个这样的对象（当对象都没有名称时，将难以区分）。

然而，私有继承所提供的特性确实比包含多。例如，假设类包含保 护成员（可以是数据成员，也可以是成员函数），则这样的成员在派生 类中是可用的，但在继承层次结构外是不可用的。如果使用组合将这样 的类包含在另一个类中，则后者将不是派生类，而是位于继承层次结构 之外，因此不能访问保护成员。但通过继承得到的将是派生类，因此它 能够访问保护成员。

另一种需要使用私有继承的情况是需要重新定义虚函数。派生类可 以重新定义虚函数，但包含类不能。使用私有继承，重新定义的函数将 只能在类中使用，而不是公有的。 

### 14.2.3 保护继承

保护继承是私有继承的变体。保护继承在列出基类时使用关键字 `protected`。使用保护继承时，基类的公有成员和保护成员都将成为派生类的保护成员。

![image-20210815194347832](https://static.fungenomics.com/images/2021/08/image-20210815194347832.png)

### 14.2.4 使用using重新定义访问权限
使用保护派生或私有派生时，基类的公有成员将成为保护成员或私 有成员。假设要让基类的方法在派生类外面可用，方法之一是定义一个使用该基类方法的派生类方法。

另一种方法是，将函数调用包装在另一个函数调用中，即使用一个 using声明（就像名称空间那样）来指出派生类可以使用特定的基类成 员，即使采用的是私有派生。



## 14.3 多重继承

![image-20210815202024284](https://static.fungenomics.com/images/2021/08/image-20210815202024284.png)

![image-20210815202112888](https://static.fungenomics.com/images/2021/08/image-20210815202112888.png)

### 14.3.1 虚基类

<img src="E:/dev/Typora-Note/C++/CPP%20Primer%20Plus%20-%206/chapter14.assets/image-20230725213417381.png" alt="image-20230725213417381" style="zoom:50%;" />

虚基类是的多个类（基类相同）派生出来的对象只继承一个基类对象。类申明中使用关键字<font color="red">virtual</font>

```C++
class singer： virtual public worker{...}
class Waiter： public virtual worker{...}

class singingWaiter: public singer, public Waiter{...}
```

SingingWaiter 只包含一个Worker 对象的副本

本质就是Singer和Waiter在共享同一个Worker对象，而不会引入自己的Worker。

所以可以使用多态。

<img src="E:/dev/Typora-Note/C++/CPP%20Primer%20Plus%20-%206/chapter14.assets/image-20230725213450129.png" alt="image-20230725213450129" style="zoom:67%;" />

### 14.3.2 新的构造函数规则

如果Worker是虚基类，那么信息自动传递讲不起作用，例如下列的构造函数

```C++
SingWaiter(Const Worker & wk, int p=0, int v = singer::other): Waiter(wk,p),singer(wk,v)
{}
```

因为有两条不同的途径将wk传递给worker对象。为了避免这种冲突，C++在基类是虚的时候，禁止信息通过中间类自动传递给基类

上述构造函数将初始化singer和waiter的成员，但是wk的参数不会传递给子对象waiter。但是编译器必须在构造派生对象之前构造基类对象组件。因此在上述情况，编译器将调用基类的默认构造函数

如果不希望默认构造函数来构造虚基类对象，则需要显示调用基类的构造函数，因此构造函数是这样的

```C++
SingWaiter(Const Worker & wk, int p=0, int v = singer::other)
:	Worker（wk）， Waiter(wk,p)，singer(wk,v)
{}
```

### 14.3.3 MI小结

当派生类使用<font color="blue">Virtual</font>来指示派生的时候，基类就成了虚基类

```C++
Class marketing:public virtual reality{...}
```

<font color="red">主要变化</font>（使用虚基类的主要原因）

从虚基类的一个或者多个实例派生类而来的类将只继承一个基类对你下个，为了实现这个特性，必须满足其他要求

![img](E:/dev/Typora-Note/C++/CPP%20Primer%20Plus%20-%206/chapter14.assets/clip_image002.gif)有间接虚基类的派生类包含直接调用间接基类构造函数的构造函数，这对于间接非虚基类来说是非法的；

![img](E:/dev/Typora-Note/C++/CPP%20Primer%20Plus%20-%206/chapter14.assets/clip_image002.gif)通过优先规则解决名称二义性。

## 14.4 类模板

定义模板

```C++
temple <class Type>
    temple <typename Type>
    
    tyupedef usigned long item;
	item items[MAX];

	Type items[MAX];
```

类限定符也需要修改

```C++
bool Stack::push(const Item & item)
{
    ...
}

temple <calss Type> 	//or temple <typename Type>
bool Stack<Type>::push(const Type & item)
{
    ...
}

```

如果类声明中，声明了内联定义，则可以忽略模版前缀和类限定符

### 14.4.4 类的多功能性

1、递归使用模版

```C++
ArrayTP//是一个数组
ArrayTP<ArrayTP<int ,5> 10> twodee;
```

twodee包含10个元素的数组，每个数组是一个包含5个int 元素的数组，等同如下

```C++
int twodee[10][5];
```



## 14.4.1 模板的具体化

### 1.隐式实例化(implicit instantiation)

即声明一个或者多个对象，指出所需的类型

```C++
ArrayTP<int ,100> stuff;		//implicit instantiation
```

在编译器需要对象之前，不会生成类的隐式实例化，第二句语句导致编译器生成类定义，利用定义创建一个对象

```C++
ArrayTP<double ,30> *pt;		//a pointer no object needed by
pt = new ArrayTP<double ,30>;	//Now an object is needed
```

### 2.显式实例化（explicit instantiation）

当使用关键字<font color="blue">template</font>并指出所需类型来声明类时，编译器将生成类声明的显式实例化（explicit instantiation）。

声明必须位于模板定义所在的名称空间中。例如，下面的声明将<font color="red">ArrayTP<string, 100></font>声明为一个类：

```C++
temple class ArrayTP<string ,100>;		//generate ArrayTP<string ,100> Class
```

在这种情况下，<font color="blue">虽然没有创建或提及类对象，编译器也将生成类声明</font>（包括方法定义）。

和隐式实例化一样，也将根据通用模板来生成具体化。



### 3. 显式具体化（explicit specialization）

显式具体化是特定类型（用于替换模板中的泛型）的定义。

有时候，可能需要在为特殊类型实例化时，对模板进行修改，使其行为不同。在这种情况下，可以创建显式具体化。例如，假设已经为用于表示排序后数组的类（元素在加入时被排序）定义了一个模板：

```C++
 template<typename T>
 class SortedArray
 {
 	...			//details omitted
 }
```

另外，假设模板使用>运算符来对值进行比较。对于数字，这管用；如果T表示一种类，则只要定义了T::operator>( )方法，这也管用；但如果T是由const char *表示的字符串，这将不管用。

实际上，模板倒是可以正常工作，但字符串将按地址（按照字母顺序）排序。这要求类定义使用strcmp( )，而不是>来对值进行比较。

在这种情况下，可以提供一个显式模板具体化，这将采用为具体类型定义的模板，而不是为泛型定义的模板。当具体化模板和通用模板都与实例化请求匹配时，编译器将使用具体化版本。

具体化类模板定义的格式如下：

```C++
 template<> class Classname<Specilized-type-name>{...}
```

早期编译器使用的是

```C++
 class Classname<Specilized-type-name>{...}
```

### 4. 部分具体化

允许部分具体化，即部分限制模版通用性

### 14.4.9 模板类与友元

非模版友元

约束模版友元

非约束模版友元



####  1. 模板类的非模版友元

在模板类中将一个常规函数声明为友元：

```C++
template<class T>
Class HasFriend
{
	public:
	friend void counts();
}
```

<font color="red">counts()</font>成为了模版所有实例的友元，例如他是类HasFriend<int>和HasFriend<string>的友元。

它可以访问全局对象；可以使用全局指针访问非全局对象；可以创建自己的对象；可以访问独立于对象的模板类静态数据成员。

#### 2. 模板类的约束模版友元

可以使友元也称为模版，具体来说，就是约束模版友元做准备，来使得类的每一个具体化都获得一个与友元匹配的具体化。

需要以下三步：

首先，在类定义的前面声明每个模版函数

```C++
template<typename T> void counts();
template<typename T> void report(T &);
```

然后，再函数中再次将模版声明为友元，这些语句根据类模板参数 的类型声明具体化

```C++
template<typename TT>
class HasFriendT
{
    ...
    friend void counts<TT>();
    friend void report<>(HasFriendT<TT> &);
}
```

声明中的<font color="red"><></font>指出这是模板具体化。对于report <font color="red"><></font>可以为空，因为可以从函数参数中推断出如下模板类型参数

```C++
HasFriendT<TT>
```

然而也可以使用

```C++
report<HasFriendT<TT> >(HasFriendT<TT> &);
```

但是<font color="red">count()没有参数</font>，<font color="blue">因此必须使用<font color="red">（<TT>）</font>来指明其具体化</font>。还需要注意的是，TT是HasFriendT类的参数类型。

同样，理解这些声明的最佳方式也是设想声明一个特定具体化的对象时，它们将变成什么样。例如，假设声明了这样一个对象：

```C++
 HasFriendT<int> squack;
```

编译器将用int替换TT，并生成下面的类定义：

```C++
template<typename TT>
class HasFriendT
{
    ...
    friend void counts<int>();
    friend void report<>(HasFriendT<int> &);
}
```

基于TT的具体化将变为int，基于HasFriend<TT>的具体化将变为 HasFriend<int>。因此，模板具体化counts<int>( )和 report<HasFriendT<int> >( )被声明为HasFriendT<int>类的友元。

最后第三个要求

<font color="blue">友元提供模板定义</font>

请注意，程序清单14.22包含1个count( )函数，它是所有HasFriend类的友元；而程序清单14.23包含两个count( )函数，它们分别是某个被实例化的类类型的友元。

因为count( )函数调用没有可被编译器用来推断出所需具体化的函数参数，所以这些调用使用 count<int>和coount<double>( )指明具体化。

但对于report( )调用，编译器可以从参数类型推断出要使用的具体化。使用<>格式也能获得同样的效果：

```C++
#include <iostream>
using std::cout;
using std::endl;

// template prototypes
template <typename T> void counts();
template <typename T> void report(T &);

// template class
template <typename TT>
class HasFriendT
{
private:
    TT item;
    static int ct;
public:
    HasFriendT(const TT & i) : item(i) {ct++;}
    ~HasFriendT() { ct--; }
    friend void counts<TT>();
    friend void report<>(HasFriendT<TT> &);
};

template <typename T>
int HasFriendT<T>::ct = 0;

// template friend functions definitions
template <typename T>
void counts()
{
    cout << "template size: " << sizeof(HasFriendT<T>) << "; ";
    cout << "template counts(): " << HasFriendT<T>::ct << endl;
}

template <typename T>
void report(T & hf)
{
    cout << hf.item << endl;
}

int main()
{
    counts<int>();
    HasFriendT<int> hfi1(10);
    HasFriendT<int> hfi2(20);
    HasFriendT<double> hfdb(10.5);
    report(hfi1);  // generate report(HasFriendT<int> &)
    report(hfi2);  // generate report(HasFriendT<int> &)
    report(hfdb);  // generate report(HasFriendT<double> &)
    cout << "counts<int>() output:\n";
    counts<int>();
    cout << "counts<double>() output:\n";
    counts<double>();
    // std::cin.get();
    return 0; 
}
```

正如您看到的，counts<double>和counts<int>报告的模板大小不同，这表明每种T类型都有自己的友元函数count( )。

#### 3. 模板类的非约束模版友元

<font color="RoyalBlue">约束模板友元函数是在类外面声明的模板的具体化</font>。

int类具体化获得int函数具体化，依此类推。<font color="blue">通过在类内部声明模板，可以创建非约束友元函数</font>，即每个函数具体化都是每个类具体化的友元。

对于非约束友元，友元模板类型参数与模板类类型参数是不同的：

```C++
template<class T>
Class ManyFriend
{
	...
	template<typename C, typename D> friend void show2(C &, D &);
}
```

其中，函数调用show2（hfi1，hfi2）与下面的具体化匹配：

```C++
void show2<ManyFriend<int> &, ManyFriend<int> &>
    (ManyFriend<int> & c, ManyFriend<int> & d);
```

因为它是所有<font color="red">ManyFriend具体化的友元</font>，所以能够访问所有具体化的item成员，但它只访问了ManyFriend<int>对象。

同样，show2(hfd, hfi2)与下面具体化匹配：

```C++
void show2<ManyFriend<double> &, ManyFriend<int> &>(ManyFriend<double> & c, ManyFriend<int> & d);
```

它也是所有ManyFriend具体化的友元，并访问了ManyFriend<int>对象的item成员和ManyFriend<double>对象的item成员。

```C++
// manyfrnd.cpp -- unbound template friend to a template class
#include <iostream>
using std::cout;
using std::endl;

template <typename T>
class ManyFriend
{
private:
    T item;
public:
    ManyFriend(const T & i) : item(i) {}
    template <typename C, typename D> friend void show2(C &, D &);
};

template <typename C, typename D> void show2(C & c, D & d)
{
    cout << c.item << ", " << d.item << endl;
}

int main()
{
    ManyFriend<int> hfi1(10);
    ManyFriend<int> hfi2(20);
    ManyFriend<double> hfdb(10.5);
    cout << "hfi1, hfi2: ";
    show2(hfi1, hfi2);
    cout << "hfdb, hfi2: ";
    show2(hfdb, hfi2);
    // std::cin.get();
    return 0;
}
```

>   hfi1 ,hfi2 : 10 ,20
>
>   hfdb ,hfi2 : 10。5 ,20



### 14.4.10 模板别名

如果能为类型指定别名，将很方便，在模板设计中尤其如此。可使用typedef为模板具体化指定别名：

```C++
typedef  std::array<double, 12> arrd;
typedef  std::array<int, 12> arri;
typedef  std::array<std::string , 12> arrst;

arrd gallons;
arri days;
arrst months;
```

但如果您经常编写类似于上述typedef的代码，您可能怀疑要么自己忘记了可简化这项任务的C++功能，要么C++没有提供这样的功能。 C++11新增了一项功能——使用模板提供一系列别名，如下所示：

```C++
template<typename T>
	using arrtype = std::array<T , 12>;
//template to create multiple aliases
```

这将arrtype定义为一个模板别名，可使用它来指定类型，如下所示：

```C++
arrtype<double> gallons;
arrtype<int> days;
arrtype<std::string> months;
```

总之，arrtype<T>表示类型std::array<T, 12>。

C++11允许将语法using =用于非模板。用于非模板时，这种语法与常规typedef等价：

```C++
typedef const char * pcl;
	using pc2 = const char *;

typedef const int * (*pcl)[10];
	using pc2 = const char *(*)[10];
```

习惯这种语法后，您可能发现其可读性更强，因为它让类型名和类型信息更清晰。

C++11新增的另一项模板功能是可变参数模板（variadic template），让您能够定义这样的模板类和模板函数，即可接受可变数量的参数。这个主题将在第18章介绍。

## 14.5 总结

C++提供了几种重用代码的手段。无论使用哪种继 承，基类的公有接口都将成为派生类的内部接口。这有时候被称为继承 实现，但并不继承接口，因为派生类对象不能显式地使用基类的接口。 因此，不能将派生对象看作是一种基类对象。

还可以通过开发包含对象成员的类来重用类代码。这种方法被称为 包含、层次化或组合，它建立的也是has-a关系。另一方面，如果需要使用某个类的几个对象，则用包含更适合。

多重继承（MI）使得能够在类设计中重用多个类的代码。

类定义(<font color="red">类实例</font>)在声明类对象并指定特定类型时生成，例如：

下面的声明导致编译器生成类声明，在类声明中的实际类型short 替换模版中的<font color="blue">所有类型参数T</font>

```C++
Class IC<short> sic;
```

这里类名是IC\<short\>而不是IC。<font color="blue">IC\<short\></font>被称为<font color="red">模版具体化</font> ，具体来说 这是一个<font color="RoyalBlue">隐式实例化</font>。

使用关键字<font color="blue">template</font>声明类的特定具体化时，将发生<font color="RoyalBlue">显示实例化</font>

```C++
template Class IC<short> ;
```

编译器将使用通用模板生成一个int具体化——IC\<int\>。虽然尚<font color="red">未请求这个类的对象</font>。

 

<font color="blue">可以提供显式具体化</font>——覆盖模板定义的具体类声明。

方法是以<font color="green"> **template\<\>**</font>打头，然后是模板类名称，再加上尖括号（<font color="red">其中包含要具体化的类型</font>）。例如，为字符指针提供专用Ic类的代码如下：

```C++
template<> Class IC<char *> 
{
	char * str
	
	public:
	Ic(const char * s): str(s){}
	...
}
```

这样，下面这样的声明将为chic使用专用定义，而不是通用模板：

```C++
 Class IC<char *> chic
```

类模板可以指定多个泛型，也可以有非类型参数：

```C++
 template<Class T,Class TT, int n> 
class Pals{...};
```

下面的声明将生成一个隐式实例化，用double代替T，用string代替TT，用6代替n：

```C++
Pals<double, string , 6> mix
```

类模板还可以包含本身就是模板的参数：

```c++
template <template <template t> class CL,typename U, int z>
    class Trophy{...}
```

其中z是一个int值，U为类型名，CL为一个使用<font color="red">template\<typename,T\></font>声明的类模板。

类模板可以被部分具体化：

## 14.6 习题

#### 1、包含

```C++
class Frabjous {
    private:
    char fab[20];
    public:
    Frabjous(const char * s = "C++") : fab(s) { }
    virtual void tell() { cout << fab; }
};
class Gloam {
    private:
    int glip;
    Frabjous fb;
    public:
    Gloam(int g = 0, const char * s = "C++");
    Gloam(int g, const Frabjous & f);
    void tell();
};
```

假设Gloam版本的tell()应显示glip和fb的值，请为这3个Gloam方法提供定义。

```C++
Gloam::Gloam(int g, const char* s) : glip(g), fb(s){}
Gloam::Gloam(int g, const Frabjous &f) : glip(g), fb(f){} //使用Frabjous的默认复制构造函数
void Golam::tell()
{
    
    
    fb.tell();
 
    
    cout << glip <<endl;
}
```

#### 2、继承：

```C++
class Frabjous {
    private:
    char fab[20];
    public:
    Frabjous(const char * s = "C++") : fab(s) { }
    virtual void tell() { cout << fab; }
};
class Gloam : private Frabjous{
    private:
    int glip;
    public:
    Gloam(int g = 0, const char * s = "C++");
    Gloam(int g, const Frabjous & f);
    void tell();
};
```

假设Gloam版本的tell()应显示glip和fab的值，请为这3个Gloam方法提供定义。

```C++
Gloam::Gloam(int g, const char* s) : glip(g), Frabjous(s){}
Gloam::Gloam(int g, const Frabjous &f) : glip(g), Frabjous(f){} //使用Frabjous的默认复制构造函数
void Golam::tell()
{
    Frabjous::tell();
    cout << glip <<endl;
}
```


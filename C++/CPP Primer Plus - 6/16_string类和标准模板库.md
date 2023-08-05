# 第十六章 string类和标准模板库

本章内容包括：

- 标准C++ string类；
- 模板auto_ptr、unique_ptr 和 shared_ptr；
- 标准模板库；
- 容器类；
- 迭代器
- 函数对象；
- STL算法
- 模板initializer_list

### 16.1 String

```c++
// str1.cpp -- introducing the string class
#include <iostream>
#include <string>
// using string constructors

int main()
{
    using namespace std;
    string one("Lottery Winner!");     // ctor #1
    cout << one << endl;               // overloaded <<
    
    string two(20, '$');               // ctor #2
    cout << two << endl;
    
    string three(one);                 // ctor #3
    cout << three << endl;
    
    one += " Oops!";                   // overloaded +=
    cout << one << endl;
    two = "Sorry! That was ";
    three[0] = 'P';
    
    string four;                       // ctor #4
    four = two + three;                // overloaded +, =
    cout << four << endl;
    char alls[] = "All's well that ends well";
    
    string five(alls,20);              // ctor #5
    cout << five << "!\n";
    
    string six(alls+6, alls + 10);     // ctor #6
    cout << six  << ", ";
    
    string seven(&five[6], &five[10]); // ctor #6 again
    cout << seven << "...\n";
    
    string eight(four, 7, 16);         // ctor #7
    cout << eight << " in motion!" << endl;
    // std::cin.get();
    return 0; 
}
```

#### 16.1.1 移动构造函数

构造函数string（string && str）类似于复制构造函数，导致新创建的string为str的副本。但与复制构造函数不同的是，它不保证将str视为 const。这种构造函数被称为移动构造函数（move constructor）。在有些情况下，编译器可使用它而不是复制构造函数，以优化性能。第18章 的“移动语义和右值引用”一节将讨论这个主题。

#### 16.1.2 String 输入

对于类，很有帮助的另一点是，知道有哪些输入方式可用。对于C-风格字符串，有3种方式：

```c++
char info[100];
cin>> info;
cin.getline(info,100);
cin.get(info,100);
```

对于string对象，有两种方式：

```c++
string stuff;
cin >> stuff;
getline(cin,stuff);
```

两个版本的getline( )都有一个可选参数，用于指定使用哪个字符来确定输入的边界：

```c++
getline(info,100,":");
getline(stuff,":");
```

读取C-风格字符串的函数是istream类的方法，而string版本是独立的函数。这就是对于C-风格字符串输入，cin是调用对象；而对于string对象输入，cin是一个函数参数的原因。这种规则也适用于>>形式，如果使用函数形式来编写代码，这一点将显而易见：

```c++
cin.operator>>(fname);
operator>>(cin,lname);
```

string版本的getline( )函数从输入中读取字符，并将其存储到目标 string中，直到发生下列三种情况之一：

>   1.  到达文件尾，在这种情况下，输入流的eofbit将被设置，这意味着方法fail( )和eof( )都将返回true
>   2.  遇到分界字符（默认为\n)，在这种情况下，将把分界字符从输入流中删除，但不存储它；
>   3.  读取的字符数达到最大允许值（string::npos和可供分配的内存字节数中较小的一个），在这种情况下，将设置输入流的failbit，这意味着方法fail( )将返回true。

#### 16.1.4 Sring其他功能

如果字符串不断增大，超过了内存块的大小，程序将分配一个大小为原来两倍的新内存块，以提供足够的增大空间，避免不断地分配新的内存块。方法capacity( )返回当前分配给字符串的内存块的大小，而reserve( )方法让您能够请求内存块的最小长度

```c++
string empty;
cout<<empty.size()<<endl;		//size
cout<<empty.capacity()<<endl;	//momey size

empty.reserve(_SIZE);			//the momey size is _SIZE
```

如果您有string对象，但需要C-风格字符串，该如何办呢？例如，您可能想打开一个其名称存储在string对象中的文件

不幸的是，open( )方法要求使用一个C-风格字符串作为参数；幸运的是，<font color="red">c_str( )</font>方法返回一个指向<font color="RoyalBlue">C-风格字符串的指针</font>，该C-风格字符串的内容与用于调用c_str( )方法的string对象相同。因此可以这样做：

```c++
string filename;
ofstream fout;
fout.open(file.cstr());
```

### 16.2 智能指针模版

16.2.1 使用智能指针

这三个智能指针模板（<font color="RoyalBlue">auto_ptr、unique_ptr和shared_ptr</font>）都定义了类似指针的对象，可以将new获得（直接或间接）的地址赋给这种对 象。当智能指针过期时，其析构函数将使用delete来释放内存。因此，

如果将new返回的地址赋给这些对象，将无需记住稍后释放这些内存：

<font color="red">在智能指针过期时，这些内存将自动被释放</font>

图16.2说明了auto_ptr和常规指针在行为方面的差别；share_ptr和unique_ptr的行为与auto_ptr相同。

<img src="E:\dev\Typora-Note\C++\CPP Primer Plus - 6\assets\image-20230803163957172.png" alt="image-20230803163957172" style="zoom:50%;" />

要创建智能指针对象，必须包含头文件memory，该文件模板定义。然后使用通常的模板语法来实例化所需类型的指针。例如，模板 auto_ptr包含如下构造函数：

```c++
temple<class X> class auto_ptr{
    public:
    explicit auto_ptr(X * p = 0) throw();
    ...
}
```

请求X类型的auto_ptr将获得一个指向X类型的auto_ptr：

```c++
auto_ptr<double> pd (new double);
//pd an auto_ptr to double(use in place of double * pd)
auto_ptr<string> pd (new string);
```

<font color="red">new double</font>是new返回的指针，指向新分配的内存块

它是构造函数<font color="RoyalBlue">auto_ptr\<double\></font>的参数，即对应于原型中形参p的实参。同样，new string也是构造函数的实参。其他两种智能指针使用同样的语法：

```c++
unique_ptr<double> pd (new double);
shared_ptr<double> pd (new double);
```

编译器必须支持C++11新增的类share_ptr和unique_ptr 

<font color=#66ff66>16.5 smrtptrs.cpp</font>

```c++
// smrtptrs.cpp -- using three kinds of smart pointers
#include <iostream>
#include <string>
#include <memory>

class Report
{
    private:
    std::string str;
    public:
    Report(const std::string s) : str(s) { std::cout << "Object created!\n"; }
    ~Report() { std::cout << "Object deleted!\n"; }
    void comment() const { std::cout << str << "\n"; }
};

int main()
{
    {
        std::auto_ptr<Report> ps (new Report("using auto_ptr"));
        ps->comment();   // use -> to invoke a member function
    }
    {
        std::shared_ptr<Report> ps (new Report("using shared_ptr"));
        ps->comment();
    }
    {
        std::unique_ptr<Report> ps (new Report("using unique_ptr"));
        ps->comment();
    }
    // std::cin.get();  
    return 0;
}
```

#### 16.2.2 有关智能指针的注意事项

```c++
auto_ptr<string> ps (new string("I am auto_ptr"));
auto_ptr<string> vocation;

vocation = ps;
```

 

上述赋值语句将完成什么工作呢？如果ps和vocation是常规指针，则两个指针将指向同一个string对象。这是不能接受的，因为程序将试图删除同一个对象两次——一次是ps过期时，另一次是vocation过期 时。要避免这种问题，方法有多种。

1.  定义赋值运算符，使之<font color="red">执行深复制</font>。这样两个指针将指向不同的对象，其中的一个对象是另一个对象的副本。
2.  建立<font color="DarkOrchid ">所有权（<font color="red">ownership</font>)</font>概念，对于特定的对象，只能有一个智能指针可拥有它，这样只有拥有对象的智能指针的构造函数会删除该对象。然后，让赋值操作转让所有权。这就是用于<font color="red">auto_ptr</font>和 <font color="red">unique_ptr</font>的策略，但<font color=#4db8ff>unique_ptr的策略更严格</font>。
3.  创建智能更高的指针，<font color="DarkOrchid ">跟踪引用特定对象的智能指针数</font>。这称为<font color="red">引用计数</font>（reference counting)。例如，赋值时，计数将加1，而指针过期时，计数将减1。仅当最后一个指针过期时，才调用delete。这是<font color=#66ff66>shared_ptr</font>采用的策略。

```c++
// fowl.cpp  -- auto_ptr a poor choice
#include <iostream>
#include <string>
#include <memory>

int main()
{
    using namespace std;
    auto_ptr<string> films[5] =
    {
        auto_ptr<string> (new string("Fowl Balls")),
        auto_ptr<string> (new string("Duck Walks")),
        auto_ptr<string> (new string("Chicken Runs")),
        auto_ptr<string> (new string("Turkey Errors")),
        auto_ptr<string> (new string("Goose Eggs"))
    };
    auto_ptr<string> pwin;
    pwin = films[2];   // films[2] loses ownership

    cout << "The nominees for best avian baseball film are\n";
    for (int i = 0; i < 5; i++)
        cout << *films[i] << endl;
    cout << "The winner is " << *pwin << "!\n";
    // cin.get();
    return 0;

}

Dug
    The nominees for best avian baseball film are
        Fowl Balls
        segmentation fault(core dumped)
```

消息core dumped表明，错误地使用auto_ptr可能导致问题（这种代码的行为是不确定的，其行为可能随系统而异）。这里的问题在于，下面的语句将所有权从films[2]转让给pwin：

```c++
  pwin = films[2];   // films[2] loses ownership
```

这导致films[2]不再引用该字符串。<font color=#4db8ff>在<font color="red">auto_ptr</font>放弃对象的所有权后，便不再能能使用它来访问该对象</font>。当程序打印films[2]指向的字符串时，却发现这是一个空指针，这显然讨厌的意外.

```c++
shared_ptr<string> pwin;
pwin = films[2];
```

<font color=#66ff66>shared_ptr</font>

 这次<font color="DarkOrchid ">pwin和films[2]</font>指向同一个对象，而<font color=#4db8ff>引用计数从1增加到2</font>。在程序末尾，后声明的pwin首先调用其析构函数，该析构函数将引用计数降低到1。然后，shared_ptr数组的成员被释放，对filmsp[2]调用析构函数时，将引用计数降低到0，并释放以前分配的空间。

<font color=#66ff66>unique_ptr</font>

unique_ptr也采用所有权模型。但使用unique_ptr时，程序不会等到运行阶段崩溃，而在编译器因下述代码行出现错误：



#### 16.2.3   <font color=#66ff66>unique_ptr</font>为何优于<font color="red">auto_ptr</font>

```c++
auto_ptr<string> p1 (new string("auto"));
auto_ptr<string> p2;
p2 = p1;									//#3
```

在语句#3中，p2接管string对象的所有权后，<font color="DarkOrchid ">p1的所有权将被剥夺</font>。前面说过，这是件好事，可防止p1和p2的析构函数试图删除同一个对象；但如果程序随后试图使用p1，这将是件坏事，因为p1不再指向有效的数据。

```c++
unique_ptr<string> p3 (new string("auto"));
unique_ptr<string> p4;
p3 = p5;									//#6
```

编译器认为语句#6非法，避免了p3<font color="DarkOrchid ">不再指向有效数据</font>的问题。因此，unique_ptr比auto_ptr更安全（编译阶段错误比潜在的程序崩溃更安全）。

但有时候，将一个智能指针赋给另一个并不会留下危险的<font color="red">悬挂指针</font>。假设有如下函数定义：

```c++
unique_ptr<string> demo (const char * s)
{
    unique_ptr<string> temp(new string(s));
    return temp;
}

//编写代码
unique_ptr<string> ps;
ps = demo("Uniquely")
```

demo( )返回一个<font color=#66ff66>临时unique_ptr</font>，然后ps接管了原本归返回的 unique_ptr所有的对象，而返回的<font color=#66ff66>unique_ptr被销毁</font>。这没有问题，因为 ps拥有了string对象的所有权。但这里的另一个好处是，<font color="red">demo( )返回的临时unique_ptr很快被销毁</font>，没有机会使用它来访问无效的数据。

程序试图将一个unique_ptr赋给另一个时，如果源unique_ptr是个<font color=#66ff66>临时右值</font>，编译器允许这样做；如果源<font color="DarkOrchid ">unique_ptr将存在一段时间</font>，编译器将禁止这样做：	

```c++
using namespace std;
unique_ptr<string> pu1(new string "Hi ho!");
unique_ptr<string> pu2;
pu2 = pu1;									//#1 not allowed
unique_ptr<string> pu3;
pu3 = unique_ptr<string>(new string "Yo!");		//#2 allowed
```

语句#1将留下悬挂的<font color="DarkOrchid ">unique_ptr（pu1）</font>，这可能导致危害。

语句#2不会留下悬挂的unique_ptr，因为它调用unique_ptr的构造函数，该构造函数创建的临时对象在其所有权转让给pu后就会被销毁。

这种随情况而异的行为表明，<font color=#66ff66>unique_ptr</font>优于允许两种赋值的<font color=#66ff66>auto_ptr</font>。这也是禁止

>   （只是一种建议，编译器并不禁止）在容器对象中使用auto_ptr，但允许使用unique_ptr的原因。如果容器算法试图对包含unique_ptr的容器执行类似于语句#1的操作，将导致编译错误；如果算法试图执行类似于语句#2的操作，则不会有任何问题。而对于auto_ptr，类似于语句#1的操作可能导致不确定的行为和神秘的崩溃。

C++有一个标准库函数<font color=#4db8ff>std::move( )</font>，让您能够将一个unique_ptr赋给另一个。下面是一个使用前述demo( )函数的例子，该函数返回一个unique_ptr\<string\>对象：

```c++
unique_ptr<string>ps1 ps2;
ps1 = demo("ps1");
ps2 = move(ps1);		//enable assignemet
```

<font color="red">警告</font>

>   使用new分配内存时，才能使用auto_ptr和shared_ptr，使用new [ ]分配内存时，不能使用它 们。不使用new分配内存时，不能使用auto_ptr或shared_ptr；不使用new或new []分配内存时，不能使用unique_ptr。

------



### 16.3   标准模板库

#### 16.3.1  可对矢量执行的操作

除分配存储空间外，vector模板还可以完成哪些任务呢？所有的STL容器都提供了一些基本方法，其中包括

>   <font color=#66ff66>size( )</font>——返回容器中元素数目
>
>   <font color=#66ff66>swap( )</font>——交换两个容器的内容
>
>   <font color=#66ff66>begin( )</font>——返回一个指向容器中第一个元素的迭代器
>
>   <font color=#66ff66>end( )</font>——返回一个表示超过容器尾的迭代器。

#### 16.3.2 <font color="red">什么是迭代器？</font>

<font color=#4db8ff>它是一个广义指针。事实上，它可以是指针，也可以是一个可对其执行类似指针的操作</font>

——如解除引用（如operator*( )）和递增（如operator++( )）——的对象。

稍后将知道，通过将指针广义化为迭代器，让STL能够为各种不同的容器类（包括那些简单指针无法处理的类）提供统一的接口。每个容器类都定义了一个合适的迭代器，该迭代器的类型是一个名为<font color="red">iterator</font>的typedef，其作用域为整个类。例 如，要为vector的double类型规范声明一个迭代器，可以这样做：

假设scores是一个vector\<double\>对象：

```c++
vector<double> scores;
```

则可以使用迭代器pd执行这样的操作：

```c++
vector<double>::iterator pd;		//pad an iterator
pd = scores.degin();
*pd = 22.3;
++pd;

//还可以这样
auto pd = scores.begin();

//以及
for(pd = scores.begin(); pd! = scores.end(); pd++)
    ...
```

<font color="red">push_back( )</font>是一个方便的方法，它将元素添加到矢量末尾。这样做时，它将负责内存管理，增加矢量的长度，使之能够容纳新的成员。

```c++
scores.push_back(temp);
```

<font color="red">erase( )</font>方法删除矢量中给定区间的元素。它接受两个迭代器参数，这些参数定义了要删除的区间。

```c++
scores.erase(scores.begin(),  scores.end());
```

<font color="red">注意</font>

>   区间[it1, it2)由迭代器it1和it2指定，其范围为it1到it2（<font color=#66ff66>不包括it2</font>）。

<font color="red">insert( )</font>方法的功能与erase( )相反。它接受3个迭代器参数，第一个参数指定了新元素的插入位置，第二个和第三个迭代器参数定义了被插入区间，该区间通常是另一个容器对象的一部分。

```c++
vector<int> old_v;
vector<int> new_v;
...
old_v.insert(old_v.begin(),new_begin() + 1 , new_v.end());    
```

<img src="E:\dev\Typora-Note\C++\CPP Primer Plus - 6\assets\image-20230804155933759.png" alt="image-20230804155933759" style="zoom:50%;" />



#### 16.3.3  对矢量可执行的其他操作

##### 1 <font color="red">for_each</font>( )

<font color="red">for_each</font>( )函数可用于很多容器类，它接受3个参数。前两个是定义容器中区间的迭代器，<font color=#4db8ff>最后一个是指向函数的指针</font>（更普遍地说，最后一个参数是一个<font color="red">函数对象</font>，函数对象将稍后介绍）。

for_each()函数将被指向的函数应用于容器区间中的各个元素。<font color="red">被指向的函数不能修改容器元素的值</font>。可以用for_each( )函数来代替for循环。

```c++
for(pr = book.begin(); pr != book.end(); pr++)
{...}

for_each(book.begin(); book.end(); showReview)
```

这样可避免显式地使用迭代器变量。

##### 2 <font color="red">Random_shuffle( )</font>

Random_shuffle( )函数接受两个指定区间的迭代器参数，并<font color=#4db8ff>随机排列该区间中的元素</font>。例如，下面的语句随机排列books矢量中所有元素：

```c++
random_shuffle(book.begin(); book.end());
```

与可用于任何容器类的for_each不同，<font color=#66ff66>该函数要求容器类允许随机访问</font>，vector类可以做到这一点。

#####  3 <font color="red">sort( )</font>

sort( )函数也要求容器<font color="red">支持随机访问</font>。该函数有两个版本

第一个版本接受两个定义区间的迭代器参数，并使用为存储在容器中的类型元素定义的<运算符，对区间中的元素进行操作。例如，下面的语句按升序对<font color="red">coolstuff的内容进行排序</font>，排序时使用内置的<运算符对值进行比较：

```c++
vector<int> coolstuff;
...
    sort(coolstuff.begin(), coolstuff.end());
```

如果容器元素是用户定义的对象，则要使用sort( )，必须定义能够处理该类型对象的operator<( )函数。例如，如果为Review提供了成员或非成员函数operator<( )，则可以对包含Review对象的矢量进行排序。

由于Review是一个结构，因此其成员是公有的，这样的非成员函数将为：

```c++
bool operator<(cons Review & r1, const Review & r2)
{
    if(r1.title < r2.title)
        return true;
    else if(r1.title = r2.title && r1.rating < r2.rating)
        return true;
    else
        return false;
}
```

有了这样的函数后，就可以对包含Review对象（如books）的矢量进行排序了：

```c++
sort(books.begin(), books.end())
```

上述版本的operator<( )函数按title成员的字母顺序排序。

如果title成员相同，则按照rating排序。然而，如果想按降序或是按rating（而不是 title）排序，该如何办呢？

可以使用另一种格式的sort( )。它接受3个参数，前两个参数也是指定区间的迭代器，最后一个参数是指向要使用的函数的指针（函数对象），而不是用于比较的operator<( )。

返回值可转换为bool，false表示两个参数的顺序不正确。下面是一个例子：

```c++
bool WorseThan(const Review & r1,const Review & r2)
{
    if(r1.rating<r2.rating)
        return true;
    else
        return false;
}
```

有了这个函数后，就可以使用下面的语句将包含Review对象的books矢量按rating升序排列：

```c++
sort(books.begin(), books.end(), WorseThan)
```

与operator<( )相比，WorseThan( )函数执行的对Review对象进行排序的工作不那么完整。如果两个对象的title成员相同，operator<()函数将按rating进行排序，而WorseThan( )将它们视为相同。

第一种排序称为<font color="red">全排序（total ordering）</font>

第二种排序称为完整弱排序<font color="red">（strict weak ordering）</font>。

在全排序中，如果a<b和b<a都不成立，则a和b必定相同。在完整弱排序中，情况就不是这样了。它们可能相同，也可能只是在某方面相同，如WorseThan( )示例中的rating成员。所以在完整弱排序中，只能说它们等价，而不是相同。



#### 16.3.4 基于范围的**for**循环（**C++11**）

```c++
for_each(book.begin(); book.end(); showReview)
    //可替换为
    for(auto x :books) showReview(x);
```

根据book的类型（vector<Review>），编译器将推断出x的类型为Review，而循环将依次将books中的每个Review对象传递给ShowReview()。

不同于for_each( )，基于范围的<font color="red">for循环可修改容器的内容</font>，诀窍是指定一个<font color=#66ff66>引用参数</font>。例如，假设有如下函数：

```c++
void InflateReview(Review &r){r.rating++}
```

可使用如下循环对books的每个元素执行该函数：

```c++
    for(auto x :books) InflateReview(x);
```

### 16.4 泛型编程

#### 16.4.1 为何使用迭代器

模板使得算法独立于存储的数据类型，而迭代器使算法独立于使用的容器类型。

要实现find函数，迭代器应具备哪些特征呢？下面是一个简短的列表。

1.  应能够对迭代器执行解除引用的操作，以便能够访问它引用的值。即如果p是一个迭代器，则应对<font color="red">*p</font>进行定义。
2.  应能够将一个迭代器赋给另一个。即如果p和q都是迭代器，则应对<font color=#4db8ff>表达式p=q</font>进行定义。
3.  应能够将一个迭代器与另一个进行比较，看它们是否相等。即如果p和q都是迭代器，则应对p= =q和p!=q进行定义。
4.  应能够使用<font color=#66ff66>迭代器遍历容器中的所有元素</font>，这可以通过为迭代器p定义++p和p++来实现。

<font color="red">超尾迭代器</font> 

<font color=#66ff66>end()返回一个指向超尾位置的迭代器</font>，其中，下面的代码行将pr的类型声明为vector<double>类的迭代器

```c++
vector<double> class;
vector<double>::iterator pr;

name.end()
```

STL通过为每个类定义适当的迭代器，并以统一的风格设计类，能够对内部表示绝然不同的容器，编写相同的代码。

 使用C++11新增的自动类型推断可进一步简化：对于矢量或列表，都可使用如下代码：

```c++
for(auto pr = scores.begin(); pr!=  scores.end(); pr++)
```

最好避免直接使用迭代器，而应尽可能使用STL函数（如for_each( )）来处理细节。也可使用C++11新增的基于范围的for循环：

```c++
for_each(auto x: scores) cout<<x<<endl;
```

首先是处理容器的算法，应尽可能用通用的术语来表达算法，使之独立于数据类型和容器类型。为使通用算法能够适用于具体情况，应定义能够满足算法需求的迭代器，并把要求加到容器设计上。即基于算法的要求，设计基本迭代器的特征和容器特征。

#### 16.4.2  迭代器类型

STL定义了5种迭代器，并根据所需的迭代器类型对算法进行了描述。这5种迭代器分别是：

<font color="red">输入迭代器、输出迭代器、正向迭代器、双向迭代器和随机访问迭代器</font>。

例如，find( )的原型与下面类似：

```c++
template<class InputIterator, class T>
    InputIterator find(InputIterator first, InputIterator last);
```

这指出，这种算法需要一个输入迭代器。

同样，下面的原型指出排序算法需要一个随机访问迭代器

```c++
template<class RandomAccessIterator, class T>
    void sort(RandomAccessIterator first, RandomAccessIterator last);
```

对于这5种迭代器，都可以执行解除引用操作（即为它们定义了*运算符），也可进行比较，看其是相等（使用= =运算符，可能被重载了）还是不相等（使用!=运算符，可能被重载了）。如果两个迭代器相同，则对它们执行解除引用操作得到的值将相同。

即如果表达式iter1== iter2为真，则下述表达式也为真：

```c++
*iter1== *iter2;
```

##### 1 输入迭代器

即来自容器的信息被视为输入，就像来自键盘的信息对程序来说是输入一样。因此，<font color="red">输入迭代器可被程序用来读取容器中的信息</font>。

具体地说，对输入迭代器解除引用将使程序能够读取容器中的值，但不一定能让程序修改值。因此，需要输入迭代器的算法将不会修改容器中的值。

并不能保证输入迭代器第二次遍历容器时，顺序不变。另外，输入迭代器被递增后，也不能保证其先前的值仍然可以被解除引用。

基于输入迭代器的任何算法都应当是单通行（single-pass）的，不依赖于前一次遍历时的迭代器值，也不依赖于本次遍历中前面的迭代器值。

>   注意，输入迭代器是单向迭代器，可以递增，但不能倒退。

##### 2  输出迭代器

STL使用术语“输出”来指用于将信息从程序传输给容器的迭代器，因此<font color="red">程序的输出就是容器的输入</font>。

输出迭代器与输入迭代器相似，只是解除引用让程序能修改容器值，而不能读取。也许您会感到奇怪，能够写，却不能读。发送到显示器上的输出就是如此，cout可以修改发送到显示器的字符流，却不能读取屏幕上的内容。

STL足够通用，其容器可以表示输出设备，因此容器也可能如此。另外，如果算法不用读取作容器的内容就可以修改它（如通过生成要存储的新值），则没有理由要求它使用能够读取内容的迭代器。

<font color=#4db8ff>简而言之，对于单通行、只读算法，可以使用输入迭代器；而对于单通行、只写算法，则可以使用输出迭代器。</font>

##### 3 正向迭代器

与输入迭代器和输出迭代器相似，<font color="red">正向迭代器只使用++运算符来遍历容器</font>，所以它每次沿容器向前移动一个元素；

然而，与输入和输出迭代器不同的是，它总是按相同的顺序遍历一系列值。另外，将正向迭代器递增后，仍然可以对前面的迭代器值解除引用（如果保存了它），并可以得到相同的值。这些特征使得多次通行算法成为可能。

<font color=#4db8ff>正向迭代器既可以使得能够读取和修改数据，也可以使得只能读取数据：</font>

```c++
int * priw;			//read-write iterator
const int * pir;	//read-only iterator
```

##### 4 双向迭代器

假设算法需要能够双向遍历容器，情况将如何呢？例如，reverse函数可以交换第一个元素和最后一个元素、将指向第一个元素的指针加1,将指向第二个元素的指针减1

并重复这种处理过程。双向迭代器具有正向迭代器的所有特性，同时支持两种（前缀和后缀）递减运算符。

##### 5 随机访问迭代器

有些算法（如标准排序和二分检索）要求能够直接跳到容器中的任何一个元素，这叫做随机访问，需要随机访问迭代器。

随机访问迭代器具有双向迭代器的所有特性，同时添加了支持随机访问的操作（如指针增加运算）和用于对元素进行排序的关系运算符。

表16.3列出了除双向迭代器的操作外，随机访问迭代器还支持的操作。其中，X表示随机迭代器类型，T表示被指向的类型，a和b都是迭代器值，n为整数，r为随机迭代器变量或引用。

表**16.3** 随机访问迭代器操作

| 表  达 式 |             描 述              |
| :-------: | :----------------------------: |
|   a + n   | 指向a所指向的元素后的第n个元素 |
|   n + a   |          与a + n相同           |
|   a - n   | 指向a所指向的元素前的第n个元素 |
|  r += n   |        等价于r = r + n         |
|  r -= n   |        等价于r  = r – n        |
|   a[n]    |         等价于*(a + n)         |
|   b - a   |  结果为这样的n值，即b = a + n  |
|   a < b   |     如果b – a  > 0，则为真     |
|   a > b   |       如果b < a，则为真        |
|  a >= b   |    如果 !(  a < b)，则为真     |
|  a <= b   |    如果 !( a  > b)，则为真     |

#### 16.4.3  迭代器层次结构

| 迭代器功能       | 输  入 | 输  出 | 正  向 | 双  向 | 随  机 访 问 |
| ---------------- | ------ | ------ | ------ | ------ | ------------ |
| 解除引用读取     | 有     | 无     | 有     | 有     | 有           |
| 解除引用写入     | 无     | 有     | 有     | 有     | 有           |
| 固定和可重复排序 | 无     | 无     | 有     | 有     | 有           |
| ++i i++          | 有     | 有     | 有     | 有     | 有           |
| − −i i − −       | 无     | 无     | 无     | 有     | 有           |

| i[n]    | 无   | 无   | 无   | 无   | 有   |
| ------- | ---- | ---- | ---- | ---- | ---- |
| i + n   | 无   | 无   | 无   | 无   | 有   |
| i - n   | 无   | 无   | 无   | 无   | 有   |
| i + = n | 无   | 无   | 无   | 无   | 有   |
| i − = n | 无   | 无   | 无   | 无   | 有   |

过使用级别最低的输入迭代器，find( )函数便可用于任何包含可读取值的容器。而 sort( )函数由于需要随机访问迭代器，所以只能用于支持这种迭代器的容器。

各种迭代器的类型并不是确定的，而只是一种概念性描述。正如前面指出的，<font color="red">每个容器类都定义了一个类级typedef名称—— iterator</font>

因此vector<int>类的迭代器类型为vector<int> :: interator。

然 而，该类的文档将指出，矢量迭代器是随机访问迭代器，它允许使用基于任何迭代器类型的算法，因为随机访问迭代器具有所有迭代器的功能。

同样，list<int>类的迭代器类型为list<int> :: iterator。STL实现了一个双向链表，它使用双向迭代器，因此不能使用基于随机访问迭代器的算法，但可以使用基于要求较低的迭代器的算法。

#### 16.4.4  概念、改进和模型

正向迭代器是一系列要求，而不是类型。所设计的迭代器类可以满足这种要求，常规指针也能满足这种要求。STL算法可以使用任何满足其要求的迭代器实现。

<font color="red">STL文献使用术语概念（concept）来描述一系列的要求</font>。因此，存在输入迭代器概念、正向迭代器概念

<font color="red">概念可以具有类似继承的关系</font>

例如，双向迭代器继承了正向迭代器的功能。然而，不能将C++继承机制用于迭代器。例如：

可以将正向迭代器实现为一个类，而将双向迭代器实现为一个常规指针。因此，对 C++而言，这种双向迭代器是一种内置类型，不能从类派生而来。然 而，从概念上看，它确实能够继承。有些STL文献使用术语<font color=#4db8ff>改进（refinement）</font>来表示这种概念上的继承，因此，双向迭代器是对正向迭代器概念的一种改进。

<font color=#4db8ff>概念的具体实现被称为模型（model）</font>。因此，指向int的常规指针是一个随机访问迭代器模型，也是一个正向迭代器模型，因为它满足该概念的所有要求。

##### **1.**    指针用作迭代器

<font color="red">迭代器是广义指针，而指针满足所有的迭代器要求</font>

迭代器是STL算法的接口，而指针是迭代器，因此STL算法可以使用指针来对基于指针的非STL容器进行操作。例如，可将STL算法用于数组。假设Receipts是一个double数组，并要按升序对它进行排序：

STL sort( )函数接受指向容器第一个元素的迭代器和指向超尾的迭代器作为参数。

&Receipts[0]（或Receipts）是第一个元素的地址

&Receipts[SIZE]（或Receipts + SIZE）是数组最后一个元素后面的元素的地址。

因此，下面的函数调用对数组进行排序：

```c++
const int SIZE = 100;
double Receipts[SIZE];
sort(Receipts , Receipts + SIZE);
```

C++确保了表达式Receipts + n是被定义的，只要该表达式的结果位于数组中。因此，C++支持将超尾概念用于数组，使得可以将STL算法用于常规数组。

由于指针是迭代器，而算法是基于迭代器的，这使得可将STL算法用于常规数组。同样，可以将STL算法用于自己设计的数组形式，只要提供适当的迭代器（可以是指针，也可以是对象）和超尾指示器即可。

```c++
Copy()、ostream和istream、istream_iterator
```

STL提供了一些预定义迭代器。为了解其中的原因，这里先介绍一些背景知识。有一种算法（名为copy( )）可以将数据从一个容器复制到另一个容器中。

<font color="red">这种算法是以迭代器方式实现的</font>，所以它可以从一种容器到另一种容器进行复制，甚至可以在数组之间复制，因为可以将指向数组的指针用作迭代器。例如，下面的代码将一个数组复制到一个矢量中：

```c++
int casts[10] = {1...10};
vector<int> dice[10];
copy(casts casts + 10 , dice.begin());
```

<font color="red">最后一个迭代器参数表示要将第一个元素复制到什么位置</font>。前两个参数必须是（或最好 是）输入迭代器，最后一个参数必须是（或最好是）输出迭代器。

如果有一个表示输出流的迭代器，则可以使用copy( )。STL为这种迭代器提供了ostream_iterator模板。用STL的话说，该模板是输出迭代器概念的一个模型，它也是一个<font color="red">适配器（adapter）</font>——

一个类或函数，可以将一些其他接口转换为STL使用的接口。可以通过包含头文件iterator（以前为iterator.h）并作下面的声明来创建这种迭代器：

```c++
#include<iterator>
...
    ostream_iterator<int, char> out_iter(cout, " ");
```

out_iter迭代器现在是一个接口，让您能够使用cout来显示信息。

模板参数（int、char）指出了被发送给输出流的数据类型；

构造函数的第一个参数（这里为cout）指出了要使用的输出流，它也可以是用于文件输出的流（参见第17章）；最后一个字符串参数是在发送给输出流的每个数据项后显示的分隔符。

可以这样使用迭代器：

```c++
*out_iter++ = 15;		//works like cout<<15<< " ";
```

对于常规指针，这意味着将15赋给指针指向的位置，然后将指针加 1。

但对于该ostream_iterator，这意味着将15和由空格组成的字符串发送到cout管理的输出流中，并为下一个输出操作做好了准备。可以将copy()用于迭代器，如下所示：

```c++
copy(dic.begin(),dice.end(),out_iter);
```

这意味着将dice容器的整个区间复制到输出流中，即显示容器的内容。

也可以不创建命名的迭代器，而直接构建一个匿名迭代器。即可以这样使用适配器：

```c++
copy(dic.begin(),dice.end(),ostream_iterator<int, char>(cout, " "));
```

iterator头文件还定义了一个istream_iterator模板，使istream输入可用作迭代器接口。它是一个输入迭代器概念的模型，可以使用两个 istream_iterator对象来定义copy( )的输入范围：

```c++
copy(ostream_iterator<int, char> (cin),ostream_iterator<int, char>(), dic.begin());
```

与ostream_iterator相似，istream_iterator也使用两个模板参数。第一个参数指出要读取的数据类型，第二个参数指出输入流使用的字符类 型。

<font color=#4db8ff>使用构造函数参数cin意味着使用由cin管理的输入流，省略构造函数参数表示输入失败</font>，因此上述代码从输入流中读取，直到文件结尾、类型不匹配或出现其他输入故障为止。

##### 2 其他有用的迭代器

除了<font color=#4db8ff>ostream_iterator</font>和<font color=#4db8ff>istream_iterator</font>之外，头文件iterator还提供了其他一些专用的预定义迭代器类型。它们是<font color=#66ff66>reverse_iterator、 back_insert_iterator、front_insert_iterator和insert_iterator</font>。



------



###### 1 <font color="red">reverse -iterator</font>

对<font color=#66ff66>reverse_iterator</font>执行<font color="red">递增操作将导致它被递减</font>。为什么不直接对常规迭代器进行递减呢？主要原因是为了简化对已有的函数的使用。

vector类有一个名为<font color=#66ff66>rbegin( )和rend( )</font>的成员函数，前者返回一个指向<font color=#4db8ff>超尾的反向迭代器</font>，后者返回一个指向<font color=#4db8ff>第一个元素的反向迭代器</font>。因为对迭代器执行递增操作将导致它被递减

```c++
ostream_iterator<int, char>(cout, " "));
copy(dic.begin(),dice.end(),out_iter);
//现在假设要反向打印容器的内容
copy(dic.rbegin(),dice.rend(),out_iter);
```

<font color="red">注意</font>

>   rbegin( )和end( )返回相同的值（超尾），但类型不同（reverse_iterator和iterator）。同样
>
>   end( )和begin( )也返回相同的值（指向第一个元素的迭代器），但类型不同。

必须对反向指针做一种特殊补偿。假设rp是一个被初始化为 dice.rbegin( )的反转指针。那么*rp是什么呢？因为rbegin( )返回超尾，因此不能对该地址进行解除引用。

同样，如果rend( )是第一个元素的位 置，则copy( )必须提早一个位置停止，因为区间的结尾处不包括在区间中。

<font color="red">反向指针通过先递减，再解除引用解决了</font>这两个问题。即<font color=#66ff66>\*rp将在\*rp的当前值之前对迭代器执行解除引用</font>。也就是说

如果<font color=#66ff66>rp指向位置 6，则*rp将是位置5的值</font>，依次类推。

程序清单16.10演示了如何使用 copy( )、istream迭代器和反向迭代器。

```c++
// copyit.cpp -- copy() and iterators
#include <iostream>
#include <iterator>
#include <vector>

int main()
{
    using namespace std;

    int casts[10] = {6, 7, 2, 9 ,4 , 11, 8, 7, 10, 5};
    vector<int> dice(10);
    // copy from array to vector
    copy(casts, casts + 10, dice.begin());

    cout << "Let the dice be cast!\n";
    // create an ostream iterator
    ostream_iterator<int, char> out_iter(cout, " ");


    // copy from vector to output
    copy(dice.begin(), dice.end(), out_iter);
    cout << endl;

    cout <<"Implicit use of reverse iterator.\n";
    copy(dice.rbegin(), dice.rend(), out_iter);

    cout << endl;
    cout <<"Explicit use of reverse iterator.\n";

    // vector<int>::reverse_iterator ri;  // use if auto doesn't work
    for (auto ri = dice.rbegin(); ri != dice.rend(); ++ri)
        cout << *ri << ' ';
    cout << endl;
    // cin.get();
    return 0; 
}

//Debug
Let the dice be cast!
    6, 7, 2, 9 ,4 , 11, 8, 7, 10, 5
    Implicit    
    逆： 6, 7, 2, 9 ,4 , 11, 8, 7, 10, 5
    Explicit    
    逆： 6, 7, 2, 9 ,4 , 11, 8, 7, 10, 5
```



另外三种迭代器（<font color=#4db8ff>back_insert_iterator、front_insert_iterator和 insert_iterator</font>）也将提高STL算法的通用性。

很多STL函数都与copy( )相似，将结果发送到输出迭代器指示的位置。前面说过，下面的语句将值复制到从dice.begin( )开始的位置：

```c++
copy(casts, casts +10, dice.begin());
```

###### 2 <font color="red">三种插入迭代器</font>

通过将复制转换为插入,插入将添加新的元素，而不会覆盖已有的数据，并使用自动内存分配来确保能够容纳新的信息。

<font color="red">back_insert_iterator</font>将元素插入到<font color=#66ff66>容器尾部</font>

<font color="red">front_insert_iterator</font>将元素插入到<font color=#66ff66>容器的前端</font>

<font color="red">insert_iterator</font>将元素插入到insert_iterator构造函数的参数<font color=#66ff66>指定的位置前面</font>

这三个插入迭代器都是输出容器概念的模型。

###### <font color="DarkOrchid ">3 限制</font>

<font color="red">back_insert_iterator</font>只能用于允许在<font color=#66ff66>尾部快速插入</font>的容器（快速插入指的是一个时间固定的算法，将在本章后面的“容器概念”一节做进一步讨论)，vector类符合这种要求。

 <font color="red">front_insert_iterator</font>只能用于允许在起始位置做时间固定插入的容器类 型，vector类不能满足这种要求，但queue满足。

<font color="red">insert_iterator</font>没有这些限制，因此可以用它把信息插入到矢量的前端。然而， <font color=#4db8ff><font color="red">front_insert_iterator</font>对于那些支持它的容器来说，完成任务的速度更快。</font>

<font color=#66ff66>提示</font>

>   可以用<font color=#66ff66>insert_iterator</font>将复制数据的算法转换为插入数据的算法。

这些迭代器将容器类型作为模板参数，将<font color="red">实际的<font color=#66ff66>容器标识</font>符作为<font color=#66ff66>构造函数参数</font></font>。也就是说，要为名为dice的vector\<int\>容器，创建一个 back_insert_iterator，可以这样做：

```c++
back_insert_iterator<vector<int>> back_iter(dice);
```

必须声明容器类型的原因是，<font color="red">迭代器必须使用合适的容器方法</font>。 

copy( )是一个独立的函数，没有<font color="DarkOrchid ">重新调整容器大小的权限</font>。但前面的声明让back_iter能够使用方法<font color=#66ff66>vector\<int\>::push_back( )</font>，该方法有这样的权限。

<font color=#66ff66>front_insert_iterator</font>的方式与此相同。对于<font color=#4db8ff>insert_iterator</font>声明，还需一个<font color="red">指示插入位置</font>的构造函数参数：

```c++
 insert_iterator<vector<int>> insert_iter(dice,dice.begin());
```

程序清单16.11演示了这两种迭代器的用法，还使用for_each( )而不是ostream迭代器进行输出。

```c++

// inserts.cpp -- copy() and insert iterators
#include <iostream>
#include <string>
#include <iterator>
#include <vector>
#include <algorithm>

void output(const std::string & s) {std::cout << s << " ";}

int main()
{
    using namespace std;
    string s1[4] = {"fine", "fish", "fashion", "fate"};
    string s2[2] = {"busy", "bats"};
    string s3[2] = {"silly", "singers"};
    vector<string> words(4);
    
    
    copy(s1, s1 + 4, words.begin());
    for_each(words.begin(), words.end(), output);
    cout << endl;

// construct anonymous back_insert_iterator object
    copy(s2, s2 + 2, back_insert_iterator<vector<string> >(words));
    for_each(words.begin(), words.end(), output);
    cout << endl;

// construct anonymous insert_iterator object
    copy(s3, s3 + 2, insert_iterator<vector<string> >(words, words.begin()));
    for_each(words.begin(), words.end(), output);
    cout << endl;
    // cin.get();
    return 0; 
}
//fine fish fashion fate
//fine fish fashion fate busy bats
//silly singers fine fish fashion fate busy bats

```

第一个copy( )从s1中复制4个字符串到words中。这之所以可行，在某种程度上说是由于words被声明为能够存储4个字符串，这等于被复制的字符串数目。

然后，<font color=#4db8ff>back_insert_iterator</font>将s2中的字符串插入到words数组的末尾，将words的长度增加到6个元素。最后，<font color=#66ff66>insert_iterator</font>将s3中的两个字符串插入到words的第一个元素的前面，将words的长度增加到8个元素。如果程序试图使用words.end( )和words.begin( )作为迭代器，将s2和s3复制到words中，words将没有空间来存储新数据，程序可能会由于内存违规而异常终止。

------



#### 16.4.5 容器种类

STL具有<font color=#4db8ff>容器概念</font>和<font color=#4db8ff>容器类型</font>

概念是具有名称（如容器、序列容器、关联容器等）的通用类别；<font color="red">容器类型是可用于创建具体容器对象的模板</font>。

以前的11个容器类型分别是<font color=#4db8ff>deque、list、queue、priority_queue、 stack、vector、map、multimap、set、multiset和bitset</font>（本章不讨论 bitset，它是在比特级处理数据的容器）；

C++11新增了<font color=#4db8ff>forward_list、 unordered_map、unordered_multimap、unordered_set</font>和 <font color=#4db8ff>unordered_multiset</font>，且不将bitset视为容器，而将其视为一种独立的类 别。因为概念对类型进行了分类，下面先讨论它们。

##### **1.**    容器概念

概念描述了所有容器类都通用的元素。它是一个<font color=#66ff66>概念化的抽象基类</font>——说它概念化，是因为<font color="red">容器类并不真正使用继承机制</font>。换句话说，容器概念指定了所有STL容器类都必须满足的一系列要求。

容器是存储其他对象的对象。<font color=#66ff66>当容器过期时，存储在容器中的数据也将过期</font>（然而，如果数据是指针的话，则它指向的数据并不一定过期）。

不能将任何类型的对象存储在容器中，不能将任何类型的对象存储在容器中

C++11改进了这些概念，添加了术语<font color=#4db8ff>可复制插入（CopyInsertable）</font>和<font color=#4db8ff>可移动插入（MoveInsertable）</font>，但这里只进行简单的概述。

<font color="red"> 基本容器不能保证其元素都按特定的顺序存储，也不能保证元素的顺序不变</font>，但对概念进行改进后，则可以增加这样的保证

表16.5对一些通用特征进行了总结。其中:

X表示容器类型，如vector；

T表示存储在容器中的对象类型；

a和b表示类型为X的值；r表示类型为X&的值；

u表示类型为X的标识符（即如果X表示vector<int>，则u是一个vector<int>对象）

| 表  达 式        | 返  回 类 型      | 说 明                                                        | 复 杂度  |
| :--------------- | :---------------- | :----------------------------------------------------------- | :------- |
| X ::  iterator   | 指向T的迭代器类型 | 满足正向迭代器要求的任何迭代器                               | 编译时间 |
| X ::  value_type | T                 | T的类型                                                      | 编译时间 |
| X u;             |                   | 创建一个名为u的空容器                                        | 固定     |
| X( );            |                   | 创建一个匿名的空容器                                         | 固定     |
| X u(a);          |                   | 调用复制构造函数后u == a                                     | 线性     |
| X u = a;         |                   | 作用同X u(a);                                                | 线性     |
| r = a;           | X&                | 调用赋值运算符后r == a                                       | 线性     |
| (&a)->~X(  )     | void              | 对容器中每个元素应用析构函数                                 | 线性     |
| a.begin( )       | 迭代器            | 返回指向容器第一个元素的迭代器                               | 固定     |
| a.end(  )        | 迭代器            | 返回超尾值迭代器                                             | 固定     |
| a.size( )        | 无符号整型        | 返回元素个数，等价于a.end( )– a.begin( )                     | 固定     |
| a.swap(b)        | void              | 交换a和b的内容                                               | 固定     |
| a = = b          | 可转换为  bool    | 如果a和b的长度相同，且a中每个元素都等于（= =  为真）b中相应的元素，则为真 | 线性     |
| a != b           | 可转换为  bool    | 返回!(a= =b)                                                 | 线性     |

------

##### 2 <font color="red">C++11</font>新增的容器要求

在这个表中

rv表示类型为X的非常量右值，如函数的返回值。要求 X::iterator满足正向迭代器的要求，而以前只要求它不是输出迭代器。

<center>表 16.6 C++11 新增的基本容器要求</center>

| 表  达 式   | 返  回 类 型   | 说 明                                       | 复  杂 度 |
| :---------- | :------------- | :------------------------------------------ | :-------- |
| X u(rv);    |                | 调用移动构造函数后，u的值与rv的原始值相同   | 线性      |
| X u = rv;   |                | 作用同X u(rv);                              |           |
| a  = rv;    | X&             | 调用移动赋值运算符后，u的值与rv的原始值相同 | 线性      |
| a.cbegin( ) | const_iterator | 返回指向容器第一个元素的const迭代器         | 固定      |
| a.cend( )   | const_iterator | 返回超尾值const迭代器                       | 固定      |

复制构造和复制赋值以及移动构造和移动赋值之间的差别在于：

<font color="red">复制操作保留源对象，而移动操作可修改源对象</font>，还可能转让所有权，而不做任何复制。如果源对象是临时的，移动操作的效率将高于常规复制。第18章将更详细地介绍移动语义。



##### 3 序列



## Last

指针要访问一个顺序容器，需要把其中每一个元素遍历出来，迭代器可以按照设计要求来做到不暴露元素就能进行查找

首先迭代器是一个类模板，意味着可以不用去考虑顺序容器的数据元素的具体类型，可以更多心思放在元素查找上面。

同时又因为是类模板，可以不暴露访问顺序容器
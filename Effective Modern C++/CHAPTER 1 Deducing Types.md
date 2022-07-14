C++98有一套类型推导的规则：用于函数模板的规则。C++11修改了其中的一些规则并增加了两套规则，一套用于`auto`，一套用于`decltype`。

**这一章是每个C++程序员都应该掌握的知识。它解释了模板类型推导是如何工作的，auto是如何依赖类型推导的，以及decltype是如何按照它自己那套独特的规则工作的。它甚至解释了你该如何强制编译器使类型推导的结果可视，这能让你确认编译器的类型推导是否按照你期望的那样进行。**

# 条款一：理解模板类型推导 
* 在模板类型推导时，有引用的实参会被视为无引用，他们的引用会被忽略 
* 对于通用引用的推导，左值实参会被特殊对待 
* 对于传值类型推导，const 和/或 volatile 实参会被认为是non- const 的和non- volatile 的
* 在模板类型推导时，数组名或者函数名实参会退化为指针，除非它们被用于初始化引用

**Item 1: Understand template type deduction**
考虑函数模板如下，
```
template<typename T> 
void f(ParamType param);
```
其调用看起来是这样的
```
f(expr);         //使用表达式调用f
```
而模板就是从`expr`中推导`T`和对`ParamType`。

## 情景一：ParamType是一个指针或引用，但不是通用引用
推导遵循如下原则，和我们自己默认的原则是一致的。
1. 如果 expr 的类型是一个引用，忽略引用部分 
2. 然后 expr 的类型与 ParamType 进行模式匹配来决定 T

### 例1
```
template<typename T> 
void f(T& param);                //param是一个引用
```
我们声明下面这些变量并调用：
```
int x=27;                   //x是int 
const int cx=x;         //cx是const int 
const int& rx=x;      //rx是指向作为const int的x的引用
```
在不同的调用中，对param和T的推导类型会是这样的：
```
f(x);           //T是int，param的类型是int&
f(cx);          //T是const int，param的类型是const int&
f(rx);           //T是const int，param的类型是const int&
```

### 例2
将`f`的形参类型`T&`改为`const T&`。
```
template<typename T> 
void f(const T& param);             //param现在是reference-to-const

int x = 27;                         //如之前一样
const int cx = x;                //如之前一样
const int& rx = x;              //如之前一样

f(x);                   //T是int，param的类型是const int& 
f(cx);                  //T是int，param的类型是const int& 
f(rx);                  //T是int，param的类型是const int&
```

### 例3
`param`是一个指针或者指向const的指针。
```
template<typename T> 
void f(T* param);               //param现在是指针 

int x = 27;                         //同之前一样
const int *px = &x;          //px是指向作为const int的x的指针

f(&x);                   //T是int，param的类型是int* 
f(px);                  //T是const int，param的类型是const int*
```

## 情景二：ParamType是一个通用引用
所谓通用引用，即**如果一个函数模板形参的类型为`T&&`，并且`T`需要被推导得知，或者如果一个对象被声明为`auto&&`，这个形参或者对象就是一个通用引用。** 

推导遵循以下两个规则：
1. 如果 expr 是左值， T 和 ParamType 都会被推导为左值引用。这非常不寻常，第一，这是模板类型推导中唯一一种 T 被推导为引用的情况。第二，虽然 ParamType 被声明为右值引用类型，但是最后推导的结果是左值引用。 
2. 如果 expr 是右值，就使用正常的（也就是情景一）推导规则。

### 例子
```
template<typename T> 
void f(T&& param);                  //param现在是一个通用引用类型

int x=27;                          //如之前一样
const int cx=x;                 //如之前一样
const int & rx=cx;           //如之前一样

f(x);               //x是左值，所以T是int&，
                     //param类型也是int&

f(cx);              //cx是左值，所以T是const int&，
                      //param类型也是const int&

f(rx);              //rx是左值，所以T是const int&，
                     //param类型也是const int&

f(27);              //27是右值，所以T是int，
                      //param类型就是int&&
```
**条款24有详细说明。**

## 情景三：ParamType既不是指针也不是引用
当 ParamType 既不是指针也不是引用时，我们通过传值（pass-by-value）的方式处理：
```
template<typename T> 
void f(T param);                //以传值的方式处理param
```
这意味着无论传递什么 param 都会成为它的一份拷贝——一个完整的新对象。

推导遵循如下两个规则：
1. 和之前一样，如果 expr 的类型是一个引用，忽略这个引用部分。
2. 如果忽略 expr 的引用性（reference-ness）之后， expr 是一个 const ，那就再忽略 const 。如果它是 volatile ，也忽略 volatile （ volatile 对象不常见，它通常用于驱动程序的开发中）。
### 例1
```
int x=27;                  //如之前一样
const int cx=x;         //如之前一样
const int & rx=cx;   //如之前一样

f(x);       //T和param的类型都是int 
f(cx);      //T和param的类型都是int
f(rx);       //T和param的类型都是int
```


### 例2
```
template<typename T> 
void f(T param);            //仍然以传值的方式处理param 

const char* const ptr = "Fun with pointers";        //ptr是一个常量指针，指向常量对象

f(ptr);             //传递const char * const类型的实参
```
`ptr`自身的值会被传给形参，根据类型推导的第三条规则， `ptr`自身的常量性 `const ness`将会被省略，所以`param`是`const char*`，也就是一个可变指针指向`const`字符串。故推导出来的`T`是`const char *`，`paramType`是`const char *`。


### 例3 数组实参
**数组退化为指针的实例：**
数组类型不同于指针类型，虽然它们两个有时候是可互换的。关于这个错觉最常见的例子是，在很多上下文中，数组会退化为指向它的第一个元素的指针。
```
const char name[] = "J. P. Briggs";   //name的类型是const char[13]
const char * ptrToName = name;    //数组退化为指针
```

**模板中数组的推导**
因为数组形参会视作指针形参，所以传值给模板的一个数组类型会被推导为一个指针类型。
```
f(name);        //name是一个数组，但是T被推导为const char*
```

**模板可以接收指向数组的引用**
```
template<typename T> 
void f(T& param);               //传引用形参的模板

//我们这样调用
f(name);                            //传数组给f
```
*<u>T 被推导为了真正的数组！这个类型包括了数组的大小，在这个例子中 T 被推导为 const char[13] ， f 的形参（对这个数组的引用）的类型则为 const char (&)[13]。</u>*

**应用**
```
//在编译期间返回一个数组大小的常量值（
//数组形参没有名字， 
//因为我们只关心数组的大小）
template<typename T, std::size_t N>
constexpr std::size_t arraySize(T (&)[N]) noexcept {
    return N; 
}
```
这使得我们可以用一个花 括号声明一个数组，然后第二个数组可以使用第一个数组的大小作为它的大小。
```
int keyVals[] = { 1, 3, 7, 9, 11, 22, 35 };     //keyVals有七个元素
int mappedVals[arraySize(keyVals)];        //mappedVals也有七个
```

### 例4 函数实参
**在C++中不只是数组会退化为指针，函数类型也会退化为一个函数指针，我们对于数组类型推导的 全部讨论都可以应用到函数类型推导和退化为函数指针上来。**
```
void someFunc(int, double);     //someFunc是一个函数， 
                                                   //类型是void(int, double)

template<typename T> 
void f1(T param);               //传值给f1 

template<typename T> 
void f2(T & param);          //传引用给f2

f1(someFunc);               //param被推导为指向函数的指针， 
                                      //类型是void(*)(int, double)   
f2(someFunc);               //param被推导为指向函数的引用， 
                                      //类型是void(&)(int, double)
```
这个实际上没有什么不同，但是如果你知道数组退化为指针，你也会知道函数退化为指针。


# 条款二：理解auto类型推导
* `auto`类型推导通常和模板类型推导相同，但是`auto`类型推导假定花括号初始化代表`std::initializer_list`，而模板类型推导不这样做 
* 在C++14中`auto`允许出现在函数返回值或者lambda函数形参中，但是它的工作机制是模板类型推导那一套方案，而不是`auto`类型推导

**Item 2: Understand auto type deduction**
以`item1`来理解，当一个变量使用`auto`进行声明时，`auto`扮演了模板中`T`的角色，变量的类型说明符扮演了`ParamType`的角色。
## auto的推导规则
* 情景一：类型说明符是一个指针或引用但不是通用引用 
* 情景二：类型说明符一个通用引用 
* 情景三：类型说明符既不是指针也不是引用

情景一和情景三：
```
auto x = 27;                    //情景三（x既不是指针也不是引用）
const auto cx = x;              //情景三（cx也一样）
const auto & rx=cx;             //情景一（rx是非通用引用）
```
情景二：
```
auto&& uref1 = x;               //x是int左值，
                                              //所以uref1类型为int&
auto&& uref2 = cx;              //cx是const int左值，
                                            //所以uref2类型为const int&
auto&& uref3 = 27;              //27是int右值，
                                            //所以uref3类型为int&&
```

## auto和模板的类型推导的不同点
```
auto x1 = 27;               //类型是int，值是27 
auto x2(27);                  //同上
auto x3 = { 27 };           //类型是std::initializer_list<int>，
                                     //值是{ 27 } 
auto x4{ 27 };               //同上
```

# 条款三：理解decltype
* decltype总是不加修改的产生变量或者表达式的类型。
* 对于T类型的不是单纯的变量名的左值表达式，decltype总是产出T的引用即T&。
* C++14支持decltype(auto)，就像auto一样，推导出类型，但是它使用decltype的规则进行推导。


**Item 3: Understand decltype**

## decltype简单理解
### decltype推导规则
* 如果 exp 是一个不被括号( )包围的表达式，或者是一个类成员访问表达式，或者是一个单独的变量，那么 decltype(exp) 的类型就和 exp 一致，这是最普遍最常见的情况。
* 如果 exp 是函数调用，那么 decltype(exp) 的类型就和函数返回值的类型一致。
* 如果 exp 是一个左值，或者被括号( )包围，那么 decltype(exp) 的类型就是 exp 的引用；假设 exp 的类型为 T，那么 decltype(exp) 的类型就是 T&。
（情景一和情景三是否可能冲突？还是说，按优先级来区分？）

### decltype的简单例子
```
const int i = 0;                          //decltype(i)是const int

bool f(const Widget& w);        //decltype(w)是const Widget&
                                                  //decltype(f)是bool(const Widget&)

struct Point{
    int x,y;                    //decltype(Point::x)是int
};                                 //decltype(Point::y)是int

Widget w;                       //decltype(w)是Widget

if (f(w))…                          //decltype(f(w))是bool

template<typename T>            //std::vector的简化版本
class vector{
public:
    …
    T& operator[](std::size_t index);
    …
};

vector<int> v;                  //decltype(v)是vector<int>
…
if (v[0] == 0)…                 //decltype(v[0])是int&

```

### decltype(auto) (C++14的例子)
**尾置返回类型语法**
函数名称前面的auto不会做任何的类型推导工作。相反的，他只是暗示使用了C++11的尾置返回类型语法，即在函数形参列表后面使用一个”->“符号指出函数的返回类型，尾置返回类型的好处是我们可以在函数返回类型中使用函数形参相关的信息。
```
template<typename Container, typename Index>    //可以工作，
auto authAndAccess(Container& c, Index i)       //但是需要改良
    ->decltype(c[i])
{
    authenticateUser();
    return c[i];
}
```
改进之后的
```
template<typename Container, typename Index>    //最终的C++14版本
decltype(auto)
authAndAccess(Container&& c, Index i)
{
    authenticateUser();
    return std::forward<Container>(c)[i];
}

template<typename Container, typename Index>    //最终的C++11版本
auto
authAndAccess(Container&& c, Index i)
->decltype(std::forward<Container>(c)[i])
{
    authenticateUser();
    return std::forward<Container>(c)[i];
}

```


### decltype的特殊情况
如果一个不是单纯变量名的左值表达式的类型是`T`，那么`decltype`会把这个表 达式的类型报告为`T&`。例如，返回左值的函数总是返回左值引用。
```
decltype(auto) f1()
{
    int x = 0;
    …
    return x;                            //decltype(x）是int，所以f1返回int
}

decltype(auto) f2()
{
    int x = 0;
    return (x);                          //decltype((x))是int&，所以f2返回int&
}

```




# 条款四：学会查看类型推导结果

* 类型推断可以从IDE看出，从编译器报错看出，从Boost TypeIndex库的使用看出
* 这些工具可能既不准确也无帮助，所以理解C++类型推导规则才是最重要的


**Item 4: Know how to view deduced types**


## 编译器诊断
1.  对于模板类只声明，而不定义，可以查看推导出来的`T`和`paramType`。
```
template<typename T>    //只对TD进行声明
class TD;                               //TD == "Type Displayer"
```
![221b3b6c378eac3e21cf24b1b46c7518.png](en-resource://database/3732:1)

2. 对模板函数，定义static，出现warning报错。（不同编译器可能不一样，我没有测试过）。
 ![22cf52f9afdfef5dac4a8a9f141f9e57.png](en-resource://database/3734:1)
 
 ## 运行时输出
1. 使用typeid和std::type_info::name
```
std::cout << typeid(x).name() << '\n';  //显示x和y的类型
std::cout << typeid(y).name() << '\n';
```

2. 使用Boost.TypeIndex得到f的类型的代码。
```
#include <boost/type_index.hpp>

template<typename T>
void f(const T& param)
{
    using std::cout;
    using boost::typeindex::type_id_with_cvr;

    //显示T
    cout << "T =     "
         << type_id_with_cvr<T>().pretty_name()
         << '\n';
    
    //显示param类型
    cout << "param = "
         << type_id_with_cvr<decltype(param)>().pretty_name()
         << '\n';
}
```

???+ example "item7"
    + 括号初始化是最广泛使用的初始化语法，它防止变窄转换，并且对于C++最令人头疼的解析有天生的免疫性
    + 在构造函数重载决议中，括号初始化尽最大可能与`std::initializer_list`参数匹配，即便其他构造函数看起来是更好的选择
    + 对于数值类型的`std::vector`来说使用花括号初始化和小括号初始化会造成巨大的不同
    + 在模板类选择使用小括号初始化或使用花括号初始化创建对象是一个挑战。

### {}的引入

```cpp
struct A {
  A(int a) { std::cout << "A(int a)" << std::endl; }
  A(const A &a) { std::cout << "A(const A& a)" << std::endl; }
};
// main函数内
A a = 10; // 一次构造一次拷贝
A b(10); // 一次构造
A c = (10);  // 一次构造一次拷贝
A d{10};  // 一次构造
A e = {10};  // 一次构造
// d和e是c++11之后提出的 d和e在c++14前等价，14及之后不等价
```

1、`A a = 10;`问题：只能接收一个参数；要额外执行一次拷贝，有些类还不可拷贝

2、`A a(10);` 问题：被用做函数参数或返回值时还是会执行拷贝

3、`A a{10};`的优势：{}可以完美解决上面问题，并且不允许缩窄转换；大大简化了聚合类的初始化；{}对c++最令人头疼的解析问题天生免疫（类内初始化不可以使用2、）

```cpp
struct people {
//  private: // private就不是聚合类了，{}初始化报错
  // people() = default;
  int age;
  // int age(10); // ()不可以类内初始化 报错，()会解析成函数
  std::string name;
  float height;
};
struct A {
  A(int a) { std::cout << "A(int a)" << std::endl; }
  A(int a, int b) { std::cout << "A(int a,int b)" << std::endl; }
  A(const A &a) { std::cout << "A(const A& a)" << std::endl; }
};
// main函数内
// A f = 10, 5; // 只能接收一个参数，错误
A g(10, 5);     //一次构造
A gg(10, 5.);   // 不安全
A i = {10, 5};  // 一次构造
// A j = {10,5.}; // 报错，不允许缩窄转换
A j = {10, static_cast<int>(5.)};
fun1(A(10, 5));  // 一次构造一次拷贝
fun1({10, 5});   // 一次构造
A k = fun2();    // 一次构造两次拷贝
A m = fun3();  //一次构造一次拷贝 最后赋值给m时会拷贝一次，可以用移动构造
people aa{19, "cxk", 191.};  // 聚合类的初始化
```

### most vexing parse(最令人头疼的解析问题)

- 对象参数的创建
- 函数类型的规约

对于下面这个例子，有两种看法：

- 新建一个int类型的变量（将value强转为int），下面的int(value)并不是强制转换，而是创建一个临时对象
- 声明一个函数int i(int value)；C语言允许参数被多余的括号包裹

编译器会解释成第二种，视作一个函数声明而非()的初始化

```cpp
void f(double value) { int i(int(value)); }
```

对于下面这个例子，同样会视为一个函数声明而非()的初始化

```cpp
struct Timer {};
struct TimerKeeper {
  TimerKeeper(Timer t) {}
};
// main函数内
TimerKeeper time_keeper(Timer()); // 视为函数声明
TimerKeeper time_keeper{Timer()}; // {}初始化
```

### 聚合类的定义与不同标准的区别

C++ 11 ：

- 所有成员都是public
- 没有定义任何构造函数 但可以写出=default
- 没有类内初始化 （C++17取消）存在问题：初始化的任务交给了用户
- 没有基类，也没有virtual函数

```cpp
struct people{
  // people() = default;
  int age;
  std::string name;
  float height;
};
```

C++17：可以有基类，但必须是公有继承且必须是非虚继承（基类可以不是聚合类）

总是假设基类是一种在所有数据成员之前声明的特殊成员

```cpp
class MyStringWithIndex : public std::string {
 public:
  int index = 0;
};
// main函数内
MyStringWithIndex s{{"hello world!"}, 1};
// MyStringWithIndex s{"hello world!", 1}; 可以省略{}
std::cout << s.index << std::endl;
```

### std::array（std::array是cpp中数组的更优选择）

```cpp
template <typename T, size_t N>
struct MyArray{
    T data[N];
};
int a1[3]{1, 2, 3};  // 简单类型，原生数组
int a2[3] = {1, 2, 3};
std::array<int, 3> a3{1, 2, 3};      // 简单类型，std::array
std::array<int, 3> a4 = {1, 2, 3};   // 简单类型，std::array
S a5[3] = {{1, 2}, {3, 4}, {5, 6}};  // 复合类型，原生数组
S a6[3] = {1, 2, 3, 4, 5, 6};        // 大括号的省略
// 复合类型，std::array，编译失败！
// std::array<S, 3> a7{{1, 2}, {3, 4}, {5, 6}}; // 报错
std::array<S, 3> a7{{{1, 2}, {3, 4}, {5, 6}}};
std::array<S, 3> a8{1, 2, 3, 4, 5, 6};  // 不推荐
```

初始化复合类型的时候需要在外面再加个{}，原因是：array里面是一个原生数组，`T data[N] ={{1, 2}, {3, 4}, {5, 6}}` 是数组的初始化，而初始化这个data还需要一个{}，很合理，也可以只写最外面一个{}，编译器会自动推导，但不推荐这么写

### 如何让一个容器支持列表初始化

`std::initializer_list<T>` 《现代c++语言核心特性解析》P84 / 《C++ Primer》 P197

注意：

- **禁止隐式缩窄转换**《现代c++语言核心特性解析》P87
- 列表初始化的优先级问题（除非万不得已，否则总是优先匹配 `std::initializer_list<T>` 甚至是报错），如果没法隐式缩窄转换，就不会匹配ta
- 空的{}，不会调用 `std::initializer_list<T>` 构造，而是无参构造但是{{}} ({}) 会调用，第一个size为1，第二个size为0

```cpp
struct A {
  A(int a) { std::cout << "A(int a)" << std::endl; }
  A(int a, float b) { std::cout << "A(int a,float b)" << std::endl; }
  A(int a, std::string b) { std::cout << "A(int a,std::string b)" << std::endl; }
  A(const A &a) { std::cout << "A(const A& a)" << std::endl; }
  A(std::initializer_list<int> a) {
    std::cout << "C(std::initializer_list<int> a)" << std::endl;
    for (const int *item = a.begin(); item != a.end(); ++item) {
    }
  }
};
// main函数内
// 报错 也不会调用A(int ,float) 而是调用 A(std::initializer_list<int> a)
// A aaaa{1, 1.2};
A{1,"ssss"}; // 不会报错，会调用A(int,std::string),因为"ssss"无法隐式转换
A aaaaaaa{{}};
A bbbbbbb({});
```


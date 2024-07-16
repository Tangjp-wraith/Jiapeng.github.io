???+ example "item9"
    + `typedef`不支持模板化，但是别名声明支持。
    + 别名模板避免了使用“`::type`”后缀，而且在模板中使用`typedef`还需要在前面加上`typename`
    + C++14提供了C++11所有*type traits*转换的别名声明版本

### typename 关键字
用来澄清模板内部的一个标识符，代表的是某种类型

!!! reference
    《C++模板（第二版）》P62 《C++ Primer》P593

    当编译器遇到类似`T::xxx`这样的代码时，它不知道`xxx`是一个类型成员还是一个数据成员，直到实例化时自己才知道。
    但是为了处理模板，编译器必须知道名字是否表示一个类型，默认情况下C++会假定通过作用域运算符访问的名字不是类型

```cpp
struct test1 {
  static int SubType;
};
template <typename T>
class MyClass {
 public:
  void foo1() {
    int ptr = 10;
    T::SubType *ptr;
  }
  void foo2() { typename T::SubType *ptr; }
};

// main
MyClass<test1> class1;
class1.foo1();
// class1.foo2(); // 运行错误
```
对于foo1函数中的 `T::SubType *ptr;` *会解释成乘运算符，T::SubType 是一个数据成员

```cpp
struct test2 {
  typedef int SubType;
};
template <typename T>
class MyClass {
 public:
  void foo2() { typename T::SubType *ptr; }
};

MyClass<test2> class2;
class2.foo2();
```
对于这里的foo2函数，由于 typename 会将 T::SubType 解释成一个类型，即 int

!!! Note
    如果没有 typename 的话，SubType 会被假设成一个非类型成员（比如 static 或者一个枚举常量，亦或是内部嵌套类或 using 声明的 public 别名

### 对于模板来说，using比typedef更好用

```cpp
// using
template <class T>
using myVector = std::vector<T>;

template <typename T>
class Widget {
 public:
  myVector<T> list;
};
// typedef
template <typename T>
struct myVector2 {
  typedef std::vector<T> type;
};

template <typename T>
class Widget2 {
 public:
  typename myVector2<T>::type list;
};

template <>
class myVector2<float> {
  enum class WineType { White, Red, Rose };
  WineType type;
};
// main
myVector<int> myvec = {1, 2, 3};
myVector2<int>::type myvec2 = {4, 5,6};  // 此处不用加typename是因为T已经确定
```
显然使用using比使用typedef方便，使用using可以直接使用myvector，而使用typedef则在使用过程中需要带上::type；如上float特化所示，type在某个特化上可能不是别名，故使用::时需要在前面加上typename将其解释成一个类型成员

### 类型萃取器
用于添加或删除模板T的修饰

将const T -> T 

std::remove_const_t<T>::type (C++11)

std::remove_const_t<T> (C++14)
```cpp
template<typename _Tp>
using remove_const_t = typename remove_const<_Tp>::type;
```


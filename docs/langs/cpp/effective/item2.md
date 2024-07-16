???+ example "item2"

    + `auto`类型推导通常和模板类型推导相同，但是`auto`类型推导假定花括号初始化代表`std::initializer_list`，而模板类型推导不这样做
    + 在C++14中`auto`允许出现在函数返回值或者*lambda*函数形参中，但是它的工作机制是模板类型推导那一套方案，而不是`auto`类型推导

万能引用的第二种写法 auto&&
(推导结果详细见[item1](./item1：理解模板类型推导.md))

{}的auto类型推导
```cpp
auto x1 = 27;   // int
auto x2(27);    // int
auto x3{27};    // int
auto x4 = {27}; // initializer_list<int>
// auto x5{10, 27}; //error
// auto x6 = {10, 27, 7.}; // error:禁止隐式缩窄转换 
```

{}的模板类型推导
```cpp
template <typename T> void f(T param) {}

template <typename T> void g(std::initializer_list<T> param) {}

// main
// f({1, 2, 3}); // error：T推导不出initializer_list
g({1, 2, 3});
```

C++14中，auto可以作为函数返回值，但规则却是模板的规则
```cpp
// auto createInitList() { return {1, 2, 3}; } // error：auto相当于T，推导不出
auto createInitList() -> std::initializer_list<int> { return {1, 2, 3}; }
```
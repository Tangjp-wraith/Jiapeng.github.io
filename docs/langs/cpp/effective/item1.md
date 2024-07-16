???+ example "item1"
    + 在模板类型推导时，有引用的实参会被视为无引用，他们的引用会被忽略
    + 对于通用引用的推导，左值实参会被特殊对待
    + 对于传值类型推导，`const`和/或`volatile`实参会被认为是non-`const`的和non-`volatile`的
    + 在模板类型推导时，数组名或者函数名实参会退化为指针，除非它们被用于初始化引用 

对于下面两种写法，是一样的，T都是用来修饰param的，这个const都作为顶层const

```cpp
template <typename T>
void f(const T param);
void f(T const param);
```

下面两个fun函数不构成重载，原因是顶层const不构成重载

```cpp
void fun(int a) {}
void fun(const int a) {}
```

指针的引用：

```cpp
int *a = &b;
int *&c = a;
```

函数指针与函数引用：

- 函数指针的底层const似乎只能由类型别名表示出来
- 函数引用的底层const会被编译器自动忽略

```cpp
int fun2(int) { return 10; }
int (*fun2ptr)(int) = fun2;
int (*const fun2ptr1)(int) = fun2;  // 顶层const
// const int (*fun2ptr2)(int) = fun2;  底层const报错
int (&fun2ref)(int) = fun2;
using F = int(int);
const F *aaa = fun2;
F *const bbb = fun2;
const F &ccc = fun2;
```

---

```cpp
template <typename T> // 写法固定
void f(ParamType param);

f(expr); // 调用
```

ParamType: 

  - T; T*;
  - T&; T&&;
  - const T; const T*;
  - const T&; const T&&;
  - T* const; const T* const;

expr:

- 上面的T全换成int; 字面量;
- int[10]; int(\*)[10];
- int(&)[10]; bool(int,int);
- bool(*)(int,int); bool(&)(int,int);

!!! Note
    1、理解顶层与底层const，缺啥补啥:
    T;T*;T&;const T;const T*;const T&;T* const;const T* const;上面的T全换成int;字面量;由于顶层const不构成重载，于是 const T = T，T* const = T*，const T* const = const T*

    2、理解数组与函数能够退化成指针即可 int[10];int(\*)[10];int(&)[10];bool(int,int);bool(*)(int,int);bool(&)(int,int);

    3、通用引用（万能引用）：传入左值   ParamType为左值引用；传入右值，ParamType为右值引用   T&&;const T&&;  注意引用折叠，当int & 和 && 可折叠为int & ，万能引用只能写作 T&& ，const T&&只是右值引用


完整示例代码

```cpp
#include <iostream>

// void fun(int a) {}
// void fun(const int a) {} // 报错，顶层const不构成重载

template <typename T>
void f(T param) {
  std::cout << param << std::endl;
}

template <typename T>
void f(T *param) {
  std::cout << param << std::endl;
}
template <typename T>
void h(const T *param) {
  std::cout << param << std::endl;
}

template <typename T>
void g(T &param) {
  std::cout << param << std::endl;
}

template <typename T>
void k(const T &param) {
  std::cout << param << std::endl;
}

template <typename T>
void q(T &&param) {
  std::cout << param << std::endl;
}

template <typename T>
void p(const T &&param) {
  std::cout << param << std::endl;
}

// void f(const T param);
// void f(T const param);

void func(int, int) {}

/*
int fun2(int) { return 10; }
int (*fun2ptr)(int) = fun2;
int (*const fun2ptr1)(int) = fun2;  // 顶层const
// const int (*fun2ptr2)(int) = fun2;  底层const报错
int (&fun2ref)(int) = fun2;
using F = int(int);
const F *aaa = fun2;
F *const bbb = fun2;
const F &ccc = fun2;
*/

int main() {
  {  // void f(T param) 对应的情况
    int a = 10;
    int *aptr = &a;
    int &aref = a;
    int &&arref = std::move(a);
    f(a);      // void f<int>(int param)
    f(aptr);   // void f<int *>(int *param)
    f(aref);   // void f<int>(int param) 引用在等号右侧，忽略&
    f(arref);  // void f<int>(int param)

    const int ca = 20;
    const int *captr = &ca;
    const int &caref = ca;
    const int &&carref = std::move(ca);
    int *const acptr = &a;
    const int *const cacptr = &a;
    f(ca);      // void f<int>(int param) 顶层const
    f(captr);   // void f<const int *>(const int *param) 底层const
    f(caref);   // void f<int>(int param)
    f(carref);  // void f<int>(int param)
    f(acptr);   // void f<int *>(int *param)
    f(cacptr);  // void f<const int *>(const int *param)
    f(10);      // void f<int>(int param)

    int array[2] = {0, 1};
    f(array);  // void f<int *>(int *param) 退化
    const char aaa[12] = "hello world";
    // 数组退化成指针,顶层const变成底层const
    f(aaa);            // void f<const char *>(const char *param)
    f("hello world");  // void f<const char *>(const char *param)
    int(*arrayptr)[2] = &array;
    int(&arrayref)[2] = array;
    f(arrayptr);  // void f<int (*)[2]>(int (*param)[2])
    f(arrayref);  // void f<int *>(int *param)

    f(func);  // void f<void (*)(int, int)>(void (*param)(int, int)) 退化为指针
    void (*funcptr)(int, int) = func;
    void (&funcref)(int, int) = func;
    f(funcptr);  // void f<void (*)(int, int)>(void (*param)(int, int))
    f(funcref);  // void f<void (*)(int, int)>(void (*param)(int, int))
  }

  {  // void f(T* param) 对应的情况
    int a = 10;
    int *aptr = &a;
    int &aref = a;
    int &&arref = std::move(a);
    // f(a);      // 报错
    f(aptr);  // void f<int>(int *param)
    // f(aref);   // 报错
    // f(arref);  // 报错

    const int ca = 20;
    const int *captr = &ca;
    const int &caref = ca;
    const int &&carref = std::move(ca);
    int *const acptr = &a;
    const int *const cacptr = &a;
    // f(ca);      // 报错
    // f(captr);   // 报错
    // f(caref);   // 报错
    // f(carref);  // 报错
    f(acptr);   // void f<int>(int *param)
    f(cacptr);  // void f<const int>(const int *param)
    // f(10);      // 报错

    int array[2] = {0, 1};
    f(array);  // void f<int>(int *param) 退化
    const char aaa[12] = "hello world";
    f(aaa);            // void f<const char>(const char *param)
    f("hello world");  // void f<const char>(const char *param)
    int(*arrayptr)[2] = &array;
    int(&arrayref)[2] = array;
    f(arrayptr);  // void f<int[2]>(int *param[2])
    f(arrayref);  // void f<int>(int *param)

    f(func);  // void f<void(int, int)>(void (*param)(int, int)) 退化为指针
    void (*funcptr)(int, int) = func;
    void (&funcref)(int, int) = func;
    f(funcptr);  // void f<void(int, int)>(void (*param)(int, int))
    f(funcref);  // void f<void(int, int)>(void (*param)(int, int))
  }

  {  // void g(T& param) 对应的情况
    int a = 10;
    int *aptr = &a;
    int &aref = a;
    int &&arref = std::move(a);
    g(a);      // void g<int>(int param)
    g(aptr);   // void g<int *>(int *param)
    g(aref);   // void g<int>(int param) 引用在等号右侧，忽略&
    g(arref);  // void g<int>(int param)

    const int ca = 20;
    const int *captr = &ca;
    const int &caref = ca;
    const int &&carref = std::move(ca);
    int *const acptr = &a;
    const int *const cacptr = &a;
    g(ca);      // void g<const int>(const int &param)
    g(captr);   // void g<const int *>(const int *&param)
    g(caref);   // void g<const int>(const int &param)
    g(carref);  // void g<const int>(const int &param)
    g(acptr);   // void g<int *const>(int *const &param)
    g(cacptr);  // void g<const int *const>(const int *const &param)
    // g(10);      // 报错

    int array[2] = {0, 1};
    g(array);  // void g<int[2]>(int &param[2]) 数组的引用不发生退化
    const char aaa[12] = "hello world";
    g(aaa);            // void g<const char[12]>(const char &param[12])
    g("hello world");  // void g<const char[12]>(const char &param[12])
    int(*arrayptr)[2] = &array;
    int(&arrayref)[2] = array;
    g(arrayptr);  // void g<int (*)[2]>(int (*&param)[2])
    g(arrayref);  // void g<int[2]>(int &param[2])

    g(func);  // void g<void(int, int)>(void (&param)(int, int)) 引用不发生退化
    void (*funcptr)(int, int) = func;
    void (&funcref)(int, int) = func;
    g(funcptr);  // void g<void (*)(int, int)>(void (*&param)(int, int))
    g(funcref);  // void g<void(int, int)>(void (&param)(int, int))
  }

  {  // void h(const T* param) 对应的情况
    int a = 10;
    int *aptr = &a;
    int &aref = a;
    int &&arref = std::move(a);
    // h(a);      // 报错
    h(aptr);  // void h<int>(const int *param)
    // h(aref);   // 报错
    // h(arref);  // 报错

    const int ca = 20;
    const int *captr = &ca;
    const int &caref = ca;
    const int &&carref = std::move(ca);
    int *const acptr = &a;
    const int *const cacptr = &a;
    // h(ca);      // 报错
    h(captr);  // void h<int>(const int *param)
    // h(caref);   // 报错
    // h(carref);  // 报错
    h(acptr);   // void h<int>(const int *param)
    h(cacptr);  // void h<int>(const int *param)
    // h(10);      // 报错

    int array[2] = {0, 1};
    h(array);  // void h<int>(const int *param)
    const char aaa[12] = "hello world";
    h(aaa);            // void h<char>(const char *param)
    h("hello world");  // void h<char>(const char *param)
    int(*arrayptr)[2] = &array;
    int(&arrayref)[2] = array;
    h(arrayptr);  // void h<int[2]>(const int *param[2])
    h(arrayref);  // void h<int>(const int *param)

    // h(func);  // 报错 推导不出来底层const
    void (*funcptr)(int, int) = func;
    void (&funcref)(int, int) = func;
    // h(funcptr);  //  报错
    // h(funcref);  //  报错
  }

  {           // void h(const T& param) 可以接任何东西
    k(func);  // void k<void(int, int)>(void (&param)(int, int)) &被忽略
  }

  {  // void q(T&& param) 引用折叠 万能引用
    int a = 10;
    int *aptr = &a;
    int &aref = a;
    int &&arref = std::move(a);
    q(a);      // void q<int &>(int &param)
    q(aptr);   // void q<int *&>(int *&param)
    q(aref);   // void q<int &>(int &param)
    q(arref);  // void q<int &>(int &param)
    const int ca = 20;
    const int *captr = &ca;
    const int &caref = ca;
    const int &&carref = std::move(ca);
    int *const acptr = &a;
    const int *const cacptr = &a;
    q(ca);     // void q<const int &>(const int &param)
    q(captr);  // void q<const int *&>(const int *&param)
    q(caref);  // void q<const int &>(const int &param)
    q(carref);  // void q<const int &>(const int &param) 右值引用仍然是左值
    q(acptr);   // void q<int *const &>(int *const &param)
    q(cacptr);  // void q<const int *const &>(const int *const &param)
    q(10);      // void q<int>(int &&param)
    // 数组函数同理
  }

  {  // void p(const T&& param) 只是右值引用
    int a = 10;
    // p(a); // 报错,只能传入右值
    p(10);
    // 其余全部报错
  }

  return 0;
}
```
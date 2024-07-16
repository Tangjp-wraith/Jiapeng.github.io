???+ example "item23"
    + `std::move`执行到右值的无条件的转换，但就自身而言，它不移动任何东西。
    + `std::forward`只有当它的参数被绑定到一个右值时，才将参数转换为右值。
    + `std::move`和`std::forward`在运行期什么也不做。

### std::move实现

```cpp
// 有问题的实现版本
template <typename T>
T &&mymove(T &&param) {
  return static_cast<T&&>(param);
}
// main
int m = 10;
// int &&n = mymove(m); //error
```
上述mymove实现的问题在于当mymove接收一个左值时，模板类型会被推导为一个左值引用，接着T&&会发生引用折叠，进而T&&也会被推导为左值引用，即`int &mymove<int &>(int &param)`，这样用一个右值接收就会出错，因此在接收这个类型T时，需要将引用删除，这就用到了[item9](./item9.md)中提及的类型萃取器，正确的实现如下：
```cpp
template <typename T>
typename std::remove_reference<T>::type &&move(T &&param)  // c++11
{
  using ReturnType = typename std::remove_reference<T>::type &&;
  return static_cast<ReturnType>(param);
}
template <typename T>
decltype(auto) move2(T &&param)  // C++14
{
  using ReturnType = std::remove_reference_t<T> &&;
  return static_cast<ReturnType>(param);
}
```

### std::move

std::move实际做的事情是将实参转化为右值（有些提议说它的名字叫rvalue_cost更好）

对一个对象使用std::move就是告诉编译器这个对象很适合被移动

std::move本质：总是将任何值转化为右值

???+Warning "对于const右值引用，只能匹配上拷贝构造"
    ```cpp
    struct A {
    A(int value) : b(value) { std::cout << "create" << std::endl; }
    A(const A &value) {
        std::cout << "copy" << std::endl;
        b = value.b;
    }
    A(A &&value) {
        std::cout << "move" << std::endl;
        b = value.b;
        value.b = 0;
    }

    int b;
    };

    class Annotation {
    public:
    explicit Annotation(const A a) : a_param(std::move(a)) {}

    private:
    A a_param;
    };

    int main() {
    Annotation aa{10};
    
    const A cc{12};
    // A &&m = std::move(cc); //error
    }
    ```
    `Annotation aa{10};`只会调用拷贝构造，即使在实现中使用了std::move，这是由于10是一个右值，因此当aa{10}被初始化后，const aa则被解释为了一个const的右值引用，这与移动构造中的右值引用并不匹配，而拷贝构造中的const &可以匹配一切，进而就会调用拷贝构造

### 初探std::forward

```cpp
void process(const A &lvalArg) { std::cout << "deal lvalArg" << std::endl; }
void process(A &&rvalArg) { std::cout << "deal rvalArg" << std::endl; }

template <typename T>
void logAndProcess(T &&param) {
  // process(param);
  // process(std::move(param));
  process(std::forward<T>(param));  // 实参用右值初始化时，转换为一个右值
}

int main() {
  A m{10};
  logAndProcess(m);
  logAndProcess(std::move(m));
}
```
无论logAndProcess传入的参数是m还是std::move(m)，logAndProcess中接收到的param最终都会变成左值，因此当我们希望将param左右值信息保留并传递的时候，就会用到std::forward

`std::forward<T>`的本质：有条件的move，只有当实参用右值初始化才转化为右值
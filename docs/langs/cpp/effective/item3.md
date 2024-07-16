???+ example "item3"
    + `decltype`总是不加修改的产生变量或者表达式的类型。
    + 对于`T`类型的不是单纯的变量名的左值表达式，`decltype`总是产出`T`的引用即`T&`。
    + C++14支持`decltype(auto)`，就像`auto`一样，推导出类型，但是它使用`decltype`的规则进行推导。

### decltype + 变量

所有信息都会被保留，数组与函数也不会退化

```cpp
int a = 10;
decltype(a) b; // b -> int
int &&arref = std::move(a);
decltype(arref) ref; // ref -> int &&
const int ca = 20;
const int &&carref = std::move(ca);
decltype(carref) cb;  // cb -> const int &&
int array[2] = {0, 1};
decltype(array) arr; // arr -> int[2]
```

### decltype + 表达式

会返回表达式结果对应的类型

表达式结果不是左值就是右值

+ 左值：得到该类型的左值引用
+ 右值：得到该类型

```cpp
int *const acptr = &a;
const int *const cacptr = &a;
decltype(*cacptr) cc;  // cc -> const int &
10;
decltype(10) aa; // aa -> int

int value = 10;
int *ptr = &value;
decltype(*ptr) ref = value;  // ref -> int &
decltype(ptr) val;           // val -> int *
decltype((value)) ref2 = value;  // ref2 ->int &
decltype((((value)))) ref3 = value;  // ref3 ->int &
decltype(value + 10) value2 = value;  // value2 -> int

const int cvalue = 20;
const int *cptr = &cvalue;
int *const ptrc = &value;
decltype(*cptr) cref = cvalue;  // cref -> const int &
decltype(*ptrc) refc = value;   // refc -> int &
decltype((cvalue + 10)) value3 = value;  // value3 -> int
```

`int *ptr = &value;` 中 *ptr 是左值，因此推导得到的是左值引用，即int &

但是 `decltype(ptr) val; ` val 的类型确实int \*，而ptr也是左值，这是因为：**decltype单独作用于对象，没有使用对象的表达式属性，而是直接获得变量类型，想要将变量作为表达式，可以加括号**

**decltype 并不会实际计算表达式的值，编译器会分析表达式并得到类型**

对于下面这个例子中，并不会执行fun函数打印，decltype会分析得到fun返回类型为int，然后推导value4为int
```cpp
int fun(int x) {
  std::cout << "fun" << std::endl;
  return x;
}
// main
int value = 10;
decltype(fun(value)) value4;
```

### decltype使用场景

某些情况模板函数的返回值是无法提前得到的

对于std::vector<>的对象，使用operator[]通常会返回T&，但是对于std::vector< bool > 却是一个例外，因此此时的模板函数的返回值必须是自动推导

C++ 11 可以使用尾置返回类型语法 auto + decltype(xxx)

C++ 14 可以直接使用auto，但是auto作为函数返回值走的是模板类型推导，所有可以使用decltype(auto)来保留xxx的所有修饰

```cpp
struct Point {
  int x, y;
  Point &operator=(Point p) {
    std::cout << "operator=" << std::endl;
    return *this;
  }
};

template <typename Container, typename Index>  // 可以工作，
auto testFun(Container &c, Index i)            // 但是需要改良
    -> decltype(c[i])  // c++11的写法 尾置返回类型语法
{
  // do something...
  return c[i];
}

template <typename Container, typename Index>  // C++14版本，
auto testFun_error(
    Container &c,
    Index i)  // 不那么正确 因为auto作为函数返回值的时候走的是模板推导的套路
{
  // do something...
  return c[i];  // 从c[i]中推导返回类型
}

template <typename Container, typename Index>  // C++14版本，
decltype(auto) testFun_right(Container &c, Index i)  // 可以工作，但是还需要改良
{
  // do something...
  return c[i];
}

template <typename Container, typename Index>
decltype(auto) testFun_better(Container &&c, Index i) {
  // do something...
  // return c[i];
  return std::forward<Container>(c)[i];
}

int main() {
  int value = 10;
  Point point{1, 2};
  std::vector<bool> vec = {true, false, true};
  decltype(vec[1]) value6;  // value6 -> std::vector<bool>::reference
  std::vector<int> vec2 = {1, 2, 3};
  decltype(vec2[1]) value7 = value;  // value7 -> int &
  std::vector<Point> vec3 = {{1, 2}, {3, 4}, {2, 3}};
  decltype(vec3[1]) value8 = point; // value8 -> Point &

  testFun(vec, 1) = true;
  testFun(vec2, 1) = 10;
  testFun(vec3, 1) = {4, 8};

  testFun_error(vec, 1) = true;
  // testFun_error(vec2, 1) = 10;  // error testFun_error(vec2, 1) 是右值
  testFun_error(vec3, 1) =
      point;  // testFun_error(vec3, 1) 也是右值，但是他在执行 Point &operator=(Point p) 函数

  testFun_right(vec, 1) = true;
  testFun_right(vec2, 1) = 20;
  testFun_right(vec3, 1) = {5, 15};
}
```

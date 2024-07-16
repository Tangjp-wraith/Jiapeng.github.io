!!! Reference
      《C++ Primer》P58：当执行对象拷贝操作时，常量的顶层const不受什么影响，而底层const必须一致

```cpp
  const int a = 10;
  int b = a;  // 拷贝a的值，顶层const加不加不受影响
  const int *const p = new int(10);
  // int *p1 = p;        // error 底层const必须一致
  // int *const p2 = p;  // error 错误同上
  const int *p3 = p;  // 顶层const可以忽略，但底层必须一致加上了第一个const
  // int *p4 = &a;  // error a虽然是顶层const，&a变成了底层const 注意！！！
  const int *p4 = &a;
  const int &r1 = a;  //常量引用如果在左侧，右侧可以接任何东西
  const int &rr1 = b;  //常量引用如果在左侧，右侧可以接任何东西
  const int &rr2 = 40;  //常量引用如果在左侧，右侧可以接任何东西
  // int &r2 = a;          // error 非常量引用 = 常量
  // int &r3 = r1;  // 引用如果在等号右侧，请忽略& ，错误同上
  int c = a;  // 等于int c = a; 非常量 = 常量引用
```

对于  `const int *const p = new int(10)`; 第一个const为底层const，第二个const为顶层const

(const) p ---> 10(const) , 若pa--->10(const) 显然，如果语句编译通过那*pa=11就可以执行，pa可以直接修改这个10，这是不对的，`int *pa = p`是错误的

而对于 `int *const p = new int(10);`来说，(const) p ---> 10,那么pa--->10是可以的，于是 `int *pa = p`是可以通过的

!!! Note
    关于引用&总结如下几点：

    - **引用不是对象且不进行拷贝，不满足上面的原则**
    - **常量引用如果在左侧，右侧可以接任何东西**
    - **非常量引用 = 常量 （报错！！！）**
    - **引用如果在等号右侧，请忽略&**
    - **非常量 = 常量引用 （正确）**
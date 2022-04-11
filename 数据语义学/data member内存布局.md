# 0 引言

在阅读本篇文章之前，此处建议可以先阅读如下两篇文章

[虚拟继承1](https://github.com/qls152/DeepUnderstandingGcc-Clang-CplusplusObjectModel/blob/main/%E6%95%B0%E6%8D%AE%E8%AF%AD%E4%B9%89%E5%AD%A6/%E8%99%9A%E6%8B%9F%E7%BB%A7%E6%89%BF(1).md)

[虚拟继承2](https://github.com/qls152/DeepUnderstandingGcc-Clang-CplusplusObjectModel/blob/main/%E6%95%B0%E6%8D%AE%E8%AF%AD%E4%B9%89%E5%AD%A6/%E8%99%9A%E6%8B%9F%E7%BB%A7%E6%89%BF(2).md)

**本篇主要讲解非继承非多态场景下 对象的数据成员的内存布局以及不同场景下，成员变量的初始化顺序。**

# 1 由Point3d类说起

本文是基于[深度探索C++对象模型](https://book.douban.com/subject/1091086/)第三章相应小节 data member内存布局 ，因此这里也采用相应的例子Point3d，其定义大致如下

```c++
#include <iostream>
#include <list>

class Point3d {
public:
  Point3d() = default;

private:
  int64_t x_{0};
  static Point3d* free_list;
  int64_t y_{0};
  static const int chunkSize{0};
  int64_t z_{0};
};
Point3d* Point3d::free_list{};

int main() {
  Point3d p;
  return 0;
}
```

在上述情景下，通过[C++ Insights](https://cppinsights.io/) 知 Point3d p 扩展为如下语句

```c++
Point3d p = Point3d();
```

通过[Compiler Explorer](https://godbolt.org/) 可知 相应的汇编实现如下

```c++
        movq    $0, -32(%rbp)
        movq    $0, -24(%rbp)
        movq    $0, -16(%rbp)
```

通过gdb确认Point3d的data member内存布局如下

```c++
(gdb) p p
$2 = {
  x_ = 0,
  y_ = 93824992235680,
  static chunkSize = 0,
  z_ = 140737488347488,
  static free_list = 0x0
}
```

通过gdb确认Point3d的大小为

```c++
(gdb) p sizeof(p)
$4 = 24
```

故综上可以得到Point3的数据成员内存布局 正如[深度探索C++对象模型](https://book.douban.com/subject/1091086/)中所描述

> Nonstatic data members 在类对象中的排列顺序和其被声明顺序一样，任何中间介入的
> static data members 都不会被放进内存布局中。
> static data members存放在程序的data segment中。

**同时，通过汇编部分可知，gcc针对类成员默认初始化的顺序与其声明顺序相反。即gcc会先将z_设置成为0，然后是y_, 最后是x_。而不是按照声明顺序进行初始化。并且gcc也会进行一定优化，省略了构造函数！**

# 2 x_成员为显示初始化

也即如下代码所示

```c++
class Point3d {
public:
  Point3d() = default;

private:
  int64_t x_;
  static Point3d* free_list;
  int64_t y_{0};
  static const int chunkSize{0};
  int64_t z_{0};
};
```

此时，通过[Compiler Explorer](https://godbolt.org/) 可知其被转换成如下实现

```c++
Point3d::Point3d() [base object constructor]:
        pushq   %rbp
        movq    %rsp, %rbp
        movq    %rdi, -8(%rbp)
        movq    -8(%rbp), %rax
        movq    $0, 8(%rax) // 初始化变量Z
        movq    -8(%rbp), %rax
        movq    $0, 16(%rax) // 初始化变量y
        nop
        popq    %rbp
        ret
main:
        pushq   %rbp
        movq    %rsp, %rbp
        subq    $32, %rsp
        leaq    -32(%rbp), %rax
        movq    %rax, %rdi
        call    Point3d::Point3d() [complete object constructor]
```

**可知，此时gcc编译器生成了相应的构造函数，且针对y_，z_的初始化依然与声明顺序相反！**

# 3 成员初始化列表

若将Point3d的例子，改为成员初始化列表的形式，即如下代码

```c++
#include <iostream>
#include <list>

class Point3d {
public:
  Point3d(int x, int y, int z) : x_(x), y_(y), z_(z) {}
  Point3d() = default;

private:
  int64_t x_;
  static Point3d* free_list;
  int64_t y_{0};
  static const int chunkSize{0};
  int64_t z_{0};
};
Point3d* Point3d::free_list{};

int main() {
  Point3d p(1,1,1);
  return 0;
}
```

此时，通过[Compiler Explorer](https://godbolt.org/) 可知其构造函数和相应的初始化如下

```c++
Point3d::Point3d(int, int, int) [base object constructor]:
        pushq   %rbp
        movq    %rsp, %rbp
        movq    %rdi, -8(%rbp)
        movl    %esi, -12(%rbp) // x
        movl    %edx, -16(%rbp) // y
        movl    %ecx, -20(%rbp) // z
        movl    -12(%rbp), %eax // x
        movslq  %eax, %rdx
        movq    -8(%rbp), %rax
        movq    %rdx, (%rax)
        movl    -16(%rbp), %eax // 初始化y
        movslq  %eax, %rdx
        movq    -8(%rbp), %rax
        movq    %rdx, 8(%rax)
        movl    -20(%rbp), %eax // 初始化z
        movslq  %eax, %rdx
        movq    -8(%rbp), %rax
        movq    %rdx, 16(%rax)
        nop
        popq    %rbp
        ret
Point3d::free_list:
        .zero   8
main:
         .....
        subq    $32, %rsp
        leaq    -32(%rbp), %rax
        movl    $1, %ecx // 从右至左入参，此为z
        movl    $1, %edx // 此为y
        movl    $1, %esi // 此为x
        movq    %rax, %rdi
        call    Point3d::Point3d(int, int, int) [complete object constructor]  
```

故由上述可知，此时，gcc针对Point3d中data member的初始化顺序与声明顺序相同！

# 4 总结

通过该部分分析，可以总结成如下几条

- Nonstatic data members 在类对象中的排列顺序和其被声明顺序一样，任何中间介入的static data members都不会被放进内存布局中。static data members存放在程序的data segment中。

- 若所有成员均显示初始化，则其初始化的顺序与声明顺序相反

- 若声明了显示构造函数用以初始化成员变量，则其初始化顺序与声明顺序一致








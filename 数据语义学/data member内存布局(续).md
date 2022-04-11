# 0 引言

本文是[data member内存布局](https://github.com/qls152/DeepUnderstandingGcc-Clang-CplusplusObjectModel/blob/main/%E6%95%B0%E6%8D%AE%E8%AF%AD%E4%B9%89%E5%AD%A6/data%20member%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80.md)的续，上篇文章，我未考虑Point3d拥有virtual函数的例子，此篇文章补上该部分。

# 1 由Point3d说起

承接上文，更改相应的Point3d，现在的实现如下

```c++
#include <iostream>
#include <list>

class Point3d {
public:
  // Point3d(int x, int y, int z) 
  //   : x_(x), y_(y), z_(z) {}
  Point3d() = default;
  virtual void print() {
    std::cout << "hello world!\n";
  }

private:
  int64_t x_{0};
  
  int64_t y_{0};
  static const int chunkSize{0};
  int64_t z_{0};
  static Point3d* free_list;
};
Point3d* Point3d::free_list{};

int main() {
  Point3d p;
  return 0;
}

```

通过gdb观察相应的内存布局如下

```c++
{
  _vptr.Point3d = 0x7ffff7d98fc8 <__exit_funcs_lock>,
  x_ = 93824992236176,
  y_ = 0,
  static chunkSize = 0,
  z_ = 93824992235712,
--Type <RET> for more, q to quit, c to continue without paging--
  static free_list = 0x0
}
```

**由上述可知，针对含有虚函数的类Point3d, 在其内存布局中，会产生一个vptr指针，且该vptr位于内存布局的开始位置。 也即针对Gcc编译器而言，vptr会放在class object的最前端。** 此处也验证了深度探索C++对象模型 中如下陈述

> 编译器可能会使用一些内部使用的data members，以支持整个对象模型，vptr就是这样的东
> 西，当前所有的编译器都把它安插在一个内含 virtual function的class 的object内。
> vptr会放在什么位置呢？传统上它被放在所有明确声明的members的最后。不过如今也有一些
> 编译器把vptr放在一个class object的最前端。

由此可知，**目前的gcc/clang便是 如今的一些编译器！**

# 2 无显示构造函数

该部分，主要深入汇编层面，看看gcc，针对含有虚函数的类是如何构造和实现的。

上述所展示的例子，通过[Compiler Explorer](https://godbolt.org/)，关于Point3d p; 的实现在汇编层面如下

```c++
       movl    $vtable for Point3d+16, %eax // 初始化vptr
        movq    %rax, -32(%rbp)
        movq    $0, -24(%rbp)
        movq    $0, -16(%rbp)
        movq    $0, -8(%rbp)

 vtable for Point3d:
        .quad   0
        .quad   typeinfo for Point3d
        .quad   Point3d::print()
```

由上述汇编语句可知，在成员变量显示初始化的场景下，gcc会进行优化，省略构造函数！ 但依然会生成Point3d的vtbale。

因此Point3d的内存布局如下

![Point3d内存布局](https://pic4.zhimg.com/80/v2-05178ed85a31c1248f35bdd12e134043_1440w.jpg)

# 3 显示构造函数

这部分实现可参考上篇文章，本部分为方便理解，将代码粘贴如下

```c++
#include <iostream>
#include <list>

class Point3d {
public:
  Point3d(int x, int y, int z) : x_(x), y_(y), z_(z) {}
  Point3d() = default;
  virtual void print() {
    std::cout << "hello world!\n";
  }

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

我们通过[Compiler Explorer - C++ (x86-64 gcc (trunk))](https://godbolt.org/clientstate/eyJzZXNzaW9ucyI6W3siaWQiOjEsImxhbmd1YWdlIjoiYysrIiwic291cmNlIjoiI2luY2x1ZGUgPGlvc3RyZWFtPlxuI2luY2x1ZGUgPGxpc3Q+XG5cbmNsYXNzIFBvaW50M2Qge1xucHVibGljOlxuICBQb2ludDNkKGludCB4LCBpbnQgeSwgaW50IHopIDogeF8oeCksIHlfKHkpLCB6Xyh6KSB7fVxuICBQb2ludDNkKCkgPSBkZWZhdWx0O1xuICB2aXJ0dWFsIHZvaWQgcHJpbnQoKSB7XG4gICAgc3RkOjpjb3V0IDw8IFwiaGVsbG8gd29ybGQhXFxuXCI7XG4gIH1cblxucHJpdmF0ZTpcbiAgaW50NjRfdCB4XztcbiAgc3RhdGljIFBvaW50M2QqIGZyZWVfbGlzdDtcbiAgaW50NjRfdCB5X3swfTtcbiAgc3RhdGljIGNvbnN0IGludCBjaHVua1NpemV7MH07XG4gIGludDY0X3Qgel97MH07XG59O1xuUG9pbnQzZCogUG9pbnQzZDo6ZnJlZV9saXN0e307XG5cbmludCBtYWluKCkge1xuICBQb2ludDNkIHAoMSwxLDEpO1xuICByZXR1cm4gMDtcbn0iLCJjb21waWxlcnMiOlt7ImlkIjoiZ3NuYXBzaG90Iiwib3B0aW9ucyI6Ii1zdGQ9YysrMTEifV19XX0=) 观察其gcc实现，相应汇编代码如下

```c++
Point3d::Point3d(int, int, int) [base object constructor]:
        pushq   %rbp
        movq    %rsp, %rbp
        movq    %rdi, -8(%rbp)
        movl    %esi, -12(%rbp) // x_
        movl    %edx, -16(%rbp) // y_
        movl    %ecx, -20(%rbp) // z_
        movl    $vtable for Point3d+16, %edx // 初始化vptr
        movq    -8(%rbp), %rax
        movq    %rdx, (%rax) // 初始化vptr
        movl    -12(%rbp), %eax // 初始化x_
        movslq  %eax, %rdx
        movq    -8(%rbp), %rax
        movq    %rdx, 8(%rax)
        movl    -16(%rbp), %eax // 初始化y_
        movslq  %eax, %rdx
        movq    -8(%rbp), %rax
        movq    %rdx, 16(%rax)
        movl    -20(%rbp), %eax // 初始化z_
        movslq  %eax, %rdx
        movq    -8(%rbp), %rax
        movq    %rdx, 24(%rax)
        nop
        popq    %rbp
        ret

main:
        pushq   %rbp
        movq    %rsp, %rbp
        subq    $32, %rsp
        leaq    -32(%rbp), %rax
        movl    $1, %ecx
        movl    $1, %edx
        movl    $1, %esi
        movq    %rax, %rdi
        call    Point3d::Point3d(int, int, int) [complete object constructor] // 调用相应的构造函数
```

由上述汇编实现，可总结出如下结论：

- gcc针对含有虚函数的类且该类具有显示构造函数，不会进行优化，会生成相应的complete 
  object constructor 和 base object constructor

- gcc会在类Point3d中安插一个vptr，且该vptr在所有参数初始化列表之前被初始化

关于vtable和base object constructor相关内容，可参考[虚拟继承1](https://github.com/qls152/DeepUnderstandingGcc-Clang-CplusplusObjectModel/blob/main/%E6%95%B0%E6%8D%AE%E8%AF%AD%E4%B9%89%E5%AD%A6/%E8%99%9A%E6%8B%9F%E7%BB%A7%E6%89%BF(1).md)

# 4 总结

通过该篇文章，便可将c++对象模型中，data members内存布局基本理解了。

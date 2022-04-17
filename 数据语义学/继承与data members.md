# 0 引言

本文将深入讲解[深度探索C++对象模型](https://book.douban.com/subject/1091086/) 第三章中 继承与Data member部分，本部分将分四篇讲解。依然会深入汇编层面，并最后给出Itanium C++ ABI (Revision: 1.83)中相应C++对象布局的约定。

本文先从最简单的场景：**只要继承不要多态说起！**

在开始本文之前，需要引入如下术语：

> the size of an object, sizeof(O); // sizeof(O), 对象O的大小

> the alignment of an object, align(O); // 对象的对其要求

其结果和C++标准的alignof一致，譬如cpprefrence中

```c++
#include <iostream>
 
struct Foo {
    int   i;
    float f;
    char  c;
};
 
// Note: `alignas(alignof(long double))` below can be simplified to simply 
// `alignas(long double)` if desired.
struct alignas(alignof(long double)) Foo2 {
    // put your definition here
}; 
 
struct Empty {};
 
struct alignas(64) Empty64 {};
 
int main()
{
    std::cout << "Alignment of"  "\n"
        "- char             : " << alignof(char)    << "\n"
        "- pointer          : " << alignof(int*)    << "\n"
        "- class Foo        : " << alignof(Foo)     << "\n"
        "- class Foo2       : " << alignof(Foo2)    << "\n"
        "- empty class      : " << alignof(Empty)   << "\n"
        "- alignas(64) Empty: " << alignof(Empty64) << "\n";
}

```
其结果如下

```c++
Alignment of
- char             : 1
- pointer          : 8
- class Foo        : 4
- class Foo2       : 16
- empty class      : 1
- alignas(64) Empty: 64
```

> the offset within O, offset(C), of each data component C, i.e. base
> or member. // offset(C)也即对象C在类O中的偏移量

其结果同offsetof一致。

> dsize(O): the data size of an object, which is the size of O without
> tail padding.

**dsize(O)指的是对象O的数据大小，不包括尾部padding（sizeof(O)会包括)**

> nvsize(O): the non-virtual size of an object, which is the size of O 
> without virtual bases.

**nvsize(O):不包括对象O的虚基类的大小，可以理解为size(O) - size(virtual bases)**

> nvalign(O): the non-virtual alignment of an object, which is the 
> alignment of O without virtual bases.

**nvalign(O):剔除对象O的虚基类后，其对齐的大小.**

# 1 由一个例子说起

即如下代码

```c++
#include <iostream>

class Point2d {
public:
  Point2d(int x = 0.0, int y = 0.0) : x_(x), y_(y) {}
  float x() {return x_;}
  float y() {return y_;}

  void x(float newX) { x_ = newX;}
  void y(float newY) { y_ = newY;}

  void operator+=(const Point2d& rhs) {
    x_ += rhs.x_;
    y_ += rhs.y_;
  }

private:
  int x_;
  int y_;
};

class Point3d : public Point2d {
public:
  Point3d(int x = 0.0, int y = 0.0, int z = 0.0) 
    : Point2d(x, y), z_(z) {}

  float z() {return z_;}
  void z(float newZ) {z_ = newZ;}
  void operator+=(const Point3d& rhs) {
    Point2d::operator+=(rhs);
    z_ += rhs.z_;
  }
private:
  int z_;
};

int main() {
  Point2d d1;
  Point3d d2;
  return 0;
}
```
通过gdb，可看到如下布局

```c++
d1 = {
  x_ = 0,
  y_ = 1431654560
}
d2 = {
  <Point2d> = {
    x_ = 21845,
--Type <RET> for more, q to quit, c to continue without paging--
    y_ = -7840
  }, 
  members of Point3d:
  z_ = 32767
}

```

可知，非多态下继承，正如深度探索C++对象模型 中所示

![](https://pic3.zhimg.com/80/v2-d53bdeaed4fa558fdc8785db566d87c2_1440w.jpg)

也可以从汇编角度看一看，所谓的构造函数到底是如何实现的，上述具体的汇编实现如下所示[Compiler Explorer](https://godbolt.org/)

```c++
Point2d::Point2d(int, int) [base object constructor]:
        pushq   %rbp
        movq    %rsp, %rbp
        movq    %rdi, -8(%rbp)
        movl    %esi, -12(%rbp)
        movl    %edx, -16(%rbp)
        movq    -8(%rbp), %rax
        movl    -12(%rbp), %edx
        movl    %edx, (%rax)
        movq    -8(%rbp), %rax
        movl    -16(%rbp), %edx
        movl    %edx, 4(%rax)
        nop
        popq    %rbp
        ret

Point3d::Point3d(int, int, int) [base object constructor]:
        pushq   %rbp
        movq    %rsp, %rbp
        subq    $32, %rsp
        movq    %rdi, -8(%rbp)
        movl    %esi, -12(%rbp)
        movl    %edx, -16(%rbp)
        movl    %ecx, -20(%rbp)
        movq    -8(%rbp), %rax
        movl    -16(%rbp), %edx
        movl    -12(%rbp), %ecx
        movl    %ecx, %esi
        movq    %rax, %rdi
        call    Point2d::Point2d(int, int) [base object constructor]
        movq    -8(%rbp), %rax
        movl    -20(%rbp), %edx
        movl    %edx, 8(%rax)
        nop
        leave
        ret

main:
        pushq   %rbp
        movq    %rsp, %rbp
        subq    $32, %rsp
        leaq    -8(%rbp), %rax
        movl    $0, %edx
        movl    $0, %esi
        movq    %rax, %rdi
        call    Point2d::Point2d(int, int) [complete object constructor]
        leaq    -20(%rbp), %rax
        movl    $0, %ecx
        movl    $0, %edx
        movl    $0, %esi
        movq    %rax, %rdi
        call    Point3d::Point3d(int, int, int) [complete object constructor]
        movl    $0, %eax
        leave
        ret
```

由上述汇编代码可知，所谓构造函数和普通函数调用方式相同。

# 2 派生类中base class subobject 的完整性

**在此，抛出一个疑问：GCC编译器是否如深度探索C++对象模型 所说：派生类中base class subobject具有完整性？**

我们通过如下代码验证

```c++
#include <iostream>
class Concrete {
private:
  int val_;
  char c1_;
  char c2_;
  char c3_;
};

class Concrete1 {
private:
  int val_;
  char c1_;
};
class Concrete2 : public Concrete1 {
private:
  char c2_;
};

class Concrete3 : public Concrete2 {
private:
  char c3_;
};

int main() {
  Concrete base;
  Concrete3 df;
  Concrete2 c2 = df;
  return 0;
}
```
通过gdb可观察如下结果

```c++
base = {
  val_ = -7840,
  c1_ = -1 '\377',
  c2_ = 127 '\177',
  c3_ = 0 '\000'
}
df = {
--Type <RET> for more, q to quit, c to continue without paging--
  <Concrete2> = {
    <Concrete1> = {
      val_ = 1431654560,
      c1_ = 85 'U'
    }, 
    members of Concrete2:
    c2_ = 85 'U'
--Type <RET> for more, q to quit, c to continue without paging--
  }, 
  members of Concrete3:
  c3_ = 0 '\000'
}
c2 = {
  <Concrete1> = {
    val_ = -7840,
--Type <RET> for more, q to quit, c to continue without paging--
    c1_ = -1 '\377'
  }, 
  members of Concrete2:
  c2_ = 127 '\177'
}

(gdb) p sizeof(c2)
$1 = 8
(gdb) p sizeof(df)
$2 = 8
(gdb) p sizeof(base)
$3 = 8
```
**由上述可知，gcc并不保证subobject 的完整性。**

**gcc编译器采用的Itanium C++ ABI (Revision: 1.83) 标准，其关于C++对象memory layout主要由如下过程构成**

![](https://pic2.zhimg.com/80/v2-c105b7446be7ba612150950f04e1a4e0_1440w.png)

# 3 gcc类对象内存分配过程

此处均指在非POD type场景下。

正如上述所示，gcc编译器针对class的内存分配由4步构成。本小节分别介绍每一步的细节及注意事项。

## 3.1 初始化

1. 首先设置sizeof(C),align(C), dsize(C)如下

```c++
sizeof(C) = 0

align(C) = 1

dsize(C) = 0
```

2. 若C为dynamic class，则需要进行如下过程

> dynamic class:A class requiring a virtual table pointer (because it 
> or its bases have one or more virtual member functions or virtual 
> base classes).

> dynamic class: 需要vptr的类(本身或者其基类有至少一个virtual成员函数或者拥有虚
> 基类)

> primary base class: For a dynamic class, the unique base class (if 
> any) with which it shares the virtual pointer at offset 0.

> primary基类：动态类的第一个非虚直接动态基类

  - 识别所有直接或间接的虚拟基类，它们是某些其他直接或间接基类的primary基类。 这些虚基类叫做间接primary基类。

  - 如果C有一个动态基类，则尝试选择一个primary基类B。它是第一个（按直接基类顺序）非虚动态基类（如果存在). 否则，它是一个几乎空的虚基类，是（前序）继承图顺序中的第一个，如果存在则不是间接primary基类，或者如果它们都是间接primary基类，则只是第一个。

  - 如果C没有主基类，则在偏移量0处为C分配vptr，并将 sizeof(C)、align(C) 和 dsize(C) 设置为指针的适当值（对于 Itanium 64，所有 8 个字节 位 ABI）。



# 参考

[offsetof](https://en.cppreference.com/w/cpp/types/offsetof)

[alignof](https://en.cppreference.com/w/cpp/language/alignof)

[Itanium C++ ABI (Revision: 1.83)](http://refspecs.linux-foundation.org/cxxabi-1.83.html#vtable)


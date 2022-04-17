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

> nearly empty class: A class that contains a virtual pointer, but no 
> other data except (possibly) virtual bases. In particular, it:

> has no non-static data members other than zero-width bitfields,

> has no direct base classes that are not either empty, nearly empty, 
> or virtual,

> has at most one non-virtual, nearly empty direct base class, and

> has no proper base class that is empty, not morally virtual, and at 
> an offset other than zero.

> 简单来说，几乎空类是指只含有vptr的指针

  a. 识别所有直接或间接的虚拟基类，它们是某些其他直接或间接基类的primary基类。 这些虚基类叫做间接primary基类。

  b. 如果C有一个动态基类，则尝试选择一个primary基类B。它是第一个（按直接基类顺序）非虚动态基类（如果存在). 否则，它是一个几乎空的虚基类，是（前序）继承图顺序中的第一个，如果存在则不是间接primary基类，或者如果它们都是间接primary基类，则只是第一个。

  c. 如果C没有主基类，则在偏移量0处为C分配vptr，并将 sizeof(C)、align(C) 和 dsize(C) 设置为指针的适当值（对于 Itanium 64，所有 8 个字节 位 ABI）。

**NOTE:上述b现在被认为是设计中的错误。 使用第一个间接主基类作为派生类的主基类并不会节省对象中的任何空间，并且会导致在基类虚表的附加副本中出现一些虚函数指针的重复。**

**好处是使用派生类的虚指针作为基类的虚指针通常会节省负载，并且调用它的虚函数不需要对this指针进行调整。**

为了更好的理解上述概念引入如下示例代码

```c++
struct A { virtual void f(){}; };
struct B : virtual public A { int i{0}; };
struct C : virtual public A { int j{}; };
struct D : public B, public C {};

int main() {
  D d;
  C c;
  return 0;
}
```

通过[Compiler Explorer](https://godbolt.org/)可知D的vtable如下所示

```c++
vtable for D:
        .quad   0
        .quad   0
        .quad   0
        .quad   typeinfo for D
        .quad   A::f()
        .quad   -16
        .quad   -16
        .quad   -16
        .quad   typeinfo for D
        .quad   0
```

通过gdb可知D的部分内存如下所示

```c++
(gdb) p d
$2 = (D) {
  <B> = {
    <A> = {
      _vptr.A = 0x555555557bf8 <vtable for D+32>
    }, 
    members of B:
    i = 0
  }, 
  <C> = {
    members of C:
    j = 0
--Type <RET> for more, q to quit, c to continue without paging--
  }, <No data fields>}
(gdb) 
(gdb) 
(gdb) p sizeof(d)
$3 = 32
```

可知gdb显示D的成员只有一个vptr，但是sizeof(d)的大小为32，**那么原因是什么呢？**

此处可确定gdb显示D的成员不全，从D的vtable和sizeof(d)的大小可知，D包含两个vptr，也即D的内存布局及vtable结构如下所示

![](https://pica.zhimg.com/80/v2-4a21157dbe2eeb09c89caf8e8cd9cf0b_1440w.png)

之所以有如此异常，是因为C++ABI中规定

**当类C的任何间接主基类E，即已被选为C的某个其他基类（直接或间接、虚拟或非虚拟）的主基类的一个，将作为该其他基类的一部分进行分配**

同时，上述示例中类A为D的间接primary基类，故其会用作primary 基类，首先分配一个vptr指向A vtable的副本（primary vtable).

## 3.2 分配成员变量/基类 

**注：此阶段不会分配虚基类**

对于每个成员组件D(首先是C的primary基类，然后是声明顺序的非primary、非虚直接基类，然后是声明顺序的非静态数据成员和未命名位域），分配如下 ：

1. 如果 D 是声明类型为 T 且声明宽度为n位的（可能未命名）位域：
   
  根据 sizeof(T) 和 n 有两种情况：

  a. 如果 sizeof(T)*8 >= n，则根据底层 C psABI 的要求分配位域，且位域永远不会放置在C的基类的尾部填充中。

  如果 dsize(C) > 0，并且偏移量 dsize(C) - 1 处的字节部分由位域填充，并且该位域也是在 C 中声明的数据成员（但不在 C 的正确基类之一中），则 下一个可用位是偏移量 dsize(C) - 1 处的未填充位。否则，下一个可用位位于偏移量 dsize(C) 处。

  如下示例可辅助理解

  ```c++
  struct B : virtual public A { public: int i{0}; char a1_{};};
  class E : public B {
    int a_{0};
    unsigned char b : 3;

    int c:3;
  };

  int main() {
    // D d;
    E e;
    return 0;
  }
  ```

  通过gdb可知，sizeof(e) = 24

  b. 如果 sizeof(T)*8 < n，则令 T' 为 sizeof(T')*8 <= n 的最大整数 POD 类型。 位域从与 T' 适当对齐的下一个偏移量开始分配，长度为 n 位。 第一个 sizeof(T)*8 位用于保存位域的值，然后是 n - sizeof(T)*8 位的填充。

  如下示例可辅助理解

  ```c++
  struct B : virtual public A { public: int i{0}; char a1_{};};

  class E : public B {
    int a_{0};
    unsigned char b : 35;
  };

  int main() {
    // D d;
    E e;
    return 0;
  }
  ```

  通过gdb可知，sizeof(e) = 32.

**在任何一种情况下，更新 dsize(C) 以包括包含位域（部分）的最后一个字节，并将 sizeof(C) 更新为 max(sizeof(C),dsize(C))。**

2. 当D是一个非空基类或者数据成员时

- 从偏移量dsize(C)开始，如果需要对齐, 则根据基类的nvalign(D)或数据成员的 align(D)增加相应偏移值, 获得新的偏移量B。 

- 将D放置在此偏移B处，若这样导致相同类型的两个组件（直接或间接）具有相同的偏移，则针对基类，将B增加nvalign(D)，针对数据成员，将B增加align(D)

- 重试，重复直到成功（不小于sizeof(C)向上取整到所需的对齐值）。

此处以前述部分的Concrete3类来举例说明，Concrete3内存分配过程如下

```c++
                             分配Concerete1
分配Concrete2---------------> 分配数据成员
                            

分配数据成员(c3_)
```

- 如果D是基类，则此步骤仅分配其非虚部分，即排除任何直接或间接虚基类。

- 如果D是基类，则
  sizeof(C) = max (sizeof(C), offset(D)+nvsize(D)) 

- 如果D是数据成员，则将
  izeof(C) = max (sizeof(C), offset(D)+sizeof(D))

- 如果D是一个基类（在这种情况下非空的），则
  dsize(C) = offset(D)+nvsize(D)，
  align(C) = max (align(C), nvalign(D))
  
- 如果D是数据成员，则
  dsize(C) = offset(D)+sizeof(D)
  align(C) = max (align(C), align(D))

# 参考

[offsetof](https://en.cppreference.com/w/cpp/types/offsetof)

[alignof](https://en.cppreference.com/w/cpp/language/alignof)

[bit filed](https://en.cppreference.com/w/cpp/language/bit_field)

[C++ bit fields](https://docs.microsoft.com/en-us/cpp/cpp/cpp-bit-fields?view=msvc-170)

[Itanium C++ ABI (Revision: 1.83)](http://refspecs.linux-foundation.org/cxxabi-1.83.html#vtable)




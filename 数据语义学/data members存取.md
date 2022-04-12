# 0 引言

本文是基于 [深度探索C++对象模型](https://book.douban.com/subject/1091086/)第三章中相应章节，该部分深入研究一个类A有一个虚基类的情况下，存取相应虚基类的成员效率的研究。

在[虚拟继承(1)](https://github.com/qls152/DeepUnderstandingGcc-Clang-CplusplusObjectModel/blob/main/%E6%95%B0%E6%8D%AE%E8%AF%AD%E4%B9%89%E5%AD%A6/%E8%99%9A%E6%8B%9F%E7%BB%A7%E6%89%BF(1).md)中，我讨论了不含成员的虚继承的场
景，本文也借此补充一个 虚基类含有成员的场景！

本篇文章所使用的代码如下

```c++
#include <iostream>
#include <string>

class Base {
public:
  explicit Base(int a) : a_{a} {}
  int a_;
};

class Derived : public  virtual Base {
public:
  explicit Derived() : Base(0) {}
  int b_{0};
};

int main() {
  Derived d, *p = &d;
  ++d.b_; // 之所以如此是为了防止编译器优化
  ++d.a_;
  ++p->a_;
  return 0;
}
```

通过本文可进一步加深 [深度探索C++对象模型](https://book.douban.com/subject/1091086/) 中关于如下描述的理解

> 以两种方式存取x坐标，像这样
> origin.x=0.0
> pt->x=0.0
> 从origin存取和从pt存取有什么重大差异？答案是，当point3d是一个派生类，而在其继承结
> 构中有一个virtual bse class，并且被存取的member(本例中的x)是一个从该virtual
> base class 继承而来的member时，就会有重大差异！

# 1 gdb感知Derived内存模型

在深入汇编代码之前先通过gdb感受一下Dervied的内存模型，然后和相应的汇编作对比.

本文例子通过gdb，可以看到Derived内存模型如下所示

```c++
$2 = (Derived) {
  <Base> = {
    a_ = 0
  }, 
  members of Derived:
  _vptr.Derived = 0x555555557d60 <VTT for Derived>,
--Type <RET> for more, q to quit, c to continue without paging--
  b_ = 0
}

(gdb) p sizeof(d)
$3 = 16
```

故由上可知，Derived成员布局为

![](https://pic3.zhimg.com/80/v2-903f76cc38dce5136276530d57c0bc3a_1440w.jpg)

# 2 两种不同方式存取a_的效率讨论

此部分不再讲述具体的构造函数初始化过程，感兴趣的可参考[虚拟继承2](https://github.com/qls152/DeepUnderstandingGcc-Clang-CplusplusObjectModel/blob/main/%E6%95%B0%E6%8D%AE%E8%AF%AD%E4%B9%89%E5%AD%A6/%E8%99%9A%E6%8B%9F%E7%BB%A7%E6%89%BF(2).md)。

本部分首先看一下，示例代码生成的vtble的具体内容，通过[Compiler Explorer](https://godbolt.org/) 可知其内容如下

```c++
vtable for Derived:
        .quad   12
        .quad   0
        .quad   typeinfo for Derived 
```

综合1中Devired的成员内存模型，最终可得Devired的内存模型如下

![](https://pic2.zhimg.com/80/v2-490af581026956065596394f64b14f35_1440w.jpg)

通过Compiler Explorer 可知

```c++
  ++d.b_; // 之所以如此是为了防止编译器优化
  ++d.a_;
  ++p->a_;
```

语句所对应的汇编实现如下所示

```c++
 // ++d.b_
        movl    -24(%rbp), %eax // 直接通过b_内存offset
        addl    $1, %eax
        movl    %eax, -24(%rbp)
        
        // ++d.a_
        movl    -20(%rbp), %eax // 直接通过a_内存offset
        addl    $1, %eax
        movl    %eax, -20(%rbp)

        // ++p->a_
        movq    -8(%rbp), %rax // 通过调整vptr取得相应a_
        movq    (%rax), %rax
        subq    $24, %rax
        movq    (%rax), %rax
        movq    %rax, %rdx
        movq    -8(%rbp), %rax
        addq    %rdx, %rax
        movl    (%rax), %edx
        addl    $1, %edx
        movl    %edx, (%rax)
```

由上述汇编知，**当存在virtual base class的场景下，通过指针存取其基类成员 效率要比通过类对象存取数据成员低很多，因为多了vptr的调整步骤.**

在本文中，gcc是如何实现p->a_的步骤如下：

- 通过movq -8(%rbp), %rax 和 movq (%rax), %rax 获得vtbl的vptr所指向的地址

- subq $24, %rax 使vptr指向vtbl的起点，也即vbase_offset处

- movq (%rax), %rax movq %rax, %rdx 这两句将vbase赋值给rdx

- addq %rdx, %rax 找到virtual base class在Derived中的位置，并获得相应的值

# 3 总结

本文通过简单的汇编层面的说明解释了[深度探索C++对象模型](https://book.douban.com/subject/1091086/)中 关于两种方式存取x效率的问题。

同时，也间接性增加了关于virtual base class含有成员变量时，Derived的内存布局的场景。

